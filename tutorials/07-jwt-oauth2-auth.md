# 第7章：JWT/OAuth2/bcrypt — 认证授权系统

## 本章学习目标

本章将深入学习密码安全和 JWT 认证，实现完整的用户注册/登录/令牌刷新和 RBAC 权限控制系统。

**学习路径：**

1. ✅ 密码安全：哈希 vs 加密、bcrypt 算法、Argon2、时序攻击
2. ✅ JWT (RFC 7519)：结构/签名算法/完整验证/安全证明
3. ✅ OAuth2 (RFC 6749)：四种授权码流程/PKCE
4. ✅ RBAC 权限控制：角色层级 + 资源级授权
5. ✅ 项目：完整认证系统实现

---

## 1. 密码安全

### 1.1 哈希 vs 加密

```text
哈希（Hash）：
           单向函数
密码 ──────────────→ 哈希值（固定长度）
                      ↓
            无法从哈希值还原密码（单向陷门函数）
            相同的输入 → 相同的输出（确定性）

加密（Encryption）：
密码 ───────→ 密文 ───────→ 密码（可还原！）
      加密密钥        解密密钥

结论：密码必须用哈希，不能用加密！
```

**为什么不能用加密存储密码？**

```text
加密 → 持有密钥就能解密出原文
如果服务器被入侵，密钥和密文同时泄露 → 所有密码明文暴露

哈希 → 不可逆
即使数据库泄露，攻击者也无法还原原始密码
（但需要用加盐哈希防止彩虹表）
```

### 1.2 常用哈希算法对比

| 算法 | 输出长度 | 特性 | 推荐用于密码？ |
| ------ | --------- | ------ | -------------- |
| MD5 | 128 bit | ❌ 已破解，可碰撞 | ❌ |
| SHA-1 | 160 bit | ❌ 已破解（SHAttered） | ❌ |
| SHA-256 | 256 bit | 快速，适合完整性校验 | ❌（太快，容易暴力破解） |
| **bcrypt** | 可变 | 慢速、加盐、可调成本 | ✅ **推荐** |
| **Argon2** | 可变 | 内存硬、抗 GPU/ASIC | ✅ **最佳** |
| **scrypt** | 可变 | 内存硬 | ✅ 好 |

### 1.3 bcrypt 算法详解

bcrypt 的设计目标是**慢**——使暴力破解的代价高到不现实。

**bcrypt 哈希结构：**

```text
$2b$10$rXUQ4YEGeBiN.EUVRjIKOeF6sX5HVxDqzgFJ9Yx9nFCTNiHd5rmKm
││  │  │
││  │  └──────────────────────────── 哈希值（31字节 Base64）
││  └──────────────────────────────── 盐（22字符 Base64）
│└─────────────────────────────────── 成本因子（cost=10 → 2^10轮）
└──────────────────────────────────── bcrypt 版本（$2b$，旧版 $2a$）
```

**成本因子对速度的影响：**

```text
cost=10:  ~100ms（推荐，2024年标准）
cost=8:   ~25ms（较弱，仍在合理范围）
cost=12:  ~400ms（高强度，敏感系统）
cost=14:  ~1.6s（极度安全但用户体验差）

每增加 1 个成本值，计算时间翻倍。
```

**为什么 bcrypt 能防 GPU？**

```text
bcrypt 算法基于 Blowfish 密码的密钥扩展阶段，需要 4KB 内存。
虽然 4KB 很小，但算法中包含了大量相互依赖的查表操作，
GPU 的并行优势无法充分发挥。

相比之下：
MD5:    2^80 MD5/sec (ASIC)  →  秒破8字符密码
SHA256: 2^68 SHA/sec (ASIC)  →  毫秒破8字符密码
bcrypt: 2^6 cost10/sec (FPGA) →  数天破8字符密码
```

**Python 实现：**

```python
from passlib.context import CryptContext

# 创建密码上下文（推荐 bcrypt）
pwd_context = CryptContext(
    schemes=["bcrypt"],
    deprecated="auto",
)

# 哈希密码
def hash_password(password: str) -> str:
    return pwd_context.hash(password)

# 验证密码
def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

# 使用
hashed = hash_password("MySecurePwd123!")
print(hashed)   # $2b$12$rXUQ4YEGeBiN...
print(verify_password("MySecurePwd123!", hashed))  # True
print(verify_password("wrong", hashed))            # False
```text

### 1.4 时序攻击证明

**问题：** 字符串比较在发现第一个不同字符时就立即返回。

```python
# 不安全的比较（时序泄露）
def insecure_compare(a: str, b: str) -> bool:
    if len(a) != len(b):
        return False
    for i in range(len(a)):
        if a[i] != b[i]:        # ← 这里！不同的字符比较时间不同！
            return False        # ← 找到不同就立即返回
    return True

# 攻击者可以测量响应时间，逐字符推断密码：
# 如果第一个字符正确，响应时间多 1 次循环
# 如果前两个字符正确，响应时间多 2 次循环
# → 通过时间差异即可暴力破解！
```

**证明：**

```text
假设密码为 "abcdef"，攻击者尝试 "a____" 和 "b____"：

尝试 "a____":
  compare: 'a' == 'a' → 继续
  compare: '_' == 'b' → 不匹配，返回 False
  耗时: 2 次比较

尝试 "b____":
  compare: 'b' == 'a' → 不匹配，返回 False
  耗时: 1 次比较

→ "a____" 耗时更长 → 第一个字符是 'a'！
接着尝试 "ab___" vs "ac___":
  "ab___" 耗时 3 次比较 → 第二个字符是 'b'！
...
逐字符破解！只需要 26 × 8 ≈ 208 次请求（而非 26^8 ≈ 2000亿次）
```

**解决方案：常数时间比较**

```python
import hmac

# 安全的比较（常数时间）
def constant_time_compare(a: str, b: str) -> bool:
    """hmac.compare_digest 保证比较时间不因内容而变化"""
    return hmac.compare_digest(a, b)

# 原理：不管在哪里发现不同，都继续比较完所有字符
# 确保相同长度的字符串比较时间恒定
```text

### 1.5 Argon2（PHC 冠军）

Argon2 是 2015 年密码哈希竞赛（PHC）的冠军，设计目标是抵御 GPU 和 ASIC 攻击。

```python
from passlib.hash import argon2

# 使用 Argon2
hasher = argon2.using(
    time_cost=3,        # 迭代次数
    memory_cost=65536,  # 内存使用（KB）= 64MB
    parallelism=4,      # 并行度
)

hashed = hasher.hash("MySecurePwd!")
print(hashed)  # $argon2id$v=19$m=65536,t=3,p=4$...

# 验证
hasher.verify("MySecurePwd!", hashed)  # True
```

