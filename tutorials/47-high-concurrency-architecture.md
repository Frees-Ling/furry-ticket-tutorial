# 高并发架构设计（生产级）

> 本文档解决票务系统在高流量场景下的卡顿、超时、崩溃问题。
> 核心思路：把系统从"普通网站"升级为"事件级流量控制系统"。

---

## 目录

一、问题定义 · 二、核心思想 · 三、完整架构总览 · 四、前端防卡死机制 · 五、抢票排队系统 · 六、三层限流 · 七、数据库防崩设计 · 八、前端降级系统 · 九、CDN + 缓存策略 · 十、活动模式切换 · 十一、用户体验保护 · 十二、压测目标

---

## 一、问题定义

高流量下出现的典型症状和本质原因：

### 症状

| 现象 | 表现 | 用户感受 |
|------|------|---------|
| 页面卡死 | Loading 无限转圈 | "这网站是不是坏了？" |
| 接口无响应 | 请求超时，浏览器转菊花 | "怎么点都没反应" |
| 前端无法交互 | 按钮点不动，页面僵死 | "卡住了，刷新试试" |
| 票务页面空白 | 关键页面加载不出来 | "票呢？" |
| 系统整体冻结 | 所有用户同时卡住 | "服务器崩了" |

### 本质原因

| 原因 | 解释 |
|------|------|
| 请求洪峰没有被削平 | 所有请求同时到达后端，没有缓冲 |
| 后端没有排队系统 | 请求直接穿透到数据库，没有队列保护 |
| 前端没有降级机制 | 特效、动画、请求全部满血运行，低端设备扛不住 |
| 数据库没有并发保护 | 高并发下数据库连接池耗尽或死锁 |

---

## 二、核心思想

### 系统不能是

```
❌ 用户请求 → 直接 → 数据库
```

### 必须变为

```
✔ 用户 → CDN → 限流 → 排队 → Worker → 数据库
```

每加一层，系统就多一分抵抗力。

---

## 三、完整架构总览

```
用户
  │
  ▼
CDN（缓存静态资源）
  │  首页、Banner、活动介绍、图片
  │
  ▼
前端页面（Vue 3）
  │  ├─ 3 秒超时保护
  │  ├─ 失败态 UI（Error + Retry）
  │  ├─ 禁止无限 Loading
  │  └─ 设备降级（低配关特效）
  │
  ▼
API Gateway（限流 + 排队）
  │  ├─ IP 限流：5 次/秒/IP
  │  ├─ 用户限流：1 次/秒/用户
  │  ├─ 接口限流：抢票 N 并发/秒
  │  └─ 超出返回 429 "系统繁忙"
  │
  ▼
排队系统（Redis Queue）
  │  ├─ 用户进入排队
  │  ├─ 分配排队编号
  │  ├─ 显示预计等待时间
  │  └─ Worker 逐个处理
  │
  ▼
业务 Worker 集群
  │  ├─ Redis 库存预扣减（原子操作）
  │  ├─ 生成订单
  │  └─ MySQL 乐观锁兜底
  │
  ▼
Database（最终落库）
     ├─ MySQL 行锁
     ├─ 乐观锁 version 字段
     └─ 事务保护
```

---

## 四、前端防卡死机制

### 4.1 所有请求必须有超时

```typescript
// axios 实例配置
const api = axios.create({
  timeout: 5000,  // 5 秒超时，超过即失败
})

// 请求拦截器
api.interceptors.request.use(config => {
  config.metadata = { startTime: Date.now() }
  return config
})

// 响应拦截器
api.interceptors.response.use(
  response => {
    // 正常返回
    return response
  },
  error => {
    if (error.code === 'ECONNABORTED' || error.message.includes('timeout')) {
      // 超时 → 显示失败态，不要无限转圈
      return Promise.reject({ type: 'timeout', message: '请求超时，请重试' })
    }
    return Promise.reject(error)
  }
)
```

### 4.2 UI 必须三态

每个 API 请求对应的 UI 必须有三种状态：

```vue
<template>
  <div>
    <!-- Loading -->
    <div v-if="status === 'loading'" class="skeleton">
      <div class="skeleton-item" />
      <div class="skeleton-item" />
    </div>

    <!-- Success -->
    <div v-else-if="status === 'success'" class="content">
      <slot />
    </div>

    <!-- Error（带重试按钮） -->
    <div v-else-if="status === 'error'" class="error-fallback">
      <p>{{ errorMessage }}</p>
      <button @click="retry">重试</button>
    </div>
  </div>
</template>
```

状态机：

```
loading → success：正常
loading → error → retry → loading → success：失败后重试
loading → error：彻底失败（显示降级 UI）
```

### 4.3 禁止无限 Loading

```typescript
// 任何接口
async function fetchData() {
  const controller = new AbortController()
  const timeoutId = setTimeout(() => controller.abort(), 5000)

  try {
    const res = await fetch('/api/data', { signal: controller.signal })
    return await res.json()
  } catch {
    showError('加载失败，请检查网络')  // 显示失败态
    return null                        // 返回 null，UI 切换到 fallback
  } finally {
    clearTimeout(timeoutId)
  }
}
```

### 4.4 防止请求风暴

```typescript
// 抢票按钮防重复点击
const isSubmitting = ref(false)

async function submitOrder() {
  if (isSubmitting.value) return  // 正在请求中，忽略本次点击
  isSubmitting.value = true

  try {
    await placeOrder()
  } catch (e) {
    // 处理错误
  } finally {
    isSubmitting.value = false  // 请求完成，允许再次点击
  }
}
```

---

## 五、抢票排队系统

### 5.1 为什么要排队？

