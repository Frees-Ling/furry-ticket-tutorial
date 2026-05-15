# 第三方 API 参考

## 概述

所有第三方 API 集成统一在 `app/services/third_party_api.py` 中管理。
通过 `ThirdPartyAPIFacade` 单例访问，调用方无需关心具体实现和配置细节。

```python
from app.services.third_party_api import third_party
# 或
from app.services import third_party  # 通过 __init__.py 导出
```

### 设计原则

| 原则 | 说明 |
|------|------|
| 单一入口 | 所有第三方 API 均通过 `third_party` 对象调用 |
| 配置集中 | API Key/Secret 等配置在 `app/config.py` 统一管理 |
| 错误统一 | 所有外部 API 异常统一抛出 `ThirdPartyAPIError` |
| Mock 友好 | 开发环境自动启用 Mock 模式，无需真实 API 凭证 |
| 日志完整 | 每次调用记录调用方 + 参数 + 结果 |

### 支持列表

| API | 类名 | 用途 | 生产依赖 |
|-----|------|------|----------|
| 支付宝 | `AlipayAPI` | 支付签名/验签/表单 | 支付宝私钥 + 支付宝公钥 |
| 短信 | `SMSAPI` | 验证码/通知短信 | 阿里云短信 / Twilio 等 |
| 实名认证 | `RealNameAPI` | 身份证一致性校验 | 阿里云实人认证 / 国政通 |

---

## 支付宝 API

```python
third_party.alipay  # -> AlipayAPI 实例
```

### build_pay_form — 构建支付表单 URL

生成带 RSA2 签名的支付宝支付链接，前端直接跳转即可支付。

```python
from app.services import third_party

pay_url = third_party.alipay.build_pay_form({
    "out_trade_no": "ORD20250101123456",  # 商户订单号
    "total_amount": "99.00",              # 金额，字符串格式
    "subject": "票务-夏日兽聚",           # 商品标题
})
# 返回: "https://openapi.alipaydev.com/gateway.do?app_id=..."
```

**参数说明:**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| out_trade_no | string | 是 | 商户订单号，需唯一 |
| total_amount | string | 是 | 订单总金额，保留两位小数 |
| subject | string | 是 | 订单标题，会展示在支付宝收银台 |

**安全特性:**
- RSA2 (SHA256WithRSA) 签名，防篡改
- 动态时间戳，防重放攻击
- 沙箱/生产自动切换（通过 `settings.IS_PRODUCTION` 控制）

**异常:**
- `ThirdPartyAPIError`: 签名失败或生产环境私钥未配置

### verify_notification — 验证支付通知签名

支付宝异步通知到达后，必须先验签再更新订单。

```python
from app.services import third_party

# 通知处理流程
async def handle_alipay_notify(form_data: dict):
    sign = form_data.pop("sign", None)
    form_data.pop("sign_type", None)

    if sign is None:
        return "failure"

    verified = third_party.alipay.verify_notification(dict(form_data), sign)
    if not verified:
        return "failure"

    # 验签通过，处理订单...
    trade_status = form_data.get("trade_status")
    if trade_status == "TRADE_SUCCESS":
        order_no = form_data["out_trade_no"]
        trade_no = form_data["trade_no"]
        # 更新订单状态...

    return "success"
```

**安全要点:**
- **必须验签**：不验签 = 任何人可伪造支付成功通知
- 必须使用**支付宝公钥**（不是应用公钥！）
- 生产环境验签失败会返回 `False`

**配置要求:**

```env
# .env
ALIPAY_APP_ID=2021000123456789
ALIPAY_APP_PRIVATE_KEY_PATH=/etc/alipay/app_private_key.pem
ALIPAY_PUBLIC_KEY_PATH=/etc/alipay/alipay_public_key.pem
ALIPAY_NOTIFY_URL=https://your-domain.com/api/v1/payments/alipay/notify
ALIPAY_RETURN_URL=https://your-domain.com/orders
```

---

## 短信 API

```python
third_party.sms  # -> SMSAPI 实例
```

### send_verification_code — 发送验证码短信

```python
from app.services import third_party

# 开发环境（Mock，验证码打印到日志）
ok = await third_party.sms.send_verification_code(
    phone="13800138000",
    code="123456",
)

# 生产环境（传入 redis 后自动执行频率限制）
ok = await third_party.sms.send_verification_code(
    phone="13800138000",
    code="123456",
    redis=redis_client,
)
```