---

## 2. JWT (RFC 7519)

### 2.1 JWT 结构

JWT（JSON Web Token）是一个自包含的令牌，包含三部分：

```text
       Header          Payload          Signature
      ┌────────┐    ┌────────────┐    ┌──────────┐
      │ {      │    │ {          │    │ 签名     │
      │  "alg":│    │  "sub": 1, │    │ 二进制   │
      │  "HS256"│    │  "exp":..,│    │ Base64URL│
      │ }      │    │  "iat":.. │    │ 编码     │
      └────────┘    └────────────┘    └──────────┘
           ↓               ↓               ↓
    Base64URL编码    Base64URL编码     HMAC-SHA256(
                                      base64(header)
                                    + . + base64(payload)
                                    , secret_key)
           ↓               ↓               ↓
    eyJhbGciOiJI...  eyJzdWIiOjEs...   SflKxwRJSMeK...

组合：eyJhbGci...eyJzdWIiOjEs...SflKxwRJSMeK...

三部分用 . 连接：
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### 2.2 Header

```json
{
    "alg": "HS256",       // 签名算法（必须）
    "typ": "JWT",         // 令牌类型（可选，建议保留）
    "kid": "key-01"       // 密钥 ID（多密钥场景）
}
```text

**支持的签名算法：**

| alg 参数 | 算法 | 密钥类型 | 对称？ |
| --------- | ------ | --------- | ------- |
| HS256 | HMAC-SHA256 | 对称密钥 | ✅ 对称 |
| HS384 | HMAC-SHA384 | 对称密钥 | ✅ 对称 |
| HS512 | HMAC-SHA512 | 对称密钥 | ✅ 对称 |
| RS256 | RSA-SHA256 | RSA 公私钥 | ❌ 非对称 |
| RS384 | RSA-SHA384 | RSA 公私钥 | ❌ 非对称 |
| RS512 | RSA-SHA512 | RSA 公私钥 | ❌ 非对称 |
| ES256 | ECDSA-P256 | ECC 公私钥 | ❌ 非对称 |
| ES384 | ECDSA-P384 | ECC 公私钥 | ❌ 非对称 |
| EdDSA | Ed25519 | EdDSA 公私钥 | ❌ 非对称 |
| **none** | 无签名 | 无 | ⚠️ 不安全 |

### 2.3 Payload

```json
{
    // === 注册声明（RFC 7519 标准字段）===
    "sub": "1",              // Subject：用户ID（必填，唯一标识）
    "iss": "ticket-system",  // Issuer：签发人
    "aud": "ticket-app",     // Audience：接收方
    "exp": 1710500000,       // Expiration：过期时间（Unix 时间戳）
    "nbf": 1710400000,       // Not Before：生效时间
    "iat": 1710400000,       // Issued At：签发时间
    "jti": "550e8400-e29b-41d4-a716-446655440000",  // JWT ID（用于重放检测）

    // === 公共声明（自定义字段）===
    "name": "张三",
    "role": "admin",
    "permissions": ["create:event", "delete:order"]
}
```

| 声明 | 全称 | 说明 |
| ------ | ------ | ------ |
| `sub` | Subject | 令牌主体（通常是用户 ID） |
| `iss` | Issuer | 签发者 |
| `aud` | Audience | 接收方（可验证此字段仅自己可用） |
| `exp` | Expiration | 过期时间——令牌在此时间后无效 |
| `nbf` | Not Before | 在此时间前令牌无效 |
| `iat` | Issued At | 签发时间 |
| `jti` | JWT ID | 令牌唯一 ID（重放攻击检测） |

### 2.4 签名算法详解

#### 2.4.1 HMAC-SHA256

```python
import hmac
import hashlib
import base64

def create_jwt_signature(header: str, payload: str, secret: str) -> str:
    """HMAC-SHA256 签名"""
    message = f"{header}.{payload}"
    signature = hmac.new(
        secret.encode(),
        message.encode(),
        hashlib.sha256,
    ).digest()
    return base64.urlsafe_b64encode(signature).rstrip("=").decode()
```text

**安全证明（HMAC 作为 PRF）：**

```

定理：如果 HMAC-SHA256 是安全的伪随机函数（PRF），
那么没有密钥的攻击者无法伪造有效的 JWT 签名。

证明（反证法）：
假设攻击者 A 可以以不可忽略的概率伪造 JWT 签名。
那么 A 可以在不知道密钥 k 的情况下，计算出 HMAC-SHA256(k, m)。

这与 HMAC-SHA256 作为 PRF 的安全性矛盾。
（如果攻击者能预测 PRF 输出而不掌握密钥，则该 PRF 不安全）

因此，在假设 HMAC-SHA256 是安全 PRF 的前提下，
JWT 的签名是不可伪造的。

```text

#### 2.4.2 RSA-SHA256

```python
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import rsa, padding

# 生成 RSA 密钥对
private_key = rsa.generate_private_key(
    public_exponent=65537,
    key_size=2048,
)
public_key = private_key.public_key()

# 使用私钥签名
signature = private_key.sign(
    message.encode(),
    padding.PKCS1v15(),
    hashes.SHA256(),
)

# 使用公钥验证
try:
    public_key.verify(
        signature,
        message.encode(),
        padding.PKCS1v15(),
        hashes.SHA256(),
    )
    print("验证成功")
except Exception:
    print("验证失败")
```

### 2.5 算法混淆攻击（最著名的 JWT 漏洞）

**问题：** 服务器使用 RS256（非对称），攻击者获取了公钥。一些 JWT 库允许攻击者将 `alg` 改为 `HS256`，**用公钥作为 HMAC 密钥**来伪造令牌。

```python
# 攻击步骤：
# 1. 获取服务器公钥（通常在 /.well-known/jwks.json）
public_key = """
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...
-----END PUBLIC KEY-----
"""

# 2. 修改 header：alg: "RS256" → alg: "HS256"
# 3. 用公钥作为 HMAC 密钥重新签名
import hmac, hashlib, base64, json

header = base64url_encode(json.dumps({"alg": "HS256", "typ": "JWT"}))
payload = base64url_encode(json.dumps({"sub": "1", "role": "admin"}))

# 关键：用公钥（字符串）作为 HMAC 密钥！
fake_sig = hmac.new(
    public_key.encode(),
    f"{header}.{payload}".encode(),
    hashlib.sha256,
).digest()

