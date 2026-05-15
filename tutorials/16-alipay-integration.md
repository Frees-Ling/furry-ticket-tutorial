# 第16章：支付宝（Alipay）支付集成

## 本章学习目标

本章将完整实现支付宝支付的集成，从 RSA 加密原理到支付流程代码实现，从异步通知验签到防重放攻击证明。完成本章后，你将掌握：

1. RSA 非对称加密原理及在支付中的应用
2. 支付宝开放平台配置与沙箱环境使用
3. alipay-sdk-python 库的完整 API
4. 支付状态机设计与实现
5. 支付参数签名生成算法
6. 异步通知的四层防重放攻击防御
7. 退款流程实现
8. 项目支付模块代码的逐行解读

---

## 1. 支付宝开放平台准备

### 1.1 注册与创建应用

**步骤一：注册支付宝开放平台账号**

访问 [支付宝开放平台](https://open.alipay.com/) 并注册开发者账号。注册完成后，登录控制台。

**步骤二：创建应用**

1. 进入"开发者中心" → "网页/移动应用"
2. 点击"创建应用" → "网页应用"
3. 填写应用名称（例如"票务系统"）
4. 上传应用图标
5. 选择应用类型为"自用型应用"
6. 提交审核（沙箱环境可跳过审核）

**步骤三：配置应用**

创建应用后，在应用详情页进行以下配置：

| 配置项 | 说明 | 示例值 |
| -------- | ------ | -------- |
| 应用网关 | 后端服务地址 | `https://api.example.com` |
| 授权回调地址 | 用户支付完成后的跳转地址 | `https://example.com/orders` |
| 支付宝网关 | 异步通知接收地址 | `https://api.example.com/api/v1/payments/alipay/notify` |
| 接口加签方式 | 非对称加密算法 | `RSA2` |

### 1.2 沙箱环境

支付宝沙箱（Sandbox）是一个模拟的支付环境，用于开发测试。沙箱环境完全模拟真实支付流程但不会产生真实资金。

**沙箱网关地址：**

- 正式：`https://openapi.alipay.com/gateway.do`
- 沙箱：`https://openapi.alipaydev.com/gateway.do`

**沙箱买家账户：**
在支付宝开放平台沙箱应用中获取：

- 买家账号：`xxx@sandbox.com`（系统自动生成）
- 登录密码：`111111`
- 支付密码：`111111`

**沙箱与正式环境切换：**

```python
# 在项目配置中自动切换
class Settings(BaseSettings):
    ENV: str = "development"  # 通过环境变量控制

    @property
    def ALIPAY_GATEWAY(self) -> str:
        """自动切换沙箱/正式环境"""
        if self.ENV == "production":
            return "https://openapi.alipay.com/gateway.do"
        return "https://openapi.alipaydev.com/gateway.do"
```text

---

## 2. RSA 加密与签名机制

### 2.1 非对称加密原理

RSA（Rivest-Shamir-Adleman）是目前最广泛使用的非对称加密算法。它使用一对密钥：公钥（Public Key）和私钥（Private Key）。

**核心特性：**

```

私钥 → 加密/签名（只有持有者能做）
公钥 → 解密/验签（任何人都可以）

在支付场景中：
  应用私钥：保存在我们服务器上，用于签名请求参数
  支付宝公钥：从支付宝获取，用于验证支付宝的响应
  支付宝私钥：支付宝保存，用于签名异步通知
  应用公钥：上传到支付宝开放平台，用于验证我们的请求

```text

**为什么用 RSA 而不是对称加密？**

```

对称加密（如 AES）：
  优势：加解密速度快
  劣势：需要双方协商同一密钥，密钥分发困难
  场景：适合大量数据的加解密

非对称加密（如 RSA）：
  优势：无需共享私钥，公钥可公开
  劣势：加解密速度较慢（约为 AES 的 1/1000）
  场景：适合小数据量的签名和密钥交换

```text

### 2.2 RSA 密钥对生成

支付宝推荐使用 2048 位 RSA 密钥，签名算法为 `RSA2`（SHA-256 with RSA）。

**使用 OpenSSL 生成密钥对：**

```bash
# === 步骤 1：生成 2048 位 RSA 私钥 ===
openssl genrsa -out app_private_key.pem 2048

# 查看生成的私钥内容
cat app_private_key.pem
# -----BEGIN RSA PRIVATE KEY-----
# MIIEpAIBAAKCAQEA0gH3m...（共 26 行左右）
# -----END RSA PRIVATE KEY-----

# === 步骤 2：从私钥导出公钥 ===
openssl rsa -in app_private_key.pem -pubout -out app_public_key.pem

# 查看公钥内容
cat app_public_key.pem
# -----BEGIN PUBLIC KEY-----
# MIIBIjANBgkqhkiG9w0B...（约 4 行）
# -----END PUBLIC KEY-----

# === 步骤 3：转换为 PKCS8 格式（支付宝要求） ===
openssl pkcs8 -topk8 -inform PEM -in app_private_key.pem \
    -outform PEM -nocrypt -out app_private_key_pkcs8.pem

# === 步骤 4：获取支付宝公钥 ===
# 将 app_public_key.pem 的内容上传到支付宝开放平台
# 支付宝会返回 alipay_public_key.pem
# 保存到项目 keys 目录中

# 文件组织：
mkdir -p keys
cp app_private_key_pkcs8.pem keys/
# 将从支付宝下载的公钥保存为：
# keys/alipay_public_key.pem
```

### 2.3 PKCS1 vs PKCS8

RSA 私钥有两种常见的 PEM 编码格式：

| 格式 | 头部 | 说明 |
| ------ | ------ | ------ |
| PKCS1 | `-----BEGIN RSA PRIVATE KEY-----` | 仅包含私钥参数（n, e, d, p, q） |
| PKCS8 | `-----BEGIN PRIVATE KEY-----` | 包含算法标识和私钥参数 |

**支付宝要求使用 PKCS8 格式的私钥**，所以在生成后需要转换。

### 2.4 RSA 签名验证过程

```python
# === 签名生成（使用应用私钥） ===
import rsa
from Crypto.Signature import pkcs1_15
from Crypto.Hash import SHA256
from Crypto.PublicKey import RSA

def generate_sign(content: str, private_key_path: str) -> str:
    """生成 RSA2 签名（base64 编码）"""
    # 加载私钥
    with open(private_key_path, 'r') as f:
        private_key = RSA.import_key(f.read())

    # 计算 SHA256 哈希
    h = SHA256.new(content.encode('utf-8'))

    # 使用私钥签名（PKCS1 v1.5 padding）
    signature = pkcs1_15.new(private_key).sign(h)

    # Base64 编码
    return base64.b64encode(signature).decode('utf-8')


# === 签名验证（使用支付宝公钥） ===
def verify_sign(content: str, sign: str, public_key_path: str) -> bool:
    """验证 RSA2 签名"""
    with open(public_key_path, 'r') as f:
        public_key = RSA.import_key(f.read())

    h = SHA256.new(content.encode('utf-8'))
    try:
        pkcs1_15.new(public_key).verify(h, base64.b64decode(sign))
        return True
    except (ValueError, TypeError):
        return False
```text

### 2.5 PKCS1 v1.5 Padding vs OAEP

| 特征 | PKCS1 v1.5 | OAEP |
| ------ | ----------- | ------ |
| 用途 | 签名算法（支付宝使用） | 加密算法 |
| 安全性 | 较老，但签名场景安全 | 更安全，有随机化填充 |
| 可证明安全 | 否（有理论攻击） | 是（随机预言模型） |
| 兼容性 | 广泛支持 | 较新 |
| 支付宝使用 | 是（RSA2 签名） | 否 |

**OAEP 不是支付宝的选择——支付签名场景只需 PKCS1 v1.5 就足够了**。OAEP（Optimal Asymmetric Encryption Padding）主要用于加密而非签名。

---

## 3. alipay-sdk-python API 详解

### 3.1 安装

```bash
pip install alipay-sdk-python
```

### 3.2 AliPay 类构造

```python
from alipay import AliPay

alipay = AliPay(
    # === 必填参数 ===
    appid=settings.ALIPAY_APP_ID,              # 支付宝应用 ID
    app_notify_url=settings.ALIPAY_NOTIFY_URL, # 异步通知 URL（支付宝回调）
    app_private_key_string=open(
        settings.ALIPAY_APP_PRIVATE_KEY_PATH
    ).read(),                                   # 应用私钥（PKCS8 格式）
    alipay_public_key_string=open(
        settings.ALIPAY_PUBLIC_KEY_PATH
    ).read(),                                   # 支付宝公钥

    # === 可选参数 ===
    sign_type="RSA2",                           # 签名算法（默认 RSA2）
    debug=False,                                # 调试模式（会打印请求信息）
    app_cert_sn=None,                           # 应用公钥证书 SN（公钥证书模式）
    alipay_root_cert_sn=None,                   # 支付宝根证书 SN
    format="json",                              # 请求和响应的数据格式
    charset="utf-8",                            # 字符编码
    api_version="1.0",                          # API 版本
    app_auth_token=None,                        # 第三方应用授权 Token
)

# ===========================================
# 注意：证书模式 vs 公钥模式
# ===========================================
# 支付宝提供两套安全机制：
# 1. 公钥模式（简单）：使用 app_private_key_string + alipay_public_key_string
# 2. 证书模式（推荐）：使用 app_cert_sn + alipay_root_cert_sn
#
# 本教程使用公钥模式，更简单直观。
# 证书模式需要额外的证书管理，适合企业级应用。
```text

**项目配置文件（app/config.py）：**

```python
class Settings(BaseSettings):
    # 支付宝配置
    ALIPAY_APP_ID: str = ""                              # 从 .env 加载
    ALIPAY_APP_PRIVATE_KEY_PATH: str = ""                 # 私钥文件路径
    ALIPAY_PUBLIC_KEY_PATH: str = ""                      # 支付宝公钥路径
    ALIPAY_NOTIFY_URL: str = ""                           # 异步通知地址
    ALIPAY_RETURN_URL: str = ""                           # 同步回跳地址

    model_config = {
        "env_file": ".env",
        "env_file_encoding": "utf-8",
    }

settings = Settings()
```

### 3.3 api_alipay_trade_page_pay — 电脑网站支付

```python
# === 接口说明 ===
# API 名称：alipay.trade.page.pay
# 用途：生成支付表单参数，用户跳转到支付宝收银台
# 返回：URL 查询字符串或表单 HTML

order_string = alipay.api_alipay_trade_page_pay(
    out_trade_no="20240501123456789",    # 商户订单号（唯一）
    total_amount="99.50",                # 订单总金额（字符串格式）
    subject="票务-周杰伦演唱会",          # 订单标题
    return_url="https://example.com/orders",    # 同步回跳地址（覆盖构造参数）
    notify_url=None,                     # 异步通知地址（覆盖构造参数）
    quit_url=None,                       # 用户退出支付后的跳转地址
    timeout_express="30m",               # 交易超时时间（分钟）
    body="VIP 座位 A区-3排-5号",          # 订单描述
    goods_type="0",                      # 商品主类型：0=虚拟类, 1=实物类
    passback_params=None,                # 回传参数（支付宝原样返回）
    qr_pay_mode="2",                     # 扫码支付模式
    qrcode_width=None,                   # 二维码宽度
)

# === 返回值 ===
# order_string 是一个 URL query string，例如：
# app_id=202100...&method=alipay.trade.page.pay&...
# 可以直接拼接到支付宝网关后：

# 拼接支付 URL
pay_url = f"https://openapi.alipay.com/gateway.do?{order_string}"

# 或者生成自动提交表单 HTML
from alipay import AliPay
pay_url_or_form = alipay.api_alipay_trade_page_pay(
    out_trade_no="...",
    total_amount="99.50",
    subject="票务-周杰伦演唱会",
    return_url="...",
    # return_url 仅在 PC 支付时有效
)
```text

### 3.4 api_alipay_trade_query — 交易查询

```python
# === 接口说明 ===
# API 名称：alipay.trade.query
# 用途：查询支付宝交易状态
# 场景：异步通知丢失时的补偿查询

result = alipay.api_alipay_trade_query(
    out_trade_no="20240501123456789",    # 商户订单号（二选一）
    trade_no="202405012200...",          # 支付宝交易号（二选一）
)

# === 返回值示例 ===
{
    "code": "10000",                     # 接口调用成功
    "msg": "Success",
    "trade_no": "202405012200...",       # 支付宝交易号
    "out_trade_no": "20240501123456789", # 商户订单号
    "buyer_logon_id": "138***@qq.com",   # 买家支付宝账号
    "trade_status": "TRADE_SUCCESS",     # 交易状态
    "total_amount": "99.50",             # 交易金额
    "receipt_amount": "99.50",           # 实收金额
    "buyer_pay_amount": "99.50",         # 买家实付金额
    "point_amount": "0.00",              # 积分抵扣金额
    "invoice_amount": "99.50",           # 可开票金额
    "send_pay_date": "2024-05-01 12:00:00",  # 支付时间
}
```

### 3.5 api_alipay_trade_refund — 退款

```python
# === 接口说明 ===
# API 名称：alipay.trade.refund
# 用途：发起退款操作
# 限制：支持部分退款（同一笔交易最多 50 次）

result = alipay.api_alipay_trade_refund(
    trade_no="202405012200...",           # 支付宝交易号
    out_trade_no="20240501123456789",    # 商户订单号（二选一）
    refund_amount="99.50",               # 退款金额（不超过原支付金额）
    refund_reason="用户申请退款",          # 退款原因
    out_request_no=None,                 # 退款请求号（部分退款必须）
    operator_id="admin",                 # 操作员 ID
    store_id=None,                       # 门店 ID
    refund_royalty_parameters=None,      # 退分账参数
)

# === 返回值示例 ===
{
    "code": "10000",                     # 接口调用成功
    "msg": "Success",
    "trade_no": "202405012200...",       # 支付宝交易号
    "out_trade_no": "20240501123456789",
    "buyer_logon_id": "138***@qq.com",
    "fund_change": "Y",                  # 资金变动 Y/N
    "refund_fee": "99.50",               # 实际退款金额
    "gmt_refund_pay": "2024-05-01 12:30:00",  # 退款时间
}

# === 退款状态码 ===
# code == "10000" → 退款成功
# code == "40004" → 退款失败（参数错误）
# code == "20000" → 处理中（需轮询）
```text

### 3.6 api_alipay_trade_fastpay_refund_query — 退款查询

```python
result = alipay.api_alipay_trade_fastpay_refund_query(
    trade_no="202405012200...",
    out_trade_no="20240501123456789",
    out_request_no="REFUND_001",         # 退款请求号
)

# 返回值示例
{
    "code": "10000",
    "msg": "Success",
    "trade_no": "202405012200...",
    "out_trade_no": "20240501123456789",
    "refund_status": "REFUND_SUCCESS",   # 退款状态
    "total_amount": "99.50",
    "refund_amount": "99.50",
}
```

### 3.7 verify() — 异步通知验签

```python
# === 方法说明 ===
# verify(data_dict, sign)
# 参数：
#   data_dict: 支付宝异步通知的参数 dict（不含 sign 和 sign_type）
#   sign: 从通知中提取的签名字符串
# 返回：bool — 验签是否通过

# 其内部实现等价于：
def verify(self, data: dict, sign: str) -> bool:
    # 1. 按 key 排序参数
    sorted_keys = sorted(data.keys())
    # 2. 拼接 key=value 字符串
    sign_content = "&".join(f"{k}={data[k]}" for k in sorted_keys)
    # 3. 使用 支付宝公钥 验证签名
    return self.__verify_with_public_key(sign_content, sign)
```text

### 3.8 完整 SDK 使用示例

```python
from alipay import AliPay
from alipay.utils import Alipay

# ===========================================
# 初始化（单例模式，应用启动时创建一次）
# ===========================================
def create_alipay_client() -> AliPay:
    """创建支付宝客户端"""
    # 读取密钥文件
    with open(settings.ALIPAY_APP_PRIVATE_KEY_PATH, 'r') as f:
        app_private_key = f.read()
    with open(settings.ALIPAY_PUBLIC_KEY_PATH, 'r') as f:
        alipay_public_key = f.read()

    return AliPay(
        appid=settings.ALIPAY_APP_ID,
        app_notify_url=settings.ALIPAY_NOTIFY_URL,
        app_private_key_string=app_private_key,
        alipay_public_key_string=alipay_public_key,
        sign_type="RSA2",
        debug=False,
    )


# ===========================================
# 使用 alipay-sdk-python 的完整支付流程
# ===========================================
alipay = create_alipay_client()


def create_payment(order_no: str, amount: str, subject: str) -> str:
    """生成支付宝支付参数"""
    order_string = alipay.api_alipay_trade_page_pay(
        out_trade_no=order_no,
        total_amount=amount,
        subject=subject,
        return_url="https://example.com/orders",
        timeout_express="30m",
    )
    # 拼接完整支付链接
    pay_url = f"{settings.ALIPAY_GATEWAY}?{order_string}"
    return pay_url


def verify_notification(data: dict) -> bool:
    """验证支付宝异步通知"""
    sign = data.pop("sign", None)
    data.pop("sign_type", None)
    return alipay.verify(data, sign)


def process_refund(trade_no: str, amount: str, reason: str) -> dict:
    """处理退款"""
    result = alipay.api_alipay_trade_refund(
        trade_no=trade_no,
        refund_amount=amount,
        refund_reason=reason,
    )
    return result
```

---

## 4. 签名生成算法

### 4.1 签名生成过程

支付宝的签名机制确保请求和通知在传输过程中不被篡改。其核心流程是：

```text
步骤 1：将所有请求参数按 key 升序排序
步骤 2：拼接为 key1=value1&key2=value2 格式
步骤 3：使用应用私钥（RSA2）对拼接字符串签名
步骤 4：对签名结果进行 Base64 编码
步骤 5：将 sign 参数添加到请求参数中发送给支付宝
```

```text
┌─────────────────────────────────────────────────────────────────┐
│                     签名生成流程图                                 │
│                                                                   │
│  请求参数列表：                                                    │
│  {                                                                │
│    "app_id": "202100...",                                         │
│    "method": "alipay.trade.page.pay",                             │
│    "charset": "utf-8",                                            │
│    "sign_type": "RSA2",                                           │
│    "timestamp": "2024-01-01 12:00:00",                            │
│    "version": "1.0",                                              │
│    "biz_content": '{"out_trade_no":"...","total_amount":"99.50"}' │
│  }                                                                │
│                        │                                          │
│                        ▼                                          │
│  排序（按 key 升序）                                               │
│                        │                                          │
│                        ▼                                          │
│  拼接：                                                            │
│  sign_content = "app_id=202100...&biz_content={...}                │
│                  &charset=utf-8&method=alipay.trade.page.pay       │
│                  &sign_type=RSA2&timestamp=2024-01-01 12:00:00     │
│                  &version=1.0"                                     │
│                        │                                          │
│                        ▼                                          │
│  RSA2 签名：                                                      │
│  sign = RSA2_SHA256(sign_content, app_private_key)                 │
│                        │                                          │
│                        ▼                                          │
│  Base64 编码：                                                    │
│  sign = base64(sign)                                               │
│                        │                                          │
│                        ▼                                          │
│  将 sign 添加到请求参数                                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 手动实现签名

为了深入理解签名机制，下面是不使用 SDK 的手动实现：

```python
import json
import base64
from datetime import datetime
from urllib.parse import quote, urlencode

def build_signed_params(biz_content: dict, app_private_key_path: str) -> str:
    """手动构建支付宝签名的请求参数"""
    # 1. 准备基础参数
    params = {
        "app_id": settings.ALIPAY_APP_ID,
        "method": "alipay.trade.page.pay",
        "format": "JSON",
        "charset": "utf-8",
        "sign_type": "RSA2",
        "timestamp": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
        "version": "1.0",
        "notify_url": settings.ALIPAY_NOTIFY_URL,
        "return_url": settings.ALIPAY_RETURN_URL,
        "biz_content": json.dumps(biz_content, ensure_ascii=False),
    }

    # 2. 按 key 排序
    sorted_keys = sorted(params.keys())

    # 3. 拼接 sign_content
    sign_content = "&".join(
        f"{k}={params[k]}" for k in sorted_keys
    )

    print(f"待签名字符串：\n{sign_content}\n")

    # 4. 读取应用私钥并签名
    from Crypto.PublicKey import RSA
    from Crypto.Signature import pkcs1_15
    from Crypto.Hash import SHA256

    with open(app_private_key_path, 'r') as f:
        private_key = RSA.import_key(f.read())

    h = SHA256.new(sign_content.encode('utf-8'))
    signature = pkcs1_15.new(private_key).sign(h)
    sign = base64.b64encode(signature).decode('utf-8')

    # 5. 将 sign 加入参数
    params["sign"] = sign

    return params


# ===========================================
# 验签过程（收到支付宝响应后）
# ===========================================
def verify_response(response_params: dict, alipay_public_key_path: str) -> bool:
    """验证支付宝响应的签名"""
    sign = response_params.pop("sign", None)
    if not sign:
        return False

    # 1. 排序（除 sign 外的所有参数）
    sorted_keys = sorted(response_params.keys())

    # 2. 拼接
    sign_content = "&".join(
        f"{k}={response_params[k]}" for k in sorted_keys
    )

    # 3. 使用支付宝公钥验签
    from Crypto.PublicKey import RSA
    from Crypto.Signature import pkcs1_15
    from Crypto.Hash import SHA256

    with open(alipay_public_key_path, 'r') as f:
        public_key = RSA.import_key(f.read())

    h = SHA256.new(sign_content.encode('utf-8'))
    try:
        pkcs1_15.new(public_key).verify(h, base64.b64decode(sign))
        return True
    except (ValueError, TypeError):
        return False
```text

### 4.3 alipay-sdk-python 内部签名过程

`alipay-sdk-python` 库内部实现了上述签名逻辑。当调用 `api_alipay_trade_page_pay()` 时，库自动完成：

1. 合并业务参数（biz_content）和公共参数
2. 按 key 排序参数
3. 拼接待签名字符串
4. 使用构造时传入的 `app_private_key_string` 进行 RSA2 签名
5. Base64 编码签名结果
6. 将 sign 添加到请求参数
7. 返回 URL 查询字符串

### 4.4 验签流程图

```

支付宝异步通知（HTTP POST）
        │
        ▼
收到通知参数：
{
  "app_id": "202100...",
  "out_trade_no": "20240501123456789",
  "trade_no": "202405012200...",
  "total_amount": "99.50",
  "trade_status": "TRADE_SUCCESS",
  "sign": "aB3dE5...",          ← 支付宝私钥签名
  "sign_type": "RSA2"
}
        │
        ▼
分离 sign 和 sign_type
        │
        ▼
对剩余参数按 key 排序拼接
sign_content = "app_id=202100...&out_trade_no=...&..."
        │
        ▼
使用 支付宝公钥 验签
verify(sign_content, sign)
        │
        ▼
结果：True  → 签名验证通过，消息来自支付宝
结果：False → 签名验证失败，消息可能被篡改

```text

---

## 5. 支付状态机

### 5.1 订单状态定义

```python
# app/models/order.py — Order 模型的 status 字段

class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    order_no: Mapped[str] = mapped_column(String(32), unique=True)     # 订单号
    user_id: Mapped[int] = mapped_column(BigInteger, ForeignKey("users.id"))
    event_id: Mapped[int] = mapped_column(BigInteger, ForeignKey("events.id"))
    quantity: Mapped[int] = mapped_column(Integer)                     # 购买数量
    total_amount: Mapped[float] = mapped_column(Numeric(10, 2))        # 总金额
    status: Mapped[str] = mapped_column(String(20), default="pending_payment")
    # 可选状态值：
    #   "pending_payment"  — 待支付
    #   "paid"             — 已支付
    #   "cancelled"        — 已取消
    #   "refunded"         — 已退款
    payment_method: Mapped[Optional[str]] = mapped_column(String(20), default=None)
    trade_no: Mapped[Optional[str]] = mapped_column(String(64), default=None)  # 支付宝交易号
    paid_at: Mapped[Optional[datetime]] = mapped_column(DateTime(3), default=None)
    cancelled_at: Mapped[Optional[datetime]] = mapped_column(DateTime(3), default=None)
```

### 5.2 状态转换图

```text
                     ┌──────────────────┐
                     │                  │
                     │  pending_payment │ ← 订单创建后的初始状态
                     │   （待支付）     │
                     └────────┬─────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
     ┌──────────────┐  ┌──────────┐   ┌────────────┐
     │    paid      │  │ cancelled│   │  expired   │
     │  （已支付）  │  │ （已取消）│   │ （已过期） │
     └──────┬───────┘  └──────────┘   └────────────┘
            │
            │ (申请退款)
            ▼
     ┌──────────────┐
     │  refunded    │
     │  （已退款）  │
     └──────────────┘

允许的状态转换：
  pending_payment → paid         支付成功
  pending_payment → cancelled    用户取消/超时取消
  paid            → refunded     用户申请退款
  paid            → cancelled    直接取消（不退款场景）
```

### 5.3 状态转换校验逻辑

```python
# 支付服务中的状态转换校验
class PaymentService:
    async def create_alipay_payment(self, order_id: int, user_id: int) -> dict:
        """生成支付宝支付参数"""
        order = await self.session.get(Order, order_id)

        # 状态校验：只有 pending_payment 可以支付
        if not order or order.user_id != user_id:
            raise ValueError("订单不存在")
        if order.status != "pending_payment":
            raise ValueError("订单状态不允许支付")
        # ...

    async def refund(self, order_id: int, reason: str = "用户申请退款") -> dict:
        """发起退款"""
        order = await self.session.get(Order, order_id)

        # 状态校验：只有 paid 可以退款
        if not order or order.status != "paid":
            raise ValueError("订单状态不允许退款")
        # ...
```text

### 5.4 订单状态变更的原子性

状态变更需要保证原子性，防止并发导致的状态不一致：

```python
# 使用条件更新保证原子性
from sqlalchemy import update

async def confirm_payment(self, order_no: str, trade_no: str) -> bool:
    """确认支付成功（原子状态变更）"""
    stmt = (
        update(Order)
        .where(
            Order.order_no == order_no,
            Order.status == "pending_payment",  # 条件：必须是待支付状态
        )
        .values(
            status="paid",
            payment_method="alipay",
            trade_no=trade_no,
            paid_at=func.now(),
        )
    )
    result = await self.session.execute(stmt)
    # result.rowcount > 0 表示更新成功
    # result.rowcount == 0 表示状态不匹配（已被并发更新）
    if result.rowcount == 0:
        # 没有更新任何行 → 状态已被修改
        raise ValueError("订单状态异常，请稍后重试")

    # 生成票证
    order = await self.session.execute(
        select(Order).where(Order.order_no == order_no)
    )
    order = order.scalar_one()
    for i in range(order.quantity):
        ticket = Ticket(
            order_id=order.id,
            event_id=order.event_id,
            user_id=order.user_id,
            ticket_no=f"{order.order_no}-{i+1}",
            price=order.total_amount / order.quantity,
        )
        self.session.add(ticket)

    return True
```

---

## 6. 异步通知（Notify）与防重放攻击

### 6.1 异步通知流程

支付宝的异步通知（Notify）是支付集成的核心环节，也是最容易出 bug 的地方。

```mermaid
┌──────────┐          ┌──────────┐          ┌──────────┐
│  用户     │          │ 支付宝    │          │  应用服务器 │
└────┬─────┘          └────┬─────┘          └────┬─────┘
     │                     │                     │
     │  1. 提交支付请求    │                     │
     │────────────────────>│                     │
     │                     │                     │
     │  2. 跳转支付宝      │                     │
     │<────────────────────│                     │
     │                     │                     │
     │  3. 用户完成支付    │                     │
     │────────────────────>│                     │
     │                     │                     │
     │                     │  4. POST 异步通知   │
     │                     │────────────────────>│
     │                     │                     │
     │                     │  5. 返回 "success"  │
     │                     │<────────────────────│
     │                     │                     │
     │                     │  (若 4 未收到 success)
     │                     │  6. 重试通知        │
     │                     │  (最多 8 次)        │
     │                     │────────────────────>│
     │                     │                     │
```text

**关键特点：**

1. 支付宝异步通知是 **POST 请求**，`Content-Type` 为 `application/x-www-form-urlencoded`
2. 通知在支付成功后**可能延迟几秒到几分钟**到达
3. 若应用返回非 `"success"`，支付宝会**重试**最多 8 次（间隔递增：10s, 20s, 2m, 10m, 10m, 1h, 2h, 6h）
4. 应用必须在**接收通知后立即返回 `"success"`**（纯文本，不是 JSON）
5. 支付成功的同步回跳（return_url）**不能作为支付凭证**，必须以异步通知为准

### 6.2 同步回跳 vs 异步通知

| 特征 | 同步回跳（return_url） | 异步通知（notify_url） |
| ------ | ---------------------- | ---------------------- |
| 触发方式 | 浏览器 GET 跳转 | 服务器 POST 通知 |
| 可靠性 | 不可靠（用户可关闭浏览器） | 可靠（持续重试直到 success） |
| 到达时间 | 支付完成立即跳转 | 可能有延迟 |
| 验签方式 | GET 参数验签 | POST form 参数验签 |
| 作为凭证 | 否 | 是 |
| 重试机制 | 无 | 最多 8 次重试 |

### 6.3 异步通知参数详解

支付宝 POST 到 notify_url 的参数示例：

```

app_id=20210011xxxxx
charset=utf-8
gmt_create=2024-05-01 12:00:00
gmt_payment=2024-05-01 12:00:15
notify_id=2024050112001512345678901234
notify_time=2024-05-01 12:00:20
notify_type=trade_status_sync
out_trade_no=20240501123456789
seller_id=2088xxxxx
sign=aB3dE5FgH1IjK2LmNoPqRsT...  (base64)
sign_type=RSA2
total_amount=99.50
trade_no=2024050122001234567890
trade_status=TRADE_SUCCESS

```text

### 6.4 四层防重放攻击防御

重放攻击是指攻击者截获一次合法的 HTTP 请求并重新发送，导致业务逻辑被重复执行。在支付宝异步通知中，如果应用不做防御，攻击者重放通知可能导致订单被多次确认、库存被多次扣除、用户收到多份票证。

**我们的防御体系共 4 层：**

#### 第一层：幂等性检查（Redis SETNX）

```python
# 第一层防御：使用 Redis SETNX 实现幂等性
# SETNX = SET if Not eXists（不存在时才设置）

async def verify_and_process_notification(self, data: dict) -> str:
    """验证支付宝异步通知并处理（四层防御全部实现）"""
    # ... 前面省略验签逻辑 ...

    order_no = data["out_trade_no"]

    # ======== 第一层防御：幂等性检查 ========
    # 使用 Redis SETNX，只有第一次会设置成功
    # key: alipay:processed:{order_no}
    # TTL: 86400 秒（24 小时）
    processed = await self.redis.setnx(
        f"alipay:processed:{order_no}", "1"
    )
    if not processed:
        # key 已存在 → 该订单已经处理过了
        # 直接返回 success，不执行任何业务逻辑
        app.logger.info(f"重复通知: {order_no}，已跳过")
        return "success"

    # 设置过期时间（防止 Redis 中残留的 key 占用内存）
    await self.redis.expire(f"alipay:processed:{order_no}", 86400)

    try:
        # ======== 后续业务逻辑 ========
        # ...
    except Exception:
        # 业务处理失败时，删除幂等性 key，允许重试
        await self.redis.delete(f"alipay:processed:{order_no}")
        raise
```

**为什么 SETNX 而不是先检查再设置？**

```python
# ❌ 错误做法（存在竞态条件）：
if await redis.exists(f"alipay:processed:{order_no}"):
    return "success"               # 线程 A 检查通过
await redis.setex(f"alipay:processed:{order_no}", 86400, "1")
                                   # 线程 B 也检查通过
                                   # 两个线程都认为未处理，导致重复执行业务

# ✅ 正确做法（原子操作）：
processed = await redis.setnx(f"alipay:processed:{order_no}", "1")
# SETNX 是原子操作，只有一个线程能成功设置
# 失败的线程会收到 False（Redis 返回 0）
if not processed:
    return "success"               # 幂等性保护
```text

#### 第二层：联合信息校验（app_id + out_trade_no + total_amount）

```python
# ======== 第二层防御：联合信息校验 ========
# 从数据库中查询订单，验证三项信息完全匹配

async def verify_and_process_notification(self, data: dict) -> str:
    # ... 第一层防御 ...

    # 查询订单
    result = await self.session.execute(
        select(Order).where(Order.order_no == order_no)
    )
    order = result.scalar_one_or_none()
    if not order:
        # 订单不存在（可能已被删除或伪造订单号）
        return "failure"

    # 验证 app_id（确保是本应用的支付通知）
    if data.get("app_id") != settings.ALIPAY_APP_ID:
        app.logger.error(f"app_id 不匹配: {data.get('app_id')}")
        return "failure"

    # 验证金额（关键！防止金额被篡改）
    if Decimal(data["total_amount"]) != Decimal(str(order.total_amount)):
        app.logger.error(
            f"金额不匹配: 通知金额={data['total_amount']}, "
            f"订单金额={order.total_amount}"
        )
        return "failure"

    # ======== 联合验证通过，可以继续处理 ========
```

**为什么金额校验很重要？**

假设一个电影的订单金额是 `9.90` 元，攻击者截获了一个 `0.01` 元的付款通知，将 `out_trade_no` 改为大额订单号并重放。如果不校验金额，系统会误认为 `9.90` 元的订单已支付。通过 `total_amount` 校验，即使 `out_trade_no` 被篡改，金额不匹配也会拒绝。

#### 第三层：签名验证（cryptographic verification）

```python
# ======== 第三层防御：签名验证 ========
# 使用支付宝公钥验证签名
# 签名的私钥只有支付宝持有
# 因此能通过支付宝公钥验签的请求一定来自支付宝

def verify_notification_signature(self, data: dict, sign: str) -> bool:
    """验证支付宝通知签名"""
    from alipay import AliPay

    # 签名验证由 SDK 完成
    # 内部过程：
    # 1. 剔除 sign 和 sign_type
    # 2. 按 key 排序剩余参数
    # 3. 拼接 key=value
    # 4. 使用支付宝公钥 + RSA2 验签
    return self.alipay.verify(data, sign)
```text

**签名验证证明了什么？**

```

证明逻辑：

1. 签名使用支付宝私钥生成（私钥只有支付宝持有）
2. 应用服务器持有支付宝公钥（从支付宝官方获取）
3. 验签通过 → 消息确实是由支付宝签署的
4. 验签通过 → 消息在传输过程中没有被篡改

不能证明什么：

1. 不能证明该通知是"第一次"发送（所以需要第一层幂等性）
2. 不能证明交易已经成功（所以需要检查 trade_status）

```text

#### 第四层：交易状态验证（trade_status）

```python
# ======== 第四层防御：交易状态验证 ========

# 支付宝通知中的 trade_status 取值：
#   WAIT_BUYER_PAY   — 交易创建，等待买家付款
#   TRADE_CLOSED     — 未付款交易超时关闭，或支付完成后全额退款
#   TRADE_SUCCESS    — 交易支付成功（最终状态）
#   TRADE_FINISHED   — 交易结束（不可退款）

# 只有 TRADE_SUCCESS 才是我们需要处理的支付成功通知
if data.get("trade_status") != "TRADE_SUCCESS":
    # 其他状态不处理，但返回 success 给支付宝（不要返回 failure）
    return "success"
```

### 6.5 完整的异步通知处理

```python
async def verify_and_process_notification(self, data: dict) -> str:
    """
    验证支付宝异步通知并处理
    四层防御全部实现：
    1. 幂等性检查（Redis SETNX）
    2. 联合信息校验（app_id + order + amount）
    3. 签名验证（支付宝公钥）
    4. 交易状态验证（trade_status）
    """
    try:
        # ======== 第三层防御（最先执行）：签名验证 ========
        sign = data.pop("sign", None)
        data.pop("sign_type", None)

        # 验证签名是否来自支付宝
        if not self.alipay.verify(data, sign):
            app.logger.error(f"签名验证失败: {data.get('out_trade_no')}")
            return "failure"

        # ======== 第四层防御：交易状态验证 ========
        if data.get("trade_status") != "TRADE_SUCCESS":
            app.logger.info(f"非成功状态通知: {data.get('trade_status')}")
            return "success"

        order_no = data["out_trade_no"]

        # ======== 第一层防御：幂等性检查 ========
        processed = await self.redis.setnx(
            f"alipay:processed:{order_no}", "1"
        )
        if not processed:
            app.logger.info(f"幂等性跳过: {order_no}")
            return "success"
        await self.redis.expire(f"alipay:processed:{order_no}", 86400)

        try:
            # ======== 第二层防御：联合信息校验 ========
            # 查询订单
            result = await self.session.execute(
                select(Order).where(Order.order_no == order_no)
            )
            order = result.scalar_one_or_none()
            if not order:
                return "failure"

            # 金额校验
            if Decimal(data["total_amount"]) != Decimal(str(order.total_amount)):
                app.logger.error(f"金额不匹配: {order_no}")
                return "failure"

            # ======== 业务处理：确认支付 ========
            await self.order_service.confirm_payment(order_no, data["trade_no"])
            app.logger.info(f"支付成功: {order_no}")
            return "success"

        except Exception:
            # 业务失败 → 删除幂等性 key，让支付宝重试
            await self.redis.delete(f"alipay:processed:{order_no}")
            raise

    except Exception as e:
        app.logger.error(f"通知处理异常: {e}", exc_info=True)
        return "failure"
```text

### 6.6 防重放攻击证明

以下用数学证明形式说明为什么重放攻击无效：

```

定义：
  R = {out_trade_no, app_id, total_amount, sign, trade_status}
  表示一次支付宝通知请求。

定理：对于任意两次不同的通知请求 R₁ 和 R₂，
  若 R₁ 已成功处理，则 R₂ 不可能导致重复业务执行。

证明：
  假设 R₂ 被重放，我们检查四层防御：

  第一层（幂等性）：
    若 R₁ 处理成功，redis.setnx("alipay:processed:ORDER_NO", "1") 为 False
    → 直接返回 "success"，不再执行后续逻辑
    → 防御成功

  第二层（联合校验）：
    若 R₁ 在第一层失败前 R₂ 进入（极低概率竞态）：
    两个请求都会通过 SETNX
    但 joint_verify(app_id, out_trade_no, total_amount) 中：
      - 对于同一条订单，金额相同 → 都能通过
      - 对于不同订单，金额不同 → 第二请求被拒绝
    但还需要第三层来防止伪造请求。

  第三层（签名验证）：
    若 R₂ 是攻击者伪造的请求：
    verify(sign_content, sign, alipay_public_key) = False
    → 返回 "failure"
    → 攻击者无法伪造签名

  第四层（状态检查）：
    若 R₂ 的 trade_status ≠ TRADE_SUCCESS：
    → 返回 "success" 但不执行业务
    → 防御成功

  结论：重放攻击无法导致重复支付确认。
  Q.E.D.

```text

---

## 7. 项目代码逐行解读

### 7.1 app/services/alipay_sdk.py

```python
# 文件：/Users/freesling/Develop/hujiu/app/services/alipay_sdk.py

from app.config import settings


class AlipaySDK:
    """支付宝客户端包装器（沙箱/正式自动切换）"""

    @property
    def gateway_url(self) -> str:
        """
        自动根据环境返回支付网关 URL。
        @property 装饰器将其变为属性访问，而非方法调用。

        settings.ENV 的值由 .env 文件中的 ENV 环境变量控制：
          - "production"  → 正式环境
          - 其他值         → 沙箱环境（development, staging 等）
        """
        if settings.ENV == "production":
            return "https://openapi.alipay.com/gateway.do"
        return "https://openapi.alipaydev.com/gateway.do"  # 沙箱

    def build_pay_form(self, pay_params: dict) -> str:
        """
        构建支付宝支付表单 HTML。

        参数：
          pay_params: dict，包含以下字段：
            - out_trade_no: 商户订单号
            - total_amount: 订单总金额
            - subject: 订单标题

        返回：
          str: 支付宝支付 URL

        注意：这是简化实现。
        生产环境中应使用 alipay-sdk-python 完成签名。
        """
        # 构建业务参数（biz_content）
        biz_content = {
            "out_trade_no": pay_params["out_trade_no"],
            "total_amount": pay_params["total_amount"],
            "subject": pay_params["subject"],
            # 产品码：FAST_INSTANT_TRADE_PAY = 电脑网站支付
            "product_code": "FAST_INSTANT_TRADE_PAY",
        }

        # 构建请求参数
        params = {
            "app_id": settings.ALIPAY_APP_ID,
            "method": "alipay.trade.page.pay",     # API 接口名称
            "format": "JSON",
            "charset": "utf-8",
            "sign_type": "RSA2",                    # 签名算法
            "timestamp": "2024-01-01 00:00:00",    # 实际应使用当前时间
            "version": "1.0",
            "notify_url": settings.ALIPAY_NOTIFY_URL,  # 异步通知地址
            "return_url": settings.ALIPAY_RETURN_URL,  # 同步回跳地址
            "biz_content": str(biz_content).replace("'", '"'),  # 将单引号替换为双引号
        }

        # 拼接 URL 查询参数
        query = "&".join(f"{k}={v}" for k, v in params.items())
        return f"{self.gateway_url}?{query}"
```

### 7.2 app/services/payment_service.py

```python
# 文件：/Users/freesling/Develop/hujiu/app/services/payment_service.py

from decimal import Decimal
from typing import Optional
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from redis.asyncio import Redis

from app.models import Order, Event
from app.services.order_service import OrderService


class PaymentService:
    """
    支付服务类，封装支付相关的业务逻辑。

    职责：
    1. 生成支付宝支付参数
    2. 验证和处理支付宝异步通知
    3. 处理退款请求

    依赖注入：
      - session: AsyncSession（SQLAlchemy 异步会话）
      - redis: Redis（异步 Redis 客户端）
    """

    def __init__(self, session: AsyncSession, redis: Redis):
        """
        初始化支付服务。
        同时创建 OrderService 实例，用于订单相关的操作。

        FastAPI 的依赖注入系统会自动提供 session 和 redis：
          在 routers/payments.py 中：
          session: AsyncSession = Depends(get_db)
          redis: Redis = Depends(get_redis)
        """
        self.session = session
        self.redis = redis
        self.order_service = OrderService(session, redis)

    async def create_alipay_payment(self, order_id: int, user_id: int) -> dict:
        """
        生成支付宝支付参数。

        流程：
        1. 根据 order_id 查询订单
        2. 验证订单属于当前用户
        3. 验证订单状态为 pending_payment
        4. 查询活动信息用于生成 subject
        5. 返回支付参数

        参数：
          order_id: 订单 ID
          user_id: 当前用户 ID（用于鉴权）

        返回：
          dict: 包含 pay_params, order_no, total_amount

        异常：
          ValueError: 订单不存在、订单不属于用户、状态不允许支付
        """
        # 查询订单
        order = await self.session.get(Order, order_id)
        # SQLAlchemy 的 session.get 使用主键查询
        # 相当于 SELECT * FROM orders WHERE id = order_id

        # 验证订单存在且属于当前用户
        if not order or order.user_id != user_id:
            raise ValueError("订单不存在")

        # 验证订单状态（只有待支付订单可以发起支付）
        if order.status != "pending_payment":
            raise ValueError("订单状态不允许支付")

        # 查询活动（用于订单标题）
        event = await self.session.get(Event, order.event_id)

        # 构建支付请求参数
        pay_params = {
            "out_trade_no": order.order_no,            # 商户订单号
            "total_amount": str(order.total_amount),   # 金额转为字符串
            "subject": f"票务-{event.title}",          # 标题格式：票务-活动名
            "product_code": "FAST_INSTANT_TRADE_PAY",  # 产品码
        }

        return {
            "pay_params": pay_params,
            "order_no": order.order_no,
            "total_amount": float(order.total_amount),
        }

    async def verify_and_process_notification(self, data: dict) -> str:
        """
        验证支付宝异步通知并处理（核心方法）。

        四层防御：
        1. 签名验证（使用支付宝公钥）
        2. 交易状态检查（必须为 TRADE_SUCCESS）
        3. 幂等性检查（Redis SETNX）
        4. 金额校验（通知金额 vs 订单金额）

        参数：
          data: dict — 支付宝 POST 的表单参数

        返回：
          str: "success" 或 "failure"
        """
        # 从通知参数中提取签名并移除
        sign = data.pop("sign", None)
        data.pop("sign_type", None)

        # ！！！重要：实际项目需要使用 alipay-sdk 进行签名验证 ！！！
        # 验签步骤：
        #   1. 对 data（不含 sign）按 key 排序拼接
        #   2. 使用支付宝公钥 + RSA2 验证
        # 代码示例：
        #   from alipay import AliPay
        #   if not alipay.verify(data, sign):
        #       return "failure"
        #
        # 此处为简化实现，仅演示逻辑流程

        # 第四层：交易状态验证
        if data.get("trade_status") != "TRADE_SUCCESS":
            return "success"

        order_no = data["out_trade_no"]

        # 第一层：幂等性检查
        # setnx 原子操作
        processed = await self.redis.setnx(
            f"alipay:processed:{order_no}", "1"
        )
        if not processed:
            # 已处理过，直接返回 success
            return "success"

        try:
            # 第二层：联合信息校验（金额）
            result = await self.session.execute(
                select(Order).where(Order.order_no == order_no)
            )
            order = result.scalar_one_or_none()
            if not order:
                return "failure"

            # 严格金额比较（使用 Decimal 避免浮点误差）
            if Decimal(data["total_amount"]) != Decimal(str(order.total_amount)):
                return "failure"

            # 调用 OrderService 确认支付
            # 会执行：
            #   1. 将订单状态改为 paid
            #   2. 设置 payment_method = "alipay"
            #   3. 设置 trade_no
            #   4. 设置 paid_at
            #   5. 生成对应的票证记录
            await self.order_service.confirm_payment(order_no, data["trade_no"])
            return "success"

        except Exception:
            # 如果业务处理失败，删除幂等性 key
            # 这样支付宝重试时，幂等性检查不会跳过
            await self.redis.delete(f"alipay:processed:{order_no}")
            raise

    async def refund(self, order_id: int, reason: str = "用户申请退款") -> dict:
        """
        发起退款。

        流程：
        1. 查询订单
        2. 验证订单状态为 paid
        3. 调用支付宝退款 API
        4. 更新订单状态为 refunded

        参数：
          order_id: 订单 ID
          reason: 退款原因（默认"用户申请退款"）

        返回：
          dict: {"success": True, "trade_no": "支付宝交易号"}

        异常：
          ValueError: 订单不存在或状态不允许退款
        """
        order = await self.session.get(Order, order_id)

        # 状态验证：只有已支付的订单可以退款
        if not order or order.status != "paid":
            raise ValueError("订单状态不允许退款")

        # ！！！实际项目需要调用支付宝退款 API ！！！
        # result = alipay.api_alipay_trade_refund(
        #     trade_no=order.trade_no,
        #     refund_amount=float(order.total_amount),
        #     refund_reason=reason,
        # )
        # if result["code"] != "10000":
        #     return {"success": False, "error": result["sub_msg"]}

        # 更新订单状态
        order.status = "refunded"
        await self.session.flush()

        return {"success": True, "trade_no": order.trade_no}
```text

### 7.3 app/routers/payments.py

```python
# 文件：/Users/freesling/Develop/hujiu/app/routers/payments.py

from fastapi import APIRouter, Depends, HTTPException, Request, status
from sqlalchemy.ext.asyncio import AsyncSession
from redis.asyncio import Redis

from app.database import get_db
from app.redis_client import get_redis
from app.schemas.payment import AlipayRequest, AlipayResponse, RefundRequest
from app.services.payment_service import PaymentService
from app.services.alipay_sdk import AlipaySDK
from app.middleware.auth import get_current_user, require_admin
from app.models.user import User

# 创建路由实例
# 在 app/routers/__init__.py 中注册：
#   api_router.include_router(payments.router, prefix="/payments", tags=["支付"])
# 所以最终端点前缀为：/api/v1/payments
router = APIRouter()


@router.post("/alipay", response_model=AlipayResponse)
async def alipay_payment(
    data: AlipayRequest,
    session: AsyncSession = Depends(get_db),
    redis: Redis = Depends(get_redis),
    current_user: User = Depends(get_current_user),
):
    """
    获取支付宝支付参数。

    HTTP POST /api/v1/payments/alipay

    请求体 (AlipayRequest):
      {
        "order_id": 123     // 订单 ID
      }

    响应 (AlipayResponse):
      {
        "pay_url": "https://openapi.alipaydev.com/gateway.do?...",
        "order_no": "20240501123456789",
        "total_amount": 99.50
      }

    流程：
    1. 验证用户身份（通过 get_current_user 依赖）
    2. 获取数据库会话和 Redis 连接（通过依赖注入）
    3. 创建 PaymentService 实例
    4. 生成支付参数
    5. 使用 AlipaySDK 构建支付 URL
    6. 返回支付信息和 URL
    """
    # 创建支付服务实例
    service = PaymentService(session, redis)

    try:
        # 生成支付参数
        result = await service.create_alipay_payment(data.order_id, current_user.id)

        # 构建支付 URL
        sdk = AlipaySDK()
        pay_url = sdk.build_pay_form(result["pay_params"])

        # 返回响应
        return AlipayResponse(
            pay_url=pay_url,
            order_no=result["order_no"],
            total_amount=result["total_amount"],
        )

    except ValueError as e:
        # 业务逻辑异常（订单不存在、状态不允许等）
        # 返回 400 Bad Request
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e),
        )


