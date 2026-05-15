# 注册安全与实名认证系统

## 概述

本系统提供三层安全防护：

1. **防机器人验证** — 注册前需通过数学题 CAPTCHA
2. **手机验证** — 注册需短信验证码验证手机号
3. **实名认证** — 购票前必须完成实名认证（姓名 + 身份证号）

---

## 一、防机器人验证

### 实现原理

使用简单的数学题 CAPTCHA（Completely Automated Public Turing test to tell Computers and Humans Apart）：

1. 前端调用 `GET /api/v1/auth/captcha` 获取验证题
2. 后端生成随机数学题（如 "23 + 45 = ?"），答案存入 Redis
3. 前端展示题目，用户输入答案
4. 注册时提交 `captcha_token` + `captcha_answer`，后端验证

### 代码位置

`app/services/captcha_service.py` — `CaptchaService` 类

### 核心方法

| 方法 | 说明 |
| ------ | ------ |
| `generate()` | 生成题目 + 答案（开发环境） |
| `generate_public()` | 生成题目（生产环境，不含答案） |
| `verify(token, answer)` | 验证答案，一次性使用 |

### 安全要点

- **一次性使用**：验证通过后立即从 Redis 删除，防止重放
- **5 分钟过期**：Redis TTL 自动清理
- **轻量级**：无需第三方服务，零成本

### 生产环境替换

将 `CaptchaService` 替换为 Google reCAPTCHA：

```python
async def verify(self, token: str, answer: str) -> bool:
    # 调用 Google reCAPTCHA API
    response = await httpx.post(
        "https://www.google.com/recaptcha/api/siteverify",
        data={"secret": RECAPTCHA_SECRET, "response": token},
    )
    return response.json().get("success", False)
```text

---

## 二、手机短信验证

### 实现原理

1. 用户输入手机号，请求发送验证码
2. 后端生成 6 位随机码，存入 Redis（Key: `sms:code:{phone}`）
3. 开发环境直接打印到日志，生产环境调用第三方 SMS API
4. 用户输入验证码，后端比对
5. 验证成功后删除 Redis Key（一次性使用）

### 代码位置

`app/services/sms_service.py` — `SMSService` 类

### 流程

```

用户 → POST /auth/send-sms → 生成 6 位码 → Redis 存储 → 发送 SMS
用户 → POST /auth/register → 校验 SMS 码 → 创建用户 (phone_verified=True)

```text

### 频率限制

同一手机号 60 秒内只能发送一次：

```python
ttl = await self.redis.ttl(key)
if ttl > SMS_CODE_TTL - 60:  # 300 - 60 = 240秒内不让重发
    raise ValueError("发送过于频繁")
```

### 安全要点

- **5 分钟有效期**：验证码 300 秒后自动过期
- **一次性使用**：验证成功后立即从 Redis 删除
- **频率限制**：同一手机号 60 秒内不能重复发送

---

## 三、实名认证

### 实现原理

1. 用户提交真实姓名 + 身份证号
2. 调用第三方 API 校验身份信息真实性
3. 校验通过后保存到数据库
4. 购票时检查 `id_verified` 字段

### 代码位置

`app/services/realname_service.py` — `RealNameService` 类

### 第三方 API 接入

```python
# 阿里云实人认证
response = await httpx.post(api_url, json={
    "real_name": real_name,
    "id_number": id_number,
    "phone": phone,
}, headers={
    "Authorization": f"APPCODE {settings.REALNAME_API_APPCODE}",
})
```text

### 开发环境 Mock

- 身份证号 `110101199001011234` 等测试号自动通过
- 其他 18 位身份证号也通过
- 生产环境通过 `REALNAME_MOCK_ENABLED=False` 关闭

### 数据脱敏

```python
# 张** → 张三
mask_real_name("张三")  # → "张*"

# 110101********1234
mask_id_number("110101199001011234")  # → "110101********1234"
```

---

## 四、注册流程

### 完整流程图