# 4. 服务器使用公钥 + HS256 验证 → 验证通过！
```text

**防御：**

```python
# 1. 永远指定预期的算法
payload = jwt.decode(
    token,
    public_key,
    algorithms=["RS256"],  # 只允许 RS256，禁止 HS256
    options={"verify_aud": False},
)

# 2. 验证 alg 不能为 "none"
if payload.get("alg") == "none":
    raise Exception("算法混淆攻击检测！")
```

### 2.6 access_token vs refresh_token

```text
access_token（访问令牌）：
    - 短有效期（15-30分钟）
    - 携带身份信息
    - 每次请求都需要验证
    
refresh_token（刷新令牌）：
    - 长有效期（7-30天）
    - 仅用于获取新的 access_token
    - 存储在安全位置（HTTP-Only Cookie）
```

```python
import jwt
from datetime import datetime, timedelta
from app.config import settings

def create_access_token(user_id: int, role: str) -> str:
    """创建访问令牌（30分钟有效）"""
    expire = datetime.utcnow() + timedelta(
        minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES
    )
    payload = {
        "sub": str(user_id),
        "role": role,
        "exp": expire,
        "iat": datetime.utcnow(),
        "type": "access",
    }
    return jwt.encode(payload, settings.SECRET_KEY, algorithm="HS256")


def create_refresh_token(user_id: int) -> str:
    """创建刷新令牌（7天有效）"""
    expire = datetime.utcnow() + timedelta(
        days=settings.REFRESH_TOKEN_EXPIRE_DAYS
    )
    payload = {
        "sub": str(user_id),
        "exp": expire,
        "iat": datetime.utcnow(),
        "type": "refresh",
        "jti": str(uuid4()),  # 唯一 ID，用于吊销
    }
    return jwt.encode(payload, settings.SECRET_KEY, algorithm="HS256")


def decode_token(token: str) -> dict | None:
    """验证并解码令牌"""
    try:
        payload = jwt.decode(
            token,
            settings.SECRET_KEY,
            algorithms=["HS256"],
            options={"require": ["exp", "sub"]},  # 必须包含 exp 和 sub
        )
        return payload
    except jwt.ExpiredSignatureError:
        return None  # 令牌过期
    except jwt.InvalidTokenError:
        return None  # 无效令牌
```text

### 2.7 令牌黑名单

```python
async def revoke_token(jti: str, redis: AsyncRedis, ttl: int = 86400):
    """将令牌加入黑名单"""
    await redis.setex(f"revoked:{jti}", ttl, "1")

async def is_token_revoked(jti: str, redis: AsyncRedis) -> bool:
    """检查令牌是否被吊销"""
    return await redis.exists(f"revoked:{jti}") == 1
```

---

## 3. OAuth2 (RFC 6749)

### 3.1 四种授权码流程

```text
OAuth2 定义了四种授权方式：

1. Authorization Code（授权码）—— ✅ 推荐
   最安全的方式，需要后端参与。

2. Implicit（隐式）—— ❌ 已弃用
   安全性不足，2019年起不再推荐。

3. Password（密码）—— ⚠️ 仅限第一方应用
   用户直接提供密码给第三方。

4. Client Credentials（客户端凭证）—— 服务间认证
   没有用户参与。

5. Refresh Token（刷新令牌）—— 续期 access_token
   配合其他方式使用。
```

### 3.2 Authorization Code + PKCE（最安全）

```text
┌─────────┐              ┌─────────────┐              ┌────────┐
│  前端    │              │   后端       │              │ 授权服  │
│ (SPA/APP)│              │  (API服务)  │              │ 务器    │
└────┬────┘              └──────┬──────┘              └───┬────┘
     │                          │                         │
     │ 1. 生成 code_verifier    │                         │
     │    和 code_challenge     │                         │
     │                          │                         │
     │ 2. 请求授权：            │                         │
     │    /authorize?           │                         │
     │    response_type=code    │                         │
     │    + code_challenge      │                         │
     │───────────────────────────────────────────────────→│
     │                          │                         │
     │ 3. 返回授权码 (code)     │                         │
     │←────────────────────────────────────────────────────│
     │                          │                         │
     │ 4. POST 授权码到后端      │                         │
     │─────────────────────────→│                         │
     │                          │                         │
     │                          │ 5. /token?              │
     │                          │    code +                │
     │                          │    code_verifier        │
     │                          │────────────────────────→│
     │                          │                         │
     │                          │ 6. 返回 access_token    │
     │                          │←────────────────────────│
     │                          │                         │
     │ 7. 返回令牌给前端        │                         │
     │←─────────────────────────│                         │
     │                          │                         │
```

**PKCE (RFC 7636) 证明：**

```text
PKCE 防止授权码拦截攻击：

1. 前端生成 code_verifier（随机字符串，43-128字符）
2. 计算 code_challenge = SHA256(code_verifier) 的 Base64URL
3. 请求授权码时附带 code_challenge
4. 换取令牌时提供 code_verifier
5. 服务器验证 SHA256(verifier) == challenge

安全性证明：
即使攻击者拦截了授权码（code），
但没有 code_verifier（从未传输！），
code_challenge 是哈希值不可逆，
因此攻击者无法换取令牌。
```

### 3.3 Refresh Token 旋转

```python
async def refresh_access_token(
    refresh_token: str,
    redis: AsyncRedis,
    session: AsyncSession,
):
    """刷新令牌（带旋转机制）"""
    payload = decode_token(refresh_token)
    if not payload:
        raise HTTPException(status_code=401, detail="刷新令牌无效")

    if payload.get("type") != "refresh":
        raise HTTPException(status_code=401, detail="令牌类型错误")

    # 检测重放攻击：检查 jti 是否已使用
    if await is_token_revoked(payload["jti"], redis):
        # 可能的令牌泄露！吊销该用户的所有令牌
        user_id = int(payload["sub"])
        await revoke_all_user_tokens(user_id, redis)
        raise HTTPException(status_code=401, detail="检测到令牌重放攻击")

    # 吊销旧刷新令牌（旋转）
    await revoke_token(payload["jti"], redis)

    # 颁发新令牌
    user_id = int(payload["sub"])
    new_access = create_access_token(user_id)
    new_refresh = create_refresh_token(user_id)

    return {
        "access_token": new_access,
        "refresh_token": new_refresh,
        "token_type": "bearer",
    }

async def revoke_all_user_tokens(user_id: int, redis: AsyncRedis):
    """吊销用户所有令牌"""
    # 记录用户版本号，所有该用户旧版本的令牌失效
    await redis.incr(f"user_token_version:{user_id}")
