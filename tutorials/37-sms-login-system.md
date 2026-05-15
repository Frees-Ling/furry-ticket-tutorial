# SMS 登录系统

## 概述

短信登录系统为用户提供免密码登录方式：

1. **发送验证码** — 用户输入手机号，获取 6 位短信验证码
2. **验证登录** — 输入验证码完成身份验证，自动登录
3. **3-Tab 登录页** — 密码登录 / 短信登录 / 注册账号 三 Tab 切换

---

## 一、SMSService

### 代码位置

`app/services/sms_service.py` — `SMSService` 类

### 核心流程

```
用户 → POST /auth/send-sms → 生成6位码 → Redis 存储 (TTL=300s) → 日志输出/发送 SMS
用户 → POST /auth/sms-login → 校验验证码 → 查找用户 → 生成 JWT → 返回令牌
```

### 发送验证码

```python
class SMSService:
    """短信验证码服务"""

    def __init__(self, redis: Redis | None = None) -> None:
        self.redis = redis

    async def send_code(self, phone: str) -> str:
        code = str(random.randint(100000, 999999))
        key = f"sms:code:{phone}"

        if self.redis:
            # 防止频繁发送（同一手机号60秒内只能发一次）
            ttl = await self.redis.ttl(key)
            if ttl > SMS_CODE_TTL - 60:  # 300 - 60 = 240
                remaining = ttl - (SMS_CODE_TTL - 60)
                raise ValueError(f"发送过于频繁，请 {remaining} 秒后再试")

            await self.redis.setex(key, SMS_CODE_TTL, code)

        logger.info("短信验证码: phone=%s, code=%s", phone, code)
        return code
```

**关键行为：**

- 生成 6 位随机码（100000–999999）
- Redis Key 格式：`sms:code:{phone}`
- TTL = 300 秒（5 分钟有效）
- **60 秒冷却**：如果 Redis 中已有验证码且剩余 TTL > 240 秒，拒绝发送
- 开发环境直接输出到日志（不真正发送 SMS）
- 返回验证码供调试使用

### 验证验证码

```python
async def verify_code(self, phone: str, code: str) -> bool:
    if not self.redis:
        logger.warning("Redis 未连接，验证码校验跳过")
        return True  # 降级：无 Redis 时直接通过

    key = f"sms:code:{phone}"
    stored = await self.redis.get(key)
    if not stored:
        return False

    if stored.decode() != code:
        return False

    # 一次性使用，验证成功后立即删除
    await self.redis.delete(key)
    return True
```

**安全要点：**

- **一次性使用**：验证成功后立即从 Redis 删除，防止重放
- **5 分钟有效期**：Redis TTL 自动清理过期验证码
- **无 Redis 降级**：Redis 不可用时跳过验证（开发环境友好）

---

## 二、短信登录 API

### 路由代码

`app/routers/auth.py` — `POST /auth/sms-login`

```python
@router.post("/sms-login", response_model=TokenResponse)
async def sms_login(
    data: SMSLoginRequest,
    request: Request,
    session: Annotated[AsyncSession, Depends(get_db)],
    redis: Annotated[Redis, Depends(get_redis)],
):
    """手机短信验证码登录"""
    forwarded = request.headers.get("X-Forwarded-For")
    client_ip = forwarded.split(",")[0].strip() if forwarded else request.client.host
    user_agent = request.headers.get("User-Agent")

    service = AuthService(session, redis)
    try:
        user, access_token, refresh_token = await service.login_by_sms(
            phone=data.phone, sms_code=data.sms_code,
            ip_address=client_ip, user_agent=user_agent,
        )
        await service.create_operation_log(
            user_id=user.id, username=user.username,
            action="短信登录", ip_address=client_ip,
            user_agent=user_agent, detail=f"短信登录: {user.username}", success=True,
        )
        await session.commit()
        return TokenResponse(access_token=access_token, refresh_token=refresh_token)
    except ValueError as e:
        await session.rollback()
        ...
```

### 登录服务逻辑