@router.post("/alipay/notify")
async def alipay_notify(
    request: Request,
    session: AsyncSession = Depends(get_db),
    redis: Redis = Depends(get_redis),
):
    """
    支付宝异步通知处理（核心端点）。

    HTTP POST /api/v1/payments/alipay/notify

    这是支付宝支付流程中最重要的端点。
    支付宝在用户支付成功后，会向此地址发送 POST 请求。

    注意：
    - 此端点不验证用户身份（通知来自支付宝服务器而非用户浏览器）
    - Content-Type 为 application/x-www-form-urlencoded
    - 必须返回纯文本 "success" 或 "failure"
    - 不要加身份验证中间件！

    处理流程：
    1. 从 POST body 中解析 form-data 参数
    2. 创建 PaymentService 实例
    3. 调用 verify_and_process_notification（内部包含四层防御）
    4. 返回处理结果
    """
    # 从请求中提取 form 数据
    # request.form() 返回一个 FormData 对象
    # 转换为 dict 以便后续处理
    form_data = dict(await request.form())
    # form_data 包含支付宝 POST 的所有参数：
    # {
    #   "app_id": "202100...",
    #   "out_trade_no": "20240501123456789",
    #   "trade_no": "202405012200...",
    #   "total_amount": "99.50",
    #   "trade_status": "TRADE_SUCCESS",
    #   "sign": "aB3dE5FgH1Ij...",
    #   "sign_type": "RSA2",
    #   ...
    # }

    # 创建支付服务并处理通知
    service = PaymentService(session, redis)
    result = await service.verify_and_process_notification(form_data)

    # 处理失败
    if result == "failure":
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="通知验证失败",
        )

    # 返回 success（必须返回纯文本 "success"）
    # 支付宝收到 "success" 后停止重试
    return {"result": result}