```text

---

## 4. RBAC 权限控制

### 4.1 角色层级

```python
# 角色定义
ROLE_HIERARCHY = {
    "admin": ["admin", "manager", "user"],       # admin 包含所有权限
    "manager": ["manager", "user"],              # manager 包含 user 权限
    "user": ["user"],                             # user 只有基本权限
}

# 权限定义
PERMISSIONS = {
    # 活动管理
    "event:create": ["admin", "manager"],
    "event:read": ["admin", "manager", "user"],
    "event:update": ["admin", "manager"],
    "event:delete": ["admin"],

    # 订单管理
    "order:create": ["admin", "user"],
    "order:read": ["admin", "user"],
    "order:cancel": ["admin", "user"],

    # 系统管理
    "user:manage": ["admin"],
    "payment:refund": ["admin"],
    "report:view": ["admin", "manager"],
}
```

### 4.2 权限验证装饰器

```python
from functools import wraps
from fastapi import HTTPException, Depends

def require_permission(permission: str):
    """权限验证装饰器"""
    async def permission_checker(current_user: User = Depends(get_current_user)):
        # 获取用户角色
        user_role = current_user.role

        # 检查角色是否拥有该权限
        allowed_roles = PERMISSIONS.get(permission, [])
        if user_role not in allowed_roles:
            raise HTTPException(
                status_code=403,
                detail=f"权限不足：需要 {permission}",
            )
        return current_user

    return permission_checker

# 使用
from typing import Annotated

AdminDep = Annotated[User, Depends(require_permission("event:create"))]
ReaderDep = Annotated[User, Depends(get_current_user)]

@app.get("/events")
async def list_events(current_user: ReaderDep):
    ...

@app.post("/events")
async def create_event(current_user: AdminDep):
    ...
```text

### 4.3 资源级授权

```python
async def check_event_ownership(
    event_id: int,
    current_user: User,
    session: AsyncSession,
):
    """检查用户是否为活动创建者"""
    event = await session.get(Event, event_id)
    if not event:
        raise HTTPException(status_code=404)

    # admin 可以管理所有活动
    if current_user.role == "admin":
        return event

    # 普通用户只能修改自己的活动
    if event.created_by != current_user.id:
        raise HTTPException(status_code=403, detail="无权操作此活动")

    return event
```

---

## 5. 项目：完整认证系统实现

### 5.1 安全工具

**app/utils/security.py：**

```python
from datetime import datetime, timedelta
from uuid import uuid4
from typing import Optional

import jwt
from passlib.context import CryptContext

from app.config import settings

# 密码上下文
pwd_context = CryptContext(
    schemes=["bcrypt"],
    deprecated="auto",
)


def hash_password(password: str) -> str:
    """bcrypt 哈希密码"""
    return pwd_context.hash(password)


def verify_password(plain_password: str, hashed_password: str) -> bool:
    """验证密码"""
    return pwd_context.verify(plain_password, hashed_password)


def create_access_token(user_id: int, role: str) -> str:
    """创建访问令牌（30分钟有效）"""
    expire = datetime.utcnow() + timedelta(
        minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES
    )
    payload = {
        "sub": str(user_id),
        "role": role,
        "exp": expire,
        "iat": datetime.utcnow(),
        "type": "access",
    }
    return jwt.encode(payload, settings.SECRET_KEY, algorithm="HS256")


def create_refresh_token(user_id: int) -> str:
    """创建刷新令牌（7天有效）"""
    expire = datetime.utcnow() + timedelta(
        days=settings.REFRESH_TOKEN_EXPIRE_DAYS
    )
    payload = {
        "sub": str(user_id),
        "exp": expire,
        "iat": datetime.utcnow(),
        "type": "refresh",
        "jti": str(uuid4()),
    }
    return jwt.encode(payload, settings.SECRET_KEY, algorithm="HS256")


def decode_token(token: str) -> Optional[dict]:
    """解码并验证令牌"""
    try:
        payload = jwt.decode(
            token,
            settings.SECRET_KEY,
            algorithms=["HS256"],
            options={"require": ["exp", "sub"]},
        )
        return payload
    except (jwt.ExpiredSignatureError, jwt.InvalidTokenError):
        return None
```text

### 5.2 Pydantic Schema

**app/schemas/auth.py：**

```python
from pydantic import BaseModel, Field, field_validator
import re


class UserCreate(BaseModel):
    """注册请求"""
    username: str = Field(..., min_length=2, max_length=50, description="用户名")
    email: str = Field(..., description="邮箱")
    password: str = Field(..., min_length=6, max_length=100, description="密码")

    @field_validator("username")
    @classmethod
    def validate_username(cls, v):
        if not re.match(r"^[a-zA-Z0-9_一-鿿]+$", v):
            raise ValueError("用户名只能包含字母、数字、下划线和中文")
        return v

    @field_validator("password")
    @classmethod
    def validate_password(cls, v):
        if not any(c.isdigit() for c in v):
            raise ValueError("密码必须包含数字")
        if not any(c.isupper() for c in v):
            raise ValueError("密码必须包含大写字母")
        return v


class LoginRequest(BaseModel):
    """登录请求"""
    username: str = Field(..., description="用户名")
    password: str = Field(..., description="密码")


class TokenResponse(BaseModel):
    """令牌响应"""
    access_token: str
    refresh_token: str
    token_type: str = "bearer"


class TokenRefresh(BaseModel):
    """刷新令牌请求"""
    refresh_token: str


class UserResponse(BaseModel):
    """用户信息响应"""
    model_config = {"from_attributes": True}

    id: int
    username: str
    email: str
    role: str
    is_active: bool
    created_at: datetime


class UserUpdate(BaseModel):
    """更新用户信息"""
    email: Optional[str] = None
    phone: Optional[str] = Field(None, pattern=r"^1[3-9]\d{9}$")