**参数说明:**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| phone | string | 是 | 目标手机号 |
| code | string | 是 | 6 位验证码 |
| redis | Redis\|None | 否 | 传入后自动检查发送频率（60 秒限制） |

**频率限制:**
- 同一手机号 60 秒内只能发送一次
- 超过频率限制抛出 `ThirdPartyAPIError`

### send_notification — 发送通知短信

```python
from app.services import third_party

ok = await third_party.sms.send_notification(
    phone="13800138000",
    message="您的订单已支付成功，订单号：ORD20250101123456",
)
```

---

## 实名认证 API

```python
third_party.realname  # -> RealNameAPI 实例
```

### verify — 执行实名认证

```python
from app.services import third_party

ok, message = await third_party.realname.verify(
    real_name="张三",
    id_number="110101199001011234",
    phone="13800138000",
)

if ok:
    print("认证通过")
else:
    print(f"认证失败: {message}")
```

**参数说明:**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| real_name | string | 是 | 真实姓名 |
| id_number | string | 是 | 身份证号 |
| phone | string | 是 | 手机号 |

**返回:**

```python
(True, "验证通过（Mock）")   # 成功
(False, "身份证号格式不正确") # 失败
```

**测试身份证号（开发环境自动通过）:**

| 身份证号 | 说明 |
|----------|------|
| 110101199001011234 | 测试用户 |
| 110101199003074477 | 测试用户 |

---

## 统一异常处理

```python
from app.services import third_party, ThirdPartyAPIError

try:
    pay_url = third_party.alipay.build_pay_form(params)
except ThirdPartyAPIError as e:
    return {"error": str(e)}

try:
    ok, msg = await third_party.realname.verify(name, id_no, phone)
except ThirdPartyAPIError as e:
    return {"error": "认证服务暂不可用，请稍后重试"}
```

---

## 生产环境配置

### 支付宝

```env
ALIPAY_APP_ID=你的APPID
ALIPAY_APP_PRIVATE_KEY_PATH=/path/to/private_key.pem
ALIPAY_PUBLIC_KEY_PATH=/path/to/alipay_public_key.pem
ALIPAY_NOTIFY_URL=https://your-domain.com/api/v1/payments/alipay/notify
ALIPAY_RETURN_URL=https://your-domain.com/orders
```

### 短信服务

```env
SMS_ACCESS_KEY=你的AccessKey
SMS_ACCESS_SECRET=你的AccessSecret
SMS_TEMPLATE_CODE=SMS_123456
```

### 实名认证

```env
REALNAME_API_URL=https://api.realname.com/verify
REALNAME_API_KEY=你的APIKey
REALNAME_API_SECRET=你的APISecret
REALNAME_MOCK_ENABLED=true
SMS_MOCK_ENABLED=true
```

---

## 配置项

| 配置项 | 对应 API | 说明 |
|--------|----------|------|
| `ALIPAY_APP_ID` | Alipay | 支付宝应用 ID |
| `ALIPAY_APP_PRIVATE_KEY_PATH` | Alipay | 应用私钥路径 |
| `ALIPAY_PUBLIC_KEY_PATH` | Alipay | 支付宝公钥路径 |
| `ALIPAY_NOTIFY_URL` | Alipay | 异步通知地址 |
| `ALIPAY_RETURN_URL` | Alipay | 同步跳转地址 |
| `IS_PRODUCTION` | 全部 | 切换沙箱/生产模式 |
| `REALNAME_MOCK_ENABLED` | RealName | 实名认证 Mock 开关 |
| `SMS_MOCK_ENABLED` | SMS | 短信 Mock 开关 |
| `ENVIRONMENT` | 全部 | 环境标识 |

---

## 迁移指南

### 旧方式（分散调用）

```python
from app.services.alipay_sdk import AlipaySDK
from app.services.sms_service import SMSService
from app.services.realname_service import RealNameService

sdk = AlipaySDK()
url = sdk.build_pay_form(params)

sms = SMSService(redis)
await sms.send_code(phone)

realname = RealNameService(session, redis)
await realname.verify(user_id, name, id_no)
```

### 新方式（统一入口）

```python
from app.services import third_party

url = third_party.alipay.build_pay_form(params)
await third_party.sms.send_verification_code(phone, code)
ok, msg = await third_party.realname.verify(name, id_no, phone)
```

**新旧混用没有问题** — 两种方式都支持，建议新代码统一使用新方式。