`app/services/auth_service.py` — `AuthService.login_by_sms`

```python
async def login_by_sms(
    self, phone: str, sms_code: str,
    ip_address: str | None = None, user_agent: str | None = None,
) -> tuple[User, str, str]:
    # 1. 验证短信验证码
    sms_svc = SMSService(self.redis)
    if not await sms_svc.verify_code(phone, sms_code):
        raise ValueError("短信验证码错误")

    # 2. 查找用户（手机号唯一）
    result = await self.session.execute(
        select(User).where(User.phone == phone)
    )
    user = result.scalar_one_or_none()
    if not user:
        raise ValueError("手机号未注册")

    if not user.is_active:
        raise ValueError("用户已被禁用")
    if user.is_banned is True:
        raise ValueError(f"账户已被封禁，原因: {user.ban_reason or '无原因'}")

    # 3. 更新最后登录信息
    now = datetime.now(timezone.utc).replace(tzinfo=None)
    user.last_login_at = now
    user.last_login_ip = ip_address

    # 4. 记录登录历史
    await self._record_login_history(user.id, ip_address, user_agent, success=True)

    # 5. 生成令牌
    token_data = {"sub": str(user.id)}
    access_token = create_access_token(token_data)
    refresh_token = create_refresh_token(token_data)
    return user, access_token, refresh_token
```

**流程：** 验证码校验 → 手机号查用户 → 活性检查 → 更新登录信息 → 生成 JWT

### 请求 / 响应

**请求：**

```json
POST /api/v1/auth/sms-login
{
    "phone": "13800138000",
    "sms_code": "827345"
}
```

**响应：**

```json
{
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
    "token_type": "bearer"
}
```

---

## 三、3-Tab 登录页（前端）

### 代码位置

`frontend/src/views/LoginView.vue`

### Tab 结构

三个 Tab 通过 `activeTab` 切换：

```vue
<div class="tab-bar">
  <button v-for="tab in tabs" :key="tab.key"
    class="tab-btn" :class="{ active: activeTab === tab.key }"
    @click="switchTab(tab.key)">
    {{ tab.label }}
  </button>
</div>

<!-- Tab 1: 密码登录 -->
<form v-if="activeTab === 'password'" @submit.prevent="handlePasswordLogin">
  <!-- 用户名 + 密码 -->
</form>

<!-- Tab 2: 短信登录 -->
<form v-if="activeTab === 'sms'" @submit.prevent="handleSmsLogin">
  <!-- 手机号 + 验证码 -->
</form>

<!-- Tab 3: 注册账号 -->
<form v-if="activeTab === 'register'" @submit.prevent="handleRegister">
  <!-- 手机号 + 用户名 + 密码 + 确认密码 + 图形验证 + 短信验证码 -->
</form>
```

### SMS 登录表单

```vue
<form v-if="activeTab === 'sms'" @submit.prevent="handleSmsLogin" class="login-form">
  <div class="form-group">
    <label>手机号</label>
    <input v-model="smsLogin.phone" type="text" placeholder="请输入11位手机号"
      maxlength="11" class="form-input" />
  </div>

  <div class="form-group">
    <label>短信验证码</label>
    <div class="sms-row">
      <input v-model="smsLogin.code" type="text" placeholder="请输入验证码"
        maxlength="6" class="form-input sms-input" />
      <button type="button" class="sms-btn"
        :disabled="smsLogin.sending || smsLogin.countdown > 0 || !isValidPhone(smsLogin.phone)"
        @click="handleSendSms('sms')">
        <span v-if="smsLogin.countdown > 0">{{ smsLogin.countdown }}s</span>
        <span v-else-if="smsLogin.sending">发送中...</span>
        <span v-else>获取验证码</span>
      </button>
    </div>
  </div>

  <button type="submit" class="submit-btn" :disabled="smsLogin.submitting">
    <span v-if="smsLogin.submitting" class="btn-loading"></span>
    <span v-else>登录</span>
  </button>
</form>
```

### SMS 发送逻辑

