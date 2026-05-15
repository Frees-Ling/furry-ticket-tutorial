# 第23章：开发中遇到的 Bug 与修复记录

## 本章学习目标

本章记录了票务系统开发过程中遇到的实际 Bug 及其修复过程。每个 Bug 都包含：问题现象、根因分析、解决方案和修复后的验证方法。学习这些 Bug 不仅能帮助你理解系统的边界情况，还能培养"防御性编程"思维。

---

## 23.1 Bug: `func.now()` 与 Python `datetime` 的比较陷阱

### 问题现象

创建订单时，如果活动开始时间在 `start_date` 字段设定的时间附近，有时会错误地报"活动已开始或已结束"，即使活动实际还未开始。

### 根因分析

在 `app/services/order_service.py` 的原始代码中使用了 SQLAlchemy 的 `func.now()` 来获取当前时间：

```python
# ❌ 有 Bug 的代码
if event.start_date <= func.now():
    raise ValueError("活动已开始或已结束")
```text

**问题：** `func.now()` 返回的是 SQLAlchemy 的 `FunctionElement` 对象，不是 Python 的 `datetime` 对象。当在 Python 层面比较 `datetime` 和 `FunctionElement` 时，SQLAlchemy 会重载比较运算符返回一个新的 SQL 表达式，而不是立即求值。这意味着：

1. 这个比较实际上生成了 SQL 条件 `start_date <= NOW()`，被延迟到了数据库查询时执行
2. 如果在 `flush()` 之后使用，可能和预期行为不同
3. 代码可读性差，其他开发者容易误判

### 修复

```python
# ✅ 修复后的代码：使用 Python datetime，确保在应用层比较
now = datetime.now(timezone.utc).replace(tzinfo=None)
if event.start_date <= now:
    raise ValueError("活动已开始或已结束")
```

关键修复记录在 `app/services/order_service.py` 第 61 行的注释中：

```text
# 关键修复：使用 Python datetime 而不是 SQLAlchemy func.now()！
```

### 验证方法

```python
# 测试：创建一个 start_date 接近当前时间的活动
# 在活动开始前 1 秒下单，应该成功
# 在活动开始后 1 秒下单，应该失败
```text

---

## 23.2 Bug: Redis Lua 脚本在 Docker 部署时的路径问题

### 问题现象

在本地开发环境运行正常，但部署到 Docker 容器后，创建订单时一直返回"活动不存在"（Lua 脚本返回 -2）。

### 根因分析

原来的代码从外部 `.lua` 文件加载脚本：

```python
# ❌ 有 Bug 的代码
async def _load_script(self) -> str:
    script_path = os.path.join(
        os.path.dirname(__file__),
        "../scripts/inventory.lua",
    )
    with open(script_path) as f:
        script = f.read()
    return await self.redis.script_load(script)
```

**问题：** 在 Docker 容器中，相对路径 `"../scripts/inventory.lua"` 依赖于文件在项目中的相对位置。如果 Docker 的工作目录配置不正确，或者文件没有正确复制到镜像中，路径解析就会失败，导致脚本无法加载。

### 修复

将 Lua 脚本定义为 Python 内联字符串常量，不再依赖外部文件路径：

```python
# ✅ 修复后的代码：内联字符串，无路径依赖
INVENTORY_LUA_SCRIPT = """
local stock_key = KEYS[1]
local limit_key = KEYS[2]
local quantity = tonumber(ARGV[1])
...
"""

class InventoryService:
    def __init__(self, redis: Redis):
        self.redis = redis
        self._sha: Optional[str] = None   # 缓存 SHA，避免重复加载

    async def load_script(self):
        if not self._sha:
            self._sha = await self.redis.script_load(INVENTORY_LUA_SCRIPT)
```text

### 经验教训

- Docker 部署中避免使用相对路径读取文件
- 小型脚本可以直接内联为字符串常量
- 如果必须使用外部文件，用 `pathlib.Path(__file__).resolve().parent` 构建绝对路径

---

## 23.3 Bug: 取消订单时 sold_count 可能变为负数

### 问题现象

在并发测试中，当同一活动的多个订单同时被取消时，`events.sold_count` 字段偶尔出现负值。