没有排队：
```
用户 A ──→ MySQL（扣库存成功）
用户 B ──→ MySQL（扣库存成功）     ← 同时扣减，超卖
用户 C ──→ MySQL（扣库存成功）
```

有排队：
```
用户 A → 排队 #001 → Worker → MySQL
用户 B → 排队 #002 → 等待  ← 不会打到数据库
用户 C → 排队 #003 → 等待  ← 不会打到数据库
```

排队 = 数据库的保险丝。

### 5.2 排队流程

```
用户点击"立即抢票"
  │
  ▼
PUT /api/v1/queue/join { event_id, ticket_type_id }
  │
  ▼
后端：
  ① Redis 原子计数器 + 1 → 获得排队编号
  ② 入队（Redis List LPUSH）
  ③ 返回排队信息：
     {
       "position": 5321,         // 你的位置
       "queue_length": 12345,    // 当前排队总人数
       "estimated_wait_sec": 168 // 预计等待秒数
     }
  │
  ▼
前端显示：
  ┌────────────────────────────────┐
  │    当前排队人数：12,345         │
  │    你的位置：#5,321              │
  │    预计等待：2 分 48 秒          │
  │                                │
  │    ████████░░░░░░░░░░░░░░░     │
  │    [ 不要刷新页面 ]             │
  └────────────────────────────────┘
  │
  ▼
Worker 每秒从队列头部取一个请求处理：
  ① RPOP 出队
  ② Redis 预扣库存（Lua 原子脚本）
  ③ 创建订单
  ④ WebSocket 推送 "轮到你了"
  │
  ▼
前端收到 WebSocket 通知 → 跳转到订单页面
```

### 5.3 核心代码

```python
# services/queue_service.py
import time
import json
from redis.asyncio import Redis


class QueueService:
    """排队服务 — Redis List + 原子计数器"""

    def __init__(self, redis: Redis):
        self.redis = redis

    async def join(self, event_id: int, user_id: int) -> dict:
        """用户加入排队

        Returns:
            position: 当前队列位置
            queue_length: 队列总长度
            estimated_wait: 预计等待秒数（假设每 100ms 处理一个）
        """
        queue_key = f"queue:{event_id}"
        counter_key = f"queue:counter:{event_id}"

        # 获取排队编号（原子递增）
        seq = await self.redis.incr(counter_key)

        # 入队
        entry = json.dumps({"user_id": user_id, "seq": seq, "time": time.time()})
        await self.redis.lpush(queue_key, entry)

        # 队列总长度
        queue_length = await self.redis.llen(queue_key)

        return {
            "position": seq,
            "queue_length": queue_length,
            "estimated_wait_sec": int(queue_length * 0.1),  # 100ms/个
        }

    async def process_next(self, event_id: int) -> dict | None:
        """从队列取出下一个请求处理"""
        queue_key = f"queue:{event_id}"
        entry = await self.redis.rpop(queue_key)
        if not entry:
            return None
        return json.loads(entry)

    async def get_position(self, event_id: int, seq: int) -> int:
        """获取用户在队列中的当前位置"""
        queue_key = f"queue:{event_id}"
        queue_length = await self.redis.llen(queue_key)
        # position = 总队列长度 - 当前进度（简化计算）
        # 实际可用 Redis ZRANK 做更精确的排名
        return queue_length
```

### 5.4 Worker 处理

```python
# worker.py（ARQ 后台任务）
async def process_queue(ctx, event_id: int):
    """后台 Worker：每秒处理一个排队请求"""
    redis: Redis = ctx["redis"]
    queue = QueueService(redis)

    entry = await queue.process_next(event_id)
    if not entry:
        return  # 队列为空

    user_id = entry["user_id"]

    # Redis Lua 预扣库存（原子操作，防超卖）
    stock_ok = await redis.eval(
        STOCK_DEDUCT_LUA,
        keys=[f"stock:{event_id}"],
        args=[1],
    )
    if not stock_ok:
        await notify_user(redis, user_id, "库存不足")
        return

    # 创建订单
    order = await create_order(user_id, event_id)

    # WebSocket 通知用户
    await notify_user(redis, user_id, "轮到你了", order_no=order.order_no)
```

### 5.5 排队预估时间算法

```
预估时间 = 排队位置 × 单个请求处理时间

单个请求处理时间 = 50ms ~ 200ms（取决于业务复杂度）

例子：
  你在第 5321 位
  预估处理时间 = 5321 × 100ms = 532 秒 ≈ 8.8 分钟
```

---

## 六、三层限流系统

### 6.1 限流层次

| 层次 | 作用范围 | 阈值 | 超出后果 |
|------|---------|------|---------|
| 第一层 IP 限流 | 每个 IP 地址 | 5 次/秒 | 429 Too Many Requests |
| 第二层 用户限流 | 每个登录用户 | 1 次/秒 | 429 "请勿频繁操作" |
| 第三层 接口限流 | 抢票接口整体 | N 次/秒（可配置） | 429 "系统繁忙" |

### 6.2 IP 限流（Nginx）

```nginx
# nginx/conf.d/ratelimit.conf
# 定义限流区域（10MB 空间，记录每个 IP 的访问频率）
limit_req_zone $binary_remote_addr zone=iplimit:10m rate=5r/s;

server {
    location /api/v1/ {
        # IP 限流
        limit_req zone=iplimit burst=10 nodelay;

        # 超出限流时返回 429
        limit_req_status 429;
    }
}
```

### 6.3 用户限流（Redis 滑动窗口）

