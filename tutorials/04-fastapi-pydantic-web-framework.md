# 第4章：FastAPI + Pydantic v2 — Web 框架基础

## 本章学习目标

本章将从零学习 Pydantic v2 和 FastAPI，掌握构建 Web API 所需的全部核心知识，最终搭建出票务系统的完整 Web 框架。

**学习路径：**

1. ✅ ASGI 协议与 Uvicorn 完全教程
2. ✅ Pydantic v2 完全教程（Field/验证/序列化/全部类型）
3. ✅ FastAPI 完全教程（路径操作/参数/响应/依赖注入）
4. ✅ 中间件、CORS、异常处理、静态文件
5. ✅ 项目：票务系统 Web 框架搭建（路由注册 + 验证测试）

---

## 1. ASGI 协议与 Uvicorn

### 1.1 WSGI vs ASGI

在理解 FastAPI 之前，需要先了解底层的协议。

**WSGI（Web Server Gateway Interface）—— Python 传统 Web 协议（PEP 3333）：**

```text
HTTP 请求 → Nginx → WSGI 服务器(Gunicorn) → Django/Flask 应用
                                  ↓
                         同步处理，一次一个请求
                         不支持 WebSocket
                         不支持 HTTP/2
```

**ASGI（Asynchronous Server Gateway Interface）—— Python 异步 Web 协议（ASGI 规范）：**

```text
HTTP 请求 → Nginx → ASGI 服务器(Uvicorn) → FastAPI 应用
                                  ↓
                         异步处理，高并发
                         支持 WebSocket
                         支持 HTTP/2
                         支持 SSE
```

**ASGI 规范核心：**

ASGI 将应用定义为一个**异步可调用对象**，接收三个参数：

```python
# ASGI 应用的最小实现
async def app(scope, receive, send):
    """
    scope:    连接信息字典（方法、路径、头等）
    receive:  异步函数，接收客户端消息
    send:     异步函数，发送消息给客户端
    """
    assert scope["type"] == "http"

    # 接收请求体
    request_body = await receive()

    # 发送响应头
    await send({
        "type": "http.response.start",
        "status": 200,
        "headers": [
            (b"content-type", b"text/plain"),
        ],
    })

    # 发送响应体
    await send({
        "type": "http.response.body",
        "body": b"Hello, ASGI!",
    })
```text

**ASGI 的生命周期协议：**

```python
# 应用生命周期
async def app(scope, receive, send):
    if scope["type"] == "lifespan":
        while True:
            message = await receive()
            if message["type"] == "lifespan.startup":
                # 应用启动时执行（连接数据库、加载缓存等）
                await send({"type": "lifespan.startup.complete"})
            elif message["type"] == "lifespan.shutdown":
                # 应用关闭时执行（关闭连接池、清理资源）
                await send({"type": "lifespan.shutdown.complete"})
            break
```

FastAPI 的 `@asynccontextmanager` lifespan 就是对上述协议的封装。

### 1.2 Uvicorn 完全教程

Uvicorn 是基于 uvloop 和 httptools 的超快 ASGI 服务器。

```bash
# 安装
pip install uvicorn[standard]
# [standard] 包含：uvloop(事件循环加速), httptools(HTTP解析), wswebsocket(WebSocket)

# 基础用法
uvicorn app.main:app

# 指定主机和端口
uvicorn app.main:app --host 0.0.0.0 --port 8000

# 热重载（开发模式）
uvicorn app.main:app --reload

# 指定日志级别
uvicorn app.main:app --log-level debug

# 指定 workers（生产模式）
uvicorn app.main:app --workers 4

# 启用 HTTPS
uvicorn app.main:app --ssl-certfile cert.pem --ssl-keyfile key.pem

# 限制定向协议
uvicorn app.main:app --http httptools --ws websockets
```text

**Uvicorn 全部参数：**

| 参数 | 默认值 | 说明 |
| ------ | -------- | ------ |
| `--host` | 127.0.0.1 | 监听地址 |
| `--port` | 8000 | 监听端口 |
| `--workers` | 1 | 工作进程数（仅 Unix） |
| `--loop` | auto | 事件循环：auto/uvloop/asyncio |
| `--http` | auto | HTTP 实现：auto/httptools/h11 |
| `--ws` | auto | WebSocket 实现：auto/websockets/wsproto |
| `--reload` | False | 热重载（开发） |
| `--reload-dir` | . | 监听更改的目录 |
| `--log-level` | info | 日志级别 |
| `--access-log` | True | 访问日志 |
| `--proxy-headers` | False | 信任代理头（X-Forwarded-For） |
| `--forwarded-allow-ips` | 127.0.0.1 | 允许转发的 IP |
| `--timeout-keep-alive` | 5 | 长连接超时（秒） |
| `--ssl-keyfile` | - | SSL 私钥路径 |
| `--ssl-certfile` | - | SSL 证书路径 |

**loop 选择：**

- **uvloop**：基于 libuv 的加速事件循环，比 asyncio 快 2 倍（仅 Unix/macOS）
- **asyncio**：Python 标准事件循环（Windows 兼容）

**workers 工作原理：**

```

Uvicorn 主进程
    │
    ├── Worker 1 (进程)
    │   └── 事件循环 → 处理请求
    │
    ├── Worker 2 (进程)
    │   └── 事件循环 → 处理请求
    │
    ├── Worker 3 (进程)
    │   └── 事件循环 → 处理请求
    │
    └── Worker 4 (进程)
        └── 事件循环 → 处理请求

```text

> **注意：** Windows 不支持 `--workers`（缺少 fork）。Windows 下需要手动启动多个进程或使用 Docker。

---

## 2. Pydantic v2 完全教程