```

### 5.3 Auth 服务

**app/services/auth_service.py：**

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from fastapi import HTTPException, status

from app.models import User
from app.utils.security import (
    hash_password,
    verify_password,
    create_access_token,
    create_refresh_token,
    decode_token,
)
from app.schemas.auth import UserCreate


class AuthService:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def register(self, data: UserCreate) -> User:
        """用户注册"""
        # 检查用户名是否已存在
        result = await self.session.execute(
            select(User).where(User.username == data.username)
        )
        if result.scalar_one_or_none():
            raise HTTPException(status_code=409, detail="用户名已存在")

        # 检查邮箱是否已存在
        result = await self.session.execute(
            select(User).where(User.email == data.email)
        )
        if result.scalar_one_or_none():
            raise HTTPException(status_code=409, detail="邮箱已被注册")

        # 创建用户
        user = User(
            username=data.username,
            email=data.email,
            password_hash=hash_password(data.password),
            role="user",
        )
        self.session.add(user)
        await self.session.flush()
        return user

    async def login(self, username: str, password: str) -> dict:
        """用户登录"""
        result = await self.session.execute(
            select(User).where(User.username == username)
        )
        user = result.scalar_one_or_none()

        if not user or not verify_password(password, user.password_hash):
            raise HTTPException(status_code=401, detail="用户名或密码错误")

        if not user.is_active:
            raise HTTPException(status_code=403, detail="账户已被禁用")

        # 生成令牌
        access_token = create_access_token(user.id, user.role)
        refresh_token = create_refresh_token(user.id)

        # 更新最后登录时间
        user.last_login_at = datetime.utcnow()

        return {
            "access_token": access_token,
            "refresh_token": refresh_token,
            "token_type": "bearer",
        }

    async def refresh_token(self, refresh_token: str) -> dict:
        """刷新令牌"""
        payload = decode_token(refresh_token)
        if not payload or payload.get("type") != "refresh":
            raise HTTPException(status_code=401, detail="无效的刷新令牌")

        user_id = int(payload["sub"])

        # 验证用户仍存在且活跃
        user = await self.session.get(User, user_id)
        if not user or not user.is_active:
            raise HTTPException(status_code=401, detail="用户不存在或已禁用")

        # 颁发新令牌
        return {
            "access_token": create_access_token(user.id, user.role),
            "refresh_token": create_refresh_token(user.id),
            "token_type": "bearer",
        }

    async def get_user_by_id(self, user_id: int) -> User:
        """获取用户信息"""
        user = await self.session.get(User, user_id)
        if not user:
            raise HTTPException(status_code=404, detail="用户不存在")
        return user
```text

### 5.4 Auth 路由

**app/routers/auth.py：**

```python
from typing import Annotated
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.dependencies import get_current_user
from app.models import User
from app.schemas.auth import (
    UserCreate, UserResponse, TokenResponse,
    LoginRequest, TokenRefresh,
)
from app.services.auth_service import AuthService

router = APIRouter()
DbSession = Annotated[AsyncSession, Depends(get_db)]


@router.post("/register", response_model=UserResponse, status_code=201)
async def register(data: UserCreate, session: DbSession):
    """用户注册"""
    service = AuthService(session)
    return await service.register(data)


@router.post("/login", response_model=TokenResponse)
async def login(data: LoginRequest, session: DbSession):
    """用户登录（返回 access_token + refresh_token）"""
    service = AuthService(session)
    return await service.login(data.username, data.password)


@router.post("/refresh", response_model=TokenResponse)
async def refresh(data: TokenRefresh, session: DbSession):
    """刷新访问令牌"""
    service = AuthService(session)
    return await service.refresh_token(data.refresh_token)


@router.get("/me", response_model=UserResponse)
async def get_me(current_user: User = Depends(get_current_user)):
    """获取当前用户信息"""
    return current_user


@router.post("/logout")
async def logout(
    current_user: User = Depends(get_current_user),
):
    """登出（客户端只需删除令牌）"""
    return {"message": "已登出"}
```

### 5.5 验证

```bash
# 1. 注册
curl -X POST http://localhost:8000/api/v1/auth/register \
    -H "Content-Type: application/json" \
    -d '{"username": "testuser", "email": "test@example.com", "password": "Test1234"}'

# 2. 登录
curl -X POST http://localhost:8000/api/v1/auth/login \
    -H "Content-Type: application/json" \
    -d '{"username": "testuser", "password": "Test1234"}'
# → {"access_token": "eyJ...", "refresh_token": "eyJ...", "token_type": "bearer"}

# 3. 访问受保护资源
TOKEN="eyJ..."
curl http://localhost:8000/api/v1/auth/me \
    -H "Authorization: Bearer $TOKEN"

# 4. 刷新令牌
REFRESH="eyJ..."
curl -X POST http://localhost:8000/api/v1/auth/refresh \
    -H "Content-Type: application/json" \
    -d "{\"refresh_token\": \"$REFRESH\"}"

# 5. 无效令牌测试
curl http://localhost:8000/api/v1/auth/me \
    -H "Authorization: Bearer invalid_token"
# → 401 Unauthorized
```text

---

## 6. 常见问题与排错

| 问题 | 原因 | 解决 |
| ------ | ------ | ------ |
| `jwt.exceptions.DecodeError` | Token 格式不对 | 检查是否包含三个用 `.` 分隔的部分 |
| `Signature verification failed` | SECRET_KEY 不匹配 | 在签名和验证时使用相同的密钥 |
| Token 过期后仍能使用 | 服务器时间不同步 | 检查 NTP 时间同步 |
| 算法混淆攻击 | 未固定预期算法 | 验证时指定 `algorithms=["HS256"]` |
| bcrypt 验证非常慢 | cost 值太高 | 默认 cost=12，生产建议 cost=10-12 |
| passlib 报错 | bcrypt 版本不匹配 | `pip install bcrypt==4.1.2` |

---

---

## 📁 项目代码参考

| 文件 | 说明 |
| ------ | ------ |
| `app/middleware/auth.py` | JWT 认证依赖：`get_current_user`、`RoleChecker`、`require_admin` |
| `app/services/auth_service.py` | 认证业务：注册/登录/令牌刷新/登出/黑名单 |
| `app/utils/jwt.py` | Token 生成/验证/黑名单（Redis 存储黑名单） |
| `app/utils/security.py` | 密码哈希（bcrypt） |
| `app/schemas/user.py` | 用户相关 Pydantic schema |
| `app/routers/auth.py` | 认证路由定义 |

**关键依赖注入路径：**

- `get_current_user` → `app.middleware.auth`
- `RoleChecker` → `app.middleware.auth`
- `decode_token` → `app.utils.jwt`
- Token 黑名单：Redis SET 存储，每次验证时检查

## 7. 本章总结

✅ **已掌握：**

- 密码安全：bcrypt 结构、成本因子、防 GPU 原理、时序攻击证明
- JWT：Header/Payload/Signature 结构、HMAC/RSA 签名、算法混淆攻防
- OAuth2：授权码+PKCE 流程、Refresh Token 旋转、重放攻击检测
- RBAC：角色层级、权限验证装饰器、资源级授权
- 完整认证系统：注册/登录/刷新/登出/用户信息

✅ **项目里程碑：**