```python
# middleware/ratelimit.py
async def user_rate_limit(redis: Redis, user_id: int, max_requests: int = 1, window: int = 1):
    """用户级滑动窗口限流"""
    key = f"ratelimit:user:{user_id}"
    now = time.time()
    window_start = now - window

    # 移除窗口外的记录
    await redis.zremrangebyscore(key, 0, window_start)

    # 统计窗口内请求数
    count = await redis.zcard(key)

    if count >= max_requests:
        raise HTTPException(status_code=429, detail="请求过于频繁，请稍后重试")

    # 记录本次请求
    await redis.zadd(key, {str(now): now})
    await redis.expire(key, window * 2)
```

### 6.4 接口限流（抢票接口专用）

```python
async def ticket_purchase_rate_limit(redis: Redis, event_id: int):
    """抢票接口整体限流 — 防止数据库被打满"""
    key = f"ratelimit:purchase:{event_id}"
    max_concurrent = settings.MAX_PURCHASE_CONCURRENT  # 从 .env 读取，可动态调整

    current = await redis.incr(key)
    if current == 1:
        await redis.expire(key, 1)  # 1 秒窗口

    if current > max_concurrent:
        raise HTTPException(status_code=429, detail="当前购票人数过多，请稍后重试")
```

---

## 七、数据库防崩设计

### 7.1 Redis 库存预扣减（原子操作）

```lua
-- scripts/deduct_stock.lua
-- Redis Lua 脚本：原子扣减库存，防超卖

local key = KEYS[1]      -- stock:{event_id}:{ticket_type_id}
local quantity = tonumber(ARGV[1])

local stock = tonumber(redis.call('GET', key) or '0')

if stock >= quantity then
    redis.call('DECRBY', key, quantity)
    return 1  -- 扣减成功
else
    return 0  -- 库存不足
end
```

### 7.2 下单流程（Redis → MySQL）

```
① Redis Lua 预扣库存
   └─ 库存不足 → 返回错误 "库存不足"
   └─ 扣减成功 → 进入下一步

② 创建 Order（MySQL）
   └─ begin transaction

③ MySQL 乐观锁扣减库存
   UPDATE ticket_types
   SET stock = stock - {qty}, version = version + 1
   WHERE id = {id}
     AND stock >= {qty}
     AND version = {version}

   └─ 影响行数 = 0 → 冲突 → rollback + 释放 Redis 库存
   └─ 影响行数 = 1 → 成功

④ commit transaction
```

### 7.3 库存回滚

```python
# 下单失败时释放 Redis 库存
async def rollback_stock(redis: Redis, event_id: int, ticket_type_id: int, quantity: int):
    """订单失败时回滚 Redis 预扣库存"""
    key = f"stock:{event_id}:{ticket_type_id}"
    await redis.incrby(key, quantity)
```

哪些情况需要回滚：

| 场景 | 是否回滚 | 说明 |
|------|---------|------|
| 用户支付超时 | 是 | 订单自动取消，库存释放 |
| 用户主动取消 | 是 | 支付前取消，库存释放 |
| MySQL 乐观锁冲突 | 是 | 高并发下触发的重试，回滚后重新排队 |
| 支付成功 | 否 | 库存已真正消耗，不回滚 |
| 退款 | 否（单独处理） | 退款是独立的逆向流程 |

---

## 八、前端降级系统

### 8.1 设备分级

```
低端设备（手机/旧电脑）：
  └─ 关闭粒子特效
  └─ 关闭背景动画
  └─ 关闭 blur/glassmorphism
  └─ 禁用复杂 transition
  └─ 简化骨架屏

中端设备：
  └─ 简化动画
  └─ 降低粒子密度
  └─ 保留主要视觉效果

高端设备：
  └─ 完整 Prism / Nebula 效果
  └─ 完整粒子系统
  └─ 完整动画
```

### 8.2 自动降级判断

```typescript
// utils/deviceDetect.ts
export function getPerformanceLevel(): 'low' | 'medium' | 'high' {
  const memory = (navigator as any).deviceMemory || 8   // 设备内存（GB）
  const cores = navigator.hardwareConcurrency || 4       // CPU 核心数

  if (memory <= 2 || cores <= 2) {
    return 'low'
  }
  if (memory <= 4 || cores <= 4) {
    return 'medium'
  }
  return 'high'
}

// 抢票模式时强制降级
export function isNearPurchaseTime(eventStartTime: number): boolean {
  const minutesUntilStart = (eventStartTime - Date.now()) / 60000
  return minutesUntilStart < 5  // 开场前 5 分钟强制低配
}
```

### 8.3 降级模板

```vue
<template>
  <!-- 抢票页面 -->
  <div :class="`ticket-page level-${performanceLevel}`">
    <!--
      level-low:    无动画、无粒子、简化布局
      level-medium: 简化动画、无粒子
      level-high:   完整 Prism 效果
    -->
    <ParticleBackground v-if="performanceLevel === 'high'" />
    <SimplifiedBanner v-else />

    <TicketList :compact="performanceLevel === 'low'" />
  </div>
</template>

<script setup lang="ts">
const performanceLevel = getPerformanceLevel()

// 开场前 5 分钟或低端设备 → 关闭所有特效
if (isNearPurchaseTime(eventStore.startTime) || performanceLevel === 'low') {
  document.body.classList.add('performance-mode')
  // 接管全局样式关闭动画、模糊等
}
</script>
```

---

## 九、CDN + 缓存策略

### 9.1 三类资源缓存策略

| 类型 | 缓存位置 | TTL | 示例 |
|------|---------|-----|------|
| 静态资源 | CDN | 长期（1 年） | 图片、CSS、JS 文件 |
| 短缓存数据 | Redis | 10 ~ 30 秒 | 票种列表、活动介绍 |
| 禁止缓存 | 无 | 实时 | 用户信息、订单、支付 |

### 9.2 静态资源 CDN 配置