### 根因分析

```sql
-- ❌ 有 Bug 的 SQL
UPDATE events SET sold_count = sold_count - :qty WHERE id = :eid
```

**问题：** 如果两个取消操作并发执行，而 `sold_count` 已经是 0，两次 `UPDATE` 都会执行，导致 `sold_count` 变成负数。数据上的负值不影响核心功能，但会：

1. 导致前端显示"已售 -5 张"的荒谬数据
2. 影响"可用库存 = total_stock - sold_count"的计算

### 修复

使用 `GREATEST` 函数确保 `sold_count` 不会低于 0，并添加 `sold_count >= :qty` 条件：

```sql
-- ✅ 修复后的 SQL
UPDATE events SET sold_count = GREATEST(sold_count - :qty, 0)
WHERE id = :eid AND sold_count >= :qty2
```text

对应代码在 `app/services/order_service.py` 的 `cancel_order()` 和 `auto_cancel_expired_orders()` 中。

### 验证方法

```python
# 并发取消测试：100 个线程同时取消同一活动的多个订单
# 验证 sold_count 永远不会 < 0
```

---

## 23.4 Bug: 订单金额浮点数精度丢失

### 问题现象

当票价是 99.90 元，购买 3 张时，订单总金额显示为 299.69999999999993 元而非 299.70 元。

### 根因分析

```python
# ❌ 有 Bug 的代码
total_amount = event.price * data.quantity  # price 是 float 类型
```text

**问题：** IEEE 754 双精度浮点数无法精确表示 0.1、0.2 等十进制小数。`99.90 * 3` 的计算结果是 `299.69999999999993`。虽然数据库存储时四舍五入可以掩盖问题，但在金额校验环节（如支付宝回调中的金额比对）会产生不一致。

### 修复

所有金额计算使用 `Decimal`：

```python
# ✅ 修复后的代码
from decimal import Decimal

total_amount = Decimal(str(event.price)) * data.quantity
```

关键原则：

- `Decimal()` 的参数必须是**字符串**，不能是 float
- 数据库存 float 是最后一步（`float(total_amount)`）
- 整个计算过程都用 `Decimal` 保证精度

对应代码在 `app/services/order_service.py` 第 79 行。

### 验证方法

```python
from decimal import Decimal
assert Decimal('99.90') * 3 == Decimal('299.70')  # ✅ 精确相等
assert 99.90 * 3 == 299.69999999999993  # ❌ float 不精确
```text

---

## 23.5 Bug: WebSocket 缺少 Origin 校验（CSWSH 漏洞）

### 问题现象

安全审计发现，WebSocket 端点没有验证连接来源，任何网站都可以通过 JavaScript 连接本系统的 WebSocket 服务。

### 根因分析

```python
# ❌ 有 Bug 的代码
@router.websocket("/ws/seats/{event_id}")
async def seat_websocket(websocket: WebSocket, event_id: int):
    await websocket.accept()  # 直接接受，未验证来源
```

**CSWSH（Cross-Site WebSocket Hijacking）：** 由于 WebSocket 不受同源策略限制，恶意网站可以构造 WebSocket 连接到本系统的端点。虽然我们的 WebSocket 不发送敏感数据（仅座位更新），但攻击者可以：

- 通过 `Sec-WebSocket-Key` 和 `Sec-WebSocket-Accept` 的握手过程确认 WebSocket 服务存在
- 在用户已登录的情况下，连接到座位更新频道（如果 Token 在 Cookie 中）

### 修复

在 WebSocket 连接时添加 Origin 检查：

```python
# ✅ 修复后的代码片段（对应 app/routers/ws.py）
@router.websocket("/ws/seats/{event_id}")
async def seat_websocket(websocket: WebSocket, event_id: int):
    # Origin 校验：防止 CSWSH 攻击
    origin = websocket.headers.get("origin", "")
    allowed_origins = ["http://localhost:5173", "http://localhost:8000"]
    if origin and origin not in allowed_origins:
        await websocket.close(code=4001)
        return

    await manager.connect(websocket, f"seats:{event_id}")
```text

同时，完整的 WebSocket 安全还包含：