Pydantic 是 FastAPI 的数据验证核心。Pydantic v2 使用 Rust 编写的核心引擎（pydantic-core），验证速度比 v1 快 5-50 倍。

### 2.1 BaseModel 基础

```python
from pydantic import BaseModel

class UserCreate(BaseModel):
    """创建用户的请求体"""
    username: str
    email: str
    password: str

# 使用
data = UserCreate(
    username="张三",
    email="zhangsan@example.com",
    password="securePwd123",
)
print(data.username)  # 张三
print(data.model_dump())  # 序列化为字典
# {'username': '张三', 'email': 'zhangsan@example.com', 'password': 'securePwd123'}
```

**数据验证——传入错误类型自动转换或报错：**

```python
class EventCreate(BaseModel):
    title: str
    price: float
    total_stock: int

# 自动类型转换
event = EventCreate(title="演唱会", price="680", total_stock="100")
print(event.price)  # 680.0（字符串自动转为 float）
print(event.total_stock)  # 100（字符串自动转为 int）

# 类型错误
try:
    EventCreate(title="演唱会", price="不是数字", total_stock="100")
except Exception as e:
    print(e)  # 详细的验证错误信息
```text

### 2.2 Field 全部参数

```python
from pydantic import BaseModel, Field
from typing import Optional

class EventCreate(BaseModel):
    title: str = Field(
        ...,                        # ... 表示必填
        min_length=1,               # 最小长度
        max_length=200,             # 最大长度
        description="活动标题",      # 描述（用于生成文档）
        examples=["周杰伦演唱会"],   # 示例值
    )
    price: float = Field(
        ...,                        # 必填
        gt=0,                       # 大于 0 (greater than)
        ge=0.01,                    # 大于等于 (greater or equal)
        lt=100000,                  # 小于 (less than)
        le=99999.99,               # 小于等于 (less or equal)
        multiple_of=0.01,           # 必须是 0.01 的倍数（金额精度）
        description="票价",
    )
    total_stock: int = Field(
        ..., ge=1, le=100000
    )
    description: Optional[str] = Field(
        None,                       # None = 可选
        max_length=2000,
    )
    category: str = Field(
        default="other",            # 默认值
        pattern=r"^(concert|sports|theater|other)$",  # 正则验证
    )
    is_active: bool = Field(
        default=True,
        alias="isActive",           # 别名（允许传入 isActive）
    )

# 使用别名
event = EventCreate.model_validate({
    "title": "演唱会",
    "price": 680,
    "total_stock": 100,
    "isActive": True,  # 使用别名
})
print(event.is_active)  # True

# 设为冻结（不可变）
class Config(BaseModel):
    model_config = {"frozen": True}  # 实例创建后不可修改
```

**Field 全部参数速查表：**

| 参数 | 类型 | 说明 |
| ------ | ------ | ------ |
| `default` | Any | 默认值 |
| `default_factory` | Callable | 默认值工厂函数 |
| `alias` | str | 字段别名 |
| `alias_priority` | int | 别名优先级 |
| `validation_alias` | str | 仅验证时的别名 |
| `serialization_alias` | str | 仅序列化时的别名 |
| `title` | str | 标题（OpenAPI 用） |
| `description` | str | 描述（OpenAPI 用） |
| `examples` | list | 示例值 |
| `exclude` | bool | 排除此字段 |
| `discriminator` | str | 判别联合的辨别字段 |
| `gt` | float | 大于 |
| `ge` | float | 大于等于 |
| `lt` | float | 小于 |
| `le` | float | 小于等于 |
| `multiple_of` | float | 倍数约束 |
| `min_length` | int | 最小长度 |
| `max_length` | int | 最大长度 |
| `pattern` | str | 正则表达式 |
| `frozen` | bool | 是否不可变 |
| `validate_default` | bool | 是否验证默认值 |
| `repr` | bool | 是否在 **repr** 中显示 |

### 2.3 model_config 全部选项

```python
from pydantic import BaseModel, ConfigDict

class UserModel(BaseModel):
    model_config = ConfigDict(
        # ===== 验证相关 =====
        extra="forbid",                  # 禁止额外字段（默认 "ignore"）
        # "ignore": 忽略额外字段
        # "forbid": 额外字段抛出错误
        # "allow":  保留额外字段

        frozen=True,                     # 冻结（不可修改）

        populate_by_name=True,           # 允许用字段名填充（即使定义了别名）

        validate_assignment=True,        # 赋值时验证

        validate_default=False,          # 是否验证默认值

        # ===== 序列化相关 =====
        from_attributes=True,            # 从 ORM 对象创建（用于 response 模型）

        # ===== 文档相关 =====
        title="User Model",              # JSON Schema 标题
        json_schema_extra={              # 额外 JSON Schema 属性
            "example": {"username": "张三", "email": "test@example.com"},
        },

        # ===== 性能相关 =====
        revalidate_never=False,          # 是否重新验证
        defer_build=True,                # 延迟构建模式（缓存架构信息）
    )

    username: str
    email: str
```text

### 2.4 验证器（Validator）

#### 2.4.1 @field_validator

