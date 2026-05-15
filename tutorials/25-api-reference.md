# API 参考文档

> 本文档列出所有 API 端点，包括请求/响应格式、认证要求和错误码。

---

## 目录

1. [通用信息](#1-通用信息)
2. [认证 API](#2-认证-api)
3. [活动 API](#3-活动-api)
4. [订单 API](#4-订单-api)
5. [支付 API](#5-支付-api)
6. [管理员 API](#6-管理员-api)
7. [WebSocket 端点](#7-websocket-端点)
8. [核销 API](#8-核销-api)
9. [健康检查](#9-健康检查)
10. [错误码汇总](#10-错误码汇总)
11. [速率限制与响应头](#11-速率限制与响应头)

---

## 1. 通用信息

### 1.1 基础 URL

```text
开发环境：http://localhost:8000/api/v1
生产环境：https://your-domain.com/api/v1
```

### 1.2 认证方式

大多数 API 需要 JWT Bearer Token 认证：

```http
Authorization: Bearer <access_token>
```text

在需要认证的端点中，如果请求头中不包含有效的 access_token，将返回 `401 Unauthorized`。

### 1.3 通用响应格式

**成功响应**：返回请求的 JSON 数据。

**错误响应**：

```json
{
  "detail": "错误描述信息"
}
```

### 1.4 通用请求头

| 头 | 值 | 说明 |
| ------ | ------- | ------ |
| `Content-Type` | `application/json` | 请求体格式 |
| `Authorization` | `Bearer <token>` | JWT 认证令牌 |
| `X-Request-ID` | 由服务端生成 | 请求追踪 ID（响应头） |

---

## 2. 认证 API

### POST `/auth/register`

注册新用户。

**请求体**：

```json
{
  "username": "test_user",
  "password": "TestPass123!",
  "email": "user@example.com",
  "phone": "13800138000"
}
```text

**字段说明**：

| 字段 | 类型 | 必填 | 约束 |
| ------ | ------ | ------ | ------ |
| username | string | 是 | 3-50 字符，字母数字下划线 |
| password | string | 是 | 8-128 字符，需含大小写字母、数字、特殊字符 |
| email | string | 是 | 有效邮箱格式 |
| phone | string | 否 | 11 位手机号，以 1 开头 |

**成功响应** (201 Created)：

```json
{
  "id": 1,
  "username": "test_user",
  "email": "user@example.com",
  "phone": "13800138000",
  "avatar": null,
  "role": "user",
  "is_active": true,
  "created_at": "2026-05-13T10:00:00Z"
}
```

**错误响应**：

| 状态码 | 场景 |
| -------- | ------ |
| 409 Conflict | 用户名或邮箱已存在 |
| 422 Unprocessable Entity | 参数校验失败（如密码强度不足） |

---

### POST `/auth/login`

用户登录，返回 JWT 令牌对。

**请求体**：

```json
{
  "username": "test_user",
  "password": "TestPass123!"
}
```text

**成功响应** (200 OK)：

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiJ9...",
  "token_type": "bearer",
  "expires_in": 1800
}
```

**令牌说明**：

| 字段 | 说明 |
| ------ | ------ |
| access_token | 访问令牌，有效期 30 分钟 |
| refresh_token | 刷新令牌，有效期 7 天 |
| token_type | 固定为 `bearer` |
| expires_in | 访问令牌过期时间（秒） |

**错误响应**：

| 状态码 | 场景 |
| -------- | ------ |
| 401 Unauthorized | 用户名或密码错误 |
| 401 Unauthorized | 账户已锁定 |

---

### POST `/auth/refresh`

刷新访问令牌（带轮换机制）。旧 refresh_token 刷新后失效。

**请求体**：

```json
{
  "refresh_token": "eyJhbGciOiJIUzI1NiJ9..."
}
```text

**成功响应** (200 OK)：

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiJ9...",
  "token_type": "bearer",
  "expires_in": 1800
}
```

**错误响应**：

| 状态码 | 场景 |
| -------- | ------ |
| 401 Unauthorized | 无效或已吊销的刷新令牌 |

---

### GET `/auth/me`

获取当前登录用户信息。

**请求头**：`Authorization: Bearer <access_token>`

**成功响应** (200 OK)：

```json
{
  "id": 1,
  "username": "test_user",
  "email": "user@example.com",
  "phone": "13800138000",
  "avatar": null,
  "role": "user",
  "is_active": true,
  "created_at": "2026-05-13T10:00:00Z"
}
```text

**错误响应**：

| 状态码 | 场景 |
| -------- | ------ |
| 401 Unauthorized | 未提供或无效的令牌 |
| 403 Forbidden | 账户已被封禁 |

---

## 3. 活动 API

### GET `/events`

浏览活动列表（公开接口，无需认证）。

**查询参数**：

| 参数 | 类型 | 必填 | 默认值 | 说明 |
| ------ | ------ | ------ | -------- | ------ |
| page | integer | 否 | 1 | 页码，>= 1 |
| size | integer | 否 | 20 | 每页数量，1-100 |
| category | string | 否 | null | 分类筛选：concert/sports/theater/other |
| keyword | string | 否 | null | 关键词搜索（匹配标题） |
| min_price | number | 否 | null | 最低价格 |
| max_price | number | 否 | null | 最高价格 |
| status | string | 否 | published | 活动状态 |

**成功响应** (200 OK)：

```json
{
  "items": [
    {
      "id": 1,
      "title": "周杰伦演唱会",
      "description": "2026 巡演",
      "venue": "国家体育场",
      "start_date": "2026-12-01T19:00:00Z",
      "end_date": "2026-12-01T23:00:00Z",
      "category": "concert",
      "price": 680.00,
      "total_stock": 10000,
      "sold_count": 3456,
      "status": "published",
      "created_by": 1,
      "created_at": "2026-03-01T10:00:00Z",
      "updated_at": "2026-03-01T10:00:00Z"
    }
  ],
  "total": 50,
  "page": 1,
  "size": 20,
  "total_pages": 3
}
```

**分页信息**：

| 字段 | 说明 |
| ------ | ------ |
| total | 总记录数 |
| page | 当前页码 |
| size | 每页数量 |
| total_pages | 总页数 = ceil(total / size) |

---

### GET `/events/{event_id}`

获取活动详情（公开接口）。

**路径参数**：

| 参数 | 类型 | 说明 |
| ------ | ------ | ------ |
| event_id | integer | 活动 ID |

**成功响应** (200 OK)：同上 `EventResponse` 格式。

**错误响应**：

| 状态码 | 场景 |
| -------- | ------ |
| 404 Not Found | 活动不存在 |

---

### POST `/events`

创建活动（需要组织者或管理员权限）。

**请求头**：`Authorization: Bearer <access_token>`

**请求体**：

```json
{
  "title": "周杰伦演唱会",
  "description": "2026 全球巡演北京站",
  "venue": "国家体育场",
  "start_date": "2026-12-01T19:00:00Z",
  "end_date": "2026-12-01T23:00:00Z",
  "category": "concert",
  "price": 680.00,
  "total_stock": 10000
}
```text

**字段说明**：

| 字段 | 类型 | 必填 | 约束 |
| ------ | ------ | ------ | ------ |
| title | string | 是 | 1-200 字符 |
| venue | string | 是 | 1-200 字符 |
| start_date | datetime | 是 | 必须早于 end_date |
| end_date | datetime | 是 | 必须晚于 start_date |
| category | string | 否 | concert/sports/theater/other |
| price | number | 是 | > 0，<= 99999.99 |
| total_stock | integer | 是 | 1-100000 |

**成功响应** (201 Created)：返回 `EventResponse`。

**错误响应**：

| 状态码 | 场景 |
| -------- | ------ |
| 401 Unauthorized | 未登录 |
| 403 Forbidden | 非组织者/管理员角色 |

---

### PUT `/events/{event_id}`

更新活动（需要组织者或管理员权限）。

**请求头**：`Authorization: Bearer <access_token>`

**请求体**（全部可选）：

```json
{
  "title": "更新后的标题",
  "price": 888.00
}
```

**安全限制**：

- 组织者只能更新自己创建的活动
- 管理员可以更新任何活动

---

### DELETE `/events/{event_id}`

删除活动（需要管理员权限）。

**请求头**：`Authorization: Bearer <access_token>`

**成功响应**：204 No Content

---

## 4. 订单 API

### POST `/orders`

创建订单（需要登录）。

**请求头**：`Authorization: Bearer <access_token>`

**请求体**：

```json
{
  "event_id": 1,
  "quantity": 2
}
```text

**行为限制**：

| 限制项 | 阈值 | 说明 |
| -------- | ------ | ------ |
| 下单频率 | 5 单/10 分钟 | 超过返回 429 |
| 并发未支付订单 | 最多 3 笔 | 超过返回 429 |
| IP 下单频率 | 20 单/小时 | 超过返回 429 |

**成功响应** (201 Created)：

```json
{
  "id": 1,
  "order_no": "TKT20260513a1b2c3d4",
  "user_id": 1,
  "event_id": 1,
  "quantity": 2,
  "total_amount": 1360.00,
  "status": "pending_payment",
  "payment_method": null,
  "trade_no": null,
  "paid_at": null,
  "cancelled_at": null,
  "created_at": "2026-05-13T10:00:00Z",
  "updated_at": "2026-05-13T10:00:00Z"
}
```

**错误响应**：

| 状态码 | 场景 |
| -------- | ------ |
| 400 Bad Request | 库存不足、活动未上架、活动已开始 |
| 401 Unauthorized | 未登录 |
| 429 Too Many Requests | 频率超限或并发订单超限 |
| 422 Unprocessable Entity | 参数校验失败 |

---

### GET `/orders`

获取当前用户的订单列表（需要登录）。

**请求头**：`Authorization: Bearer <access_token>`

**查询参数**：

| 参数 | 类型 | 默认值 | 说明 |
| ------ | ------ | -------- | ------ |
| page | integer | 1 | >= 1 |
| size | integer | 20 | 1-100 |

**成功响应** (200 OK)：

```json
{
  "items": [...],
  "total": 10,
  "page": 1,
  "size": 20
}
```text

---

### GET `/orders/{order_id}`

获取订单详情（需要登录，只能查看自己的订单）。

**请求头**：`Authorization: Bearer <access_token>`

**错误响应**：

| 状态码 | 场景 |
| -------- | ------ |
| 404 Not Found | 订单不存在（或不属于当前用户） |

---

### POST `/orders/{order_id}/cancel`

取消订单（需要登录，只能取消自己的订单）。

**请求头**：`Authorization: Bearer <access_token>`

**限制**：只有 `pending_payment` 或 `paid` 状态的订单可以取消。

**成功响应** (200 OK)：返回 `OrderResponse`，status 变为 `cancelled`。

---

## 5. 支付 API

### POST `/payments/alipay`

获取支付宝支付 URL（需要登录）。

**请求头**：`Authorization: Bearer <access_token>`

**请求体**：

```json
{
  "order_id": 1
}
```

**成功响应** (200 OK)：

```json
{
  "pay_url": "https://openapi.alipaydev.com/gateway.do?app_id=...&sign=...",
  "order_no": "TKT20260513a1b2c3d4",
  "total_amount": 1360.00
}
```text

前端可直接将 `pay_url` 在新窗口打开，跳转到支付宝支付页面。

---

### POST `/payments/alipay/notify`

支付宝异步通知处理（无需认证，支付宝 POST 回调专用）。

**请求体**：支付宝 POST 表单数据（`application/x-www-form-urlencoded`）

**安全措施**：

1. RSA2 签名验证
2. 交易状态检查（仅处理 TRADE_SUCCESS）
3. Redis SETNX 幂等键（防止重复处理）
4. 金额校验 + 订单状态机检查

**成功响应** (200 OK)：

```json
{
  "result": "success"
}
```

**错误响应**：

| 状态码 | 场景 |
| -------- | ------ |
| 400 Bad Request | 通知验证失败 |

---

### POST `/payments/refund`

退款（需要管理员权限）。

**请求头**：`Authorization: Bearer <access_token>`

**请求体**：

```json
{
  "order_id": 1,
  "reason": "用户申请退款"
}
```text

**限制**：只有 `paid` 状态的订单可以退款。

---

## 6. 管理员 API

所有管理员 API 需要 `admin` 角色。

### GET `/admin/users`

用户列表。

**请求头**：`Authorization: Bearer <access_token>`

**查询参数**：

| 参数 | 类型 | 说明 |
| ------ | ------ | ------ |
| page | integer | 页码 |
| size | integer | 每页数量 |
| role | string | 角色过滤：user/organizer/admin |
| is_banned | boolean | 封禁状态过滤 |
| keyword | string | 搜索关键词（用户名/邮箱/手机号） |

---

### GET `/admin/users/{user_id}`

获取用户详细信息（含订单统计）。

---

### POST `/admin/users/{user_id}/ban`

封禁用户。

**请求体**：

```json
{
  "reason": "恶意刷票"
}
```

---

### POST `/admin/users/{user_id}/unban`

解封用户。

---

### PUT `/admin/users/{user_id}`

修改用户信息。

**请求体**：

```json
{
  "role": "organizer",
  "is_active": true
}
```text

---

### GET `/admin/users/{user_id}/login-history`

获取用户登录历史。

| 参数 | 类型 | 默认值 |
| ------ | ------ | -------- |
| limit | integer | 20 |

---

## 7. WebSocket 端点

### WS `/shows/{event_id}/seats`

实时座位更新。通过 JWT token 查询参数认证。

**连接**：

```

ws://localhost:8000/api/v1/shows/1/seats?token=<access_token>

```text

**客户端 → 服务器消息**：

```json
// 选择座位
{"type": "select", "seat": "A1"}

// 释放座位
{"type": "release", "seat": "A1"}

// 心跳
{"type": "ping"}
```

**服务器 → 客户端消息**：

```json
// 连接确认
{"type": "connected", "authenticated": true, "event_id": 1}

// 座位被选
{"type": "seat_selected", "seat": "A1", "user": "test_user"}

// 座位释放
{"type": "seat_released", "seat": "A1"}

// 心跳响应
{"type": "pong"}

// 错误
{"type": "error", "message": "seat 格式无效"}
```text

---

### WS `/user/orders`

用户订单状态实时更新。**需要 JWT token 认证**。

**连接**：

```

ws://localhost:8000/api/v1/user/orders?token=<access_token>

```text

**服务器 → 客户端消息**：

```json
// 连接确认
{"type": "connected", "user_id": 42}

// 订单状态变更
{"type": "order_status", "order_id": 1, "status": "paid", "order_no": "TKT20260513..."}
```

**安全措施**：

1. Origin 头校验（CSWSH 防护）
2. JWT token 认证（查询参数传递）
3. 消息大小限制（64KB）
4. 消息类型白名单

---

## 8. 核销 API

核销 API 用于活动入场时的票证验证。所有核销接口需要 `organizer` 或 `admin` 角色。

### POST `/verification/verify`

核销单张票证（扫码或手动输入验证码）。

**请求头**：`Authorization: Bearer <access_token>`（需要 organizer/admin 角色）

**请求体**：

```json
{
  "code": "TC250513A3F8K2M9X1"
}
```text

**成功响应** (200 OK)：

```json
{
  "id": 42,
  "order_id": 10,
  "event_id": 1,
  "user_id": 5,
  "ticket_no": "TKT20260513a1b-1",
  "price": 680.00,
  "verification_code": "TC250513A3F8K2M9X1",
  "verified_at": "2026-05-13T18:30:00Z",
  "verified_by": 1,
  "created_at": "2026-05-13T10:00:00Z"
}
```

**错误响应**：

| 状态码 | 场景 |
| -------- | ------ |
| 404 | 票证不存在或验证码无效 |
| 409 | 票证已核销 或 订单未支付 |
| 403 | 非组织者/管理员角色 |

---

### POST `/verification/batch-verify`

批量核销票证。

**请求头**：`Authorization: Bearer <access_token>`（需要 organizer/admin 角色）

**请求体**：

```json
{
  "codes": ["TC250513A3F8K2M9X1", "TC250513B7C4D1E2F3"]
}
```text

**成功响应** (200 OK)：

```json
{
  "success": [
    { "id": 42, "ticket_no": "TKT20260513a1b-1", "verification_code": "TC250513A3F8K2M9X1", ... }
  ],
  "failed": [
    { "code": "TC250513B7C4D1E2F3", "reason": "票证已于 2026-05-13T18:30:00Z 核销" }
  ],
  "success_count": 1,
  "failed_count": 1
}
```

---

### GET `/verification/ticket/{code}`

通过验证码查询票证信息。

**请求头**：`Authorization: Bearer <access_token>`（需要 organizer/admin 角色）

**成功响应** (200 OK)：返回 `TicketResponse`，与核销接口响应格式一致。

---

### GET `/events/{event_id}/verification/tickets`

获取活动票证列表（核销管理）。

**请求头**：`Authorization: Bearer <access_token>`（需要 organizer/admin 角色）

**查询参数**：

| 参数 | 类型 | 默认值 | 说明 |
| ------ | ------ | -------- | ------ |
| page | integer | 1 | 页码 |
| size | integer | 20 | 每页数量，1-100 |
| verified | boolean | null | 过滤已核销/未核销 |

---

### GET `/events/{event_id}/verification/stats`

获取活动核销统计。

**请求头**：`Authorization: Bearer <access_token>`（需要 organizer/admin 角色）

**成功响应** (200 OK)：

```json
{
  "event_id": 1,
  "total_tickets": 500,
  "verified_count": 320,
  "unverified_count": 180,
  "verification_rate": 64.0
}
```text

---

## 9. 健康检查

### GET `/health`

服务健康检查（无需认证）。

```json
{"status": "ok"}
```

### GET `/health/ready`

就绪检查（无需认证）。

```json
{"status": "ok"}
```text

这两个端点不经过速率限制中间件。

---

## 10. 错误码汇总

| 状态码 | 含义 | 常见场景 |
| -------- | ------ | --------- |
| 200 | 成功 | 请求正常处理 |
| 201 | 创建成功 | 注册、创建活动/订单 |
| 204 | 无内容 | 删除成功 |
| 400 | 请求错误 | 库存不足、参数无效、核销失败 |
| 401 | 未认证 | 缺少/无效的 JWT 令牌 |
| 403 | 权限不足 | 非管理员访问管理接口、非 organizer 调用核销接口 |
| 404 | 资源不存在 | 活动/订单/票证不存在 |
| 409 | 冲突 | 用户名/邮箱已存在、票证已核销 |
| 413 | 请求体过大 | 超过 1MB |
| 422 | 参数校验失败 | 字段格式错误 |
| 429 | 请求过频 | 速率限制触发 |

---

## 11. 速率限制与响应头

### 11.1 速率限制策略

| 限制级别 | 配额 | 范围 |
| --------- | ------ | ------ |
| 认证端点（login/register） | 20 次/分钟 | 每用户/IP |
| 通用 API | 100 次/分钟 | 每用户/IP |
| 下单频率 | 5 单/10 分钟 | 每用户 |
| 并发未支付订单 | 3 笔 | 每用户 |
| IP 下单频率 | 20 单/小时 | 每 IP |

### 11.2 速率限制响应头

当速率限制触发时，响应头包含：

```

Retry-After: 30
X-RateLimit-Limit: 20
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1680000000

```text

| 响应头 | 说明 |
| -------- | ------ |
| `X-RateLimit-Limit` | 配额上限 |
| `X-RateLimit-Remaining` | 剩余配额 |
| `X-RateLimit-Reset` | 配额重置的 Unix 时间戳 |
| `Retry-After` | 建议等待秒数（仅限流时返回） |