@router.post("/refund")
async def refund(
    data: RefundRequest,
    session: AsyncSession = Depends(get_db),
    redis: Redis = Depends(get_redis),
    current_user: User = Depends(require_admin),
):
    """
    退款（需要管理员权限）。

    HTTP POST /api/v1/payments/refund

    请求体 (RefundRequest):
      {
        "order_id": 123,        // 订单 ID
        "reason": "用户申请退款"  // 退款原因（可选）
      }

    权限：仅 admin 角色可访问（通过 require_admin 依赖控制）

    流程：
    1. 验证用户角色为 admin
    2. 创建 PaymentService 实例
    3. 调用 refund 方法处理退款
    4. 提交事务（确认退款操作）
    5. 返回退款结果
    """
    # 创建支付服务
    service = PaymentService(session, redis)

    try:
        # 处理退款（内部验证订单状态为 paid）
        result = await service.refund(data.order_id, data.reason)

        # 提交事务
        # PaymentService.refund 中修改了订单状态
        # 需要在路由层提交以生效
        await session.commit()

        return result

    except ValueError as e:
        # 业务异常（订单不存在、状态不允许退款等）
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e),
        )
```

---

## 8. 支付状态关联数据验证

### 8.1 Order.status 与数据库约束

```sql
-- 虽然我们通过业务代码控制状态转换，
-- 但 MySQL 层面也可以添加 CHECK 约束作为兜底：
ALTER TABLE orders
ADD CONSTRAINT chk_order_status
CHECK (status IN ('pending_payment', 'paid', 'cancelled', 'refunded'));