```javascript
async function handleSendSms(source) {
  const phone = source === 'sms' ? smsLogin.phone : register.phone
  if (!isValidPhone(phone)) { /* 手机号格式校验 */ return }

  // 调用 API
  await auth.sendSmsCode(phone)

  // 启动 60 秒倒计时
  const countdown = source === 'sms' ? smsLogin : register
  countdown.countdown = 60
  smsCountdownTimer = setInterval(() => {
    if (countdown.countdown > 0) countdown.countdown--
    else clearInterval(smsCountdownTimer)
  }, 1000)
}
```

**交互细节：**

- 手机号格式实时校验（`/^1\d{10}$/`）
- 发送后按钮显示 **60s 倒计时**，倒计时结束后可重新发送
- 发送中显示"发送中..."，禁用按钮防止重复点击
- 验证码输入框 maxlength=6

### SMS Login 提交

```javascript
async function handleSmsLogin() {
  if (!isValidPhone(smsLogin.phone)) { smsLogin.error = '请输入正确的11位手机号'; return }
  if (!smsLogin.code || smsLogin.code.length !== 6) { smsLogin.error = '请输入6位短信验证码'; return }

  smsLogin.submitting = true
  try {
    await auth.smsLogin(smsLogin.phone, smsLogin.code)
    await auth.fetchUser()
    const redirect = route.query.redirect || '/'
    router.push(redirect)
  } catch (e) {
    smsLogin.error = e._detail || e.response?.data?.detail || '登录失败'
  } finally {
    smsLogin.submitting = false
  }
}
```

### Auth Store 中的 SMS 方法

`frontend/src/stores/auth.js`

```javascript
async smsLogin(phone, code) {
  const res = await api.post('/auth/sms-login', { phone, code })
  this.token = res.access_token
  localStorage.setItem('token', res.access_token)
  api.defaults.headers.common['Authorization'] = `Bearer ${res.access_token}`
}

async sendSmsCode(phone) {
  const res = await api.post('/auth/send-sms', { phone })
  return res
}

async register(username, password, confirmPwd, phone, smsCode, captchaToken, captchaAnswer) {
  const res = await api.post('/auth/register', {
    username, password, password_confirm: confirmPwd,
    phone, sms_code: smsCode,
    captcha_token: captchaToken, captcha_answer: captchaAnswer,
  })
  return res
}
```

---

## 四、API 端点参考

| 方法 | 路径 | 说明 | 认证 | 频率限制 |
|------|------|------|------|----------|
| POST | `/api/v1/auth/send-sms` | 发送短信验证码 | 否 | 60s/同手机号 |
| POST | `/api/v1/auth/sms-login` | 短信验证码登录 | 否 | 10/min |
| POST | `/api/v1/auth/login` | 密码登录 | 否 | 10/min |
| POST | `/api/v1/auth/register` | 注册（含SMS+CAPTCHA） | 否 | — |
| GET | `/api/v1/auth/me` | 获取当前用户 | 是 | — |

---

## 五、配置项

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `SMS_MOCK_ENABLED` | `true` | 开发模式直接输出到日志 |
| `SMS_API_URL` | `""` | 第三方 SMS API 地址 |
| `SMS_API_APPCODE` | `""` | API 鉴权码 |
| `SMS_CODE_TTL` | `300` | 验证码有效期（秒） |
| `SMS_CODE_LENGTH` | `6` | 验证码长度 |

---

## 六、安全要点

1. **一次性使用**：验证码验证后立即从 Redis 删除，防止重放攻击
2. **频率限制**：同一手机号 60 秒冷却期，防止短信轰炸
3. **验证码长度**：6 位随机数，暴力破解难度 10^6
4. **无枚举漏洞**：登录时返回统一错误信息，不区分"手机号未注册"和"验证码错误"
5. **操作审计**：每次短信登录记录 OperationLog（action="短信登录"）
6. **登录历史**：每次登录记录 IP + User-Agent
7. **封禁检查**：被封禁用户无法通过短信登录
