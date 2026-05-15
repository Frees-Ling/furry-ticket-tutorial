# 算法深度解析

> 本文深入剖析票务系统中的每个核心算法，涵盖设计思想、数学原理、代码实现和边界情况。

---

## 目录

1. [双层防超卖算法](#1-双层防超卖算法)
2. [Redis Lua 原子库存扣减](#2-redis-lua-原子库存扣减)
3. [MySQL 乐观锁库存更新](#3-mysql-乐观锁库存更新)
4. [滑动窗口速率限制](#4-滑动窗口速率限制)
5. [支付宝防重放四层防护](#5-支付宝防重放四层防护)
6. [JWT 令牌轮换与黑名单](#6-jwt-令牌轮换与黑名单)
7. [订单号生成算法](#7-订单号生成算法)
8. [Redis 库存回滚算法](#8-redis-库存回滚算法)

---

## 1. 双层防超卖算法

### 1.1 问题定义

**超卖**（Overselling）是指系统售出的票数超过了活动设定的总库存。例如一个活动有 100 张票，但系统卖出了 101 张。

在并发场景下，两个用户同时购买最后一张票时，如果系统不做特殊处理，两人都可能看到"库存为 1"然后同时下单成功，导致超卖。

### 1.2 双层防御架构

本项目采用两层防御机制，每一层都能独立防止超卖：

```text
请求
  │
  ▼
┌──────────────────────────────────────────┐
│ 第一层：Redis Lua 原子扣减（快速路径）      │
│ 耗时：~0.1ms                              │
│ 作用：拦截大部分并发请求，快速返回库存不足    │
├──────────────────────────────────────────┤
│ 成功                                      │
│  ▼                                        │
│ ┌────────────────────────────────────────┐│
│ │ 第二层：MySQL 乐观锁更新（慢速路径）      ││
│ │ 耗时：~1-5ms                            ││
│ │ 作用：最终一致性保证，兜底防御             ││
│ └────────────────────────────────────────┘│
├──────────────────────────────────────────┤
│ 任一失败 → Redis 库存回滚                  │
└──────────────────────────────────────────┘
```

### 1.3 数学验证

定义事件：

- A = Redis 扣减成功（返回 1）
- B = MySQL 乐观锁更新成功（rowcount > 0）

系统允许下单的条件：A ∧ B（两者都成功）

**超卖发生条件**：¬A ∧ B（Redis 失败但 MySQL 成功）

但是，`¬A` 意味着 Redis 返回了非 1 的值（0=库存不足, -1=超限购, -2=键不存在）。如果 Redis 返回库存不足，那 MySQL 的 `sold_count + quantity <= total_stock` 条件也应该不满足。**所以在设计上，超卖的概率趋近于 0**。

唯一可能的超卖场景：Redis 成功扣减后，MySQL 更新时 version 冲突，但库存实际上足够，且旧数据还在。此时 Redis 库存已扣减但订单未创建，然后库存被归还，不会造成超卖。

**结论**：两层防御下，超卖概率在数学上趋近于 0。

### 1.4 失败概率分析

| 场景 | Redis 结果 | MySQL 结果 | 最终结果 | 用户看到 |
| ------ | ----------- | ----------- | --------- | --------- |
| 正常 | 成功(1) | 成功(rowcount=1) | 下单成功 | 订单创建成功 |
| 库存不足 | 失败(0) | — | 拒绝下单 | "库存不足" |
| 并发冲突 | 成功(1) | 失败(rowcount=0) | 回滚 Redis，拒绝 | "库存不足" |
| 超限购 | 失败(-1) | — | 拒绝下单 | "超过限购数量" |

### 1.5 代码走查

**下单流程（OrderService.create_order）:**

```python
async def create_order(self, user_id: int, data: OrderCreate) -> Order:
    # 1. 参数校验
    event = await self.session.get(Event, data.event_id)
    if not event:
        raise ValueError("活动不存在")
    if event.status != "published":
        raise ValueError("活动未上架")

    # 2. Redis Lua 原子扣减（第一层防御）
    result = await self.inventory.deduct_stock(
        event_id=data.event_id, user_id=user_id,
        quantity=data.quantity, max_per_user=10,
    )
    if result != 1:  # 0=库存不足, -1=超限购, -2=键不存在
        error_map = {0: "库存不足", -1: "超过限购数量", -2: "系统错误"}
        raise ValueError(error_map.get(result, "下单失败"))

    try:
        # 3. 创建订单对象
        order = Order(...)
        self.session.add(order)
        await self.session.flush()

        # 4. MySQL 原子 UPDATE（第二层防御）
        stmt = text("""
            UPDATE events
            SET sold_count = sold_count + :qty,
                version = version + 1
            WHERE id = :eid AND version = :ver
              AND sold_count + :qty2 <= total_stock
        """)
        result = await self.session.execute(stmt, { ... })

        if result.rowcount == 0:
            raise ValueError("库存不足（乐观锁拦截）")

        await self.session.flush()
        return order

    except Exception:
        # 任一环节失败 → 归还 Redis 库存
        await self.inventory.restore_stock(data.event_id, data.quantity)
        raise
```text

**关键要点**：

1. **第 70-79 行** — 先做 Redis 扣减，快速拦截大部分失败请求
2. **第 100-114 行** — MySQL 原子 UPDATE，使用 `sold_count + qty <= total_stock` 条件确保不会超卖
3. **第 128-130 行** — 任何异常都归还 Redis 库存，保证数据一致

### 1.6 改进空间

1. **Redis 回滚原子性**：目前使用 WATCH 事务回滚库存，极端并发下可能降级为 INCRBY
2. **库存上限保护**：回滚时的 `restore_stock` 没有硬性上限（仅用 WATCH 做乐观检测）
3. **Lua 脚本统一**：可以将扣减和回滚都改为 Lua 脚本，减少网络往返

---

## 2. Redis Lua 原子库存扣减

### 2.1 为什么要用 Lua？

Redis 的 Lua 脚本是**原子执行**的。在执行整个脚本期间，不会有其他命令插入。这完美解决了竞态条件问题：

```

不使用 Lua（有竞态条件）：
    客户端1: READ stock → 1
    客户端2: READ stock → 1  ← 两个客户端都读到 1！
    客户端1: WRITE stock → 0
    客户端2: WRITE stock → 0  ← 超卖！

使用 Lua（原子执行）：
    Redis: 执行脚本（在一个原子操作中完成读取、判断、写入）

```text

### 2.2 Lua 脚本源码

```lua
-- KEYS[1] = stock_key (库存键，例如 "stock:100")
-- KEYS[2] = limit_key (限购键，例如 "user_bought:100:42")
-- ARGV[1] = quantity (购买数量)
-- ARGV[2] = max_per_user (每人限购数量，0=不限)
-- ARGV[3] = user_id (用户 ID)

local stock_key = KEYS[1]
local limit_key = KEYS[2]
local quantity = tonumber(ARGV[1])
local max_per_user = tonumber(ARGV[2])
local user_id = ARGV[3]

-- 第1步：检查库存键是否存在
local stock = redis.call("GET", stock_key)
if not stock then
    return -2  -- 库存键不存在（活动可能未初始化）
end

-- 第2步：检查限购限制（如果 max_per_user > 0）
if max_per_user > 0 then
    local bought = redis.call("GET", limit_key)
    if bought and tonumber(bought) + quantity > max_per_user then
        return -1  -- 超过限购数量
    end
end

-- 第3步：检查库存是否充足
if tonumber(stock) >= quantity then
    -- 库存充足 → 扣减
    redis.call("DECRBY", stock_key, quantity)

    -- 如果有限购，更新已购数量
    if max_per_user > 0 then
        redis.call("INCRBY", limit_key, quantity)
        redis.call("EXPIRE", limit_key, 86400)  -- 24小时过期
    end
    return 1  -- 成功！
end

-- 库存不足
return 0
```

### 2.3 返回值语义

| 返回值 | 含义 | 处理方式 |
| -------- | ------ | --------- |
| 1 | 扣减成功 | 继续 MySQL 下单流程 |
| 0 | 库存不足 | 返回 "库存不足" |
| -1 | 超过限购 | 返回 "超过限购数量" |
| -2 | 库存键不存在 | 返回 "系统错误"（需检查活动初始化） |

### 2.4 为什么限购键 TTL 设为 86400 秒？

限购键记录用户在 24 小时内购买了多少张票。`EXPIRE 86400` 的语义是：

- 如果用户已经买了票，24 小时内计入限购额度
- 24 小时后计数器自动过期清除
- 每次购买时 `INCRBY` 会刷新 TTL

### 2.5 脚本加载与执行

```python
async def load_script(self) -> None:
    """将 Lua 脚本加载到 Redis，获取 SHA 缓存"""
    if not self._sha:
        self._sha = await self.redis.script_load(INVENTORY_LUA_SCRIPT)

async def deduct_stock(self, ...) -> int:
    await self.load_script()

    try:
        # 优先使用 EVALSHA（节省带宽）
        return int(await self.redis.evalsha(self._sha, len(keys), *keys, *args))
    except (ConnectionError, TimeoutError):
        # 降级到 EVAL（传递完整脚本）
        return int(await self.redis.eval(INVENTORY_LUA_SCRIPT, len(keys), *keys, *args))
```text

`EVALSHA` 比 `EVAL` 更高效：不需要每次都传递完整的 Lua 脚本，只需传递 SHA 摘要（20 字节 vs 几百字节）。

---

## 3. MySQL 乐观锁库存更新

### 3.1 乐观锁原理

乐观锁假设并发冲突不常发生（"乐观"），只在更新时检查数据是否被修改过。

```sql
-- 标准乐观锁模式
UPDATE events
SET sold_count = sold_count + :qty,
    version = version + 1          -- version 递增
WHERE id = :eid
  AND version = :ver               -- 检查 version 是否与读取时一致
  AND sold_count + :qty <= total_stock  -- 同时检查库存上限
```

如果 `version` 不匹配（说明其他线程已经修改过），或 `sold_count + qty > total_stock`（库存不足），则 `rowcount = 0`，不会更新任何行。

### 3.2 为什么还需要乐观锁？Lua 脚本不是已经原子扣减了吗？

Redis Lua 是快速路径（fast path），MySQL 是慢速路径（slow path）。两层防御的原因：

1. **Redis 可能丢失数据**：如果 Redis 宕机且未持久化，库存数据会丢失
2. **Redis 主从切换**：异步复制可能导致库存数据不一致
3. **MySQL 是最终数据源**：确保数据的最终一致性

同时，Redis 的 `DECRBY` 操作没有库存上限检查（MySQL 的 UPDATE 有 `sold_count + qty <= total_stock` 条件）。

### 3.3 代码解析

```python
stmt = text(
    "UPDATE events SET sold_count = sold_count + :qty, "
    "version = version + 1 "
    "WHERE id = :eid AND version = :ver "
    "AND sold_count + :qty2 <= total_stock"
)
result = await self.session.execute(stmt, {
    "qty": data.quantity,
    "eid": data.event_id,
    "ver": event.version,       # 读取时的版本号
    "qty2": data.quantity,      # 重复参数（text() 不支持命名参数复用）
})

if result.rowcount == 0:
    # 乐观锁冲突或库存不足
    raise ValueError("库存不足（乐观锁拦截）")
```text

**为什么用 `text()` 原生 SQL 而不是 ORM 查询？**

因为这个 UPDATE 必须是**原子操作**。如果先 SELECT 再 UPDATE，中间会有竞态条件窗口。`text()` 可以编写直接执行的原生 SQL，确保原子性。

---

## 4. 滑动窗口速率限制

### 4.1 为什么不用固定窗口？

固定窗口算法：每分钟最多 100 次请求，每分钟重置计数器。

```

固定窗口的缺陷：
   请求数
   100 ┤███████████████████████
    0 ┤───────────────────────────▶ 时间
      0          1min         2min

   如果用户在 1:00:59 发了 100 个请求，在 1:01:00（重置后）又发了 100 个请求
   ⇒ 200 个请求在 2 秒内涌入！

```text

滑动窗口解决了这个问题：以当前时间回溯一个窗口长度，统计窗口内的请求数。

### 4.2 Redis Sorted Set 实现

```

                 滑动窗口（60秒）
                 ◄──────────────►
请求时间戳：  ...  10  20  30  40  50  60  (秒前)
                  ↑              ↑
               最早的请求       当前请求

Redis Key: ratelimit:api:user:42
ZSET 成员：<时间戳> (例如 "1680000000.123")
ZSET 分值：时间戳（与成员相同）

```text

### 4.3 实现代码

```python
async def _check_rate_limit(
    self, key: str, max_req: int, window: int
) -> tuple[bool, int, int]:
    now = time.time()
    window_start = now - window

    async with self.redis.pipeline() as pipe:
        # 第1步：移除窗口外的旧记录（清理过期数据）
        await pipe.zremrangebyscore(key, 0, window_start)

        # 第2步：统计窗口内的请求数
        await pipe.zcard(key)

        # 第3步：添加当前请求
        await pipe.zadd(key, {str(now): now})

        # 第4步：设置键过期时间（避免内存泄漏）
        await pipe.expire(key, window + 60)

        # 原子执行所有命令
        results = await pipe.execute()

    count = results[1]  # zcard 结果

    if count > max_req:
        # 超出限制 — 计算需要等待的时间
        earliest = await self.redis.zrange(key, 0, 0, withscores=True)
        if earliest:
            wait_time = int(window - (now - earliest[0][1])) + 1
        else:
            wait_time = window
        return False, 0, wait_time

    return True, max_req - count, 0
```

### 4.4 Pipeline 的优势

使用 Pipeline 将四个 Redis 命令打包发送，减少网络往返（4 次命令只需 1 次网络往返）。

### 4.5 时间复杂度分析

| 操作 | 时间复杂度 |
| ------ | ----------- |
| `ZREMRANGEBYSCORE` | O(log N + M)，N=集合大小, M=移除的元素数 |
| `ZCARD` | O(1) |
| `ZADD` | O(log N) |
| `EXPIRE` | O(1) |

总体复杂度：O(log N + M)。由于窗口大小固定（如 60 秒），窗口内的请求数有上限（如 100），所以可视为 O(1)。

### 4.6 滑动窗口 vs 固定窗口对比

| 特性 | 固定窗口 | 滑动窗口 |
| ------ | --------- | --------- |
| 实现复杂度 | 简单（计数器+TTL） | 中等（Sorted Set） |
| 内存开销 | O(1) | O(N)，N=窗口内请求数 |
| 边界突刺 | 有（窗口切换时） | 无 |
| 精度 | 分钟级 | 秒级 |
| Redis 命令 | 1 个（INCR+EXPIRE） | 4 个（Pipeline） |

### 4.7 优雅降级

```python
except (ConnectionError, TimeoutError, OSError):
    # Redis 不可用时放行（降级处理）
    return True, max_req, 0
```text

当 Redis 不可用时，限流中间件会放行所有请求，确保服务可用性不受影响。

---

## 5. 支付宝防重放四层防护

### 5.1 问题背景

支付宝异步通知是 HTTP POST 回调，可能因为网络问题**重复发送**。同时，攻击者可能**伪造**通知来模拟支付成功。

系统需要同时防御：**重放攻击**（replay attack）和**伪造攻击**（forgery attack）。

### 5.2 四层防护架构

```

支付宝异步通知
    │
    ▼
┌─────────────────────────────────────┐
│ 第一层：RSA2 签名验证                 │ ← 确保消息确实来自支付宝
│ 使用支付宝公钥验证签名                  │
├─────────────────────────────────────┤
│ 通过？                               │
│  ▼                                   │
│ ┌───────────────────────────────────┐│
│ │ 第二层：交易状态检查               ││
│ │ 仅处理 TRADE_SUCCESS 的通知       ││
│ └───────────────────────────────────┘│
├─────────────────────────────────────┤
│ 通过？                               │
│  ▼                                   │
│ ┌───────────────────────────────────┐│
│ │ 第三层：Redis SETNX 幂等键         ││
│ │ 确保每个 out_trade_no 只处理一次   ││
│ └───────────────────────────────────┘│
├─────────────────────────────────────┤
│ 通过？                               │
│  ▼                                   │
│ ┌───────────────────────────────────┐│
│ │ 第四层：金额校验 + 状态机检查       ││
│ │ 金额匹配 + 订单必须是待支付状态      ││
│ └───────────────────────────────────┘│
├─────────────────────────────────────┤
│ 全部通过 → 确认支付 / 任一失败 → 拒绝 │
└─────────────────────────────────────┘

```text

### 5.3 第一层：RSA2 签名验证

```python
def verify_notification(self, data: dict, sign: str) -> bool:
    """使用支付宝公钥验证 RSA2 签名"""

    # 1. 剔除 sign 和 sign_type，剩余参数排序拼接
    sign_str = self._sort_params(data)

    # 2. 加载支付宝公钥
    with open(public_key_path) as f:
        public_key_pem = f.read()

    public_key = serialization.load_pem_public_key(
        public_key_pem.encode("utf-8"), backend=default_backend()
    )

    # 3. 验证签名
    signature = base64.b64decode(sign)
    public_key.verify(
        signature,
        sign_str.encode("utf-8"),
        padding.PKCS1v15(),
        hashes.SHA256(),
    )
    return True
```

**为什么 RSA2 签名不可伪造？**

- 支付宝使用自己的私钥对通知签名
- 系统使用支付宝公钥验证签名
- 攻击者没有支付宝私钥，无法伪造有效签名
- 签名覆盖所有通知参数，任一参数修改都会导致验签失败

### 5.4 第二层：交易状态检查

```python
trade_status = data.get("trade_status")
if trade_status != "TRADE_SUCCESS":
    logger.info("支付宝通知非成功状态: trade_status=%s", trade_status)
    return "success"  # 对非成功状态也返回 success
```text

只处理 `TRADE_SUCCESS` 状态。对其他状态（如 `TRADE_FINISHED`、`WAIT_BUYER_PAY`）直接返回 `success`，避免支付宝重复推送。

### 5.5 第三层：Redis SETNX 幂等键

```python
idempotency_key = f"alipay:processed:{order_no}"
processed = await self.redis.setnx(idempotency_key, "1")
if not processed:
    logger.info("重复通知（幂等键已存在）: order_no=%s", order_no)
    return "success"  # 已处理过，直接返回成功

await self.redis.expire(idempotency_key, IDEMPOTENCY_TTL)
```

**幂等键设计**：

- `SETNX`（SET if Not eXists）是原子操作，确保只有一个线程能设置成功
- `SETNX` 返回 False 表示键已存在，说明该订单已处理过
- 设置 TTL（24 小时），超过此时间的重复通知视为新通知

**为什么使用 `SETNX` 而不是 `GET + SET`？**

```python
# 错误！非原子操作有竞态条件
if not await self.redis.get(key):  # 检查
    await self.redis.set(key, "1")  # 设置 ← 中间可能被其他线程插入
    process_order()  # 可能重复处理！

# 正确！SETNX 是原子操作
if await self.redis.setnx(key, "1"):  # 检查+设置 原子完成
    process_order()  # 仅执行一次
```text

### 5.6 第四层：金额校验 + 状态机检查

```python
# 金额校验（字符串转 Decimal 避免浮点精度问题）
notify_decimal = Decimal(str(notify_amount))
order_decimal = Decimal(str(order.total_amount))
if notify_decimal != order_decimal:
    logger.error("金额不匹配")
    return "failure"

# 状态机检查（由 confirm_payment 完成）
if not order or order.status != "pending_payment":
    return None  # 只有待支付订单才能确认支付
```

### 5.7 攻击场景模拟

| 攻击方式 | 第一层(验签) | 第二层(状态) | 第三层(幂等) | 第四层(金额) | 结果 |
| --------- | ------------ | ------------ | ------------ | ------------ | ------ |
| 篡改金额 | 拦截(签名不匹配) | — | — | — | 失败 |
| 重放旧通知 | 通过 | 通过 | 拦截(SETNX) | — | 失败 |
| 伪造通知 | 拦截(无法伪造签名) | — | — | — | 失败 |
| 修改订单号 | 拦截(签名不匹配) | — | — | — | 失败 |

---

## 6. JWT 令牌轮换与黑名单

### 6.1 JWT 令牌结构

```text
Header:
{
  "alg": "HS256",
  "typ": "JWT"
}

Payload:
{
  "sub": "42",            ← 用户 ID
  "exp": 1712345678,       ← 过期时间（Unix 时间戳）
  "iat": 1712343878,       ← 签发时间
  "type": "access",       ← 令牌类型（access/refresh）
  "jti": "a1b2c3d4e5",   ← 令牌唯一 ID（用于黑名单）
  "iss": "ticket-booking" ← 签发者
}
```

### 6.2 访问令牌 vs 刷新令牌

| 属性 | 访问令牌 (access_token) | 刷新令牌 (refresh_token) |
| ------ | ------------------------ | ------------------------ |
| 有效期 | 30 分钟（短期） | 7 天（长期） |
| 用途 | 访问受保护的 API | 获取新的访问令牌 |
| 黑名单 | 不需要（过期快） | 需要（有效期长） |
| 轮换 | 不需要 | 每次使用后轮换 |

### 6.3 令牌刷新流程

```text
客户端                          服务端
  │                               │
  │  1. POST /auth/refresh        │
  │     {refresh_token: "xxx"}    │
  │ ─────────────────────────────►│
  │                               │
  │                               │  2. 解码 refresh_token
  │                               │  3. 检查 type=refresh
  │                               │  4. 检查黑名单（jti）
  │                               │  5. 查询用户状态
  │                               │
  │                               │  6. 旧令牌加入黑名单（轮换）
  │                               │  7. 生成新令牌对
  │                               │
  │  8. 返回新令牌对               │
  │ ◄─────────────────────────────│
  │    新 access_token            │
  │    新 refresh_token           │
  │                               │
  │  9. 旧 access_token 过期      │
  │  10. 旧 refresh_token 失效    │
```

### 6.4 轮换机制的安全性

```python
async def refresh_token(self, refresh_token_str: str) -> tuple[str, str] | None:
    payload = decode_token(refresh_token_str)
    if not payload or payload.get("type") != "refresh":
        return None

    # 检查黑名单
    jti = payload.get("jti")
    if jti and await is_token_blacklisted(self.redis, jti):
        return None  # 已吊销

    user_id = int(payload["sub"])

    # 轮换：旧令牌加入黑名单
    if settings.REFRESH_TOKEN_ROTATION and jti:
        exp = payload.get("exp", 0)
        now = datetime.now(UTC).timestamp()
        ttl = max(int(exp - now), 3600)  # 至少保留 1 小时
        await blacklist_token(self.redis, jti, ttl)

    # 生成新令牌对
    new_access = create_access_token({"sub": str(user_id)})
    new_refresh = create_refresh_token({"sub": str(user_id)})
    return new_access, new_refresh
```text

**轮换机制解决了什么问题？**

如果没有轮换，refresh_token 被泄露后：

1. 攻击者拿到 refresh_token
2. 攻击者可以一直刷新获取新的 access_token
3. 合法用户无法察觉

有轮换后：

1. 攻击者使用 refresh_token 刷新
2. 旧 refresh_token 被加入黑名单
3. 合法用户下次刷新时会发现 token 已被吊销
4. 合法用户可立即重新登录，使所有会话失效

### 6.5 JWT 黑名单实现

```python
async def is_token_blacklisted(redis: Redis, jti: str) -> bool:
    """检查 JWT 是否已被吊销"""
    result = await redis.exists(f"jwt:blacklist:{jti}")
    return bool(result)

async def blacklist_token(redis: Redis, jti: str, ttl: int = 86400) -> None:
    """将 JWT 加入黑名单"""
    await redis.setex(f"jwt:blacklist:{jti}", ttl, "1")
```

### 6.6 安全性总结

```text
攻击者窃取了 refresh_token → 尝试使用 →
    1. 服务器解码 token → 有效
    2. 检查黑名单 → 不在黑名单（首次使用）
    3. 生成新令牌 → 旧 token 加入黑名单
    4. 攻击者获得新令牌

合法用户后续使用旧 refresh_token →
    发现已被吊销 →
    意识到 token 泄露 →
    立即重新登录 → 所有旧 token 作废
```

---

## 7. 订单号生成算法

### 7.1 格式设计

```python
def generate_order_no() -> str:
    """生成唯一订单号：TKT + YYYYMMDD + 8位十六进制随机数"""
    date_part = datetime.now(timezone.utc).strftime("%Y%m%d")
    random_part = secrets.token_hex(4)  # 8 hex chars
    return f"TKT{date_part}{random_part}"
```text

生成的订单号示例：`TKT202605131a2b3c4d`

### 7.2 格式解析

| 部分 | 长度 | 示例 | 说明 |
| ------ | ------ | ------ | ------ |
| `TKT` | 3 位 | `TKT` | 固定前缀，标识票务系统 |
| `YYYYMMDD` | 8 位 | `20260513` | 日期，便于按日分表 |
| 随机部分 | 8 位 | `1a2b3c4d` | 16^8 = 42 亿种组合 |

### 7.3 唯一性保证

1. **日期前缀**：不同日期的订单号前缀不同，天然隔离
2. **secrets.token_hex(4)**：使用密码学安全的随机数生成器（非 `random` 模块）
3. **数据库唯一约束**：`order_no` 列有 `UNIQUE` 约束，最后一道防线
4. **碰撞概率**：同一天内碰撞概率约为 `n^2 / (2 * 16^8)`，即使一天产生 100 万订单，碰撞概率仍小于 0.0001%（实际上只需使用 MD5 截断级的安全保障）

### 7.4 为什么不用 UUID？

```python
# UUID 示例：550e8400-e29b-41d4-a716-446655440000
# 问题：太长（36 位）、无序、不易读

# 本项目的订单号：TKT202605131a2b3c4d
# 优点：短（19 位）、有序、可读性强（含日期）
```

### 7.5 为什么不用自增 ID？

1. **安全**：自增 ID 暴露订单数量，攻击者可遍历
2. **分布式**：多机环境自增 ID 可能冲突
3. **可推测**：攻击者可推算出其他用户的订单号

---

## 8. Redis 库存回滚算法

### 8.1 回滚场景

当 Redis 成功扣减库存但 MySQL 下单失败时，需要回滚 Redis 库存。

### 8.2 实现代码

```python
async def restore_stock(self, event_id: int, quantity: int) -> None:
    """归还库存（取消订单 / 支付超时）"""
    if quantity <= 0:
        return

    key = self._stock_key(event_id)

    try:
        # 使用 WATCH 事务保证原子性 + 上限保护
        async with self.redis.pipeline(transaction=True) as pipe:
            await pipe.watch(key)
            current = await pipe.get(key)
            current_val = int(current) if current else 0

            # 上限保护：不让库存超过初始值
            # 假设初始库存是 100，已卖 30，还剩下 70 + 归还 2 = 72（不能超过 100）
            new_val = current_val + quantity

            await pipe.multi()
            await pipe.set(key, new_val)
            await pipe.execute()

    except (ConnectionError, TimeoutError):
        # WATCH 冲突或网络错误时降级
        await self.redis.incrby(key, quantity)
```text

### 8.3 WATCH 事务机制

Redis 的 WATCH 是一种**乐观锁**实现：

1. `WATCH key` — 监视 key
2. 读取 key 的当前值
3. `MULTI` — 开始事务
4. 执行命令
5. `EXEC` — 执行事务（如果 key 在 WATCH 后被修改过，EXEC 返回 nil，事务失败）

本例中，如果 WATCH 冲突（其他线程修改了库存），事务失败，走 `except` 分支降级为 `INCRBY`。

### 8.4 降级处理的风险

当 WATCH 事务失败时，代码降级为简单 `INCRBY`。这可能导致**超量归还**：

```

场景：多次取消同一订单

1. 用户下单 → Redis 扣减 2 → MySQL 失败 → restore_stock(2)
2. 重试时再次 restore_stock(2)（因为 WATCH 事务会检测到冲突）
   → INCRBY 2 导致库存多了 2

```text

**缓解措施**：实际项目中，应该确保 `restore_stock` 不会被多次调用（幂等设计），或在 MySQL 层面做去重。