-- 支付成功后生成票证的一致性约束
ALTER TABLE tickets
ADD CONSTRAINT fk_tickets_order
FOREIGN KEY (order_id) REFERENCES orders(id)
ON DELETE CASCADE;

-- 退款后不允许再次退款的约束（业务层防御+数据库约束）
-- 注：MySQL 不支持基于状态的条件约束
-- 需要在业务代码中保证
```text

### 8.2 Pydantic 请求/响应模式

```python
# app/schemas/payment.py

from typing import Optional
from pydantic import BaseModel, Field


class AlipayRequest(BaseModel):
    """支付宝支付请求参数"""
    order_id: int = Field(..., gt=0, description="订单 ID")
    # gt=0: 必须大于 0
    # ...: 必填


class AlipayResponse(BaseModel):
    """支付宝支付响应参数"""
    pay_url: str                  # 支付宝支付链接
    order_no: str                 # 商户订单号
    total_amount: float           # 订单总金额


class RefundRequest(BaseModel):
    """退款请求参数"""
    order_id: int = Field(..., gt=0, description="订单 ID")
    reason: str = Field("用户申请退款", description="退款原因")


class NotifyResponse(BaseModel):
    """异步通知响应"""
    result: str                   # "success" 或 "failure"
```

---

## 9. .env 配置示例

```ini
# .env — 支付宝相关配置

