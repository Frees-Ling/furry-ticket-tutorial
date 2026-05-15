# 第三方 API 实现详解

> 本文档详细讲解第三方 API 的实现原理、安全设计、工具用法。
> 配合 `45-third-party-api-reference.md` 阅读效果最佳。

---

## 一、架构关系

你只需要在 `.env` 里填好配置，系统自动加载并使用。

```
编辑 .env 文件（填上各平台的 Key、Secret、URL）
    │  ALIPAY_APP_ID=你的APPID
    │  ALIPAY_APP_PRIVATE_KEY_PATH=/etc/alipay/private.pem
    │  SMS_MOCK_ENABLED=true
    │  ENVIRONMENT=development
    │
    ▼
app/config.py（Pydantic Settings 读取 .env）
    │  settings.ALIPAY_APP_ID = "你的APPID"
    │  settings.ALIPAY_APP_PRIVATE_KEY_PATH = "/etc/alipay/private.pem"
    │  settings.IS_PRODUCTION = False
    │
    ▼
app/services/third_party_api.py（使用配置实现 API 调用）
    │  AlipayAPI.build_pay_form()     → 读 settings 里的支付宝配置
    │  SMSAPI.send_verification_code() → 读 settings 里的短信配置
    │  RealNameAPI.verify()            → 读 settings 里的实名认证配置
    │
    ▼
业务 Service 层调用
    │  PaymentService.create_alipay_payment()
    │  SMSService.send_code()
    │  RealNameService.verify()
    │
    ▼
用户前端看到结果
```

**一个配置在系统中的完整流动路径：**

```
你在 .env 写:        ALIPAY_APP_ID=2021000123456789
    │
config.py 变成:     settings.ALIPAY_APP_ID = "2021000123456789"
    │
third_party_api.py: params["app_id"] = settings.ALIPAY_APP_ID
    │
最终 URL 里出现:    https://openapi.alipaydev.com/gateway.do?app_id=2021000123456789&...
```

**你不需要手动调用 `third_party_api.py`。填好 `.env` 后，各业务 Service 会自动使用配置。**

---

## 二、支付宝支付安全设计

### 2.1 支付完整生命周期

```
用户点击"去支付"
  → 前端 POST /api/v1/payments/alipay { order_id }
  → 后端创建 Order（status = pending_payment）
  → 后端调用 AlipayAPI.build_pay_form()
  → 返回支付 URL，前端跳转支付宝
  → 用户在支付宝 APP 确认支付
  → 支付宝发异步通知到你的 notify_url
  → 后端处理通知、验签、更新订单
  → WebSocket 通知用户"支付成功"
```

关键：**只能用异步通知（notify_url）更新订单**，不能用同步返回（return_url）。因为同步返回到用户浏览器，任何人都可以伪造这个跳转。

### 2.2 四层防重放设计

为什么要四层？因为每一层解决不同的问题：

```
第一层：RSA2 签名验证
  ┌─ 验签不通过 → 请求不是支付宝发的 → 拒绝
  └─ 验签通过   → 请求确实是支付宝发的
  
  问题：支付宝可能重复推送同一通知（最多 8 次）
  
第二层：Redis SETNX 幂等键
  ┌─ SETNX 失败 → 已经处理过了 → 跳过
  └─ SETNX 成功 → 第一次处理 → 继续
  
  问题：攻击者截获了合法通知，篡改金额后再放行
  
第三层：金额校验
  ┌─ 通知金额 ≠ 订单金额 → 数据被篡改 → 拒绝
  └─ 通知金额 = 订单金额 → 一致 → 继续
  
  问题：订单已经支付过了，攻击者伪造另一笔"支付"进来
  
第四层：状态机保护
  ┌─ 订单不是 pending_payment → 状态异常 → 拒绝
  └─ 订单是 pending_payment  → 可以处理 → 更新为 paid
```

#### 第一层：RSA2 签名

非对称加密原理：

```
对称加密：一个钥匙开锁和关锁
非对称加密：两把钥匙，公钥公开，私钥保密

私钥签名 → 公钥验签（谁都能验证是不是你签的）
公钥加密 → 私钥解密（只有你能看）

你的私钥 → 签名的请求 → 支付宝用你的公钥验签
支付宝私钥 → 签名的通知 → 你用支付宝公钥验签
```