1. JWT 令牌验证（通过查询参数传递）
2. 消息大小限制（防止内存耗尽）
3. Seat ID 格式校验（仅允许纯数字）
4. 消息类型白名单（仅允许 `subscribe`、`unsubscribe`、`ping` 三种类型）

### 验证方法

```javascript
// 尝试从恶意页面连接
const ws = new WebSocket('ws://localhost:8000/ws/seats/1');
ws.onopen = () => console.log('连接成功（不应该成功！）');
ws.onclose = (e) => console.log('连接被拒绝:', e.code);  // 期望 4001
```

---

## 23.6 Bug: Token 刷新竞态条件

### 问题现象

当多个 API 请求同时返回 401 时，前端会触发多次 Token 刷新请求，导致 Refresh Token 被轮换多次而失效。

### 根因分析

```javascript
// ❌ 有 Bug 的代码（简化）
api.interceptors.response.use(
    (response) => response,
    async (error) => {
        if (error.response?.status === 401) {
            const newToken = await refreshToken();  // 多个请求同时进入这里！
            error.config.headers.Authorization = `Bearer ${newToken}`;
            return api(error.config);
        }
        return Promise.reject(error);
    }
);
```text

假设 3 个请求同时返回 401：

1. 请求 A 触发 Refresh → 成功，Token1 被轮换为 Token2
2. 请求 B 触发 Refresh → **Token2 被轮换为 Token3**（Token1 已失效）
3. 请求 C 触发 Refresh → **Token3 被轮换为 Token4**

结果：Token1 被轮换 3 次，只有最后一个 Token 是有效的，前面的全部失效。

### 修复

使用 `_retry` 标志位防止重入：

```javascript
// ✅ 修复后的代码（对应 frontend/src/utils/api.js）
api.interceptors.response.use(
    (response) => response.data,
    async (error) => {
        const originalRequest = error.config;

        if (error.response?.status === 401 && !originalRequest._retry) {
            originalRequest._retry = true;  // 标记为已重试

            try {
                const newToken = await refreshToken();
                originalRequest.headers.Authorization = `Bearer ${newToken}`;
                return api(originalRequest);
            } catch (refreshError) {
                // Refresh 也失败了 → 登出
                localStorage.removeItem('token');
                localStorage.removeItem('refresh_token');
                window.location.href = '/login';
                return Promise.reject(refreshError);
            }
        }
        return Promise.reject(error);
    }
);
```

**核心思路：** 每个请求的 `config` 对象是独立的，通过设置 `_retry = true` 来确保一个请求只会触发一次 Token 刷新，不会形成循环。

---

## 23.7 Bug: Redis 库存未在 MySQL 乐观锁失败时恢复

### 问题现象

在极端高并发场景下（如 1000 个用户同时抢 100 张票），偶尔出现库存"丢失"的情况——活动页面显示有库存但下单总是返回库存不足。

### 根因分析

```python
# ❌ 有 Bug 的代码逻辑
async def create_order(user_id, event_id, quantity):
    # 1. Redis 扣减库存（成功）
    result = await inventory.deduct_stock(event_id, user_id, quantity)
    if result != 1:
        raise ValueError("库存不足")

    # 2. MySQL 乐观锁检查（可能失败）
    event = await session.get(Event, event_id)
    # ... 检查版本号 ...

    # 3. 如果 MySQL 检查失败，Redis 库存没有归还！
    # → 这部分库存"永远消失"了
```text

**问题根因：** Redis 扣减是"快速门卫"，MySQL 乐观锁是"最终检查"。当 MySQL 检查拦截了请求（version 冲突或库存不足），Redis 已经扣减的库存需要原路归还。如果没有归还逻辑，库存就会持续减少，最终所有库存都从 Redis 中消失但实际并没有卖掉。

### 修复

使用 `try/except` 确保 MySQL 失败时一定归还 Redis 库存：

```python
# ✅ 修复后的代码（对应 app/services/order_service.py 第 121-123 行）
try:
    # 生成订单 + MySQL 乐观锁更新
    order = Order(...)
    self.session.add(order)
    await self.session.flush()

    stmt = text(
        "UPDATE events SET sold_count = sold_count + :qty, "
        "version = version + 1 "
        "WHERE id = :eid AND version = :ver "
        "AND sold_count + :qty2 <= total_stock"
    )
    result = await self.session.execute(stmt, {...})

    if result.rowcount == 0:
        raise ValueError("库存不足（乐观锁拦截）")

    return order

except Exception:
    # ❗ 任何失败都归还 Redis 库存
    await self.inventory.restore_stock(event_id, quantity)
    raise
```