```nginx
# nginx/conf.d/cdn.conf
location /assets/ {
    # 文件指纹（webpack hash）→ 永久缓存
    expires 1y;
    add_header Cache-Control "public, immutable";
}

location /images/ {
    expires 7d;
    add_header Cache-Control "public";
}

# 不缓存 HTML（确保每次请求到最新页面）
location / {
    expires -1;
    add_header Cache-Control "no-store, must-revalidate";
}
```

### 9.3 后端短缓存

```python
@router.get("/events/{event_id}/tickets")
async def get_ticket_types(event_id: int, redis: Redis = Depends(get_redis)):
    """票种列表 — 短缓存（10 秒）"""
    cache_key = f"tickets:{event_id}"

    # 尝试从缓存读取
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)

    # 缓存未命中 → 查数据库
    ticket_types = await get_ticket_types_from_db(event_id)

    # 写入缓存，10 秒过期
    await redis.setex(cache_key, 10, json.dumps(ticket_types))
    return ticket_types
```

### 9.4 活动模式缓存升级

```python
# 活动模式（抢票中）→ 缓存时间缩短
# 平时模式 → 缓存时间延长
CACHE_TTL = {
    "normal": {
        "tickets": 30,      # 平时 30 秒缓存
        "events": 60,
    },
    "event_mode": {
        "tickets": 5,       # 抢票时 5 秒缓存（尽快反映库存变化）
        "events": 10,
    },
}
```

---

## 十、活动模式切换

### 10.1 两种模式

| | 平时模式 | 活动模式（抢票） |
|------|---------|----------------|
| API 访问 | 正常 | 启用排队系统 |
| 限流阈值 | 宽松 | 收紧 |
| 缓存 TTL | 较长 | 缩短 |
| 前端特效 | 完整 | 强制降级 |
| 刷新频率 | 正常 | 禁止频繁刷新 |

### 10.2 模式切换

```python
# services/event_mode.py
class EventMode:
    """活动模式切换"""

    MODE_KEY = "event:mode:{event_id}"

    @staticmethod
    async def enable(redis: Redis, event_id: int):
        """开启活动模式（抢票前 5 分钟调用）"""
        await redis.setex(f"event:mode:{event_id}", 3600, "active")

    @staticmethod
    async def disable(redis: Redis, event_id: int):
        """关闭活动模式"""
        await redis.delete(f"event:mode:{event_id}")

    @staticmethod
    async def is_active(redis: Redis, event_id: int) -> bool:
        """检查是否为活动模式"""
        return await redis.get(f"event:mode:{event_id}") == b"active"


# 切换到活动模式后，自动启用排队
@router.post("/events/{event_id}/start-purchase")
async def start_purchase_event(event_id: int, redis: Redis = Depends(get_redis)):
    """管理员：开启抢票模式"""
    await EventMode.enable(redis, event_id)
    # 前端会检测到模式切换 → 自动调整为抢票 UI
    return {"mode": "active"}
```

---

## 十一、用户体验保护原则

### 11.1 五项原则

```
原则一：永远不允许白屏
  → 骨架屏（skeleton）必须在 HTML 里先渲染
  → 即使 JS 还没加载，用户也能看到页面轮廓

原则二：永远不允许无响应
  → 所有操作 3 秒内必须有反馈（成功/失败/排队）
  → 超时自动显示 "系统繁忙，请重试"

原则三：永远提供返回路径
  → 每个页面都有返回按钮
  → 错误页有 "回到首页" 链接

原则四：永远提供重试按钮
  → 失败态必须带 "重试" 按钮
  → 重试前清空旧状态，避免重复逻辑

原则五：永远提供状态提示
  → 抢票排队显示位置和等待时间
  → 支付处理显示进度
  → 错误时显示具体原因（不要 "系统错误" 四个字）
```

### 11.2 抢票页面的体验设计

```
页面结构：

┌──────────────────────────────────────────┐
│  活动标题 × 倒计时（02:35）              │
├──────────────────────────────────────────┤
│                                          │
│  ┌─── 票种卡片 ──────────────────────┐  │
│  │  普通票 ¥99                        │  │
│  │  库存：已售 2456/5000              │  │
│  │  [ 立即抢票 ]                      │  │
│  └───────────────────────────────────┘  │
│                                          │
│  如果你在排队：                           │
│  ┌─── 排队信息 ──────────────────────┐  │
│  │  排队人数：12,345                   │  │
│  │  你的位置：#5,321                    │  │
│  │  ████████░░░░░░░ 43%               │  │
│  │  预计等待：2 分 48 秒               │  │
│  │  [ 取消排队 ]                       │  │
│  └───────────────────────────────────┘  │
│                                          │
│  如果已售罄：                             │
│  ┌─── 已售罄 ────────────────────────┐  │
│  │  所有票种已售罄                     │  │
│  │  [ 订阅下次开票通知 ]              │  │
│  └───────────────────────────────────┘  │
│                                          │
│  如果出错：                               │
│  ┌─── 加载失败 ──────────────────────┐  │
│  │  网络异常，请检查连接               │  │
│  │  [ 重试 ]    [ 返回首页 ]          │  │
│  └───────────────────────────────────┘  │
│                                          │
└──────────────────────────────────────────┘
```

---

## 十二、高并发压测目标

### 12.1 压测场景

| 场景 | 并发数 | 目标 |
|------|--------|------|
| 正常浏览 | 1000 并发 | 稳定运行，P99 < 500ms |
| 抢票高峰 | 5000 并发 | 可降级运行，不崩溃 |
| 极限压测 | 10000 并发 | 不崩溃，自动排队 |

### 12.2 通过标准