```python
from pydantic import BaseModel, field_validator
import re

class UserCreate(BaseModel):
    username: str
    password: str
    confirm_password: str

    # 单字段验证器
    @field_validator("username")
    @classmethod
    def validate_username(cls, v: str) -> str:
        """自定义用户名验证"""
        if len(v) < 2:
            raise ValueError("用户名至少2个字符")
        if not re.match(r"^[a-zA-Z0-9_一-龥]+$", v):
            raise ValueError("用户名只能包含字母、数字、下划线和中文")
        return v

    # mode="before"：在 Pydantic 内置验证之前执行
    @field_validator("password", mode="before")
    @classmethod
    def validate_password_before(cls, v):
        """传入原始值时先处理"""
        if isinstance(v, str) and len(v) < 6:
            raise ValueError("密码至少6个字符")
        return v

    # mode="after"（默认）：在 Pydantic 内置验证之后执行
    @field_validator("password")
    @classmethod
    def validate_password_after(cls, v: str) -> str:
        if not any(c.isdigit() for c in v):
            raise ValueError("密码必须包含数字")
        if not any(c.isupper() for c in v):
            raise ValueError("密码必须包含大写字母")
        return v

    # mode="wrap"：包裹模式，可以自定义验证流程
    @field_validator("confirm_password", mode="wrap")
    @classmethod
    def validate_confirm(cls, v, handler, info):
        """先执行默认验证，再执行自定义逻辑"""
        # handler 执行内置验证
        validated = handler(v)
        # 访问其他字段
        values = info.data
        if "password" in values and validated != values["password"]:
            raise ValueError("两次密码不一致")
        return validated

    # mode="plain"：完全替代默认验证
    @field_validator("username", mode="plain")
    @classmethod
    def validate_username_plain(cls, v):
        """完全自定义验证，不执行内置类型转换"""
        if isinstance(v, str):
            v = v.strip()
        return v
```

#### 2.4.2 @model_validator

```python
from pydantic import BaseModel, model_validator
from typing import Any

class OrderCreate(BaseModel):
    quantity: int
    unit_price: float
    total_amount: float | None = None

    # mode="before"：验证整个输入字典
    @model_validator(mode="before")
    @classmethod
    def check_input_type(cls, data: Any) -> Any:
        """验证前检查：解析或转换输入格式"""
        if isinstance(data, dict) and "total" in data:
            data["total_amount"] = data.pop("total")
        return data

    # mode="after"：验证后处理（可以访问已验证的字段）
    @model_validator(mode="after")
    def calculate_total(self) -> "OrderCreate":
        """自动计算总价"""
        if self.total_amount is None:
            self.total_amount = self.quantity * self.unit_price
        return self
```text

### 2.5 序列化

```python
from pydantic import BaseModel, field_serializer, model_serializer
from datetime import datetime
from typing import Any

class EventResponse(BaseModel):
    id: int
    title: str
    price: float
    created_at: datetime

    # 字段序列化器：自定义输出格式
    @field_serializer("created_at")
    def serialize_datetime(self, v: datetime, _info) -> str:
        return v.strftime("%Y-%m-%d %H:%M:%S")

    @field_serializer("price")
    def serialize_price(self, v: float, _info) -> str:
        return f"¥{v:.2f}"

class OrderResponse(BaseModel):
    id: int
    total_amount: float
    items: list[dict]

    # 模型级序列化器：完全控制输出
    @model_serializer
    def serialize_model(self) -> dict[str, Any]:
        return {
            "order_id": self.id,
            "amount": self.total_amount,
            "item_count": len(self.items),
        }
```

**model_dump 与 model_dump_json：**

```python
user = UserCreate(username="张三", email="test@example.com", password="Test1234")

# 序列化为字典
user.model_dump()
# {"username": "张三", "email": "test@example.com", "password": "Test1234"}

# 排除某些字段
user.model_dump(exclude={"password"})
# {"username": "张三", "email": "test@example.com"}

# 只包含某些字段
user.model_dump(include={"username", "email"})
# {"username": "张三", "email": "test@example.com"}

# 序列化为 JSON（自动处理 datetime/Decimal 等）
user.model_dump_json(indent=2)
# {
#   "username": "张三",
#   "email": "test@example.com",
#   "password": "Test1234"
# }

# 排除未设置的值
class Profile(BaseModel):
    name: str
    bio: str | None = None

p = Profile(name="张三")
p.model_dump(exclude_unset=True)
# {"name": "张三"}  # bio 被排除（因为没设置）

p.model_dump(exclude_none=True)
# {"name": "张三"}  # bio 被排除（因为为 None）
```text

### 2.6 @computed_field

```python
from pydantic import BaseModel, computed_field

class OrderResponse(BaseModel):
    quantity: int
    unit_price: float

    @computed_field
    @property
    def total_amount(self) -> float:
        """计算字段：不在输入中，只在输出中出现"""
        return round(self.quantity * self.unit_price, 2)

    @computed_field(alias="totalAmount")
    @property
    def total_amount_camel(self) -> float:
        return round(self.quantity * self.unit_price, 2)

order = OrderResponse(quantity=3, unit_price=680)
print(order.model_dump())
# {"quantity": 3, "unit_price": 680.0, "total_amount": 2040.0, "totalAmount": 2040.0}
```

### 2.7 Pydantic 全部类型

```python
from pydantic import BaseModel
from typing import Annotated
from pydantic import (
    EmailStr,
    SecretStr,
    HttpUrl,
    PastDate,
    FutureDate,
    UUID4,
    Color,
    IPvAnyAddress,
    ConstrainedStr,
    constr,
    conint,
    confloat,
    condecimal,
    Field,
)

# 常用类型
class UserModel(BaseModel):
    # 字符串
    email: EmailStr                              # 自动验证邮箱格式
    password: SecretStr                          # 密码（repr 时不显示原文）

    # URL
    website: HttpUrl | None = None               # 自动验证 URL 格式
    avatar: HttpUrl | None = None

    # 日期
    birthday: PastDate | None = None             # 必须是过去的日期
    event_date: FutureDate                       # 必须是未来的日期

    # UUID
    token: UUID4                                 # UUID v4

    # 颜色
    favorite_color: Color | None = None          # CSS 颜色

    # 网络
    ip_address: IPvAnyAddress                    # IP v4 或 v6
    port: conint(ge=1, le=65535)                # 约束整数：1-65535

    # 约束字符串（方法一：ConstrainedStr 子类）
    class PhoneNumber(ConstrainedStr):
        min_length = 11
        max_length = 11
        pattern = r"^1[3-9]\d{9}$"
    phone: PhoneNumber | None = None

    # 约束字符串（方法二：constr）
    username: constr(min_length=2, max_length=50, pattern=r"^\w+$")

    # 约束数值
    age: conint(ge=0, le=150)                    # 0-150 的整数
    score: confloat(ge=0.0, le=100.0)            # 0-100 的浮点数
    price: condecimal(max_digits=10, decimal_places=2)  # 精确金额

    # 约束列表
    tags: list[str] = []
    scores: list[conint(ge=0, le=100)] = []
```text