---

## 23.8 Bug: ARQ Worker 在 Docker 中无法导入应用模块

### 问题现象

部署到 Docker 后，ARQ Worker 启动时报告 `ModuleNotFoundError: No module named 'app'`。

### 根因分析

**问题：** ARQ Worker 的启动命令是 `arq app.worker.WorkerSettings`，它需要从 `app` 包导入模块。在 Docker 容器中，如果工作目录（WORKDIR）不正确，或者在 `sys.path` 中没有包含项目根目录，Python 就无法找到 `app` 包。

### 修复

在 Dockerfile 中确保工作目录设置正确，并添加 `PYTHONPATH` 环境变量：

```dockerfile
# ✅ 修复后的 Dockerfile 配置（对应项目 Dockerfile）
WORKDIR /app
ENV PYTHONPATH=/app
COPY app/ ./app/
```text

并在 `docker-compose.prod.yml` 中显式覆盖 Worker 的工作目录。

### 验证方法

```bash
# 在容器中测试导入
docker exec -it ticket-worker python -c "from app.worker import WorkerSettings; print('OK')"
# 应该输出 OK 而不是 ModuleNotFoundError
```

---

## 23.9 其他小 Bug 与陷阱

| Bug | 现象 | 根因 | 修复 |
| ----- | ------ | ------ | ------ |
| 分页总数不准 | 翻到最后一页显示还有更多但实际没有 | `total_pages` 计算未考虑 last page 不完整 | `(total + size - 1) // size` 向上取整 |
| Token 在 Redis 黑名单中但已过期 | 每次请求都查黑名单，浪费性能 | 未检查 token 本身是否已过期 | 在 decode 阶段已校验过期，黑名单只查未过期的 token |
| 支付宝通知 JSON 解析错误 | 支付宝返回 `sign` 字段在验签时干扰 | 验签前需要移除 `sign` 和 `sign_type` | `data.pop("sign")` 后再验签 |
| CORS 预检请求返回 404 | Nginx 未处理 OPTIONS 请求 | 前端发 OPTIONS，Nginx 没有对应处理 | 添加 `if ($request_method = OPTIONS)` 处理 |
| 前端路由刷新后 404 | 直接访问 `/orders` 返回 Nginx 404 | SPA 路由需要 fallback 到 `index.html` | `try_files $uri $uri/ /index.html` |
| WebSocket 连接数持续增长 | 用户关闭页面后连接未清理 | 未监听 `WebSocket.onclose` | `onUnmounted` 中调用 `ws.close()` |

---

## 本章总结

| Bug | 影响模块 | 严重程度 | 修复文件 |
| ----- | --------- | --------- | --------- |
| func.now() 比较陷阱 | 订单服务 | 高 | `app/services/order_service.py` |
| Lua 脚本路径问题 | 库存服务 | 高 | `app/services/inventory_service.py` |
| sold_count 变负数 | 订单服务 | 中 | `app/services/order_service.py` |
| 浮点数精度丢失 | 订单服务 | 高 | `app/services/order_service.py` |
| WebSocket CSWSH | WebSocket 路由 | 高 | `app/routers/ws.py` |
| Token 刷新竞态 | 前端 Axios | 中 | `frontend/src/utils/api.js` |
| Redis 库存未归还 | 订单服务 | 高 | `app/services/order_service.py` |
| Worker 导入错误 | Docker/部署 | 高 | `Dockerfile` |

**经验教训：**

1. 不要在 Python 层面混用 `func.now()` 和 `datetime` 对象
2. Docker 部署要格外注意路径问题
3. 并发场景下的"失败回滚"必须在所有分支都执行
4. 金额计算永远用 `Decimal`，不用 `float`
5. WebSocket 端点同样需要鉴权和来源校验
6. 前端 Token 刷新必须防重入

**下一章预告：** （本附录为最终章节）