为什么叫"不可否认性"？

因为私钥只有你有。带合法签名的请求必然是你发出的，你不能事后说"我没发过这个请求"。

#### 第二层：幂等键

```python
# Redis SETNX 原子操作
# 只在键不存在时设置成功，返回 1
# 键已存在时设置失败，返回 0
idempotency_key = f"alipay:processed:{order_no}"
processed = await self.redis.setnx(idempotency_key, "1")
if not processed:
    return "success"  # 已处理过，跳过

await self.redis.expire(idempotency_key, 86400)  # 24小时 TTL
```

为什么用 `SETNX` 而不是 `GET + SET`？因为 `GET` 和 `SET` 是两步操作，中间可能有另一个请求也 `GET` 到空值，导致两个请求都去处理同一个订单。`SETNX` 是原子操作，只有一个请求能成功。

#### 第三层：金额校验

```python
notify_amount = Decimal(str(form_data.get("total_amount")))
order_amount = Decimal(str(order.total_amount))
if notify_amount != order_amount:
    await self.redis.delete(idempotency_key)  # 释放幂等键，允许重试
    return "failure"
```

用 `Decimal` 而不是 `float`：因为 `float` 有精度问题（0.1 + 0.2 != 0.3），金额比较必须精确。

#### 第四层：状态机保护

```python
# 支付前只允许 pending_payment 的订单
if order.status != "pending_payment":
    raise ValueError("订单状态不允许支付")

# 通知处理时也只接受 pending_payment
if order.status != "pending_payment":
    return "success"  # 已支付，不重复处理
```

订单状态机：

```
pending_payment → paid → refunded
                → cancelled
                → expired
```

每个状态转换都受代码保护，不能跳过中间状态。

### 2.3 攻击场景模拟

```
攻击者知道了你的 notify_url
  → 构造 POST 请求模拟支付宝通知
  → 第 1 层：没有支付宝私钥 → 签名验证失败 → ❌

攻击者录制了一条支付宝通知，原样重放
  → 第 1 层：签名合法 → ✅
  → 第 2 层：幂等键已存在 → ❌

攻击者截获通知，把 total_amount 从 9.90 改成 0.01
  → 第 1 层：签名是针对原内容签的 → 改后验签失败 → ❌
  
攻击者自己生成了私钥/公钥对伪造通知
  → 第 1 层：用的是支付宝公钥验签 → 你的公钥和支付宝的不匹配 → ❌
```

**除非支付宝私钥泄露，否则以上所有攻击路径都不成立。**

---

## 三、短信验证码原理

### 3.1 验证码生命周期

```
T+0s    用户点击"获取验证码"
         SMSService.send_code(phone)
           → 生成 6 位随机码（random.randint(100000, 999999)）
           → Redis SETEX key=sms:code:{phone}, TTL=300
           → 日志输出 "短信验证码: phone=138..., code=123456"

T+30s   用户输入验证码 → 提交表单
         SMSService.verify_code(phone, code)
           → Redis GET sms:code:{phone}
           → 取出 "123456" → 比对一致
           → Redis DELETE sms:code:{phone}
           → return True（一次性使用，验证后立即删除）

T+70s   用户再次点击"获取验证码"
         Redis TTL 检查 → key 不存在（60 秒限频期已过）
         → 重新生成验证码 → 正常发送

T+350s  Redis key 自动过期
          验证码失效，即使从未使用
```

### 3.2 频率限制原理

```python
key = f"sms:code:{phone}"
ttl = await self.redis.ttl(key)

# 如果 TTL > 240 秒 → 说明验证码发出不到 60 秒
if ttl > SMS_CODE_TTL - 60:  # 300 - 60 = 240
    remaining = ttl - 240
    raise ValueError(f"发送过于频繁，请 {remaining} 秒后再试")
```

巧妙之处：利用验证码 key 的 TTL 来判断时间，不需要额外的 Redis key。

```
验证码 TTL = 300 秒（5 分钟后过期）
限频窗口 = 60 秒（同一手机号 1 分钟内只能发一次）

逻辑：
  如果剩余 TTL > 240 → 说明 key 刚创建不久 → 距上次发送不到 60 秒
  如果剩余 TTL ≤ 240 → 说明已经过了 60 秒 → 可以再次发送
```