### 2.8 泛型模型

```python
from pydantic import BaseModel
from typing import Generic, TypeVar, List

# 定义泛型类型参数
T = TypeVar("T")
ID = TypeVar("ID")

class PaginatedResponse(BaseModel, Generic[T]):
    """分页响应的泛型模型"""
    items: List[T]
    total: int
    page: int
    page_size: int
    total_pages: int

# 不同类型的响应
class UserOut(BaseModel):
    id: int
    username: str

class EventOut(BaseModel):
    id: int
    title: str
    price: float

# 使用
user_page: PaginatedResponse[UserOut]  # UserOut 列表的分页响应
event_page: PaginatedResponse[EventOut]  # EventOut 列表的分页响应
```

### 2.9 判别联合（Discriminated Union）

```python
from pydantic import BaseModel, Field
from typing import Literal, Union

# 不同的支付方式有不同的参数
class AlipayPayment(BaseModel):
    method: Literal["alipay"]
    app_id: str
    order_string: str

class WechatPayment(BaseModel):
    method: Literal["wechat"]
    app_id: str
    open_id: str
    prepay_id: str

class WalletPayment(BaseModel):
    method: Literal["wallet"]
    balance: float

# 判别联合
Payment = Annotated[
    Union[AlipayPayment, WechatPayment, WalletPayment],
    Field(discriminator="method"),
]

class OrderCreate(BaseModel):
    event_id: int
    quantity: int
    payment: Payment  # 根据 method 字段自动判断类型

# 使用
order = OrderCreate(
    event_id=1,
    quantity=2,
    payment={"method": "alipay", "app_id": "xxx", "order_string": "xxx"},
)
print(type(order.payment).__name__)  # AlipayPayment
```text

### 2.10 TypeAdapter 和 RootModel

```python
from pydantic import TypeAdapter, RootModel

# TypeAdapter：直接验证一个类型（不创建模型）
IntList = TypeAdapter(list[int])
result = IntList.validate_python([1, 2, 3])
print(result)  # [1, 2, 3]

# 验证原始类型
StringMap = TypeAdapter(dict[str, int])
data = StringMap.validate_python({"a": 1, "b": 2})

# RootModel：模型只有一个字段
class OrderIDs(RootModel):
    root: list[int]

oids = OrderIDs([1, 2, 3])
print(oids.root)  # [1, 2, 3]
```

### 2.11 Pydantic 错误处理

```python
from pydantic import ValidationError

try:
    UserCreate(
        username="A",            # 太短
        email="invalid-email",   # 不是邮箱
        password="short",        # 太短
        age=200,                 # 超范围
    )
except ValidationError as e:
    print(e.errors())
    # 输出详细错误列表：
    # [
    #   {"type": "string_too_short", "loc": ("username",), "msg": "...", "input": "A"},
    #   {"type": "value_error", "loc": ("email",), "msg": "...", "input": "invalid"},
    #   ...
    # ]
```text

### 2.12 model_validate 从 ORM 对象创建

```python
from pydantic import BaseModel, ConfigDict

# FastAPI 响应模型的标准做法
class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)  # 关键：允许从 ORM 对象创建

    id: int
    username: str
    email: str

# 从 SQLAlchemy ORM 对象创建
# user_orm = db_session.get(User, 1)  # SQLAlchemy 模型对象
# user_resp = UserResponse.model_validate(user_orm)
```

---

## 3. FastAPI 完全教程

### 3.1 创建 FastAPI 应用

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

# ========== lifespan（应用生命周期） ==========
@asynccontextmanager
async def lifespan(app: FastAPI):
    """管理应用启动和关闭时的资源"""
    # 启动时执行
    print("🚀 应用启动中...")
    # 连接数据库、Redis、加载配置等
    yield
    # 关闭时执行
    print("👋 应用关闭中...")
    # 关闭连接池、清理资源

# ========== 创建应用 ==========
app = FastAPI(
    title="TicketBooking",          # API 标题
    description="票务系统 API 文档", # API 描述
    version="1.0.0",                # 版本
    lifespan=lifespan,              # 生命周期
    docs_url="/docs",               # Swagger UI 地址
    redoc_url="/redoc",             # ReDoc 地址
    openapi_url="/openapi.json",    # OpenAPI JSON 地址
)

# ========== 路径操作 ==========
@app.get("/")
async def root():
    return {"message": "票务系统 API 服务"}

@app.get("/health")
async def health():
    return {"status": "ok", "service": "TicketBooking"}
```text

**FastAPI 应用参数速查表：**

| 参数 | 默认值 | 说明 |
| ------ | -------- | ------ |
| `title` | "FastAPI" | API 标题 |
| `description` | None | API 描述 |
| `version` | "0.1.0" | API 版本 |
| `lifespan` | None | 生命周期函数 |
| `docs_url` | "/docs" | Swagger UI |
| `redoc_url` | "/redoc" | ReDoc |
| `openapi_url` | "/openapi.json" | OpenAPI Schema |
| `root_path` | "" | 反向代理路径前缀 |
| `servers` | [] | 服务器列表 |
| `dependencies` | [] | 全局依赖注入 |
| `default_response_class` | JSONResponse | 默认响应类 |
| `swagger_ui_parameters` | {} | Swagger UI 参数 |

### 3.2 路径操作装饰器

```python
from fastapi import FastAPI, status