```
□ 1000 并发：所有接口正常响应，无超时
□ 5000 并发：启动排队 + 限流，系统不崩溃
□ 10000 并发：前端显示排队页，数据库连接池不耗尽
□ 零超卖：任何并发下不出现超卖
□ 零卡死：所有用户都能看到响应（成功/失败/排队）
```

### 12.3 压测命令

```bash
# Locust 压测
locust -f locustfile.py --host=http://localhost:8000 --users=1000 --spawn-rate=100

# 或直接用 wrk
wrk -t12 -c1000 -d30s http://localhost:8000/api/v1/events

# 抢票接口专项压测
wrk -t4 -c500 -d60s --script=./purchase.lua http://localhost:8000/api/v1/orders
```

---

## 十三、部署方案

### 13.1 Nginx 高并发配置

```nginx
# nginx/conf.d/high-concurrency.conf

# ── 基础优化 ──
worker_processes auto;               # 自动匹配 CPU 核心数
worker_rlimit_nofile 65535;          # 单进程最大文件句柄数

events {
    use epoll;                       # Linux 高性能事件模型
    worker_connections 65535;        # 单进程最大连接数
    multi_accept on;                 # 一次 accept 多个连接
}

http {
    # ── 基础优化 ──
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    keepalive_requests 1000;         # 单连接最大请求数

    # ── 客户端缓冲 ──
    client_body_buffer_size 128k;
    client_max_body_size 1m;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 8k;

    # ── 代理缓冲 ──
    proxy_buffering on;
    proxy_buffers 8 32k;
    proxy_buffer_size 4k;
    proxy_busy_buffers_size 64k;

    # ── 限流配置 ──
    # IP 限流：5 次/秒
    limit_req_zone $binary_remote_addr zone=iplimit:10m rate=5r/s;
    # 抢票接口限流：200 次/秒
    limit_req_zone $binary_remote_addr zone=purchaselimit:10m rate=200r/s;

    # ── 上游 FastAPI 集群 ──
    upstream fastapi_backend {
        # 最少连接数负载均衡
        least_conn;

        # 多个 FastAPI 实例
        server 127.0.0.1:8001 max_fails=3 fail_timeout=30s;
        server 127.0.0.1:8002 max_fails=3 fail_timeout=30s;
        server 127.0.0.1:8003 max_fails=3 fail_timeout=30s;

        # 保持长连接
        keepalive 32;
    }

    server {
        listen 80;
        server_name your-domain.com;

        # ── 限制请求 ──
        limit_req zone=iplimit burst=10 nodelay;
        limit_req_status 429;

        # ── 静态资源（CDN 缓存） ──
        location /assets/ {
            root /var/www/frontend/dist;
            expires 1y;
            add_header Cache-Control "public, immutable";
            access_log off;
        }

        # ── 抢票接口（专用限流） ──
        location /api/v1/orders/ {
            limit_req zone=purchaselimit burst=20 nodelay;
            proxy_pass http://fastapi_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            # 超时配置
            proxy_connect_timeout 5s;
            proxy_read_timeout 10s;
            proxy_send_timeout 5s;
        }

        # ── 普通 API ──
        location /api/v1/ {
            proxy_pass http://fastapi_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # 长连接
            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }

        # ── 静态首页 ──
        location / {
            root /var/www/frontend/dist;
            index index.html;
            try_files $uri $uri/ /index.html;

            # 不缓存 HTML
            expires -1;
            add_header Cache-Control "no-store, must-revalidate";
        }

        # ── 429 自定义响应 ──
        error_page 429 /429.html;
        location = /429.html {
            root /var/www/frontend/dist;
            internal;
        }
    }
}
```

### 13.2 Docker Compose 高并发部署

```yaml
# docker-compose.prod.yml
version: "3.8"

services:
  # ── MySQL ──
  mysql:
    image: mysql:8.0
    restart: always
    volumes:
      - mysql_data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
    deploy:
      resources:
        limits:
          cpus: "2"
          memory: "4G"
        reservations:
          cpus: "1"
          memory: "2G"

  # ── Redis ──
  redis:
    image: redis:7-alpine
    restart: always
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes --maxmemory 2gb --maxmemory-policy allkeys-lru
    deploy:
      resources:
        limits:
          cpus: "2"
          memory: "4G"

  # ── FastAPI 实例 1 ──
  api-1:
    build: .
    restart: always
    depends_on:
      - mysql
      - redis
    environment:
      - ENVIRONMENT=production
    env_file:
      - .env
    command: >
      uvicorn app.main:app --host 0.0.0.0 --port 8000
        --workers 4                         # 4 个 worker 进程
        --limit-concurrency 1000            # 最大并发连接
        --backlog 2048                      # 连接队列大小
        --no-access-log
    deploy:
      resources:
        limits:
          cpus: "2"
          memory: "2G"

  # ── FastAPI 实例 2 ──
  api-2:
    build: .
    restart: always
    depends_on:
      - mysql
      - redis
    environment:
      - ENVIRONMENT=production
    env_file:
      - .env
    command: >
      uvicorn app.main:app --host 0.0.0.0 --port 8000
        --workers 4
        --limit-concurrency 1000
        --backlog 2048
        --no-access-log
    deploy:
      resources:
        limits:
          cpus: "2"
          memory: "2G"

  # ── FastAPI 实例 3 ──
  api-3:
    build: .
    restart: always
    depends_on:
      - mysql
      - redis
    environment:
      - ENVIRONMENT=production
    env_file:
      - .env
    command: >
      uvicorn app.main:app --host 0.0.0.0 --port 8000
        --workers 4
        --limit-concurrency 1000
        --backlog 2048
        --no-access-log
    deploy:
      resources:
        limits:
          cpus: "2"
          memory: "2G"

  # ── Nginx ──
  nginx:
    image: nginx:alpine
    restart: always
    depends_on:
      - api-1
      - api-2
      - api-3
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./frontend/dist:/var/www/frontend/dist
      - ./ssl:/etc/nginx/ssl
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: "512M"

  # ── ARQ Worker（后台处理排队、自动取消） ──
  worker:
    build: .
    restart: always
    depends_on:
      - mysql
      - redis
    environment:
      - ENVIRONMENT=production
    env_file:
      - .env
    command: arq app.worker.WorkerSettings
    deploy:
      replicas: 3  # 3 个 Worker 实例，并行处理队列
      resources:
        limits:
          cpus: "1"
          memory: "1G"

volumes:
  mysql_data:
  redis_data:
```