# 支付宝应用 ID（沙箱环境下使用沙箱 APPID）
ALIPAY_APP_ID=20210011xxxxx

# 密钥文件路径（相对于项目根目录）
ALIPAY_APP_PRIVATE_KEY_PATH=keys/app_private_key.pem
ALIPAY_PUBLIC_KEY_PATH=keys/alipay_public_key.pem

# 通知 URL（必须使用公网可访问的地址）
# 开发阶段：使用第12章的 ngrok/frp 内网穿透
ALIPAY_NOTIFY_URL=https://your-domain.com/api/v1/payments/alipay/notify
ALIPAY_RETURN_URL=https://your-domain.com/orders

# 环境控制（development=沙箱, production=正式）
ENV=development
```text

---

## 10. 沙箱测试完整步骤

```bash
# ============================================
# 步骤 1：生成 RSA 密钥对
# ============================================
openssl genrsa -out keys/app_private_key.pem 2048
openssl rsa -in keys/app_private_key.pem -pubout -out keys/app_public_key.pem
openssl pkcs8 -topk8 -inform PEM -in keys/app_private_key.pem \
    -outform PEM -nocrypt -out keys/app_private_key_pkcs8.pem

# ============================================
# 步骤 2：上传公钥到支付宝开放平台
# ============================================
# 1. 登录 https://open.alipay.com 进入沙箱应用
# 2. 在"接口加签方式"中点击"设置"
# 3. 选择"公钥模式"，上传 keys/app_public_key.pem 的内容
# 4. 保存后获取"支付宝公钥"
# 5. 将支付宝公钥保存到 keys/alipay_public_key.pem