app = FastAPI()

# HTTP 方法映射
@app.get("/items/{id}")        # GET 请求
@app.post("/items")            # POST 请求
@app.put("/items/{id}")        # PUT 请求
@app.patch("/items/{id}")      # PATCH 请求
@app.delete("/items/{id}")     # DELETE 请求
@app.head("/items/{id}")       # HEAD 请求
@app.options("/items")         # OPTIONS 请求

# 路径操作装饰器全部参数
@app.post(
    "/items",
    # ===== 响应相关 =====
    response_model=ItemOut,           # 响应模型（自动过滤/转换）
    response_model_exclude_unset=True, # 排除未设置的字段
    response_model_exclude_none=True,  # 排除 None 字段
    status_code=status.HTTP_201_CREATED,  # 响应状态码
    response_description="创建成功",   # 响应描述

    # ===== 标签和分类 =====
    tags=["items"],                   # 标签（用于文档分组）
    summary="创建新物品",              # 摘要
    description="完整的创建物品描述",   # 描述（支持 Markdown）

    # ===== 弃用标记 =====
    deprecated=False,                 # 是否标记为弃用

    # ===== 响应头 =====
    response_headers={
        "X-Custom-Header": "value",
    },

    # ===== 回调函数 =====
    callbacks=None,                   # OpenAPI 回调
)
async def create_item():
    return {"message": "created"}
```

### 3.3 路径参数

```python
from fastapi import FastAPI, Path

app = FastAPI()

# 基本路径参数
@app.get("/events/{event_id}")
async def get_event(event_id: int):  # 类型自动转换 + 验证
    return {"event_id": event_id}

# 路径约束
@app.get("/items/{item_id}")
async def get_item(
    item_id: int = Path(
        ...,                         # 必填
        title="The ID of the item",  # 描述
        ge=1,                        # >= 1
        le=1000,                     # <= 1000
        example=42,                  # 示例值
    ),
):
    return {"item_id": item_id}

# 路径枚举
from enum import Enum

class EventCategory(str, Enum):
    concert = "concert"
    sports = "sports"
    theater = "theater"
    other = "other"

@app.get("/events/category/{category}")
async def get_events_by_category(category: EventCategory):
    if category == EventCategory.concert:
        return {"category": "演唱会"}
    return {"category": category.value}
```text

### 3.4 查询参数

```python
from fastapi import FastAPI, Query
from typing import Optional

app = FastAPI()

# 基本查询参数
@app.get("/events/")
async def list_events(
    category: str | None = None,    # 可选：/events/?category=concert
    min_price: float | None = None,  # 可选
    max_price: float | None = None,
    status: str = "published",       # 默认值
):
    return {
        "category": category,
        "min_price": min_price,
        "max_price": max_price,
        "status": status,
    }

# 查询参数验证
@app.get("/items/")
async def list_items(
    q: str = Query(
        None,                       # 可选
        min_length=3,               # 最小长度
        max_length=50,              # 最大长度
        pattern=r"^[a-zA-Z0-9]+$", # 正则
        description="搜索关键字",
        deprecated=False,
    ),
    page: int = Query(
        1, ge=1, description="页码",
    ),
    size: int = Query(
        20, ge=1, le=100, description="每页条数",
    ),
    sort: str = Query(
        "created_at",
        pattern=r"^(created_at|price|title)$",
    ),
):
    return {"q": q, "page": page, "size": size, "sort": sort}

# 列表查询参数
@app.get("/events/by_ids/")
async def get_events_by_ids(
    ids: list[int] = Query(..., description="活动ID列表"),
):
    return {"ids": ids}
# 请求：/events/by_ids/?ids=1&ids=2&ids=3

# 布尔查询参数
@app.get("/items/")
async def list_items(
    include_deleted: bool = Query(False, alias="include_deleted"),
    # 别名：客户端可以传 ?include_deleted=true
):
    pass
```

### 3.5 请求体

```python
from fastapi import FastAPI, Body
from pydantic import BaseModel

app = FastAPI()

# Pydantic 模型作为请求体
class EventCreate(BaseModel):
    title: str
    description: str | None = None
    price: float
    total_stock: int

@app.post("/events/")
async def create_event(event: EventCreate):
    """FastAPI 自动验证请求体 JSON 是否符合 EventCreate 模型"""
    return {
        "title": event.title,
        "price": event.price,
        "stock": event.total_stock,
    }

# 多个请求体参数
class UserCreate(BaseModel):
    username: str
    email: str

class ProfileCreate(BaseModel):
    bio: str
    avatar: str | None = None

@app.post("/users/")
async def create_user(
    user: UserCreate,
    profile: ProfileCreate,  # FastAPI 自动区分两个模型
):
    return {"user": user, "profile": profile}

# Body 参数：单值作为请求体
@app.post("/items/{item_id}")
async def update_item(
    item_id: int,
    importance: int = Body(..., ge=1, le=5),
    q: str | None = None,
):
    return {"item_id": item_id, "importance": importance}

# 嵌入参数
@app.post("/events/")
async def create_event(
    event: EventCreate = Body(..., embed=True),
):
    # 请求体：{"event": {"title": "...", "price": 100}}
    pass
```text

### 3.6 表单和文件

```python
from fastapi import FastAPI, Form, File, UploadFile

app = FastAPI()

# 表单数据
@app.post("/login/")
async def login(
    username: str = Form(...),  # application/x-www-form-urlencoded
    password: str = Form(...),
):
    return {"username": username}

# 单文件上传
@app.post("/upload/")
async def upload_file(
    file: UploadFile = File(...),  # multipart/form-data
):
    content = await file.read()  # 读取文件内容
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(content),
    }