- [ ] POST /auth/register — 用户注册（密码 bcrypt 哈希）
- [ ] POST /auth/login — 登录（返回 access + refresh token）
- [ ] POST /auth/refresh — 刷新令牌
- [ ] GET /auth/me — 获取用户信息
- [ ] RBAC 权限验证生效

**下一章预告：** 第8章将实现 WebSocket 实时座位更新、SSE 用户通知流、ARQ 后台任务队列。

**练习：**

1. 注册新用户并验证密码哈希是否正确
2. 使用 JWT 访问受保护的 API
3. 等待 access_token 过期后用 refresh_token 刷新
4. 实现令牌黑名单
5. 测试不同角色的权限隔离

---

## 进阶：用户封禁与行为限制

### 封号系统

#### 封号字段设计

在 User 模型中新增字段：

```python
is_banned: bool = False          # 封号标记
ban_reason: str | None = None    # 封号原因
banned_at: datetime | None       # 封号时间
banned_by: int | None            # 操作管理员 ID
```

#### 封号检查流程

```text
用户请求 API
  ↓
JWT 解析 → 获取 user_id
  ↓
数据库查询用户
  ↓
is_banned = True？
  ├── 是 → 返回 403 "账户已被封禁"
  └── 否 → 继续处理
```

#### 管理员封号 API

`POST /api/v1/admin/users/{id}/ban` — 需要管理员权限，记录封号原因和时间

### 行为限制系统

防止滥用和机器人的多层防护：

| 限制类型 | 阈值 | 惩罚措施 |
| --------- | ------ | --------- |
| 下单频率 | 5 单/10 分钟 | 429 Too Many Requests |
| 并发未支付订单 | 最多 3 笔 | 拒绝新订单 |
| IP 下单频率 | 20 单/小时 | 429 + 临时 IP 封禁 |
| 可疑行为 | 大量取消/高频操作 | 临时冻结 |

#### 实现原理（基于 Redis 滑动窗口）

```python
key = f"behavior:order_freq:{user_id}"
# ZREMRANGEBYSCORE key 0 (now - window)
# ZCARD key → 当前窗口内请求数
# 超过阈值 → 拒绝
# ZADD key now now
# EXPIRE key window + 60
```text

### 实时订单状态推送

支付成功/超时取消后，通过 WebSocket 向用户推送状态变更：

```

┌─────────┐     ┌──────────┐     ┌───────────┐
│ 支付服务  │ ──→ │ WebSocket │ ──→ │ 用户浏览器 │
│ 确认支付  │     │ 广播通知  │     │ 实时更新   │
└─────────┘     └──────────┘     └───────────┘

```text

用户通过 `/ws/user/orders?token=xxx` 订阅自己的订单状态变更。

---

## 8. 原理深入（补充）

### 8.1 JWT 签名算法对比：HS256 vs RS256

#### 核心区别

| 特性 | HS256 (HMAC-SHA256) | RS256 (RSA-SHA256) |
| ------ | ------------------- | ------------------- |
| 密钥类型 | 对称密钥 | 非对称密钥对 |
| 签名密钥 | 同一个密钥 | 私钥签名 |
| 验签密钥 | 同一个密钥 | 公钥验签 |
| 密钥管理 | 需严格保密，泄露 = 可伪造 | 私钥保密，公钥公开 |
| 性能 | 签名快，验签快 | 签名较慢(约HS256的1/10)，验签快 |
| 密钥长度 | 256位(32字节) | 2048位 |
| 分发难度 | 各方共享同一密钥，难分发 | 公钥可随意分发 |
| 适用场景 | 单体应用、内部服务 | 微服务、第三方集成 |
| 密钥轮换 | 轮换时所有签发/验签方需同步更新 | 只需轮换私钥，公钥持有者不受影响 |

#### 安全性对比

```

算法安全性假设：

HS256 的安全性建立在：
  HMAC-SHA256 是安全的伪随机函数（PRF）
  密钥是随机生成的强密钥（≥256 位熵）
  密钥从未泄露

RS256 的安全性建立在：
  RSA 问题是计算上困难的（大整数分解）
  私钥从未泄露
  公钥可信（通过证书链或可信渠道获取）

实际安全风险：

HS256 风险：
  密钥共享方越多，泄露风险越大
  如果密钥被攻破，所有过去和未来的令牌都可伪造
    → 解决方案：定期轮换密钥 + 限制密钥知晓范围

RS256 风险：
  公钥被替换（如果通过不安全渠道获取公钥）
  量子计算机可破解 RSA（远期风险）
    → 解决方案：证书固定（Certificate Pinning）+ 密切关注量子安全

```text

#### 为什么本项目用 HS256

```

本项目的选择理由：

1. 单体架构
   FastAPI 后端是单体服务，不存在多服务密钥分发问题。
   同一个密钥由同一服务器签发和验证，HS256 足够。

2. 性能优势
   HS256 签名和验签速度都快于 RS256。
   在高并发场景下（每秒数百次请求），性能差异明显。

3. 实现简单
   无需管理证书、公钥文件。
   SECRET_KEY 一行配置即可。

4. 密钥管理
   单个密钥存储在环境变量中，通过 .env 注入。
   不使用复杂的密钥管理系统 (KMS)。

如果本项目扩展为微服务架构，每个服务独立验证 JWT，
就应该切换到 RS256，让认证服务持有私钥签发令牌，
其他服务只持有公钥验证令牌，无需共享密钥。

```text

#### 算法混淆攻击与防御

```python
# 算法混淆攻击（Algorithm Confusion Attack）：
# 攻击者将 header 中的 alg 从 RS256 改为 HS256，
# 然后用已知的公钥作为 HMAC 密钥来伪造签名。

# 防御方案：

# 方案1（推荐）：验证时固定算法列表
payload = jwt.decode(
    token,
    public_key_or_secret,
    algorithms=["HS256"],  # 明确只接受 HS256
)

# 方案2：验证 header 中的 alg 字段
header = jwt.get_unverified_header(token)
if header["alg"] != "HS256":
    raise ValueError("Invalid algorithm")

# 方案3：使用不同的密钥验证不同算法（防止 HS256 vs RS256 混淆）
def decode_token(token: str, secret: str) -> dict:
    """
    安全的 JWT 解码（防御算法混淆）
    """
    # 1. 先获取未验证的 header
    header = jwt.get_unverified_header(token)

    # 2. 检查算法是否在白名单中
    allowed_algorithms = {"HS256", "HS384"}
    if header.get("alg") not in allowed_algorithms:
        raise ValueError(f"不允许的签名算法: {header.get('alg')}")

    # 3. 禁止 "none" 算法
    if header.get("alg") == "none":
        raise ValueError("禁止无签名令牌")

    # 4. 用预期算法解码
    try:
        payload = jwt.decode(
            token,
            secret,
            algorithms=[header["alg"]],
            options={"require": ["exp", "sub", "type"]},
        )
        return payload
    except jwt.PyJWTError as e:
        raise ValueError(f"令牌验证失败: {e}")
```