```text
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ 1.获取验证题 │ → │ 2.发送短信 │ → │ 3.提交注册 │ → │ 4.注册成功 │
│ GET captcha │   │ POST send │   │POST register│  │ 用户创建   │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
                                       │
                                   ┌───┴───┐
                                   │ CAPTCHA│ ← 防机器人
                                   │ 验证   │
                                   └───┬───┘
                                       │
                                   ┌───┴───┐
                                   │ 短信码 │ ← 手机验证
                                   │ 验证   │
                                   └───┬───┘
                                       │
                                   ┌───┴───┐
                                   │ 创建   │
                                   │ 用户   │
                                   └───────┘
```

### 注册请求体

```json
{
    "username": "testuser",
    "password": "Test123!@#",
    "email": "test@example.com",
    "phone": "13800138000",
    "sms_code": "123456",
    "captcha_token": "a1b2c3d4e5f6...",
    "captcha_answer": "68"
}
```text

### 注册后提示

注册成功后前端可以提示：
> "注册成功！建议您立即完成实名认证，否则无法购票。"

---

## 五、购票流程中的实名检查

### 源码位置

`app/middleware/auth.py` — `require_real_name_verified`

```python
async def require_real_name_verified(current_user: User = Depends(get_current_user)):
    if not current_user.id_verified:
        raise HTTPException(status_code=403, detail="请先完成实名认证后再购票")
    if not current_user.phone_verified:
        raise HTTPException(status_code=403, detail="请先验证手机号后再购票")
    return current_user
```

在订单创建接口中使用：

```python
@router.post("/orders")
async def create_order(
    current_user: User = Depends(require_real_name_verified),
    ...
):
```text

### 认证链路

```

JWT 认证 → 实名检查 → 手机验证检查 → 频率限制 → 防超卖 → 创建订单

```text

---

## 六、API 端点参考

| 方法 | 路径 | 说明 | 认证 |
| ------ | ------ | ------ | ------ |
| GET | `/api/v1/auth/captcha` | 获取防机器人验证题 | 否 |
| POST | `/api/v1/auth/send-sms` | 发送短信验证码 | 否 |
| POST | `/api/v1/auth/register` | 注册（含验证码+防机器人） | 否 |
| POST | `/api/v1/auth/legacy-register` | 旧版注册（兼容测试） | 否 |
| POST | `/api/v1/auth/verify-phone` | 验证手机号 | 是 |
| POST | `/api/v1/auth/real-name-verify` | 实名认证 | 是 |
| GET | `/api/v1/auth/real-name-status` | 获取实名认证状态 | 是 |
| GET | `/api/v1/auth/events/{id}/purchase-notes` | 获取购票须知 | 否 |

---

## 七、数据库字段

### users 表新增字段

| 字段 | 类型 | 说明 |
| ------ | ------ | ------ |
| `phone_verified` | Boolean | 手机号是否已验证 |
| `real_name` | String(50) | 真实姓名 |
| `id_number` | String(18) | 身份证号 |
| `id_verified` | Boolean | 是否已实名认证 |
| `id_verified_at` | DateTime | 实名认证时间 |

### events 表新增字段

| 字段 | 类型 | 说明 |
| ------ | ------ | ------ |
| `purchase_notes` | Text | 购票须知（Markdown） |

---

## 八、配置项

在 `.env` 文件中：

```bash
# 实名认证
REALNAME_MOCK_ENABLED=true
REALNAME_API_URL=https://api.verify.com/v3
REALNAME_API_APPCODE=your_appcode

# 短信服务
SMS_MOCK_ENABLED=true
SMS_API_URL=https://api.sms.com/send
SMS_API_APPCODE=your_appcode
SMS_CODE_TTL=300
SMS_CODE_LENGTH=6
```

---

## 九、安全最佳实践

1. **HTTPS**：所有认证接口必须使用 HTTPS
2. **验证码一次性**：验证成功后立即从 Redis 删除
3. **身份证加密**：生产环境建议对 `id_number` 字段进行 AES 加密存储
4. **频率限制**：短信发送和注册接口需做频率限制
5. **脱敏展示**：不在前端明文展示身份证号和真实姓名
6. **审计日志**：记录所有认证尝试
