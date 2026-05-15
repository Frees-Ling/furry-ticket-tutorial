# 密码策略与注册安全增强

## 概述

系统提供四层密码与注册安全防护：

1. **后端密码策略** — `PasswordPolicy` 校验密码复杂度
2. **前端密码强度** — 实时强度条 + 弱/中/强视觉反馈
3. **确认密码** — 前后端双重校验
4. **防机器人验证** — CAPTCHA 数学题

---

## 一、PasswordPolicy

### 代码位置

`app/services/auth_service.py` — `PasswordPolicy` 类

```python
class PasswordPolicy:
    """密码策略校验."""

    MIN_LENGTH = settings.PASSWORD_MIN_LENGTH       # 8
    REQUIRE_UPPER = settings.PASSWORD_REQUIRE_UPPER  # True
    REQUIRE_LOWER = settings.PASSWORD_REQUIRE_LOWER  # True
    REQUIRE_DIGIT = settings.PASSWORD_REQUIRE_DIGIT  # True
    REQUIRE_SPECIAL = settings.PASSWORD_REQUIRE_SPECIAL  # True

    @classmethod
    def validate(cls, password: str) -> str | None:
        """校验密码强度，返回错误消息（None 表示通过）."""
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
```

### 配置项

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `PASSWORD_MIN_LENGTH` | `8` | 最小长度 |
| `PASSWORD_REQUIRE_UPPER` | `true` | 需要大写字母 |
| `PASSWORD_REQUIRE_LOWER` | `true` | 需要小写字母 |
| `PASSWORD_REQUIRE_DIGIT` | `true` | 需要数字 |
| `PASSWORD_REQUIRE_SPECIAL` | `true` | 需要特殊字符 |

### 注册时调用

```python
class AuthService:
    async def register(self, data: UserCreate) -> User:
        # 密码复杂度校验
        pwd_error = PasswordPolicy.validate(data.password)
        if pwd_error:
            raise ValueError(pwd_error)

        # 用户名/邮箱唯一性检查
        existing = await self.session.execute(
            select(User).where(
                (User.username == data.username) | (User.email == data.email)
            )
        )
        if existing.scalar_one_or_none():
            raise ValueError("用户名或邮箱已存在")

        # 创建用户（bcrypt 加盐哈希）
        user = User(
            username=data.username,
            password_hash=hash_password(data.password),
            email=data.email,
            phone=data.phone,
            role="user",
        )
        self.session.add(user)
        await self.session.flush()
        return user
```

---

## 二、注册路由中的密码校验

### 代码位置

`app/routers/auth.py` — `POST /auth/register`

路由层在调用 `AuthService.register` 之前进行额外的密码一致性校验：

```python
@router.post("/register", response_model=UserResponse, status_code=201)
async def register(data: RegisterRequest, ...):
    # 0. 密码一致性校验
    if data.password != data.password_confirm:
        raise HTTPException(status_code=400, detail="两次输入的密码不一致")

    # 密码长度校验（前端策略的后备）
    if len(data.password) < 8:
        raise HTTPException(status_code=400, detail="密码长度不能少于8位")

    # 密码特殊字符校验（前端策略的后备）
    if not re.search(r'[!@#$%^&*(),.?":{}|<>]', data.password):
        raise HTTPException(status_code=400, detail="密码必须包含至少一个特殊字符")

    # 1. 防机器人验证（CAPTCHA）
    captcha_svc = CaptchaService(redis)
    if not await captcha_svc.verify(data.captcha_token, data.captcha_answer):
        raise HTTPException(status_code=400, detail="验证码错误")

    # 2. 手机验证码验证
    sms_svc = SMSService(redis)
    if not await sms_svc.verify_code(data.phone, data.sms_code):
        raise HTTPException(status_code=400, detail="短信验证码错误或已过期")

    # 3. 创建用户（含 PasswordPolicy 校验）
    service = AuthService(session, redis)
    try:
        user = await service.register(create_data)
        user.phone_verified = True  # 标记手机已验证
        await service.create_operation_log(...)
        await session.commit()
        return user
    except ValueError as e:
        await session.rollback()
        raise HTTPException(status_code=409, detail=str(e))
```

**校验链：**

```
确认密码一致 → 密码长度 ≥ 8 → 含特殊字符 → CAPTCHA 验证 → SMS 验证 → PasswordPolicy → 唯一性检查 → 创建用户
```

---

## 三、前端密码强度条

### 代码位置

`frontend/src/views/LoginView.vue` — 注册 Tab

### 评分算法

```javascript
function getPasswordScore(pwd) {
  let score = 0
  if (pwd.length >= 8) score += 25     // 长度 ≥ 8
  if (/[!@#$%^&*(),.?":{}|<>_\-]/.test(pwd)) score += 25  // 特殊字符
  if (/\d/.test(pwd)) score += 20      // 数字
  if (/[A-Z]/.test(pwd)) score += 15   // 大写字母
  if (/[a-z]/.test(pwd)) score += 15   // 小写字母
  return score
}
```

### 强度映射

```javascript
const passwordStrengthClass = computed(() => {
  const s = passwordStrengthPercent.value
  if (s < 30) return 'weak'     // 红色
  if (s < 60) return 'medium'   // 黄色
  return 'strong'               // 绿色
})

const passwordStrengthLabel = computed(() => {
  const s = passwordStrengthPercent.value
  if (s < 30) return '弱'
  if (s < 60) return '中'
  return '强'
})
```

### 模板