# 多文件上传
@app.post("/upload_multiple/")
async def upload_files(
    files: list[UploadFile] = File(...),
):
    return [f.filename for f in files]

# 表单 + 文件混合
@app.post("/create_event/")
async def create_event(
    title: str = Form(...),
    price: float = Form(...),
    cover_image: UploadFile | None = File(None),
):
    return {"title": title, "price": price}
```

### 3.7 Cookie 和 Header

```python
from fastapi import FastAPI, Cookie, Header

app = FastAPI()

@app.get("/items/")
async def read_items(
    # Cookie 参数
    session_id: str | None = Cookie(None),
    # Header 参数
    user_agent: str | None = Header(None),
    # 自定义头
    x_token: str | None = Header(None, alias="X-Token"),
    # 重复头（逗号分隔的值用 list 接收）
    x_tags: list[str] | None = Header(None, alias="X-Tags"),
):
    return {
        "session_id": session_id,
        "user_agent": user_agent,
        "x_token": x_token,
        "x_tags": x_tags,
    }
```text

### 3.8 响应类型

```python
from fastapi import FastAPI, Response
from fastapi.responses import (
    JSONResponse,
    HTMLResponse,
    PlainTextResponse,
    RedirectResponse,
    StreamingResponse,
    FileResponse,
)
from fastapi import status

app = FastAPI()

# 默认 JSON 响应
@app.get("/json/")
async def json_response():
    return {"message": "Hello"}

# 指定状态码
@app.post("/items/", status_code=status.HTTP_201_CREATED)
async def create_item():
    return {"message": "created"}

# HTML 响应
@app.get("/html/", response_class=HTMLResponse)
async def html_response():
    return "<h1>Hello World</h1>"

# 纯文本
@app.get("/text/", response_class=PlainTextResponse)
async def text_response():
    return "Hello, World!"

# 重定向
@app.get("/redirect/")
async def redirect():
    return RedirectResponse(url="/new-url")

# 流式响应（大文件/SSE）
@app.get("/stream/")
async def stream_response():
    async def generate():
        for i in range(10):
            yield f"data: chunk {i}\n\n"
    return StreamingResponse(
        generate(),
        media_type="text/event-stream",
    )

# 文件响应
@app.get("/download/")
async def download():
    return FileResponse(
        "path/to/file.pdf",
        filename="document.pdf",
        media_type="application/pdf",
    )

# 自定义 Response
@app.get("/custom/")
async def custom_response():
    content = b"custom binary data"
    return Response(
        content=content,
        media_type="application/octet-stream",
        headers={"X-Custom": "value"},
    )
```

### 3.9 异常处理

```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse
from fastapi.exception_handlers import (
    http_exception_handler,
    request_validation_exception_handler,
)
from starlette.exceptions import HTTPException as StarletteHTTPException
from pydantic import ValidationError

app = FastAPI()

# ========== HTTPException（标准用法） ==========
@app.get("/items/{item_id}")
async def get_item(item_id: int):
    if item_id < 1:
        raise HTTPException(
            status_code=400,
            detail="item_id 必须大于 0",  # 错误信息
            headers={"X-Error": "Invalid ID"},  # 自定义响应头
        )
    if item_id > 1000:
        raise HTTPException(
            status_code=404,
            detail="物品不存在",
        )
    return {"item_id": item_id}

# ========== 自定义异常 ==========
class InsufficientStockError(Exception):
    def __init__(self, available: int, requested: int):
        self.available = available
        self.requested = requested

@app.exception_handler(InsufficientStockError)
async def insufficient_stock_handler(request: Request, exc: InsufficientStockError):
    return JSONResponse(
        status_code=409,
        content={
            "error": "库存不足",
            "available": exc.available,
            "requested": exc.requested,
        },
    )

@app.post("/orders/")
async def create_order():
    # 触发自定义异常
    raise InsufficientStockError(available=5, requested=10)

# ========== 覆盖默认异常处理器 ==========
@app.exception_handler(StarletteHTTPException)
async def custom_http_exception_handler(request, exc):
    print(f"HTTP 异常: {exc.status_code}")
    return await http_exception_handler(request, exc)

@app.exception_handler(ValidationError)
async def validation_exception_handler(request, exc):
    return JSONResponse(
        status_code=422,
        content={"error": "数据验证失败", "details": exc.errors()},
    )

# ========== 全局异常处理器 ==========
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    """兜底：捕获所有未处理的异常"""
    return JSONResponse(
        status_code=500,
        content={"error": "服务器内部错误", "detail": str(exc)},
    )
```text

### 3.10 依赖注入（Depends）

依赖注入是 FastAPI 最强大的特性之一。

```python
from fastapi import FastAPI, Depends, Query

app = FastAPI()

# ========== 函数依赖 ==========
async def common_parameters(
    q: str | None = None,
    page: int = Query(1, ge=1),
    size: int = Query(20, ge=1, le=100),
):
    """公共查询参数"""
    return {"q": q, "page": page, "size": size}

@app.get("/events/")
async def list_events(params: dict = Depends(common_parameters)):
    return params

@app.get("/orders/")
async def list_orders(params: dict = Depends(common_parameters)):
    return params

# ========== 类依赖 ==========
class Pagination:
    def __init__(self, page: int = Query(1, ge=1), size: int = Query(20, ge=1, le=100)):
        self.page = page
        self.size = size
        self.offset = (page - 1) * size

@app.get("/items/")
async def list_items(pagination: Pagination = Depends()):
    # FastAPI 自动识别 Depends() 无参数 = Depends(Pagination)
    return {"page": pagination.page, "offset": pagination.offset}

# ========== 生成器依赖（带资源清理） ==========
async def get_db():
    """数据库会话依赖，自动管理事务边界"""
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()