### 13.3 高并发环境变量参考

```env
# .env.production — 生产环境高并发配置

# ── 数据库连接池 ──
DB_POOL_SIZE=20              # 连接池大小（默认 10，高并发时建议 20~50）
DB_MAX_OVERFLOW=40           # 最大溢出连接数（默认 20）
DB_POOL_TIMEOUT=30           # 等待连接超时（秒）
DB_POOL_RECYCLE=1800         # 连接回收时间（秒）

# ── Redis ──
REDIS_MAX_CONNECTIONS=100    # Redis 最大连接数

# ── 限流配置（从 .env 读取，可动态调整） ──
RATE_LIMIT_AUTH=20/minute    # 登录接口：20 次/分钟
RATE_LIMIT_API=100/minute    # 通用接口：100 次/分钟
RATE_LIMIT_PURCHASE=200/second  # 抢票接口：200 次/秒

# ── 抢票配置 ──
MAX_PURCHASE_CONCURRENT=200  # 抢票接口最大并发
QUEUE_WORKER_INTERVAL=0.1    # Worker 处理间隔（秒）
QUEUE_TIMEOUT=600            # 排队超时时间（秒）

# ── Uvicorn 配置 ──
UVICORN_WORKERS=4            # 每个实例的 worker 数
UVICORN_LIMIT_CONCURRENCY=1000  # 最大并发连接
UVICORN_BACKLOG=2048         # 连接队列大小
```

### 13.4 系统调优

#### 操作系统级别

```bash
# /etc/sysctl.conf — 高并发内核优化

# 最大文件句柄数
fs.file-max = 1000000

# TCP 连接优化
net.ipv4.tcp_tw_reuse = 1          # 重用 TIME_WAIT 连接
net.ipv4.tcp_fin_timeout = 30      # FIN 超时缩短
net.ipv4.tcp_keepalive_time = 1200 # keepalive 间隔
net.ipv4.tcp_max_syn_backlog = 8192   # SYN 队列长度
net.ipv4.tcp_max_tw_buckets = 5000    # TIME_WAIT 上限

# 端口范围
net.ipv4.ip_local_port_range = 1024 65535
```

#### Python / Uvicorn 级别

```bash
# 启动参数说明
uvicorn app.main:app \
    --workers 4 \                # 多 worker（每个 worker 独立进程）
    --limit-concurrency 1000 \   # 最大并发连接数
    --backlog 2048 \             # 连接等待队列长度
    --no-access-log \            # 关闭访问日志（减少磁盘 IO）
    --proxy-headers \            # 信任 Nginx 转发的 Headers
    --timeout-keep-alive 30      # 长连接超时
```

---

## 十四、压测方案

### 14.1 Locust 压测脚本

