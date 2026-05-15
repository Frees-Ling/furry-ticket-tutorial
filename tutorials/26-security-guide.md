# 安全指南

> 本文档详细说明票务系统在各个层面实施的安全措施。

---

## 目录

1. [安全架构总览](#1-安全架构总览)
2. [JWT 认证安全](#2-jwt-认证安全)
3. [密码安全](#3-密码安全)
4. [账户锁定与防暴力破解](#4-账户锁定与防暴力破解)
5. [速率限制](#5-速率限制)
6. [CORS 配置](#6-cors-配置)
7. [安全响应头](#7-安全响应头)
8. [请求验证与大小限制](#8-请求验证与大小限制)
9. [SQL 注入防护](#9-sql-注入防护)
10. [XSS 防护](#10-xss-防护)
11. [CSRF 与 CSWSH 防护](#11-csrf-与-cswsh-防护)
12. [支付宝支付安全](#12-支付宝支付安全)
13. [WebSocket 安全](#13-websocket-安全)
14. [Docker 安全配置](#14-docker-安全配置)
15. [Nginx 安全配置](#15-nginx-安全配置)
16. [依赖漏洞扫描](#16-依赖漏洞扫描)
17. [日志安全](#17-日志安全)
18. [生产环境 Checklist](#18-生产环境-checklist)

---

## 1. 安全架构总览

本系统的安全采用**纵深防御**（Defense in Depth）策略，在每一层都设置安全措施：

```text
┌───────────────────────────────────────────────┐
│                 网络层                         │
│  ┌───────────────────────────────────────────┐│
│  │ Docker 内部网络（不对外暴露数据库）          ││
│  │ Nginx 反向代理 + SSL 终止                  ││
│  │ 端口仅监听 127.0.0.1（生产环境）            ││
│  └───────────────────────────────────────────┘│
├───────────────────────────────────────────────┤
│                 应用层                         │
│  ┌───────────────────────────────────────────┐│
│  │ CORS 白名单                                ││
│  │ TrustedHost 防护                          ││
│  │ 请求体大小限制（1MB）                       ││
│  │ 速率限制（滑动窗口）                        ││
│  └───────────────────────────────────────────┘│
├───────────────────────────────────────────────┤
│                 认证层                         │
│  ┌───────────────────────────────────────────┐│
│  │ JWT Bearer 认证                            ││
│  │ RBAC 角色权限控制                          ││
│  │ 刷新令牌轮换                              ││
│  │ JWT 黑名单                                ││
│  └───────────────────────────────────────────┘│
├───────────────────────────────────────────────┤
│                 数据层                         │
│  ┌───────────────────────────────────────────┐│
│  │ bcrypt 密码哈希                            ││
│  │ 参数化查询（防 SQL 注入）                  ││
│  │ Decimal 金额精度保护                       ││
│  │ 乐观锁防超卖                              ││
│  └───────────────────────────────────────────┘│
├───────────────────────────────────────────────┤
│                 日志层                         │
│  ┌───────────────────────────────────────────┐│
│  │ 敏感字段自动脱敏（密码/token/密钥）         ││
│  │ 请求 ID 追踪                              ││
│  │ 结构化日志（JSON 格式）                    ││
│  └───────────────────────────────────────────┘│
└───────────────────────────────────────────────┘
```

---

## 2. JWT 认证安全

### 2.1 令牌结构

```json
{
  "sub": "42",            ← 用户 ID（字符串，非自增整数暴露）
  "exp": 1712345678,       ← 过期时间
  "iat": 1712343878,       ← 签发时间
  "type": "access",       ← 令牌类型（access / refresh）
  "jti": "a1b2c3d4e5",   ← 令牌唯一 ID（用于黑名单）
  "iss": "ticket-booking" ← 签发者
}
```text

### 2.2 安全措施

| 措施 | 说明 |
| ------ | ------ |
| HS256 签名 | 使用 `SECRET_KEY` 签名，防篡改 |
| 短过期时间 | access_token 30 分钟，refresh_token 7 天 |
| 令牌类型隔离 | access_token 不能用于刷新，refresh_token 不能用于 API 访问 |
| JTI 黑名单 | 已吊销的令牌加入 Redis 黑名单 |
| 刷新令牌轮换 | 每次刷新生成新令牌，旧令牌作废 |
| 黑名单 TTL | 与令牌剩余有效期一致（至少 1 小时） |

### 2.3 令牌验证流程

```python
async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    session: AsyncSession = Depends(get_db),
    redis: Redis = Depends(get_redis),
) -> User:
    # 1. 解码 JWT（验证签名 + 过期时间）
    payload = decode_token(credentials.credentials)
    if payload is None or payload.get("type") != "access":
        raise HTTPException(status_code=401, detail="无效或过期的访问令牌")

    # 2. 检查黑名单
    jti = payload.get("jti")
    if jti and await is_token_blacklisted(redis, jti):
        raise HTTPException(status_code=401, detail="令牌已被吊销")

    # 3. 检查用户状态
    sub = payload.get("sub")
    user = await session.get(User, int(sub))
    if user is None or not user.is_active:
        raise HTTPException(status_code=401, detail="用户不存在或已被禁用")

    # 4. 检查封禁状态
    if user.is_banned:
        raise HTTPException(status_code=403, detail="账户已被封禁")

    return user
```

### 2.4 RBAC 角色层级

```text
admin（管理员）→ organizer（组织者）→ user（普通用户）
  可以管理所有        可以管理自己的        仅能操作自己的
  资源和用户          活动                  订单
```

```python
class RoleChecker:
    def __init__(self, allowed_roles: list[str]):
        self.allowed_roles = allowed_roles

    async def __call__(self, current_user: User = Depends(get_current_user)):
        if current_user.role not in self.allowed_roles:
            raise HTTPException(
                status_code=403,
                detail=f"需要 {'/'.join(self.allowed_roles)} 角色",
            )
        return current_user

require_admin = RoleChecker(["admin"])
require_organizer = RoleChecker(["admin", "organizer"])
```text

---

## 3. 密码安全

### 3.1 bcrypt 加密

本项目使用 `bcrypt` 直接进行密码哈希（不通过 passlib，因为 passlib 已不再维护）。

```python
import bcrypt

def hash_password(password: str, rounds: int = 12) -> str:
    """对密码进行 bcrypt 加盐哈希"""
    salt = bcrypt.gensalt(rounds=rounds)
    hashed = bcrypt.hashpw(password.encode("utf-8"), salt)
    return hashed.decode("utf-8")

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """恒定时间比较，防止时序攻击"""
    return bcrypt.checkpw(
        plain_password.encode("utf-8"),
        hashed_password.encode("utf-8"),
    )
```

### 3.2 bcrypt 工作因子

- `rounds=12` — 约 250ms 计算时间（平衡安全性与性能）
- 每增加 1 个 round，计算时间翻倍
- 自动加盐：每次哈希结果不同，即使相同密码

### 3.3 密码复杂度策略

```python
class PasswordPolicy:
    MIN_LENGTH = 8                    # 最小长度
    REQUIRE_UPPER = True              # 需要大写字母
    REQUIRE_LOWER = True              # 需要小写字母
    REQUIRE_DIGIT = True              # 需要数字
    REQUIRE_SPECIAL = True            # 需要特殊字符
```text

---

## 4. 账户锁定与防暴力破解

### 4.1 锁定策略

```

5 次连续失败 → 锁定 15 分钟

```text

### 4.2 Redis 实现

```python
class AccountLockout:
    def __init__(self, redis):
        self.redis = redis
        self.max_attempts = 5           # 最大失败次数
        self.lockout_minutes = 15       # 锁定时间

    async def record_failed_attempt(self, username: str) -> int:
        key = f"lockout:{username}"
        attempts = await self.redis.incr(key)
        if attempts == 1:
            await self.redis.expire(key, self.lockout_minutes * 60)
        return attempts

    async def is_locked(self, username: str) -> bool:
        key = f"lockout:{username}"
        attempts = await self.redis.get(key)
        if attempts and int(attempts) >= self.max_attempts:
            return True
        return False
```

### 4.3 统一错误信息

为了防止**用户名枚举攻击**，登录失败时返回统一错误消息，不区分"用户不存在"和"密码错误"：

```python
if not user or not verify_password(password, user.password_hash):
    # 统一提示，不泄露是用户名还是密码错误
    raise ValueError(f"用户名或密码错误，还有 {remaining} 次尝试机会")
```text

---

## 5. 速率限制

### 5.1 多层限流

| 层级 | 实现位置 | 限制 |
| ------ | --------- | ------ |
| Nginx 限流 | Nginx `limit_req_zone` | 30 req/s（API），5 req/m（登录） |
| 应用级限流 | `RateLimitMiddleware` | 100 req/min（API），20 req/min（认证） |
| 业务级限流 | `BehaviorService` | 5 单/10min（用户），20 单/h（IP） |
| 并发控制 | `BehaviorService` | 最多 3 笔未支付订单 |

### 5.2 限流响应头

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 20
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1680000000
Retry-After: 30
```

### 5.3 优雅降级

当 Redis 不可用时，限流中间件自动放行所有请求，确保服务可用性。

```python
except (ConnectionError, TimeoutError, OSError):
    return True, max_req, 0  # Redis 不可用时放行
```text

---

## 6. CORS 配置

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,  # 白名单来源
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
)
```

默认白名单（开发环境）：

```python
CORS_ORIGINS = ["http://localhost:5173", "http://localhost:8000"]
```text

生产环境应设置为具体的域名，不允许使用 `["*"]`。

---

## 7. 安全响应头

所有响应自动添加以下安全头：

| 响应头 | 值 | 作用 |
| -------- | ------ | ------ |
| `X-Content-Type-Options` | `nosniff` | 防止 MIME 类型嗅探 |
| `X-Frame-Options` | `DENY` | 防止点击劫持（无法被 iframe 嵌入） |
| `X-XSS-Protection` | `1; mode=block` | 启用浏览器 XSS 过滤 |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | HSTS，强制 HTTPS（仅 HTTPS 时添加） |
| `Content-Security-Policy` | `default-src 'self'; ...` | 内容安全策略，限制资源加载来源 |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | 控制 Referer 头信息量 |
| `Permissions-Policy` | `camera=(), microphone=(), ...` | 禁用不必要的浏览器权限 |

响应头中同时移除 `Server` 和 `X-Powered-By` 头，防止信息泄露。

### CSP 详解

```python
csp = (
    "default-src 'self'; "          # 默认只允许同源
    "script-src 'self'; "           # 脚本只允许同源
    "style-src 'self' 'unsafe-inline'; "  # 样式允许内联
    "img-src 'self' data:; "        # 图片允许同源和 data: URI
    "font-src 'self'; "             # 字体只允许同源
    "frame-src 'none'; "            # 禁止 iframe
    "object-src 'none'; "           # 禁止插件
    "base-uri 'self'; "             # 限制 base URL
    "form-action 'self'"            # 表单提交只允许同源
)
```

---

## 8. 请求验证与大小限制

### 8.1 请求体大小限制

```python
MAX_REQUEST_BODY_SIZE = 1_048_576  # 1MB

class RequestValidationMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        if request.method in ("POST", "PUT", "PATCH"):
            content_length = request.headers.get("content-length")
            if content_length and int(content_length) > MAX_REQUEST_BODY_SIZE:
                return JSONResponse(
                    status_code=413,
                    content={"detail": "请求体过大，最大允许 1MB"},
                )
```text

### 8.2 参数校验

Pydantic schema 自动校验所有请求参数：

```python
class UserCreate(BaseModel):
    username: str = Field(..., min_length=3, max_length=50, pattern=r"^[a-zA-Z0-9_]+$")
    password: str = Field(..., min_length=8, max_length=128)
    email: EmailStr
    phone: str | None = Field(None, pattern=r"^1\d{10}$")
```

### 8.3 Host 头攻击防护

```python
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["localhost", "127.0.0.1", "*.ngrok.io", "*.frp.com", "*.app"],
)
```text

---

## 9. SQL 注入防护

### 9.1 ORM 查询（自动参数化）

SQLAlchemy 的 ORM 查询默认使用参数化查询，自动防止 SQL 注入：

```python
# 安全：ORM 自动参数化
result = await session.execute(
    select(User).where(User.username == username)
)

# 安全：参数化 text() SQL
stmt = text(
    "UPDATE events SET sold_count = sold_count + :qty WHERE id = :eid"
)
result = await session.execute(stmt, {"qty": quantity, "eid": event_id})
```

### 9.2 LIKE 查询的手动转义

对于 `LIKE` 查询，`%` 和 `_` 是通配符，需要手动转义：

```python
# 转义 LIKE 通配符
safe_keyword = keyword.replace("%", "\\%").replace("_", "\\_")
conditions.append(Event.title.like(f"%{safe_keyword}%"))
```text

### 9.3 绝对不使用字符串拼接

```python
# 危险！绝不要这样做
result = await session.execute(f"SELECT * FROM users WHERE id = {user_id}")

# 安全：使用参数化
result = await session.execute(
    select(User).where(User.id == user_id)
)
```

---

## 10. XSS 防护

### 10.1 后端措施

- **Content-Security-Policy** 响应头限制脚本来源
- **Pydantic 校验**限制输入格式（正则表达式）
- **响应输出**由 FastAPI JSON 序列化自动转义

### 10.2 前端措施

- **Vue 3 自动转义**：所有模板插值 `{{ }}` 自动转义 HTML
- **避免 `v-html`**：除非必要，不使用 `v-html` 渲染用户输入
- **输入校验**：前端和后端双重验证

### 10.3 测试验证

```python
# XSS 输入测试
xss_payloads = [
    '<script>alert("xss")</script>',
    '"><script>alert(1)</script>',
    "'; DROP TABLE users; --",
]

for payload in xss_payloads:
    resp = await client.post("/api/v1/auth/register", json={
        "username": payload[:50],
        "password": "TestPass123!",
        "email": "test@test.com",
    })
    assert resp.status_code != 500  # 不应导致服务端异常
```text

---

## 11. CSRF 与 CSWSH 防护

### 11.1 CSRF 防护

本项目作为 REST API，使用 Token-Based 认证（JWT Bearer Token），天然抵御 CSRF 攻击：

- 浏览器不会自动在跨域请求中附带 `Authorization` 头
- CORS 白名单限制允许的来源
- `SameSite` Cookie 策略（未使用 Cookie，所以不适用）

### 11.2 CSWSH 防护（WebSocket）

WebSocket 端点的 Origin 校验：

```python
async def verify_ws_origin(websocket: WebSocket) -> bool:
    origin = websocket.headers.get("origin", "")
    if not origin:
        return True  # 非浏览器客户端放行
    allowed = {"http://localhost:5173", "http://localhost:8000"}
    return any(origin.startswith(a.rstrip("/")) for a in allowed)
```

如果 Origin 不在白名单中，WebSocket 连接被拒绝（close code 4001）。

---

## 12. 支付宝支付安全

详见 [ALGORITHMS.md](./ALGORITHMS.md#5-支付宝防重放四层防护) 的详细说明，核心安全措施包括：

| 层级 | 措施 | 作用 |
| ------ | ------ | ------ |
| 1 | RSA2 签名验证 | 确保通知来自支付宝 |
| 2 | 交易状态检查 | 仅处理 TRADE_SUCCESS |
| 3 | Redis SETNX 幂等键 | 防止重复通知处理 |
| 4 | 金额校验 + 状态机 | 防止金额篡改，确保订单状态正确 |

---

## 13. WebSocket 安全

| 措施 | 说明 |
| ------ | ------ |
| Origin 校验 | 防止 CSWSH 攻击 |
| JWT 认证 | 通过查询参数传递 token |
| 消息大小限制 | 最大 64KB，防 DoS |
| 消息类型白名单 | 仅处理 select/release/ping |
| 用户身份注入 | 从 JWT 提取，拒绝客户端伪造 |
| 座位 ID 格式校验 | 防止注入攻击 |

座位 ID 格式校验：

```python
def _is_valid_seat_id(seat: str) -> bool:
    """只允许大写字母 + 数字的组合"""
    if not seat or len(seat) > 5:
        return False
    if not seat.isalnum():
        return False  # 只允许字母和数字
    has_alpha = any(c.isalpha() for c in seat)
    has_digit = any(c.isdigit() for c in seat)
    alpha_part = "".join(c for c in seat if c.isalpha())
    if alpha_part != alpha_part.upper():
        return False  # 字母必须大写
    return has_alpha and has_digit
```text

---

## 14. Docker 安全配置

### 14.1 内部网络隔离

```yaml
networks:
  backend:
    driver: bridge
    internal: true          # 外部无法访问后端网络
```

### 14.2 端口锁定

```yaml
ports:
  - "127.0.0.1:3306:3306"   # 仅监听本地回环
```text

### 14.3 资源限制

```yaml
deploy:
  resources:
    limits:
      memory: 512M          # 内存上限
      cpus: "1.0"           # CPU 上限
```

### 14.4 健康检查

所有服务都配置了健康检查，确保只有就绪的服务才会接收流量。

### 14.5 非 root 用户策略

MySQL 容器创建独立的 `ticket_app` 用户，而非直接使用 root。

---

## 15. Nginx 安全配置

### 15.1 漏桶限流

```nginx
# API 限流：每秒 30 个请求
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=30r/s;

# 登录限流：每分钟 5 次
limit_req_zone $binary_remote_addr zone=login_limit:10m rate=5r/m;
```text

### 15.2 安全头

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

### 15.3 客户端限制

```nginx
client_max_body_size 10m;        # 请求体上限
client_body_buffer_size 128k;    # 请求体缓冲区
keepalive_timeout 65;            # 连接超时
keepalive_requests 100;          # 单连接最大请求数
```text

---

## 16. 依赖漏洞扫描

### 16.1 pip-audit

```bash
pip install pip-audit
pip-audit -r requirements.txt
```

### 16.2 Safety

```bash
pip install safety
safety check
```text

### 16.3 Bandit（代码安全扫描）

```bash
pip install bandit
bandit -r app/ -f json
```

### 16.4 npm audit（前端）

```bash
cd frontend
npm audit
```text

---

## 17. 日志安全

### 17.1 敏感信息脱敏

日志中自动过滤密码、令牌、密钥等敏感字段：

```python
SENSITIVE_FIELDS = re.compile(
    r"password|secret|token|key|authorization|credential|credit_card|cvv",
    re.IGNORECASE,
)

def _mask_sensitive_events(logger, log_method, event_dict):
    for key in list(event_dict.keys()):
        if SENSITIVE_FIELDS.search(key):
            event_dict[key] = "[REDACTED]"
    return event_dict
```

### 17.2 生产环境 JSON 格式

生产环境使用 JSON 格式日志，便于日志聚合工具（如 ELK、Splunk）处理：

```python
if settings.IS_PRODUCTION:
    processors.append(structlog.processors.JSONRenderer())
```text

### 17.3 请求 ID 追踪

每个请求分配唯一 ID，贯穿整个请求生命周期，方便问题追踪：

```python
request_id = uuid4().hex[:8]  # 8 位十六进制 ID
request.state.request_id = request_id
response.headers["X-Request-ID"] = request_id
```

---

## 18. 生产环境 Checklist

### 18.1 必须配置

- [ ] `SECRET_KEY` — 设置为强随机字符串（至少 32 位）
- [ ] `DB_PASSWORD` — 强密码
- [ ] `REDIS_PASSWORD` — 强密码
- [ ] `CORS_ORIGINS` — 设置为具体的前端域名
- [ ] `ENV=production` — 关闭 `/docs` 和 `/redoc`
- [ ] HTTPS 证书 — SSL 终止（Nginx）

### 18.2 推荐配置

- [ ] `ALIPAY_APP_ID` / `ALIPAY_APP_PRIVATE_KEY_PATH` / `ALIPAY_PUBLIC_KEY_PATH` — 支付宝生产环境密钥
- [ ] `ALIPAY_NOTIFY_URL` — 可公网访问的异步通知地址
- [ ] `JWT_ISSUER` — 设置为实际域名
- [ ] `BCRYPT_ROUNDS=12` — 密码哈希工作因子
- [ ] `SHOW_DOCS=False` — 关闭 API 文档暴露

### 18.3 安全检查清单

- [ ] 运行 `pip-audit` 扫描依赖漏洞
- [ ] 运行 `bandit -r app/` 扫描代码安全
- [ ] 运行 `ruff check --select S .` 检查安全相关规则
- [ ] 确认所有数据库连接使用参数化查询
- [ ] 确认 CORS 白名单正确
- [ ] 确认安全响应头已添加
- [ ] 确认 Docker 端口不对外暴露
- [ ] 确认 `.env` 文件不在版本控制中
- [ ] 确认生产环境日志级别为 `INFO`（非 `DEBUG`）
- [ ] 确认 WebSocket Origin 校验正确