### 3.3 为什么验证码设 5 分钟过期？

- 太短（1 分钟）：用户刚收到还没输入就过期了
- 太长（30 分钟）：验证码泄露后长期有效，安全隐患大
- 5 分钟是行业标准：支付宝、微信、银行都是这个时间

### 3.4 为什么一次性使用？

```python
# 验证成功后立即删除
await self.redis.delete(key)
return True
```

防止验证码被重复使用。即使验证码被攻击者截获，也只能用一次。

---

## 四、实名认证原理

### 4.1 数据脱敏规则

| 字段 | 普通用户看到他人 | 管理员看到 | 超级管理员看到 |
|------|-----------------|-----------|--------------|
| 手机号 | 不可见 | 138\*\*\*\*1234 | 完整（需操作理由） |
| 真实姓名 | 不可见 | 张\* | 完整 |
| 身份证号 | 不可见 | 不可见 | 完整（加密传输） |
| IP 地址 | 不可见 | 192.168.\*.\* | 完整 |

### 4.2 脱敏实现

```python
@staticmethod
def mask_real_name(name: str | None) -> str | None:
    """张三 → 张*"""
    if not name:
        return None
    if len(name) <= 1:
        return name
    return name[0] + "*" * (len(name) - 1)

@staticmethod
def mask_id_number(id_number: str | None) -> str | None:
    """110101199001011234 → 110101********1234"""
    if not id_number or len(id_number) < 8:
        return id_number
    return id_number[:6] + "*" * (len(id_number) - 10) + id_number[-4:]
```

### 4.3 数据库加密

身份证号在数据库中 AES-256 加密存储：

```python
from cryptography.fernet import Fernet

key = base64.urlsafe_b64decode(settings.ENCRYPTION_KEY)
f = Fernet(key)

# 存储时加密
encrypted = f.encrypt(id_number.encode())

# 读取时解密
decrypted = f.decrypt(encrypted).decode()
```

加密密钥在 `.env` 中配置，不在代码里硬编码。即使数据库被拖走，攻击者也拿不到明文身份证号。

### 4.4 bcrypt 科普

用户密码使用 bcrypt 存储：

```
MD5:     普通 GPU 每秒可计算数十亿次 → 暴力破解极快
SHA256:  普通 GPU 每秒可计算数亿次  → 仍然很快
bcrypt:  单次验证约 100ms          → GPU 难以加速
```

bcrypt 自动加盐：即使两个用户密码相同，存储的 hash 值也不同。

---

## 五、开发环境 Mock 指南

### 5.1 Mock 开关总览

| API | 触发条件 | Mock 行为 |
|-----|---------|----------|
| 支付宝 | `ALIPAY_APP_PRIVATE_KEY_PATH` 未配置或文件不存在 | sign 填 `demo_signature`，支付 URL 正常生成 |
| 短信 | `SMS_MOCK_ENABLED=true` 或 `ENVIRONMENT=development` | 验证码打印到日志，不真实发送 |
| 实名认证 | `REALNAME_MOCK_ENABLED=true` 或 `ENVIRONMENT=development` | 本地校验身份证号格式，不调外部 API |

### 5.2 测试方法

#### 支付宝

```bash
# 1. 获取沙箱配置
#    去 https://open.alipay.com/develop/sandbox 注册
#    拿到沙箱版支付宝 APP + 沙箱买家账号

# 2. 启动 ngrok 暴露 notify_url
ngrok http 8000
# → https://xxxx.ngrok-free.app

# 3. 配置 notify_url
ALIPAY_NOTIFY_URL=https://xxxx.ngrok-free.app/api/v1/payments/alipay/notify

# 4. 正常下单支付，用沙箱版支付宝扫码
```

#### 短信

```bash
# 查看服务端日志
uvicorn app.main:app --reload
# 日志输出：
# 短信验证码: phone=13800138000, code=123456（开发环境请勿泄露）

# 在前端输入 123456 即可通过验证
```

#### 实名认证

测试身份证号自动通过：

| 姓名 | 身份证号 | 结果 |
|------|----------|------|
| 张三 | 110101199001011234 | 自动通过 |
| 李四 | 110101199003074477 | 自动通过 |
| 任意 | 格式正确 | Mock 通过 |