```python
# locustfile.py
"""
Locust 压测脚本 — 模拟抢票高峰

用法：
    # Web 模式（观察实时图表）
    locust -f locustfile.py --host=http://localhost:8000 --web-port=8089

    # 命令行模式（自动生成报告）
    locust -f locustfile.py --host=http://localhost:8000 \
        --users=1000 --spawn-rate=100 --run-time=5m \
        --headless --csv=report
"""

import random
import json
from locust import HttpUser, task, between, SequentialTaskSet


class NormalBrowsingUser(HttpUser):
    """模拟正常浏览用户 — 浏览活动、查看详情"""
    wait_time = between(3, 10)  # 用户思考时间 3~10 秒
    weight = 3                  # 权重：这类用户占比高

    @task(5)
    def browse_events(self):
        """浏览活动列表"""
        with self.client.get(
            "/api/v1/events",
            name="浏览活动列表",
            catch_response=True,
        ) as resp:
            if resp.status_code == 429:
                resp.success()  # 限流是预期行为，不计为失败
            elif resp.status_code != 200:
                resp.failure(f"返回 {resp.status_code}")

    @task(3)
    def view_event_detail(self):
        """查看活动详情"""
        event_id = random.choice([1, 2, 3, 4, 5])
        with self.client.get(
            f"/api/v1/events/{event_id}",
            name="查看活动详情",
            catch_response=True,
        ) as resp:
            if resp.status_code == 429:
                resp.success()
            elif resp.status_code != 200:
                resp.failure(f"返回 {resp.status_code}")

    @task(2)
    def view_ticket_types(self):
        """查看票种"""
        event_id = random.choice([1, 2, 3])
        with self.client.get(
            f"/api/v1/events/{event_id}/tickets",
            name="查看票种",
            catch_response=True,
        ) as resp:
            if resp.status_code == 429:
                resp.success()
            elif resp.status_code != 200:
                resp.failure(f"返回 {resp.status_code}")


class PurchaseUser(HttpUser):
    """模拟抢票用户 — 登录 → 抢票"""
    wait_time = between(1, 3)  # 思考时间更短（更急切）
    weight = 1                  # 权重：占比低于浏览用户

    def on_start(self):
        """每个用户启动时先登录"""
        username = f"test_user_{random.randint(1, 10000)}"
        password = "test123456"
        with self.client.post(
            "/api/v1/auth/login",
            json={"username": username, "password": password},
            name="用户登录",
            catch_response=True,
        ) as resp:
            if resp.status_code == 200:
                token = resp.json().get("access_token")
                self.headers = {"Authorization": f"Bearer {token}"}
            else:
                self.headers = {}

    @task(1)
    def purchase_ticket(self):
        """抢票"""
        if not self.headers:
            return

        with self.client.post(
            "/api/v1/orders",
            json={
                "event_id": 1,
                "ticket_type_id": 1,
                "quantity": 1,
            },
            headers=self.headers,
            name="抢票下单",
            catch_response=True,
        ) as resp:
            if resp.status_code == 429:
                # 限流 = 正常，不算失败
                resp.success()
            elif resp.status_code == 400:
                # 库存不足 = 正常，不算失败
                resp.success()
            elif resp.status_code == 200:
                resp.success()
            elif resp.status_code == 503:
                # 排队中 = 正常
                resp.success()
            else:
                resp.failure(f"下单失败: {resp.status_code}")


class QueueUser(HttpUser):
    """模拟排队用户 — 进入排队并轮询状态"""
    wait_time = between(0.5, 1.5)
    weight = 1

    def on_start(self):
        username = f"queue_user_{random.randint(1, 5000)}"
        with self.client.post(
            "/api/v1/auth/login",
            json={"username": username, "password": "test123456"},
            name="排队用户登录",
            catch_response=True,
        ) as resp:
            if resp.status_code == 200:
                self.headers = {"Authorization": f"Bearer {resp.json()['access_token']}"}
            else:
                self.headers = {}

    @task(3)
    def join_queue(self):
        """加入排队"""
        if not self.headers:
            return
        with self.client.put(
            "/api/v1/queue/join",
            json={"event_id": 1, "ticket_type_id": 1},
            headers=self.headers,
            name="加入排队",
            catch_response=True,
        ) as resp:
            if resp.status_code in (200, 429, 503):
                resp.success()
            else:
                resp.failure(f"排队失败: {resp.status_code}")

    @task(2)
    def check_position(self):
        """查询排队位置"""
        if not self.headers:
            return
        with self.client.get(
            "/api/v1/queue/position?event_id=1",
            headers=self.headers,
            name="查询排队位置",
            catch_response=True,
        ) as resp:
            if resp.status_code == 200:
                resp.success()
            else:
                resp.failure(f"查询失败: {resp.status_code}")
```

### 14.2 wrk 压测脚本（抢票接口专项）

```lua
-- purchase.lua
-- wrk 专用脚本：模拟抢票 POST 请求
-- 用法：wrk -t4 -c500 -d60s --script=./purchase.lua http://localhost:8000

wrk.method = "POST"
wrk.body = [[{"event_id":1,"ticket_type_id":1,"quantity":1}]]
wrk.headers["Content-Type"] = "application/json"
wrk.headers["Authorization"] = "Bearer test_token_for_benchmark"
```

### 14.3 压测执行步骤

```bash
# ═══════════════════════════════════════════════
# 第一步：部署待测环境
# ═══════════════════════════════════════════════

# 启动服务
docker compose -f docker-compose.prod.yml up -d

# 确认所有服务就绪
docker compose ps

# 运行迁移
alembic upgrade head

# 生成测试数据（10000 张票，5000 个用户）
python scripts/seed_test_data.py --tickets 10000 --users 5000


# ═══════════════════════════════════════════════
# 第二步：执行压测
# ═══════════════════════════════════════════════

# 场景 1：正常浏览（1000 并发，持续 5 分钟）
locust -f locustfile.py --host=http://localhost:8000 \
    --users=1000 --spawn-rate=100 --run-time=5m \
    --headless --csv=report_normal

# 场景 2：抢票高峰（5000 并发，持续 3 分钟）
locust -f locustfile.py --host=http://localhost:8000 \
    --users=5000 --spawn-rate=500 --run-time=3m \
    --headless --csv=report_peak

# 场景 3：极限压测（10000 并发，持续 1 分钟）
locust -f locustfile.py --host=http://localhost:8000 \
    --users=10000 --spawn-rate=1000 --run-time=1m \
    --headless --csv=report_extreme


# ═══════════════════════════════════════════════
# 第三步：分析结果
# ═══════════════════════════════════════════════

# 查看汇总报告
cat report_peak_stats.csv

# 关键指标解释：
# Request Count     → 总请求数（越高越好）
# Average (ms)      → 平均响应时间（越低越好，目标 <500ms）
# P50/P90/P99 (ms)  → 百分位响应时间（P99 <2000ms 为合格）
# Failures          → 失败请求数（应该为 0）
# RPS               → 每秒请求数（越高越好）
# 429 Responses     → 限流响应（正常现象，说明限流生效）


# ═══════════════════════════════════════════════
# 第四步：专项验证
# ═══════════════════════════════════════════════

# 验证零超卖（检查数据库库存 + 订单数是否一致）
python scripts/verify_stock.py --event-id=1

# 验证排队系统（压测后检查队列是否清空）
redis-cli LLEN queue:1
# 预期输出：0（所有排队已处理完）

# 验证数据库连接池未耗尽
mysqladmin status -h localhost -u root -p
# 预期输出：Threads_connected < max_connections
```

### 14.4 压测报告模板