@app.get("/users/{user_id}")
async def get_user(user_id: int, session=Depends(get_db)):
    user = await session.get(User, user_id)
    return user

# ========== 嵌套依赖 ==========
def verify_token(token: str = Header(...)):
    """验证 token"""
    if token != "valid-token":
        raise HTTPException(status_code=401)
    return {"user_id": 1, "role": "admin"}

def verify_admin(auth: dict = Depends(verify_token)):
    """验证管理员权限（在 verify_token 基础上）"""
    if auth["role"] != "admin":
        raise HTTPException(status_code=403)
    return auth

@app.get("/admin/events/")
async def admin_list_events(auth: dict = Depends(verify_admin)):
    return {"admin": auth["user_id"]}

# ========== 全局依赖 ==========
from fastapi import FastAPI, Depends

app = FastAPI(dependencies=[Depends(verify_token)])  # 所有路由都需要 token

# ========== 路由级依赖 ==========
router = APIRouter(dependencies=[Depends(verify_admin)])
```

### 3.11 Annotated 模式（推荐）

FastAPI 从 2.0+ 强烈推荐使用 `Annotated` 模式：

```python
from typing import Annotated
from fastapi import FastAPI, Depends, Query, Path, Header

app = FastAPI()

# 定义类型别名
PaginationDep = Annotated[
    dict,
    Depends(common_parameters)
]

DbSessionDep = Annotated[
    AsyncSession,
    Depends(get_db)
]

PageQuery = Annotated[int, Query(ge=1, description="页码")]

SizeQuery = Annotated[int, Query(ge=1, le=100, description="每页条数")]

AuthDep = Annotated[dict, Depends(verify_token)]

# 使用
@app.get("/events/")
async def list_events(
    params: PaginationDep,
    session: DbSessionDep,
    auth: AuthDep,
):
    ...

@app.get("/users/{user_id}")
async def get_user(
    user_id: Annotated[int, Path(ge=1)],
    session: DbSessionDep,
):
    ...

# 推荐的 Annotated 模式 = 更少的重复代码
```text

### 3.12 中间件

```python
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware
import time

app = FastAPI()

# ========== 自定义中间件（函数方式） ==========
@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    """计算请求处理时间"""
    start_time = time.time()
    response = await call_next(request)  # 调用下一个中间件或路由
    process_time = time.time() - start_time
    response.headers["X-Process-Time"] = str(process_time)
    return response

# ========== 自定义中间件（类方式） ==========
class RequestLogMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # 请求前
        print(f"→ {request.method} {request.url.path}")

        response = await call_next(request)

        # 请求后
        print(f"← {response.status_code}")
        return response

app.add_middleware(RequestLogMiddleware)

# ========== CORS 中间件 ==========
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],                        # 允许的来源
    allow_credentials=True,                     # 允许 Cookie
    allow_methods=["*"],                        # 允许的 HTTP 方法
    allow_headers=["*"],                        # 允许的请求头
    expose_headers=["X-Process-Time"],          # 暴露给客户端的响应头
    max_age=600,                                # 预检请求缓存时间（秒）
)

# ========== 其他常用中间件 ==========
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware.gzip import GZipMiddleware
from starlette.middleware.httpsredirect import HTTPSRedirectMiddleware

# 信任的主机（防止 Host 头攻击）
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["localhost", "127.0.0.1", "*.example.com"],
)

# GZip 压缩（响应体压缩）
app.add_middleware(GZipMiddleware, minimum_size=1000)  # 超过 1KB 的响应才压缩

# HTTPS 重定向（生产环境）
# app.add_middleware(HTTPSRedirectMiddleware)
```

### 3.13 BackgroundTasks

```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

# 后台任务函数
def send_email(email: str, subject: str, body: str):
    """发送邮件（同步函数也可以）"""
    print(f"发送邮件到 {email}: {subject}")

async def write_log(user_id: int, action: str):
    """写入日志（异步函数也可以）"""
    print(f"用户 {user_id} 执行了 {action}")

@app.post("/users/{user_id}/register")
async def register(user_id: int, tasks: BackgroundTasks):
    # 注册用户（主逻辑）
    # ...

    # 添加后台任务
    tasks.add_task(send_email, "user@example.com", "欢迎注册", "...")
    tasks.add_task(write_log, user_id, "register")

    return {"message": "注册成功"}

# 注意：BackgroundTasks 在响应发送后执行
# 如果需要在 WebSocket 或长连接中使用，用 asyncio.create_task
```text

### 3.14 StaticFiles

```python
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles

app = FastAPI()

# 挂载静态文件目录
app.mount("/static", StaticFiles(directory="static"), name="static")

# 访问：http://localhost:8000/static/image.png
# 注意：mount 的路径优先级最低，放在最后
```

---

## 4. 项目：票务系统 Web 框架搭建

### 4.1 更新 main.py

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import settings
from app.routers import api_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    """应用生命周期管理"""
    # 启动
    print(f"🚀 {settings.APP_NAME} 启动中...")
    # 可在此处初始化数据库连接池、Redis 等
    yield
    # 关闭
    print("👋 应用关闭中...")


app = FastAPI(
    title=settings.APP_NAME,
    description="票务系统 API 文档 — 支持活动管理、购票、支付等完整功能",
    version="1.0.0",
    lifespan=lifespan,
    docs_url="/docs",
    redoc_url="/redoc",
)

# 中间件：CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


# 根路径
@app.get("/")
async def root():
    return {
        "service": settings.APP_NAME,
        "version": "1.0.0",
        "docs": "/docs",
        "health": "/health",
    }


# 健康检查
@app.get("/health")
async def health():
    return {
        "status": "ok",
        "service": settings.APP_NAME,
        "env": settings.ENV,
    }


# 注册路由
app.include_router(api_router, prefix="/api/v1")
```text