### 8.2 刷新令牌轮换设计

#### 为什么要轮换

```text
没有轮换的风险：
  用户登录后获得 access_token(30min) + refresh_token(7天)。
  如果 refresh_token 被窃取：
  - 攻击者可在 7 天内任意刷新获取新的 access_token
  - 合法用户无感知（攻击者和合法用户都能用）
  - 无法确定哪个 token 是合法的

有轮换的防御：
  每次刷新时，旧的 refresh_token 立即失效。
  如果 refresh_token 被窃取：
  - 攻击者刷新后，合法用户下次刷新会失败
  - 合法用户发现刷新失败 → 知道 token 被盗
  - 系统可以检测到同一用户短时间内多次刷新（异常行为）
```

#### 轮换实现（带重放检测）

```python
async def refresh_with_rotation(
    self,
    refresh_token: str,
    user_id: int,
) -> dict:
    """
    刷新令牌（带轮换 + 重放检测）

    流程：
    1. 解码 refresh_token，提取 jti
    2. 检查 jti 是否在黑名单中（重放检测）
    3. 若 jti 已被使用 → 可能是 token 泄露！
       → 吊销该用户所有 refresh_token（强制重新登录）
    4. 颁发新的 access_token + refresh_token
    5. 将旧 refresh_token 的 jti 加入黑名单（使旧令牌失效）
    """
    # 1. 解码
    payload = decode_token(refresh_token)
    if not payload or payload.get("type") != "refresh":
        raise ValueError("无效的刷新令牌")

    # 2. 验证用户
    user = await self.session.get(User, user_id)
    if not user or not user.is_active:
        raise ValueError("用户不存在或已禁用")

    jti = payload.get("jti")

    # 3. 重放检测（核心！）
    if jti and await is_token_revoked(self.redis, jti):
        # 检测到重放攻击！
        # 说明攻击者在使用旧的 refresh_token
        # 此时应该吊销该用户所有令牌
        app.logger.warning(
            f"检测到 refresh_token 重放攻击: user_id={user_id}, jti={jti}"
        )
        await self.revoke_all_user_tokens(user_id)

        # 记录安全事件
        await self.record_security_event(
            user_id=user_id,
            event_type="token_replay_detected",
            detail=f"Refresh token被重放, jti={jti}",
        )

        raise ValueError("检测到令牌重放，所有令牌已吊销，请重新登录")

    # 4. 轮换：吊销旧令牌
    if jti:
        # 旧 refresh_token 的剩余有效期作为黑名单 TTL
        old_exp = payload.get("exp", 0)
        ttl = max(0, old_exp - int(time.time()))
        await revoke_token(self.redis, jti, ttl)

    # 5. 颁发新令牌
    new_access = create_access_token(user_id, user.role)
    new_refresh = create_refresh_token(user_id)

    return {
        "access_token": new_access,
        "refresh_token": new_refresh,
        "token_type": "bearer",
    }
```text

#### 令牌家族树

```

refresh_token 每次轮换形成一条"家族链"：

  登录签发:
    refresh_token_1 (jti_1)
      │
  第一次刷新:
    refresh_token_2 (jti_2)  ← jti_1 加入黑名单
      │
  第二次刷新:
    refresh_token_3 (jti_3)  ← jti_2 加入黑名单
      │
  第三次刷新:
    refresh_token_4 (jti_4)  ← jti_3 加入黑名单

  如果攻击者窃取了 refresh_token_2 并在刷新后重放：
    系统发现 jti_2 已在黑名单中
    → 检测到重放攻击
    → 吊销 jwt_3, jti_4 等所有后续令牌
    → 用户重新登录

```text

### 8.3 黑名单实现原理

#### 黑名单数据结构

```python
# 黑名单使用 Redis String 存储
# Key: jwt:blacklist:{jti}
# Value: "1"
# TTL: 令牌剩余有效期

async def revoke_token(redis: AsyncRedis, jti: str, ttl: int = 86400):
    """
    将 JWT 加入黑名单

    参数:
        jti: JWT ID（令牌唯一标识）
        ttl: 黑名单有效时间 ≈ 令牌剩余有效期
             过期后 Redis 自动删除，避免内存泄漏
    """
    await redis.setex(f"jwt:blacklist:{jti}", ttl, "1")

async def is_token_revoked(redis: AsyncRedis, jti: str) -> bool:
    """检查令牌是否被吊销"""
    return await redis.exists(f"jwt:blacklist:{jti}") == 1
```

#### 黑名单在认证中间件中的使用

```python
# app/middleware/auth.py

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    session: AsyncSession = Depends(get_db),
    redis: AsyncRedis = Depends(get_redis),
) -> User:
    """
    获取当前用户（JWT 认证 + 黑名单检查）

    完整流程：
    1. 从 Authorization header 提取 token
    2. 解码 JWT，验证签名
    3. 检查 token 是否在黑名单中（登出/被吊销）
    4. 检查 token 类型（必须是 access_token）
    5. 查询用户信息
    6. 检查用户是否被封禁
    """
    try:
        # 1. 解码 JWT
        payload = decode_token(token)
        if payload is None:
            raise HTTPException(status_code=401, detail="无效的访问令牌")

        # 2. 检查令牌类型
        if payload.get("type") != "access":
            raise HTTPException(status_code=401, detail="令牌类型错误")

        # 3. 检查黑名单
        jti = payload.get("jti")
        if jti and await is_token_revoked(redis, jti):
            raise HTTPException(
                status_code=401,
                detail="令牌已被吊销",
                headers={"WWW-Authenticate": "Bearer"},
            )

        # 4. 查询用户
        user_id = int(payload["sub"])
        user = await session.get(User, user_id)
        if user is None:
            raise HTTPException(status_code=401, detail="用户不存在")

        # 5. 检查封禁
        if not user.is_active or getattr(user, 'is_banned', False):
            raise HTTPException(status_code=403, detail="账户已被禁用")

        return user

    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=401, detail="认证失败")
```text

#### 黑名单的几种应用场景