---

## 六、工具用法

### 6.1 OpenSSL — 生成支付宝密钥对

```bash
# 第一步：生成 2048 位 RSA 私钥
openssl genrsa -out app_private_key.pem 2048

# 第二步：从私钥导出公钥
openssl rsa -in app_private_key.pem -pubout -out app_public_key.pem

# 第三步：把 app_public_key.pem 上传到支付宝开放平台
# 第四步：从支付宝开放平台下载支付宝公钥 → 保存到服务器
```

注意：
- `app_private_key.pem` — 你的私钥，**绝对不要给任何人**（包括支付宝）
- `app_public_key.pem` — 上传到开放平台，支付宝用这个验证你的签名
- `alipay_public_key.pem` — 支付宝的公钥，你用来验证支付宝的通知签名
- 三个文件缺一不可，放错了支付流程会失败

### 6.2 Redis CLI — 调试第三方 API

```bash
# 连接到 Redis
redis-cli -h localhost -p 6379

# 查看短信验证码
GET sms:code:13800138000
# → "123456"

# 查看验证码剩余时间
TTL sms:code:13800138000
# → (integer) 245  # 还有 245 秒过期

# 查看支付幂等键
GET alipay:processed:ORD20250101001
# → "1"

# 手动删除测试数据（调试用）
DEL sms:code:13800138000
```

### 6.3 MySQL — 查看订单和数据

```bash
# 连接到 MySQL
mysql -h localhost -u root -p ticket_dev

# 查看订单状态
SELECT id, order_no, status, total_amount, created_at 
FROM orders 
ORDER BY created_at DESC 
LIMIT 10;

# 查看操作日志（支付通知记录）
SELECT * FROM operation_logs 
WHERE action LIKE '%alipay%' 
ORDER BY created_at DESC;
```

### 6.4 Docker — 管理基础设施

```bash
# 启动数据库和缓存
docker compose up -d

# 查看日志
docker compose logs -f mysql
docker compose logs -f redis

# 停止所有服务
docker compose down

# 重建（清空数据重新开始）
docker compose down -v
docker compose up -d

# 重新运行数据库迁移
alembic upgrade head
```

### 6.5 pytest — 测试支付宝等各种 API

```bash
# 跑所有单元测试
pytest -v

# 只跑支付相关测试
pytest -v -k "alipay or payment"

# 只跑短信相关测试
pytest -v -k "sms"

# 只跑实名认证测试
pytest -v -k "realname"

# 带覆盖率
pytest --cov=app --cov-report=term-missing -v
```

---

## 七、生产环境上线清单

### 7.1 配置检查

```env
# 生产环境必须填写的配置
ALIPAY_APP_ID=                 # 必填
ALIPAY_APP_PRIVATE_KEY_PATH=   # 必填
ALIPAY_PUBLIC_KEY_PATH=        # 必填
ALIPAY_NOTIFY_URL=             # 必填（公网可达）
ALIPAY_RETURN_URL=             # 必填
ENVIRONMENT=production         # 切换为生产模式
SECRET_KEY=                    # 修改为强随机字符串
```

### 7.2 Mock 开关检查

```env
SMS_MOCK_ENABLED=false         # 关闭 Mock
REALNAME_MOCK_ENABLED=false    # 关闭 Mock
```

### 7.3 安全检查清单

```
□ 支付宝私钥未提交到 git（.gitignore 已排除 *.pem）
□ 生产环境的 SECRET_KEY 已改为随机值（32 位以上）
□ 默认管理员密码已修改
□ CORS_ORIGINS 已限制为具体域名（不是 *）
□ HTTPS 已配置
□ notify_url 公网可达
□ 沙箱 APPID 已替换为生产 APPID
```

### 7.4 部署后验证

```bash
# 1. 健康检查
curl https://your-domain.com/api/v1/health

# 2. 触发一笔支付，确认异步通知能收到
#    在日志中搜索 "alipay_notify" 

# 3. 检查测试结果
pytest -v 2>&1 | grep -c "SKIP"
# 输出必须为 0（零跳过规则）

pytest -v 2>&1 | grep -c "FAILED"
# 输出必须为 0（零失败规则）
```