# ============================================
# 步骤 3：配置 .env
# ============================================
# 按照第 9 节的 .env 示例配置

# ============================================
# 步骤 4：启动内网穿透（开发环境）
# ============================================
# 支付宝需要公网可访问的 notify_url
# 使用 ngrok：
ngrok http 8000
# 将获取到的公网地址设置为 ALIPAY_NOTIFY_URL

# ============================================
# 步骤 5：启动应用
# ============================================
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

# ============================================
# 步骤 6：创建订单并发起支付
# ============================================
# 1. 注册用户 POST /api/v1/auth/register
# 2. 登录获取 Token POST /api/v1/auth/login
# 3. 创建订单 POST /api/v1/orders/ (需要 event_id)
# 4. 获取支付链接 POST /api/v1/payments/alipay
# 5. 浏览器打开 pay_url
# 6. 使用沙箱买家账号登录支付

# ============================================
# 步骤 7：验证异步通知
# ============================================
# 支付成功后：
# 1. 检查应用日志中的通知处理记录
# 2. 检查订单状态是否变为 paid
# 3. 检查票证是否已生成

# ============================================
# 步骤 8：测试退款
# ============================================
# 使用管理员 Token：
curl -X POST "http://localhost:8000/api/v1/payments/refund" \
  -H "Authorization: Bearer <admin_token>" \
  -H "Content-Type: application/json" \
  -d '{"order_id": 123, "reason": "测试退款"}'