### 4.2 创建路由器

**app/routers/**init**.py：**

```python
from fastapi import APIRouter

from app.routers import events, auth, orders, payments

api_router = APIRouter()

# 注册子路由
api_router.include_router(auth.router, prefix="/auth", tags=["认证"])
api_router.include_router(events.router, prefix="/events", tags=["活动"])
api_router.include_router(orders.router, prefix="/orders", tags=["订单"])
api_router.include_router(payments.router, prefix="/payments", tags=["支付"])
```

**app/routers/events.py：**

```python
from fastapi import APIRouter, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession
from typing import Annotated

from app.database import get_db
from app.schemas.event import EventResponse, EventCreate, EventListResponse
from app.services.event_service import EventService

router = APIRouter()
DbSession = Annotated[AsyncSession, Depends(get_db)]


@router.get("/", response_model=EventListResponse)
async def list_events(
    session: DbSession,
    page: int = Query(1, ge=1),
    size: int = Query(20, ge=1, le=100),
    category: str | None = None,
    status: str = "published",
):
    """获取活动列表（分页）"""
    service = EventService(session)
    return await service.list_events(page=page, size=size, category=category, status=status)


@router.get("/{event_id}", response_model=EventResponse)
async def get_event(event_id: int, session: DbSession):
    """获取活动详情"""
    service = EventService(session)
    return await service.get_event(event_id)
```text

**app/routers/auth.py：**

```python
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from typing import Annotated

from app.database import get_db
from app.schemas.auth import UserCreate, UserResponse, TokenResponse, LoginRequest

router = APIRouter()
DbSession = Annotated[AsyncSession, Depends(get_db)]


@router.post("/register", response_model=UserResponse)
async def register(data: UserCreate, session: DbSession):
    """用户注册"""
    ...


@router.post("/login", response_model=TokenResponse)
async def login(data: LoginRequest, session: DbSession):
    """用户登录"""
    ...
```

### 4.3 运行并验证

```bash
# 启动开发服务器
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# 在浏览器中访问：
# Swagger 文档：http://localhost:8000/docs
# ReDoc 文档：  http://localhost:8000/redoc
# 健康检查：    http://localhost:8000/health
```text

**验证结果：**

```bash
# 1. 健康检查
curl http://localhost:8000/health
# 预期：{"status":"ok","service":"TicketBooking","env":"development"}

# 2. API 根路径
curl http://localhost:8000/
# 预期：{"service":"TicketBooking","version":"1.0.0","docs":"/docs","health":"/health"}

# 3. Swagger 文档（在浏览器中打开）
open http://localhost:8000/docs

# 4. OpenAPI JSON
curl http://localhost:8000/openapi.json
```

---

## 5. 常见问题与排错

| 问题 | 原因 | 解决 |
| ------ | ------ | ------ |
| `uvicorn` 命令找不到 | 虚拟环境未激活 | 激活虚拟环境 `.venv\Scripts\Activate.ps1` |
| `ImportError: cannot import name 'app'` | 运行目录不对 | 在项目根目录运行 `uvicorn app.main:app` |
| Swagger 页面空白 | CDN 资源被墙 | 设置 `swagger_ui_parameters={"url": "/openapi.json"}` |
| CORS 错误（前端跨域） | 后端未配置 CORS | 添加 CORSMiddleware，allow_origins=["*"] |
| `422 Validation Error` | 请求体不匹配 Pydantic 模型 | 检查 JSON 字段名和类型 |
| `custom_openapi` 配置不生效 | OpenAPI 生成顺序问题 | 在路由注册后设置 |

---

---

## 📁 项目代码参考

| 文件 | 说明 |
| ------ | ------ |
| `app/main.py` | FastAPI 入口：中间件注册顺序、lifespan、路由器挂载 |
| `app/middleware/logging.py` | 请求日志中间件（记录方法/路径/状态码/耗时） |
| `app/middleware/security.py` | 安全标头中间件（CSP/HSTS/X-Frame-Options） |
| `app/middleware/validation.py` | 请求体大小校验中间件 |
| `app/middleware/ratelimit.py` | 全局限流中间件（基于 Redis 令牌桶） |
| `app/config.py` | 配置管理（`CORS_ORIGINS`、`SHOW_DOCS` 等） |

**中间件栈执行顺序**（定义在 `app/main.py`）：

1. CORSMiddleware → TrustedHostMiddleware → LoggingMiddleware → SecurityHeadersMiddleware → RequestValidationMiddleware

## 6. 本章总结

✅ **已掌握：**

- ASGI 协议原理与 Uvicorn 全部参数
- Pydantic v2：BaseModel、Field、验证器、序列化、判别联合、泛型模型
- FastAPI：路径操作、参数（路径/查询/请求体/表单/文件）、响应、异常处理
- 依赖注入（Depends）、Annotated 模式、中间件、BackgroundTasks
- 票务系统 Web 框架搭建：main.py、路由注册、Swagger 自动文档

✅ **项目里程碑：**

- [ ] FastAPI 应用启动成功
- [ ] `/health` 和 `/` 端点可访问
- [ ] Swagger `/docs` 页面正常显示
- [ ] CORS 配置正确
- [ ] 路由按模块注册（auth/events/orders/payments）

**下一章预告：** 第5章将深入学习 REST API 设计原则，实现活动管理的完整 CRUD，包括 Service 层、Repository 模式、分页、排序和过滤。

**练习：**

1. 完成本章所有路由文件的创建
2. 在 Swagger 页面测试所有端点
3. 添加一个自定义中间件记录请求时间
4. 添加一个全局异常处理器，统一错误格式
5. 使用 BackgroundTasks 实现访问日志记录