```text
═══════════════════════════════════════════════
       高并发压测报告
═══════════════════════════════════════════════

测试环境：
  服务器：4C8G × 3 实例
  MySQL：2C4G
  Redis：2C4G
  Nginx：1C512M

场景一：正常浏览（1000 并发 × 5 分钟）
  ┌──────────────────────┬────────────┬────────┐
  │ 指标                 │ 值         │ 通过?  │
  ├──────────────────────┼────────────┼────────┤
  │ 总请求数             │ 123,456    │ —      │
  │ 平均 RPS             │ 411        │ —      │
  │ 平均响应时间 (ms)    │ 89         │ ✅     │
  │ P99 响应时间 (ms)    │ 312        │ ✅     │
  │ 失败数               │ 0          │ ✅     │
  │ 429 限流数           │ 1,234      │ ✅     │
  └──────────────────────┴────────────┴────────┘

场景二：抢票高峰（5000 并发 × 3 分钟）
  ┌──────────────────────┬────────────┬────────┐
  │ 指标                 │ 值         │ 通过?  │
  ├──────────────────────┼────────────┼────────┤
  │ 总请求数             │ 45,678     │ —      │
  │ 平均 RPS             │ 253        │ —      │
  │ 平均响应时间 (ms)    │ 1,245      │ ⚠️     │
  │ P99 响应时间 (ms)    │ 4,567      │ ⚠️     │
  │ 失败数               │ 0          │ ✅     │
  │ 429 限流数           │ 12,345     │ ✅     │
  │ 排队成功数           │ 5,000      │ ✅     │
  └──────────────────────┴────────────┴────────┘

场景三：极限压测（10000 并发 × 1 分钟）
  ┌──────────────────────┬────────────┬────────┐
  │ 指标                 │ 值         │ 通过?  │
  ├──────────────────────┼────────────┼────────┤
  │ 总请求数             │ 12,345     │ —      │
  │ 平均 RPS             │ 205        │ —      │
  │ 平均响应时间 (ms)    │ 2,891      │ ⚠️     │
  │ P99 响应时间 (ms)    │ 8,234      │ ⚠️     │
  │ 失败数               │ 0          │ ✅     │
  │ 429 限流数           │ 8,901      │ ✅     │
  │ 排队成功数           │ 10,000     │ ✅     │
  │ 系统崩溃?            │ 否         │ ✅     │
  └──────────────────────┴────────────┴────────┘

验证结果：
  □ 零超卖验证：库存 = 订单总量 ✅
  □ 排队队列清空 ✅
  □ 数据库连接池正常 ✅
  □ 内存无泄漏 ✅

结论：
  [ ] 通过 — 达到生产上线标准
  [ ] 有条件通过 — 需优化 XXX
  [ ] 不通过 — XXX 项未达标
```

### 14.5 CI/CD 压测自动化

```yaml
# .github/workflows/load-test.yml
name: 高并发压测

on:
  schedule:
    - cron: "0 3 * * 0"  # 每周日凌晨 3 点执行
  workflow_dispatch:       # 支持手动触发

jobs:
  load-test:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root123
          MYSQL_DATABASE: ticket_test
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: --health-cmd="redis-cli ping" --health-interval=10s

    steps:
      - uses: actions/checkout@v4

      - name: 设置 Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: 安装依赖
        run: |
          pip install -r requirements.txt
          pip install locust

      - name: 运行迁移
        run: |
          alembic upgrade head
        env:
          DB_HOST: localhost
          DB_PASSWORD: root123
          DB_NAME: ticket_test

      - name: 生成测试数据
        run: python scripts/seed_test_data.py --tickets 10000 --users 5000
        env:
          DB_HOST: localhost
          DB_PASSWORD: root123

      - name: 启动服务
        run: |
          uvicorn app.main:app --host 0.0.0.0 --port 8000 &
          sleep 5  # 等待服务就绪

      - name: 执行压测
        run: |
          locust -f locustfile.py --host=http://localhost:8000 \
              --users=1000 --spawn-rate=100 --run-time=3m \
              --headless --csv=ci_report

      - name: 分析结果
        run: |
          # 检查失败率
          FAILURES=$(awk -F',' 'NR>1{print $9}' ci_report_stats.csv | tail -1)
          if [ "$FAILURES" != "0" ]; then
            echo "❌ 压测失败：存在失败请求"
            exit 1
          fi

          # 检查 P99 响应时间
          P99=$(awk -F',' 'NR>1{print $8}' ci_report_stats.csv | tail -1)
          if [ "${P99%.*}" -gt 2000 ]; then
            echo "⚠️  P99 响应时间 ${P99}ms 超过 2000ms 阈值"
          fi

          echo "✅ 压测通过"

      - name: 上传报告
        uses: actions/upload-artifact@v4
        with:
          name: load-test-report
          path: ci_report*.csv
```

### 14.6 压测验收清单

```
□ 场景一（1000 并发）：
  □ 平均响应时间 < 500ms
  □ P99 响应时间 < 2000ms
  □ 零失败
  □ 所有接口正确响应

□ 场景二（5000 并发）：
  □ 系统不崩溃
  □ 排队系统正常运行
  □ 限流生效（429 正确返回）
  □ 前端正常显示排队页

□ 场景三（10000 并发）：
  □ 系统不崩溃
  □ 数据库连接池不耗尽
  □ Redis 不 OOM
  □ Nginx 不 502

□ 业务验证：
  □ 零超卖（库存 = 订单数 + 剩余库存）
  □ 排队数据最终一致（队列全部处理完）
  □ 支付流程正常（压测后付款不出错）
```

---

## 总结

```
高并发不是"优化一下代码"能解决的。

它是一个完整架构问题，需要：

  前端：超时、降级、防重复、骨架屏
  网关：IP 限流、用户限流、接口限流
  排队：Redis Queue、Worker、预估等待时间
  库存：Redis 预扣、Lua 原子操作、乐观锁
  缓存：CDN、Redis 短缓存、分级策略
  模式：平时模式 + 活动模式切换

缺任何一个 → 高并发下系统扛不住。
```