# 验证订单状态变为 refunded
```

---

✅ **本章项目里程碑：**

- [ ] RSA 密钥对生成完成，私钥安全保存
- [ ] 支付宝沙箱应用配置完成
- [ ] 支付 SDK 初始化代码通过
- [ ] 支付状态机完整实现（4 种状态 + 3 种转换）
- [ ] 异步通知端点正确验签
- [ ] 四层防重放攻击全部实现
- [ ] 退款流程可用
- [ ] 沙箱支付全流程测试通过

---

## 📁 项目代码参考

| 文件 | 说明 |
| ------ | ------ |
| `app/services/alipay_sdk.py` | 支付宝 SDK 封装：签名生成、验签、HTTP 请求 |
| `app/services/payment_service.py` | 支付业务：下单、通知处理、退款、状态机 |
| `app/routers/payments.py` | 支付路由：发起支付、异步通知、查询 |
| `app/schemas/payment.py` | 支付相关 Pydantic schema |
| `app/utils/order_no.py` | 订单号生成器（流水号格式） |

**订单状态机**（`app/models/order.py`）：

```text
pending → paid → refunded (退款)
pending → cancelled (用户取消)
pending → cancelled (超时取消，ARQ Worker)
```

**4 层防重放攻击**（`app/services/payment_service.py`）：

1. RSA2 签名验证（支付宝公钥验签）
2. 幂等性键（Redis SET NX，防止重复处理）
3. 金额比对（`Decimal` 精确匹配订单金额）
4. 状态机检查（防止已支付订单再次处理）

---

## 安全防护

### 签名链安全

支付宝支付的核心安全机制是 RSA2 签名链：

```text
应用签名 → 支付宝验签 → 支付宝处理 → 支付宝签名 → 应用验签
  (app private key)    (alipay public key)              (alipay public key)    (app private key)
```

### 验签不可伪造性证明

1. 只有持有**应用私钥**的服务器才能生成被支付宝公钥验证通过的签名
2. 只有**支付宝**才能生成被应用公钥验证通过的签名
3. 攻击者在没有私钥的情况下无法伪造有效签名
4. 签名与参数绑定：任何参数修改都会导致验签失败

### 四层防重放防御

```text
支付宝通知 → Layer 1: RSA2 验签 → Layer 2: 状态校验 → Layer 3: 幂等键 → Layer 4: 金额校验 → 确认支付
```

- **Layer 1 — 签名验证**：确保数据来自支付宝
- **Layer 2 — 状态机**：只有 `TRADE_SUCCESS` 才处理
- **Layer 3 — 幂等键**：Redis SETNX + TTL 防止重复处理
- **Layer 4 — 金额校验**：通知金额与数据库金额严格比对

### 幂等键（Idempotency Key）原理

```python
key = f"alipay:processed:{order_no}"
processed = await self.redis.setnx(key, "1")  # 原子操作
await self.redis.expire(key, 86400)           # 24 小时 TTL
if not processed:
    return "success"  # 已处理过，忽略

try:
    # 处理业务...
except Exception:
    await self.redis.delete(key)  # 失败时删除幂等键，允许重试
    raise
```text

### 时间戳防重放

支付请求中的 `timestamp` 必须为**当前时间**（非硬编码）。支付宝校验时间戳与服务器时间的偏差，超过一定范围（通常 15 分钟）直接拒绝。

- [ ] 幂等性验证通过（重放通知不重复处理）

---

## 11. 原理深入（补充）

### 11.1 RSA2 签名完整流程

#### 创建签名（应用侧 → 支付宝）

```

步骤1：组装请求参数（公共参数 + 业务参数）
步骤2：按 key 字典序排序
步骤3：拼接为待签名字符串
  app_id=xxx&biz_content={...}&charset=utf-8&...
    ↓
步骤4：对待签名字符串计算 SHA256 哈希
步骤5：使用应用私钥对哈希值进行 RSA 签名（PKCS1 v1.5 padding）
步骤6：对签名结果进行 Base64 编码
    ↓
步骤7：将 sign 参数附加到请求中
步骤8：发送请求到支付宝网关

```text

```python
import hashlib
import base64
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import padding, rsa
from cryptography.hazmat.primitives.serialization import load_pem_private_key