```vue
<div class="form-group">
  <label>密码</label>
  <div class="input-suffix-wrap">
    <input v-model="register.password" :type="register.showPwd ? 'text' : 'password'"
      placeholder="至少8位，含特殊字符" class="form-input"
      @input="validateRegisterField('password')" />
    <button type="button" class="suffix-btn" @click="register.showPwd = !register.showPwd">
      {{ register.showPwd ? '隐藏' : '显示' }}
    </button>
  </div>
  <!-- Password strength bar -->
  <div class="strength-bar-wrap" v-if="register.password.length > 0">
    <div class="strength-bar" :class="passwordStrengthClass">
      <div class="strength-fill" :style="{ width: passwordStrengthPercent + '%' }"></div>
    </div>
    <span class="strength-label">{{ passwordStrengthLabel }}</span>
  </div>
</div>
```

### 视觉样式

```css
.strength-bar {
  flex: 1; height: 4px; border-radius: 3px;
  background: #f0f0f0; overflow: hidden;
}
.strength-fill {
  height: 100%; border-radius: 3px;
  transition: width 0.4s ease, background 0.4s ease;
}
.strength-bar.weak .strength-fill    { background: #ff4d4f; }  /* 红 */
.strength-bar.medium .strength-fill  { background: #faad14; }  /* 黄 */
.strength-bar.strong .strength-fill  { background: #52c41a; }  /* 绿 */
```

**交互体验：**

- 输入时实时计算强度
- 强度条宽度平滑过渡（`transition: width 0.4s ease`）
- 颜色随强度变化：弱=红 / 中=黄 / 强=绿

### 确认密码实时校验

```javascript
function validateRegisterField(field) {
  switch (field) {
    case 'password':
      if (register.password && register.password.length < 8)
        register.errors.password = '密码长度不能少于8位'
      else if (register.password && !/[!@#$%^&*(),.?":{}|<>_\-]/.test(register.password))
        register.errors.password = '密码必须包含至少一个特殊字符 (!@#$%^&*)'
      // 同时重新校验确认密码
      if (register.confirmPwd && register.password !== register.confirmPwd)
        register.errors.confirmPwd = '两次密码输入不一致'
      break
    case 'confirmPwd':
      if (register.confirmPwd && register.password !== register.confirmPwd)
        register.errors.confirmPwd = '两次密码输入不一致'
      break
  }
}
```

**特点：** 修改密码时自动触发确认密码的重新校验，无需用户额外操作。

---

## 四、防机器人验证（CAPTCHA）

### 代码位置

`app/services/captcha_service.py` — `CaptchaService`

### 验证流程

```
前端 → GET /auth/captcha → 返回 { captcha_token, question, expired_at }
前端 → 用户输入答案 → POST /auth/register 提交 { ..., captcha_token, captcha_answer }
后端 → Redis 查找 token → 比对答案 → 删除（一次性）
```

### 实现要点

- 生成随机数学题（如 "23 + 45 = ?"）
- 答案存入 Redis（Key: `captcha:{token}`）
- **一次性使用**：验证后立即删除，防止重放
- **5 分钟过期**：Redis TTL 自动清理
- 开发环境返回正确答案方便调试

### 前端解析

```vue
<div class="captcha-row">
  <div class="captcha-question" @click="loadCaptcha">
    <span v-if="register.captcha.question">{{ register.captcha.question }}</span>
    <span v-else>点击加载验证</span>
  </div>
  <input v-model="register.captcha.answer" type="text" placeholder="输入答案"
    maxlength="6" class="form-input captcha-input" />
  <button type="button" class="captcha-refresh" @click="loadCaptcha">⟳</button>
</div>
```

每次注册失败后自动刷新验证题：
```javascript
catch (e) {
  register.generalError = '注册失败'
  loadCaptcha()  // 刷新验证题
}
```

---

## 五、完整注册安全链

```
      前端                       后端
  ┌──────────┐            ┌──────────────────┐
  │ 密码强度条 │            │ PasswordPolicy   │
  │ 实时评分   │            │ 长度/大小写/数字/ │
  │ 弱/中/强  │            │ 特殊字符         │
  └──────────┘            └──────────────────┘
  ┌──────────┐            ┌──────────────────┐
  │ 确认密码   │            │ password_confirm  │
  │ 实时比对   │            │ 后端二次确认      │
  └──────────┘            └──────────────────┘
  ┌──────────┐            ┌──────────────────┐
  │ CAPTCHA   │ ─────────→ │ CaptchaService   │
  │ 数学题    │            │ 一次性/5分钟过期   │
  └──────────┘            └──────────────────┘
  ┌──────────┐            ┌──────────────────┐
  │ SMS验证码  │ ─────────→ │ SMSService       │
  │ 6位码     │            │ 一次性/60s限流    │
  └──────────┘            └──────────────────┘
                           ┌──────────────────┐
                           │ 唯一性检查        │
                           │ 用户名/邮箱/手机号 │
                           └──────────────────┘
                           ┌──────────────────┐
                           │ bcrypt 加盐哈希    │
                           │ rounds=12        │
                           └──────────────────┘
```

---

## 六、安全最佳实践

1. **纵深防御**：前端强度条仅用于用户体验，后端 `PasswordPolicy` 才是真正的强度保障
2. **一致性校验**：密码和确认密码在前后端各校验一次，防止绕过
3. **防机器人**：CAPTCHA 用于注册和敏感操作，防止自动化攻击
4. **密码存储**：使用 bcrypt 加盐哈希（工作因子 12），拒绝明文存储
5. **唯一性约束**：用户名和邮箱有数据库唯一索引，防重注册
6. **操作日志**：每次注册记录 OperationLog（action="注册"），用于审计