```

1. 登出（Logout）:
   用户点击登出 → 将当前 access_token 的 jti 加入黑名单
   用户必须使用新的 access_token 才能访问 API

2. 令牌轮换（Refresh Token Rotation）:
   刷新令牌时，将旧 refresh_token 的 jti 加入黑名单
   旧 refresh_token 立即失效

3. 重放攻击检测（Replay Detection）:
   如果某个 jti 已存在于黑名单，却被再次使用来刷新
   检测为重放攻击，吊销所有令牌

4. 强制登出（Force Logout）:
   管理员封禁用户时，吊销该用户所有令牌
   → 用户所有活跃会话立即失效

```text

#### 黑名单性能与容量

```

每个黑名单条目：
  Key: jwt:blacklist:{jti} (约 50 字节)
  Value: "1" (4 字节)
  TTL: 令牌剩余有效期（最多 7 天）

10 万用户 × 每人平均 2 个活跃令牌 = 20 万条黑名单
内存占用：20万 × 100 字节 ≈ 20MB（完全可以接受）

Redis 自动清理过期条目，不会持续增长。

黑名单查询性能：
  每次请求都进行一次 Redis EXISTS 查询（微秒级）
  对整体请求延迟影响可忽略

```text

---

## 9. 短信验证码登录与多因素认证

> **对应代码**: `app/routers/auth.py`、`app/services/auth_service.py`、`app/services/sms_service.py`

### 9.1 短信验证码登录流程

短信验证码登录是一种无密码认证方式，用户通过手机号 + 验证码完成身份验证：

```

用户输入手机号 → 点击"发送验证码" → 接收短信 → 输入验证码 → 登录
                      ↓
               SMSService.send_code()
               生成6位随机码
               存入 Redis（5分钟过期）
               调用短信网关发送

```text

**与密码登录的主要区别**：

| 对比项 | 密码登录 | 短信验证码登录 |
| -------- | --------- | -------------- |
| 凭证 | 用户名+密码 | 手机号+验证码 |
| 验证 | bcrypt 密码哈希校验 | SMSService 验证码校验 |
| 安全性 | 依赖密码强度 | 依赖手机号控制+验证码时效 |
| 适用场景 | 常规登录 | 忘记密码、便捷登录、新设备验证 |
| 账户锁定 | 连续失败 N 次锁定 | 发送频率限制（60秒内只能发一次） |

### 9.2 SMS 验证码服务

```python
# app/services/sms_service.py（示意）
class SMSService:
    def __init__(self, redis: Redis):
        self.redis = redis

    async def send_code(self, phone: str) -> str:
        """发送验证码（含频率限制）"""
        # 1. 检查频率限制（60秒内不能重复发送）
        key = f"sms:code:{phone}"
        if await self.redis.exists(key):
            raise ValueError("验证码已发送，请60秒后再试")

        # 2. 生成6位随机验证码
        code = str(random.randint(100000, 999999))

        # 3. 存入 Redis（5分钟过期）
        await self.redis.setex(key, 300, code)

        # 4. 调用短信网关发送（开发环境直接返回 code）
        # await sms_gateway.send(phone, f"您的验证码是：{code}")
        return code

    async def verify_code(self, phone: str, code: str) -> bool:
        """验证验证码"""
        key = f"sms:code:{phone}"
        stored = await self.redis.get(key)
        if stored and stored == code:
            await self.redis.delete(key)  # 一次性使用
            return True
        return False
```

**设计亮点**：

- **频率限制**：同一手机号 60 秒内只能发送一次验证码，防止短信轰炸
- **一次性使用**：验证码验证通过后立即从 Redis 删除，防止重放
- **自动过期**：验证码 5 分钟后自动过期，平衡安全性和用户体验
- **开发调试**：开发环境直接返回验证码，方便接口调试

### 9.3 登录路由对比

```text
密码登录：   POST /api/v1/auth/login     → 需要 username + password
短信登录：   POST /api/v1/auth/sms-login  → 需要 phone + sms_code

两种方式都返回 TokenResponse（access_token + refresh_token）
前端可以共用同一套 JWT 管理逻辑
```

### 9.4 多因素认证策略

本系统在实现中展示了多因素认证的思想，虽然不是强制 MFA，但提供了多个认证因素的选择：

| 因素 | 实现 | 说明 |
| ------ | ------ | ------ |
| 知识因素 | 密码登录 | 你知道的（密码） |
| 持有因素 | 短信验证码 | 你拥有的（手机号） |
| 生物因素 | 实名认证 | 你是什么（真实姓名+身份证号） |

**实际应用建议**：

- 高安全场景（如大额支付、修改关键信息）：要求同时完成密码 + 短信验证码验证
- 低风险操作（如查看订单）：单一认证因素即可
- 本系统的核销验证码（TC+YYMMDD+14hex）也属于一种一次性令牌认证

### 9.5 前端焦点：3-Tab 登录页

登录页使用 `v-show` 实现三标签切换布局，各标签页的组件逻辑相互独立：

```vue
<div class="tab-bar">
  <button v-for="tab in tabs" :key="tab.key"
    class="tab-btn" :class="{ active: activeTab === tab.key }"
    @click="switchTab(tab.key)">
    {{ tab.label }}
  </button>
</div>

<!-- Tab 1: 密码登录 -->
<form v-show="activeTab === 'password'" @submit.prevent="handlePasswordLogin">
  <input v-model="passwordLogin.username" placeholder="用户名" />
  <input v-model="passwordLogin.password" type="password" placeholder="密码" />
  <div v-if="passwordLogin.error" class="error-box">
    <span>{{ passwordLogin.error }}</span>  <!-- 中文错误提示 -->
  </div>
  <button type="submit">登录</button>
</form>

<!-- Tab 2: 短信登录 -->
<form v-show="activeTab === 'sms'" @submit.prevent="handleSmsLogin">
  <input v-model="smsLogin.phone" placeholder="手机号" maxlength="11" />
  <div class="sms-row">
    <input v-model="smsLogin.code" placeholder="验证码" maxlength="6" />
    <button type="button" @click="sendSmsCode" :disabled="smsLogin.countdown > 0">
      {{ smsLogin.countdown > 0 ? `${smsLogin.countdown}s` : '获取验证码' }}
    </button>
  </div>
  <button type="submit">登录</button>
</form>

<!-- Tab 3: 注册 -->
<form v-show="activeTab === 'register'" @submit.prevent="handleRegister">
  <!-- 用户名 + 密码 + 确认密码 + 邮箱 + 手机号 + 验证码 + 防机器人 -->
</form>
```text

**为什么用 v-show 而不是 v-if？** `v-show` 只切换 CSS 的 `display` 属性，各表单的 DOM 保持存在，输入状态不会丢失。`v-if` 会销毁重建 DOM，切换标签时用户输入会丢失。