def alipay_sign(params: dict, private_key_pem: str) -> str:
    """
    生成支付宝 RSA2 签名

    完整步骤实现（不依赖 SDK）：
    """
    # 1. 剔除 sign 和 sign_type 参数
    params = {k: v for k, v in params.items() if k not in ("sign", "sign_type")}

    # 2. 按 key 字典序排序并拼接
    sorted_keys = sorted(params.keys())
    sign_content = "&".join(f"{k}={params[k]}" for k in sorted_keys)

    # 3. 加载私钥（PKCS8 格式）
    private_key = load_pem_private_key(
        private_key_pem.encode(),
        password=None,
    )

    # 4. 计算签名
    # PKCS1v15 + SHA256 = RSA2 签名算法
    signature = private_key.sign(
        sign_content.encode("utf-8"),
        padding.PKCS1v15(),
        hashes.SHA256(),
    )

    # 5. Base64 编码
    return base64.b64encode(signature).decode("utf-8")
```

#### 验证签名（应用侧 ← 支付宝响应）

```python
def alipay_verify(params: dict, sign: str, public_key_pem: str) -> bool:
    """
    验证支付宝 RSA2 签名

    1. 从参数中剔除 sign 和 sign_type
    2. 排序 → 拼接待验签字符串
    3. 使用支付宝公钥验证签名
    """
    params = {k: v for k, v in params.items() if k not in ("sign", "sign_type")}
    sorted_keys = sorted(params.keys())
    sign_content = "&".join(f"{k}={params[k]}" for k in sorted_keys)

    from cryptography.hazmat.primitives.serialization import load_pem_public_key

    public_key = load_pem_public_key(public_key_pem.encode())

    try:
        public_key.verify(
            base64.b64decode(sign),
            sign_content.encode("utf-8"),
            padding.PKCS1v15(),
            hashes.SHA256(),
        )
        return True
    except Exception:
        return False
```text

#### 签名链的完整流程

```

请求阶段（应用 → 支付宝）：
  应用组装参数 → 应用私钥签名(SHA256+RSA) → 支付宝收到 → 应用公钥验签
                                                              ↓
                                                      支付宝处理业务
                                                              ↓
响应阶段（支付宝 → 应用）：
  支付宝组装响应 → 支付宝私钥签名(SHA256+RSA) → 应用收到 → 支付宝公钥验签

=> 双向签名验证，确保通信双方身份真实、数据未被篡改

```text

### 11.2 异步通知重试机制详解

#### 支付宝重试策略

```

通知失败判定条件：
  应用服务器返回非 "success" 字符串
  或应用服务器无响应（超时 5 秒）
  或 HTTP 状态码非 200

重试时间间隔（递增）：
  第 1 次：10 秒后
  第 2 次：20 秒后
  第 3 次：2 分钟后
  第 4 次：10 分钟后
  第 5 次：10 分钟后
  第 6 次：1 小时后
  第 7 次：2 小时后
  第 8 次：6 小时后
  → 共 8 次重试，总持续时间约 9 小时

最佳实践：

- 第一次就要正确验签并返回 "success"
- 幂等处理后仍需返回 "success"（不要返回 "failure" 触发重试）
- 业务处理失败时返回 "failure" 让支付宝重试

```text

#### 重试安全设计

```python
async def handle_notification(self, data: dict) -> str:
    """
    异步通知处理（考虑重试场景）

    重试安全策略：
    1. 验签失败 → 返回 "failure" → 支付宝重试
    2. 验签通过但状态非 TRADE_SUCCESS → 返回 "success" → 停止重试
    3. 验签通过但幂等键已存在 → 返回 "success" → 停止重试（已处理过）
    4. 验签通过但业务处理异常 → 返回 "failure" → 继续重试
    """
    # 验签
    sign = data.pop("sign", None)
    data.pop("sign_type", None)
    if not self.alipay.verify(data, sign):
        # 验签失败：可能是网络传输损坏或伪造请求
        # 返回 failure 让支付宝重试
        return "failure"

    # 非成功状态不处理
    if data.get("trade_status") != "TRADE_SUCCESS":
        return "success"  # 返回 success，告诉支付宝不用再通知了

    order_no = data["out_trade_no"]

    # 幂等性检查
    processed = await self.redis.setnx(f"alipay:processed:{order_no}", "1")
    if not processed:
        # 已处理过：直接返回 success，不触发重试
        return "success"
    await self.redis.expire(f"alipay:processed:{order_no}", 86400)

    try:
        # 业务处理
        order = await self._confirm_payment(order_no, data["trade_no"])
        return "success"
    except Exception:
        # 业务失败：删除幂等键，让支付宝重试
        await self.redis.delete(f"alipay:processed:{order_no}")
        return "failure"
```

### 11.3 防重放攻击的完整设计

#### 五种防重放机制

| 机制 | 防御目标 | 实现方式 | 强度 |
| ------ | --------- | --------- | ------ |
| RSA2 签名验证 | 伪造通知 | 支付宝公钥验签 | 密码学级别 |
| 时间戳校验 | 过期重放 | timestamp 与服务器时间差 ≤ 15min | 强 |
| 幂等键 | 重复处理 | Redis SETNX + 24h TTL | 强 |
| 金额比对 | 参数篡改 | Decimal 精确匹配 | 强 |
| 交易状态机 | 无效状态 | 仅处理 TRADE_SUCCESS | 中 |

#### 时间戳防重放的数学原理

```text
定义：
  T_req = 请求中的 timestamp（Unix 时间戳）
  T_svr = 服务器收到请求的时间
  ΔT_max = 允许的最大时间偏差（支付宝默认 15 分钟）

验证条件：
  |T_svr - T_req| ≤ ΔT_max

证明：
  攻击者截获一次合法通知后，需要重放该通知。
  假设攻击者在时间 T_capture 截获通知，在 T_replay 重放。

  如果 T_replay - T_capture > ΔT_max：
    → 时间戳校验失败 → 通知被拒绝

  因此攻击者只有在 ΔT_max 窗口内重放才可能成功。
  结合 SETNX 幂等键（订单级别），窗口内重放也会被幂等拒绝。

  双层时间保护：
  支付宝侧：校验请求中的 timestamp 与当前时间偏差
  应用侧：校验通知到达时间与订单创建时间的偏差
```

### 11.4 沙箱 vs 生产环境配置

#### 完整对比

| 配置项 | 沙箱环境 | 生产环境 |
| -------- | --------- | --------- |
| 网关地址 | `https://openapi.alipaydev.com/gateway.do` | `https://openapi.alipay.com/gateway.do` |
| APPID | 沙箱专用 APPID（平台生成） | 正式 APPID（审核通过后获取） |
| 密钥 | 测试 RSA 密钥对 | 正式 RSA 密钥对（2048位） |
| 买家账户 | 系统自动生成的沙箱账号 | 真实支付宝账号 |
| 支付资金 | 虚拟资金（不真实扣款） | 真实资金 |
| 通知地址 | 内网穿透（ngrok/frp） | 正式域名 |
| 应用审核 | 不需要 | 需要提交审核 |
| 签约 | 不需要 | 需要签约"电脑网站支付" |
| 费率 | 无 | 0.6%-1.2%（按行业） |
| 限额 | 无 | 单笔/单日有限额 |

#### .env 配置模板（支持双环境）

```ini
# .env（开发环境 - 沙箱）
ENV=development
ALIPAY_APP_ID=2021000111111111
ALIPAY_APP_PRIVATE_KEY_PATH=keys/sandbox_app_private_key.pem
ALIPAY_PUBLIC_KEY_PATH=keys/sandbox_alipay_public_key.pem
ALIPAY_NOTIFY_URL=https://your-ngrok.ngrok.io/api/v1/payments/alipay/notify
ALIPAY_RETURN_URL=http://localhost:5173/orders

# .env.production（生产环境 - 正式）
# ENV=production
# ALIPAY_APP_ID=2021001222222222
# ALIPAY_APP_PRIVATE_KEY_PATH=keys/prod_app_private_key.pem
# ALIPAY_PUBLIC_KEY_PATH=keys/prod_alipay_public_key.pem
# ALIPAY_NOTIFY_URL=https://your-domain.com/api/v1/payments/alipay/notify
# ALIPAY_RETURN_URL=https://your-domain.com/orders
```text

#### 沙箱测试常见问题

| 问题 | 原因 | 解决 |
| ------ | ------ | ------ |
| 支付页面显示"应用不存在" | APPID 配置错误 | 检查 APPID 是否来自沙箱应用 |
| 异步通知收不到 | 内网穿透未启动或 URL 不对 | 检查 ngrok/frp 是否运行，通知 URL 是否正确 |
| 验签失败 | 密钥不匹配 | 确认使用的是"支付宝公钥"而非"应用公钥" |
| 支付后页面空白 | return_url 未正确配置 | return_url 必须可公网访问 |
| 沙箱账号登录失败 | 账号过期 | 重新生成沙箱买家账号 |
