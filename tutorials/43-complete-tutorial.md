# Ticket-Sell 完整开发教程

> **目标读者**: 刚学完 Python 基础、想了解一个完整项目如何从零开发到上线的开发者。
> **前置知识**: Python 基础语法、HTTP 基本概念。
> **学习路线**: 按章节顺序阅读，每章都以前一章为基础。

---

## 目录

1. [项目环境搭建](#1-项目环境搭建)
2. [项目结构总览](#2-项目结构总览)
3. [ORM 模型层详解](#3-orm-模型层详解)
4. [Pydantic Schema 层详解](#4-pydantic-schema-层详解)
5. [数据库连接与会话管理](#5-数据库连接与会话管理)
6. [服务层（业务逻辑）详解](#6-服务层业务逻辑详解)
7. [路由层（API 端点）详解](#7-路由层api-端点详解)
8. [中间件详解](#8-中间件详解)
9. [Async/Await 异步编程模式](#9-asyncawait-异步编程模式)
10. [SQLAlchemy 2.0 ORM 模式](#10-sqlalchemy-20-orm-模式)
11. [Pydantic v2 模式详解](#11-pydantic-v2-模式详解)
12. [如何添加一个新功能](#12-如何添加一个新功能)
13. [如何编写和运行测试](#13-如何编写和运行测试)
14. [常见陷阱与避坑指南](#14-常见陷阱与避坑指南)

---

## 1. 项目环境搭建

### 1.1 前置条件

在开始之前，确保你的电脑上已经安装了以下工具：

| 工具 | 版本要求 | 验证命令 |
| ------ | --------- | --------- |
| Python | >= 3.10 | `python --version` |
| Git | 任意 | `git --version` |
| Docker | 最新 | `docker --version` |
| Docker Compose | v2+ | `docker compose version` |

### 1.2 克隆代码

```bash
git clone <仓库地址>
cd ticket-sell
```text

### 1.3 创建虚拟环境

```bash
# Windows
python -m venv .venv
.venv\Scripts\activate

# macOS / Linux
python3 -m venv .venv
source .venv/bin/activate
```

**为什么需要虚拟环境？** 每个 Python 项目依赖的包版本可能不同。虚拟环境将本项目的依赖隔离，避免与系统或其他项目的包冲突。激活后，命令行前面会出现 `(.venv)` 字样。

### 1.4 安装依赖

```bash
pip install -r requirements.txt
```text

如果下载速度慢，可配置国内镜像源：

```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

### 1.5 配置环境变量

```bash
cp .env.example .env
```text

然后用文本编辑器打开 `.env` 文件。开发环境基本可以直接使用默认值。

**重要**: `.env` 文件包含数据库密码、JWT 密钥等敏感信息，已被 `.gitignore` 排除，**切勿提交到 Git 仓库**。

### 1.6 启动基础设施（MySQL + Redis）

本项目使用 Docker Compose 一键启动 MySQL 8.0 和 Redis 7：

```bash
docker compose up -d
```

这个命令会：

1. 从 Docker Hub 下载 MySQL 8.0 和 Redis 7 镜像（仅首次执行时下载）
2. 创建两个容器：`ticket-mysql` 和 `ticket-redis`
3. 在 Docker 内部网络中运行，不绑定到宿主机端口
4. 自动执行健康检查，确保服务可用

验证服务是否启动成功：

```bash
docker ps
# 应该看到两个容器状态均为 "Up"

docker compose logs mysql
docker compose logs redis
```text

**Docker Compose 配置文件解析** (`docker-compose.yml`):

```yaml
version: "3.8"

services:
  mysql:
    image: mysql:8.0          # 使用 MySQL 8.0 镜像
    container_name: ticket-mysql  # 容器名称
    restart: unless-stopped    # 崩溃时自动重启
    expose:
      - "3306"                # 只在内部网络暴露，不绑定宿主机端口
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD:-root123}  # 从 .env 读取，默认 root123
      MYSQL_DATABASE: ${DB_NAME:-ticket_dev}        # 创建默认数据库
      MYSQL_USER: ${DB_USER:-ticket_app}            # 创建应用用户
      MYSQL_PASSWORD: ${DB_PASSWORD:-root123}       # 应用用户密码
    volumes:
      - mysql_data:/var/lib/mysql  # 持久化数据
    healthcheck:                   # 健康检查
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine      # 使用 Redis 7 Alpine 轻量版
    container_name: ticket-redis
    restart: unless-stopped
    expose:
      - "6379"
    command: redis-server --requirepass ${REDIS_PASSWORD:-redispass123} --appendonly yes
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD:-redispass123}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### 1.7 运行数据库迁移

```bash
alembic upgrade head
```text

这个命令会根据 `alembic/versions/` 目录下的迁移文件，在 MySQL 中创建项目所需的所有表结构。

**什么是 Alembic 迁移？**

- 每次修改数据库表结构（添加字段、修改类型等），都需要生成一个迁移文件
- 迁移文件记录了"从版本 A 到版本 B 需要执行的 SQL"
- `alembic upgrade head` 将所有未应用的迁移依次执行，使数据库达到最新状态

### 1.8 启动开发服务器

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

参数说明：

- `app.main:app` — 启动 `app/main.py` 文件中的 `app` 对象（FastAPI 实例）
- `--reload` — 代码变更时自动重启（仅开发环境使用）
- `--host 0.0.0.0` — 监听所有网络接口
- `--port 8000` — 监听 8000 端口

### 1.9 验证安装

打开浏览器访问 <http://localhost:8000/docs，如果看到> Swagger UI 交互式 API 文档，说明环境搭建成功。

---

## 2. 项目结构总览

### 2.1 分层架构

本项目采用**分层架构**（Layered Architecture），每层各司其职，从顶到底依次为：

```text
┌─────────────────────────────────────────┐
│             中间件层 (Middleware)          │ ← CORS、认证、限流、日志、安全头
├─────────────────────────────────────────┤
│             路由层 (Routers)              │ ← API 端点定义、HTTP 请求/响应处理
├─────────────────────────────────────────┤
│             服务层 (Services)             │ ← 核心业务逻辑
├─────────────────────────────────────────┤
│             模型层 (Models)               │ ← 数据库表结构（ORM）
├─────────────────────────────────────────┤
│             模式层 (Schemas)              │ ← API 请求/响应数据格式（Pydantic）
└─────────────────────────────────────────┘
```

### 2.2 目录结构详解

```text
ticket-sell/
├── app/                      # 后端 Python 代码
│   ├── main.py              # 应用入口：创建 FastAPI 实例、注册中间件、异常处理器
│   ├── config.py            # 配置管理：从 .env 文件读取所有配置
│   ├── database.py          # 数据库连接：SQLAlchemy 异步引擎与会话工厂
│   ├── redis_client.py      # Redis 连接：连接池管理与 FastAPI 依赖注入
│   ├── worker.py            # ARQ 后台 Worker：定时自动取消超时订单
│   ├── exceptions.py        # 自定义异常类：分层错误码映射
│   ├── models/              # ORM 模型：定义数据库表结构
│   │   ├── base.py          # 声明式基类（DeclarativeBase）
│   │   ├── user.py          # 用户模型
│   │   ├── event.py         # 活动模型（含乐观锁 version）
│   │   ├── order.py         # 订单模型（含状态机）
│   │   ├── ticket.py        # 票证模型
│   │   └── login_history.py # 登录历史模型
│   ├── schemas/             # Pydantic 模型：API 请求/响应格式
│   │   ├── user.py          # 用户相关 Schema
│   │   ├── event.py         # 活动相关 Schema
│   │   ├── order.py         # 订单相关 Schema
│   │   └── payment.py       # 支付相关 Schema
│   ├── routers/             # API 路由
│   │   ├── __init__.py      # 路由聚合
│   │   ├── auth.py          # 认证：注册/登录/刷新
│   │   ├── events.py        # 活动：CRUD + 列表
│   │   ├── orders.py        # 订单：创建/取消/列表
│   │   ├── payments.py      # 支付：支付宝支付/通知/退款
│   │   ├── admin.py         # 管理员：用户管理/封号
│   │   └── ws.py            # WebSocket：实时座位/订单更新
│   ├── services/            # 业务逻辑层
│   │   ├── auth_service.py        # 认证服务：注册/登录/令牌刷新/账户锁定
│   │   ├── order_service.py       # 订单服务：下单/取消/防超卖
│   │   ├── payment_service.py     # 支付服务：支付宝支付/通知验证/退款
│   │   ├── inventory_service.py   # 库存服务：Redis Lua 原子扣减
│   │   ├── behavior_service.py    # 行为限制：频率控制/防滥用
│   │   ├── event_service.py       # 活动服务：CRUD
│   │   ├── admin_service.py       # 管理服务：封号/解封
│   │   ├── alipay_sdk.py          # 支付宝 SDK：签名/验签
│   │   ├── connection_manager.py  # WebSocket 连接管理
│   │   └── notification_service.py # 通知服务
│   ├── middleware/          # 中间件
│   │   ├── auth.py          # JWT 认证 + RBAC 角色检查
│   │   ├── ratelimit.py     # 滑动窗口速率限制
│   │   ├── security.py      # 安全响应头
│   │   ├── logging.py       # 请求日志
│   │   └── validation.py    # 请求体大小限制
│   └── utils/               # 工具函数
│       ├── jwt.py           # JWT 令牌创建/验证/黑名单
│       ├── security.py      # bcrypt 密码哈希
│       ├── order_no.py      # 订单号生成
│       └── logging_config.py # 结构化日志配置
├── tests/                   # 测试代码
├── frontend/                # Vue 3 前端
├── nginx/                   # Nginx 配置文件
├── docker-compose.yml       # 开发环境 Docker Compose
├── docker-compose.prod.yml  # 生产环境 Docker Compose
├── .env.example             # 环境变量模板
└── requirements.txt         # Python 依赖
```

---

## 3. ORM 模型层详解

### 3.1 什么是 ORM？

ORM（Object-Relational Mapping，对象关系映射）让你可以用 Python 对象来操作数据库，而不需要写 SQL 语句。

```python
# 不用 ORM 的话，你需要写：
cursor.execute("SELECT * FROM users WHERE id = 1")
user = cursor.fetchone()

# 用 ORM 的话，你只需要写：
user = await session.get(User, 1)
```text

### 3.2 SQLAlchemy 2.0 声明式模型

本项目使用 SQLAlchemy 2.0 的新式声明式语法。每个模型类对应一张数据库表，每个类属性对应一个表字段。

**基类** (`app/models/base.py`):

```python
from sqlalchemy import MetaData
from sqlalchemy.orm import DeclarativeBase

# 命名约定：自动生成约束名称
convention = {
    "ix": "ix_%(column_0_label)s",           # 索引
    "uq": "uq_%(table_name)s_%(column_0_name)s",  # 唯一约束
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",  # 外键
    "pk": "pk_%(table_name)s",               # 主键
}

class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=convention)
```

所有模型类都继承自 `Base`。

### 3.3 各模型详解

#### 用户模型 (`app/models/user.py`)

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, index=True)
    password_hash: Mapped[str] = mapped_column(String(255))
    email: Mapped[str] = mapped_column(String(255), unique=True)
    phone: Mapped[str | None] = mapped_column(String(20), default=None)
    avatar: Mapped[str | None] = mapped_column(String(500), default=None)
    role: Mapped[str] = mapped_column(String(20), default="user")
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    is_banned: Mapped[bool] = mapped_column(Boolean, default=False)
    ban_reason: Mapped[str | None] = mapped_column(Text, default=None)
    # ...更多字段
```text

关键点：

- `Mapped[类型]` — SQLAlchemy 2.0 的类型注解语法，告诉 ORM 字段的类型
- `mapped_column(...)` — 定义列的属性（类型、约束、默认值等）
- `unique=True` — 创建唯一约束
- `index=True` — 创建数据库索引（加快查询速度）
- `ForeignKey("users.id")` — 外键关联
- `relationship(...)` — 定义对象之间的关联关系，方便跨表查询

#### 活动模型 (`app/models/event.py`) — 含乐观锁

```python
class Event(Base):
    __tablename__ = "events"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    title: Mapped[str] = mapped_column(String(200))
    # ...
    total_stock: Mapped[int] = mapped_column(Integer, default=0)  # 总库存
    sold_count: Mapped[int] = mapped_column(Integer, default=0)   # 已售数量
    price: Mapped[float] = mapped_column(Numeric(10, 2))          # 价格（精确到分）
    version: Mapped[int] = mapped_column(Integer, default=1)      # 乐观锁版本号
    # ...
```

`version` 字段是**乐观锁**的关键。当多个请求同时更新同一条记录时，乐观锁确保只有一个请求能成功。详见后续"防超卖"章节。

#### 订单模型 (`app/models/order.py`) — 含状态机

```python
class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    order_no: Mapped[str] = mapped_column(String(32), unique=True)
    user_id: Mapped[int] = mapped_column(BigInteger, ForeignKey("users.id"))
    event_id: Mapped[int] = mapped_column(BigInteger, ForeignKey("events.id"))
    quantity: Mapped[int] = mapped_column(Integer)
    total_amount: Mapped[float] = mapped_column(Numeric(10, 2))
    status: Mapped[str] = mapped_column(String(20), default="pending_payment")
    # ...
```text

订单状态机：

```

pending_payment  →  paid  →  refunded
        ↓
    cancelled

```text

允许的状态转换：

- `pending_payment → paid`：支付成功
- `pending_payment → cancelled`：用户取消或超时自动取消
- `paid → refunded`：管理员退款

#### 票证模型 (`app/models/ticket.py`)

```python
class Ticket(Base):
    __tablename__ = "tickets"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    order_id: Mapped[int] = mapped_column(BigInteger, ForeignKey("orders.id"))
    event_id: Mapped[int] = mapped_column(BigInteger, ForeignKey("events.id"))
    user_id: Mapped[int] = mapped_column(BigInteger, ForeignKey("users.id"))
    ticket_no: Mapped[str] = mapped_column(String(32), unique=True)
    # ...
```

支付成功后，系统会为订单中的每张票生成独立的 Ticket 记录。

#### 登录历史模型 (`app/models/login_history.py`)

```python
class LoginHistory(Base):
    __tablename__ = "login_histories"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    user_id: Mapped[int] = mapped_column(Integer, ForeignKey("users.id"), index=True)
    ip_address: Mapped[str] = mapped_column(String(45))
    user_agent: Mapped[str | None] = mapped_column(Text, default=None)
    login_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(3))
    success: Mapped[bool] = mapped_column(Boolean, default=True)
    fail_reason: Mapped[str | None] = mapped_column(String(255), default=None)
```text

每条登录尝试都会被记录，包括成功和失败的，用于安全审计。

### 3.4 模型间关系

```

User ──1:N──> Order ──1:N──> Ticket
  │              │
  │              └──N:1── Event
  │
  └──1:N──> LoginHistory
  
Event ──1:N──> Order
  │
  └──1:N──> Ticket

```text

用 SQLAlchemy 的关系定义：

```python
# User 模型中的关系
orders: Mapped[list[Order]] = relationship(back_populates="user")
tickets: Mapped[list[Ticket]] = relationship(back_populates="user")

# Order 模型中的关系
user: Mapped[User] = relationship(back_populates="orders")
event: Mapped[Event] = relationship(back_populates="orders")
tickets: Mapped[list[Ticket]] = relationship(
    back_populates="order",
    cascade="all, delete-orphan",  # 级联删除
)
```

---

## 4. Pydantic Schema 层详解

### 4.1 Schema 的作用

Pydantic Schema 在项目中承担三个职责：

1. **请求校验**：确保客户端发来的数据格式正确
2. **响应序列化**：将 ORM 模型转换为 JSON 响应
3. **自动文档**：FastAPI 自动从 Schema 生成 OpenAPI 文档

### 4.2 请求 Schema 示例

```python
from pydantic import BaseModel, Field, field_validator

class OrderCreate(BaseModel):
    event_id: int = Field(..., gt=0, description="活动 ID")
    quantity: int = Field(..., ge=1, le=10, description="购票数量")
```text

- `Field(...)` — `...` 表示必填，`gt=0` 表示必须大于 0
- `ge=1, le=10` — quantity 取值范围 1-10
- `description` — 用于生成 API 文档

### 4.3 响应 Schema 示例

```python
class OrderResponse(BaseModel):
    model_config = {"from_attributes": True}  # 允许从 ORM 模型创建

    id: int
    order_no: str
    user_id: int
    event_id: int
    quantity: int
    total_amount: float
    status: str
    # ...
```

- `from_attributes = True` — 关键配置，允许 `OrderResponse.model_validate(order_orm_obj)` 从 ORM 对象创建

### 4.4 字段校验器

Pydantic v2 使用 `@field_validator` 装饰器：

```python
class EventCreate(BaseModel):
    start_date: datetime
    end_date: datetime

    @field_validator("end_date")
    @classmethod
    def check_dates(cls, v, info):
        if "start_date" in info.data and v <= info.data["start_date"]:
            msg = "结束时间必须大于开始时间"
            raise ValueError(msg)
        return v

    @field_validator("category")
    @classmethod
    def check_category(cls, v):
        allowed = ["concert", "sports", "theater", "other"]
        if v not in allowed:
            msg = f"分类必须是 {allowed} 之一"
            raise ValueError(msg)
        return v
```text

### 4.5 Schema 与 ORM 的分离

**为什么需要两套模型？** ORM 模型对应数据库表，可能包含敏感字段（如 `password_hash`）。Schema 模型只暴露需要的数据。此外，Schema 还可以：

- 定义不同的请求/响应格式
- 添加字段校验规则
- 隐藏敏感字段

---

## 5. 数据库连接与会话管理

### 5.1 异步引擎与会话工厂

```python
# app/database.py
from sqlalchemy.ext.asyncio import (
    AsyncSession, async_sessionmaker, create_async_engine,
)

# 创建异步引擎（管理数据库连接池）
engine = create_async_engine(
    settings.DATABASE_URL,
    echo=settings.DB_ECHO,      # 是否打印 SQL 日志
    pool_size=10,               # 连接池大小
    max_overflow=5,             # 最大额外连接数
    pool_pre_ping=True,         # 使用前检查连接是否有效
    pool_recycle=3600,          # 连接回收时间（1小时）
)

# 创建异步会话工厂
async_session_factory = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,     # 提交后不自动过期
)
```

### 5.2 FastAPI 依赖注入

```python
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    """FastAPI 依赖：获取数据库会话"""
    async with async_session_factory() as session:
        try:
            yield session          # 将 session 注入给路由函数
            await session.commit() # 成功时自动提交
        except Exception:
            await session.rollback() # 失败时自动回滚
            raise
        finally:
            await session.close()    # 最终关闭连接
```text

**使用方式**：路由函数声明 `session: AsyncSession = Depends(get_db)`，FastAPI 自动注入。

### 5.3 Redis 连接

```python
# app/redis_client.py
_redis_pool: ConnectionPool | None = None

def get_redis_pool() -> ConnectionPool:
    """获取 Redis 连接池（单例模式）"""
    global _redis_pool
    if _redis_pool is None:
        _redis_pool = ConnectionPool.from_url(
            settings.REDIS_URL,
            max_connections=20,         # 最大连接数
            socket_timeout=5,           # Socket 超时
            socket_connect_timeout=3,   # 连接超时
            health_check_interval=30,   # 健康检查间隔
            decode_responses=True,      # 自动解码为字符串
        )
    return _redis_pool
```

---

## 6. 服务层（业务逻辑）详解

### 6.1 服务层的角色

服务层是业务逻辑的核心，它：

- 接收来自路由层的参数
- 调用模型层进行数据库操作
- 执行业务规则校验
- 调用外部服务（支付宝、Redis 等）
- 返回处理结果给路由层

### 6.2 典型服务：订单服务

```python
# app/services/order_service.py
class OrderService:
    def __init__(self, session: AsyncSession, redis: Redis) -> None:
        self.session = session
        self.redis = redis
        self.inventory = InventoryService(redis)

    async def create_order(self, user_id: int, data: OrderCreate) -> Order:
        # 1. 验证活动存在且已上架
        event = await self.session.get(Event, data.event_id)
        if not event or event.status != "published":
            raise ValueError("活动不存在或未上架")

        # 2. Redis 原子扣减库存（第一层防超卖）
        result = await self.inventory.deduct_stock(
            event_id=data.event_id, user_id=user_id,
            quantity=data.quantity, max_per_user=10,
        )
        if result != 1:
            raise ValueError("库存不足")

        try:
            # 3. 创建订单记录
            order = Order(order_no=generate_order_no(), ...)
            self.session.add(order)
            await self.session.flush()

            # 4. MySQL 乐观锁更新（第二层防超卖）
            stmt = text("""UPDATE events
                SET sold_count = sold_count + :qty, version = version + 1
                WHERE id = :eid AND version = :ver AND sold_count + :qty2 <= total_stock""")
            result = await self.session.execute(stmt, {...})
            if result.rowcount == 0:
                raise ValueError("库存不足（乐观锁拦截）")

            return order
        except Exception:
            # 失败时归还 Redis 库存
            await self.inventory.restore_stock(data.event_id, data.quantity)
            raise
```text

### 6.3 服务如何被路由调用

```python
# 路由层
@router.post("/", response_model=OrderResponse)
async def create_order(
    data: OrderCreate,
    session: AsyncSession = Depends(get_db),
    redis: Redis = Depends(get_redis),
    current_user: User = Depends(get_current_user),
):
    # 1. 行为检查（频率限制）
    behavior = BehaviorService(redis)
    allowed, wait = await behavior.check_order_frequency(current_user.id)
    if not allowed:
        raise HTTPException(status_code=429, detail=f"请 {wait} 秒后再试")

    # 2. 调用服务
    service = OrderService(session, redis)
    try:
        order = await service.create_order(current_user.id, data)
        await session.commit()
        return order
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
```

**设计原则**：

- 路由层负责 HTTP 相关的处理（参数提取、响应序列化、错误转换）
- 服务层负责纯业务逻辑（不依赖 HTTP 细节）
- 服务层抛出 `ValueError`，路由层捕获并转换为 `HTTPException`

---

## 7. 路由层（API 端点）详解

### 7.1 路由定义

```python
from fastapi import APIRouter, Depends, HTTPException

router = APIRouter()

@router.get("/events", response_model=EventListResponse)
async def list_events(
    page: int = Query(1, ge=1),
    size: int = Query(20, ge=1, le=100),
    session: AsyncSession = Depends(get_db),
):
    ...
```text

### 7.2 API 路由映射

| 路由器 | 前缀 | 端点 |
| -------- | ------ | ------ |
| auth | `/api/v1/auth` | `/register`, `/login`, `/refresh`, `/me` |
| events | `/api/v1/events` | `/`, `/{id}`, `/{id}/stock` |
| orders | `/api/v1/orders` | `/`, `/{id}`, `/{id}/cancel` |
| payments | `/api/v1/payments` | `/alipay`, `/alipay/notify`, `/refund` |
| admin | `/api/v1/admin` | `/users`, `/users/{id}/ban`, etc. |
| ws | `/api/v1` | `/shows/{id}/seats`, `/user/orders` |

### 7.3 路由聚合

```python
# app/routers/__init__.py
api_router = APIRouter()

api_router.include_router(auth.router, prefix="/auth", tags=["认证"])
api_router.include_router(events.router, prefix="/events", tags=["活动"])
api_router.include_router(orders.router, prefix="/orders", tags=["订单"])
api_router.include_router(payments.router, prefix="/payments", tags=["支付"])
api_router.include_router(ws.router, prefix="", tags=["WebSocket"])
api_router.include_router(admin.router, prefix="/admin", tags=["管理员"])
```

然后在 `main.py` 中注册到应用：

```python
app.include_router(api_router, prefix="/api/v1")
```text

---

## 8. 中间件详解

### 8.1 中间件执行顺序

在 `app/main.py` 中，中间件的注册顺序决定了它们的执行顺序。**注意**：FastAPI/Starlette 的中间件是"洋葱模型"，注册顺序决定了外层到内层的顺序：

```python
# 第一个注册的中间件是最外层的
app.add_middleware(CORSMiddleware, ...)          # 1. CORS（最先执行）
app.add_middleware(TrustedHostMiddleware, ...)    # 2. 可信主机
app.add_middleware(LoggingMiddleware)             # 3. 请求日志
app.add_middleware(SecurityHeadersMiddleware)     # 4. 安全头
app.add_middleware(RequestValidationMiddleware)    # 5. 请求验证
app.add_middleware(RateLimitMiddleware, ...)      # 6. 速率限制（最后执行，最靠近应用）
```

### 8.2 各中间件功能

| 中间件 | 功能 | 失败时 |
| -------- | ------ | -------- |
| CORS | 允许跨域请求 | 返回 CORS 错误 |
| TrustedHost | 防止 Host 头攻击 | 返回 400 |
| Logging | 记录请求日志（含耗时） | 记录异常日志 |
| SecurityHeaders | 添加安全响应头 | 不影响响应 |
| RequestValidation | 限制请求体大小（1MB） | 返回 413 |
| RateLimit | 滑动窗口限流 | 返回 429 |

### 8.3 自定义中间件示例

```python
# app/middleware/security.py
class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response: Response = await call_next(request)

        # 防止 MIME 类型嗅探
        response.headers["X-Content-Type-Options"] = "nosniff"
        # 防止点击劫持
        response.headers["X-Frame-Options"] = "DENY"
        # Content Security Policy
        response.headers["Content-Security-Policy"] = (
            "default-src 'self'; script-src 'self'; ..."
        )
        # 其他安全头...

        return response
```text

---

## 9. Async/Await 异步编程模式

### 9.1 为什么使用异步？

本项目的所有 I/O 操作（数据库查询、Redis 操作、HTTP 请求）都是异步的。异步编程允许在等待 I/O 时切换执行其他任务，提高并发处理能力。

**同步 vs 异步**：

```

同步：请求1 → DB查询(等1秒) → 响应1 → 请求2 → DB查询(等1秒) → 响应2   (总耗时：2+秒)
异步：请求1 → DB查询(等1秒) → (等待时处理请求2) → DB查询(等1秒) → ... (总耗时：~1秒)

```text

### 9.2 async/await 语法

```python
# 定义异步函数
async def get_user(user_id: int) -> User:
    # await 等待一个异步操作完成
    user = await session.get(User, user_id)
    return user

# 调用异步函数
user = await get_user(1)
```

### 9.3 异步上下文管理器

```python
# 数据库会话使用异步上下文管理器
async with async_session_factory() as session:
    user = await session.get(User, 1)

# Redis pipeline 也类似
async with self.redis.pipeline(transaction=True) as pipe:
    await pipe.watch(key)
    # ...
```text

### 9.4 并发执行任务

```python
import asyncio

# 并发执行多个独立的异步任务
results = await asyncio.gather(
    get_user(1),
    get_user(2),
    get_user(3),
)
```

### 9.5 常见模式和陷阱

**陷阱 1：在同步函数中调用异步函数**

```python
# 错误！不能在同步函数中直接 await
def get_user_sync(user_id: int):
    return await session.get(User, user_id)  # SyntaxError!

# 正确：必须定义为 async
async def get_user_sync(user_id: int):
    return await session.get(User, user_id)
```text

**陷阱 2：忘记 await**

```python
# 错误！没有 await，得到一个 coroutine 对象
result = session.execute(stmt)
# result 是 coroutine，不是查询结果

# 正确
result = await session.execute(stmt)
```

**陷阱 3：在 FastAPI 路由中混用同步和异步**

FastAPI 支持同步和异步路由函数。但如果路由是异步的，其中的 I/O 操作也必须是异步的：

```python
@router.get("/events")
async def list_events():  # 异步路由
    events = await get_all_events()  # 必须 await
    return events
```text

---

## 10. SQLAlchemy 2.0 ORM 模式

### 10.1 新式声明语法

SQLAlchemy 2.0 引入了基于类型注解的新式语法：

```python
# 1.x 旧语法
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)

# 2.0 新语法（本项目使用）
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
```

### 10.2 异步查询

```python
# 获取单个对象
user = await session.get(User, 1)

# 查询
result = await session.execute(select(User).where(User.role == "admin"))
users = result.scalars().all()

# 聚合查询
count_stmt = select(func.count(Event.id)).where(Event.status == "published")
total = await session.scalar(count_stmt)

# 分页查询
stmt = select(Event).order_by(Event.start_date).offset(0).limit(20)
result = await session.execute(stmt)
events = result.scalars().all()
```text

### 10.3 事务管理

本项目中，数据库会话的生命周期由 `get_db()` 依赖管理：

```python
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session         # 路由函数执行期间使用 session
            await session.commit() # 成功 → 提交事务
        except Exception:
            await session.rollback() # 失败 → 回滚事务
            raise
        finally:
            await session.close()    # 最终关闭连接
```

### 10.4 乐观锁

乐观锁用于解决并发更新冲突：

```python
# 更新时检查 version 字段
stmt = text("""
    UPDATE events
    SET sold_count = sold_count + :qty, version = version + 1
    WHERE id = :eid AND version = :ver AND sold_count + :qty2 <= total_stock
""")

result = await session.execute(stmt, {
    "qty": quantity, "eid": event_id,
    "ver": event.version, "qty2": quantity,
})

if result.rowcount == 0:
    # version 不匹配或库存不足 → 冲突
    raise ValueError("并发冲突，库存不足")
```text

### 10.5 N+1 查询问题

ORM 的一个常见陷阱是 N+1 查询问题：查询 N 条记录后，又对每条记录执行额外查询。

```python
# N+1 问题示例
orders = await session.execute(select(Order).limit(10))
for order in orders.scalars():  # 循环 10 次
    print(order.user.name)      # 每次触发一次额外查询！

# 解决：使用 selectinload 预加载关联对象
from sqlalchemy.orm import selectinload

stmt = select(Order).options(selectinload(Order.user)).limit(10)
orders = await session.execute(stmt)
for order in orders.scalars():
    print(order.user.name)  # 不会触发额外查询
```

---

## 11. Pydantic v2 模式详解

### 11.1 Pydantic v2 核心概念

Pydantic v2 使用 Rust 核心（pydantic-core）重写，比 v1 快 5-50 倍。

```python
from pydantic import BaseModel, Field, EmailStr, field_validator

class UserCreate(BaseModel):
    username: str = Field(
        ...,
        min_length=3,
        max_length=50,
        pattern=r"^[a-zA-Z0-9_]+$",
        description="用户名（字母数字下划线）",
    )
    password: str = Field(
        ...,
        min_length=8,
        max_length=128,
        description="密码（至少8位）",
    )
    email: EmailStr
    phone: str | None = Field(None, pattern=r"^1\d{10}$")
```text

### 11.2 Field 参数详解

| 参数 | 含义 | 示例 |
| ------ | ------ | ------ |
| `...` | 必填字段 | `Field(...)` |
| `None` / 默认值 | 可选字段 | `Field(None)` |
| `min_length` | 字符串最小长度 | `Field(min_length=3)` |
| `max_length` | 字符串最大长度 | `Field(max_length=200)` |
| `gt` / `ge` | 大于 / 大于等于 | `Field(gt=0)` |
| `lt` / `le` | 小于 / 小于等于 | `Field(le=10)` |
| `pattern` | 正则表达式 | `Field(pattern=r"^[a-z]+$")` |
| `description` | 文档描述 | `Field(description="用户名")` |

### 11.3 响应模型配置

```python
class UserResponse(BaseModel):
    model_config = {"from_attributes": True}

    id: int
    username: str
    email: str
    phone: str | None = None
    role: str
    is_active: bool
    created_at: datetime
```

- `from_attributes = True` — 关键配置！允许 `UserResponse.model_validate(db_user)` 直接从 ORM 对象创建

### 11.4 model_validate 与 model_dump

```python
# 从 ORM 对象创建 Schema 实例（替换 v1 的 from_orm）
response = EventResponse.model_validate(db_event)

# Schema 实例转字典（替换 v1 的 dict()）
data = response.model_dump()

# 排除未设置字段的字典
update_data = data.model_dump(exclude_unset=True)
```text

### 11.5 v1 到 v2 迁移要点

| v1 语法 | v2 语法 |
| --------- | --------- |
| `class Config: orm_mode = True` | `model_config = {"from_attributes": True}` |
| `obj.dict()` | `obj.model_dump()` |
| `obj.schema()` | `obj.model_json_schema()` |
| `@validator` | `@field_validator` |
| `@root_validator` | `@model_validator(mode='before')` |

---

## 12. 如何添加一个新功能

下面以"添加活动收藏功能"为例，演示完整的开发流程。

### 12.1 第一步：创建新的 ORM 模型

在 `app/models/` 下新建 `favorite.py`：

```python
from __future__ import annotations
from datetime import datetime
from sqlalchemy import DateTime, ForeignKey, Integer, UniqueConstraint, func
from sqlalchemy.orm import Mapped, mapped_column
from .base import Base

class Favorite(Base):
    __tablename__ = "favorites"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    user_id: Mapped[int] = mapped_column(Integer, ForeignKey("users.id"))
    event_id: Mapped[int] = mapped_column(Integer, ForeignKey("events.id"))
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(3)
    )

    __table_args__ = (
        UniqueConstraint("user_id", "event_id", name="uq_favorite_user_event"),
    )
```

在 `app/models/__init__.py` 中注册：

```python
from .favorite import Favorite
__all__ = ["Base", "Event", "LoginHistory", "Order", "Ticket", "User", "Favorite"]
```text

### 12.2 第二步：创建 Alembic 迁移

```bash
alembic revision --autogenerate -m "add favorites table"
```

检查生成的迁移文件，然后应用：

```bash
alembic upgrade head
```text

### 12.3 第三步：创建 Pydantic Schema

在 `app/schemas/` 下新建 `favorite.py`：

```python
from pydantic import BaseModel, Field

class FavoriteCreate(BaseModel):
    event_id: int = Field(..., gt=0, description="活动 ID")

class FavoriteResponse(BaseModel):
    model_config = {"from_attributes": True}
    id: int
    user_id: int
    event_id: int
    created_at: datetime
```

### 12.4 第四步：创建服务

在 `app/services/` 下新建 `favorite_service.py`：

```python
from sqlalchemy import select, and_
from sqlalchemy.ext.asyncio import AsyncSession

from app.models import Favorite, Event

class FavoriteService:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def add_favorite(self, user_id: int, event_id: int) -> Favorite:
        # 检查活动存在
        event = await self.session.get(Event, event_id)
        if not event:
            raise ValueError("活动不存在")

        # 检查是否已收藏
        existing = await self.session.execute(
            select(Favorite).where(
                and_(Favorite.user_id == user_id, Favorite.event_id == event_id)
            )
        )
        if existing.scalar_one_or_none():
            raise ValueError("已收藏该活动")

        favorite = Favorite(user_id=user_id, event_id=event_id)
        self.session.add(favorite)
        await self.session.flush()
        return favorite

    async def remove_favorite(self, user_id: int, event_id: int) -> None:
        result = await self.session.execute(
            select(Favorite).where(
                and_(Favorite.user_id == user_id, Favorite.event_id == event_id)
            )
        )
        favorite = result.scalar_one_or_none()
        if not favorite:
            raise ValueError("未收藏该活动")
        await self.session.delete(favorite)
        await self.session.flush()
```text

### 12.5 第五步：创建路由

在 `app/routers/` 下新建 `favorites.py`：

```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.middleware.auth import get_current_user
from app.models.user import User
from app.schemas.favorite import FavoriteCreate, FavoriteResponse
from app.services.favorite_service import FavoriteService

router = APIRouter()

@router.post("/", response_model=FavoriteResponse, status_code=201)
async def add_favorite(
    data: FavoriteCreate,
    session: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    service = FavoriteService(session)
    try:
        favorite = await service.add_favorite(current_user.id, data.event_id)
        await session.commit()
        return favorite
    except ValueError as e:
        await session.rollback()
        raise HTTPException(status_code=400, detail=str(e))

@router.delete("/{event_id}", status_code=204)
async def remove_favorite(
    event_id: int,
    session: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    service = FavoriteService(session)
    try:
        await service.remove_favorite(current_user.id, event_id)
        await session.commit()
    except ValueError as e:
        await session.rollback()
        raise HTTPException(status_code=400, detail=str(e))
```

### 12.6 第六步：注册路由

在 `app/routers/__init__.py` 中添加：

```python
from app.routers import favorites
api_router.include_router(favorites.router, prefix="/favorites", tags=["收藏"])
```text

### 12.7 第七步：编写测试

```python
# tests/test_favorites.py
from unittest.mock import AsyncMock, MagicMock
import pytest

@pytest.mark.asyncio
async def test_add_favorite():
    from app.services.favorite_service import FavoriteService

    mock_session = AsyncMock()
    mock_session.get = AsyncMock(return_value=MagicMock(spec=Event))
    mock_session.execute = AsyncMock(return_value=MagicMock(
        scalar_one_or_none=MagicMock(return_value=None)
    ))

    service = FavoriteService(mock_session)
    fav = await service.add_favorite(user_id=1, event_id=1)
    assert fav is not None
```

### 12.8 第八步：测试并提交

```bash
# 运行新功能的测试
pytest tests/test_favorites.py -v

# 运行全部测试确保没有回归
pytest

# 代码检查
ruff check .
pyright .
```text

---

## 13. 如何编写和运行测试

### 13.1 测试架构

本项目使用 pytest + unittest.mock 进行测试：

- **单元测试**：Mock 数据库和 Redis，不依赖外部服务，可以离线运行
- **集成测试**：需要真实的 MySQL 和 Redis 服务，使用 `--integration` 标记
- **并发测试**：测试防超卖逻辑在并发下的正确性

### 13.2 测试配置

`pyproject.toml` 中的 pytest 配置：

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"           # 自动识别异步测试
testpaths = ["tests"]           # 测试目录
python_files = ["test_*.py"]    # 测试文件命名模式
markers = [
    "integration: 标记集成测试，需要真实 MySQL 和 Redis",
]
```

- `asyncio_mode = "auto"` — 自动将 `async def test_*` 函数识别为异步测试

### 13.3 Mock 模式详解

**Mock 数据库会话**：

```python
from unittest.mock import AsyncMock, MagicMock

# 创建 Mock 会话
mock_session = AsyncMock()

# Mock 查询返回结果
mock_result = MagicMock()
mock_result.scalars.return_value.all.return_value = [mock_order]
mock_session.execute = AsyncMock(return_value=mock_result)

# Mock 获取单个对象
mock_session.get = AsyncMock(return_value=mock_user)
```text

**Mock Redis**：

```python
mock_redis = AsyncMock()
mock_redis.zcard = AsyncMock(return_value=3)  # 模拟 3 次请求
mock_redis.get = AsyncMock(return_value=b"2")  # 模拟 2 笔并发订单
```

**Mock HTTP 客户端**：

```python
# conftest.py 中提供 client fixture
@pytest.fixture
def client():
    """创建 Mock HTTP 客户端"""
    from httpx import ASGITransport, AsyncClient
    from app.main import app

    transport = ASGITransport(app=app)
    return AsyncClient(transport=transport, base_url="http://localhost")
```text

### 13.4 测试编写最佳实践

**原则 1：测试行为而非实现**

```python
# 好的测试 — 测试输出
def test_order_no_format():
    order_no = generate_order_no()
    assert order_no.startswith("TKT")
    assert len(order_no) == 20  # TKT + 8位日期 + 8位随机

# 不好的测试 — 测试内部实现
def test_order_no_calls_token_hex():
    with patch("secrets.token_hex") as mock:
        generate_order_no()
        mock.assert_called_once_with(4)
```

**原则 2：每个测试只测一个功能**

```python
# 好的做法 — 分开测试
def test_create_order_success(): ...
def test_create_order_insufficient_stock(): ...
def test_create_order_event_not_found(): ...

# 不好的做法 — 一个测试测多个功能
def test_create_order(): ...  # 又测成功又测失败
```text

**原则 3：使用有意义的断言消息**

```python
assert success_count <= total_stock, (
    f"超卖！成功 {success_count} 单，但库存只有 {total_stock}"
)
```

### 13.5 运行测试

```bash
# 运行所有测试
pytest

# 输出详细信息
pytest -v

# 运行特定测试文件
pytest tests/test_orders.py -v

# 运行特定测试函数
pytest tests/test_orders.py::test_create_order_validation -v

# 运行匹配名称的测试
pytest -k "order" -v

# 显示覆盖率
pytest --cov=app --cov-report=term-missing -v

# 要求覆盖率不低于 80%
pytest --cov=app --cov-fail-under=80 -v

# 运行集成测试（需 MySQL + Redis）
pytest --integration -v

# 首次失败即停止（调试用）
pytest -x
```text

---

## 14. 常见陷阱与避坑指南

### 14.1 Async/await 相关

**陷阱：在 pytest 中测试异步代码**

如果 `asyncio_mode = "auto"` 没配好，异步测试会报错：

```python
# 正确：asyncio_mode = "auto" 时自动处理
@pytest.mark.asyncio
async def test_async_function():
    result = await some_async_func()
    assert result
```

### 14.2 SQLAlchemy 相关

**陷阱：func.now() 不可与 Python datetime 直接比较**

```python
# 错误！func.now() 是 SQLAlchemy 函数对象，不是 datetime
# if event.start_date <= func.now():

# 正确：使用 Python datetime
from datetime import datetime, timezone
now = datetime.now(timezone.utc).replace(tzinfo=None)
if event.start_date <= now:
    raise ValueError("活动已开始")
```text

**陷阱：提交后对象过期**

```python
# 如果在 expire_on_commit=True 时访问属性会报错
# session.commit() 后，对象属性需要重新查询

# 本项目的 session 设置为 expire_on_commit=False 避免了此问题
```

### 14.3 Redis 相关

**陷阱：Redis 连接超时**

```python
# 生产环境应设置合理的超时和重试
ConnectionPool.from_url(
    settings.REDIS_URL,
    socket_timeout=5,          # Socket 超时 5 秒
    socket_connect_timeout=3,  # 连接超时 3 秒
    health_check_interval=30,  # 每 30 秒检查一次连接健康状态
)
```text

**陷阱：Pipeline 不支持所有模式**

```python
# inventory_service.py 中 restore_stock 使用 pipeline(transaction=True)
# 如果 Redis 集群模式，pipeline 的 WATCH 可能不支持
# 降级方案：使用简单 incrby
try:
    async with self.redis.pipeline(transaction=True) as pipe:
        await pipe.watch(key)
        # ...
except (ConnectionError, TimeoutError):
    await self.redis.incrby(key, quantity)
```

### 14.4 Pydantic 相关

**陷阱：忘记 from_attributes = True**

```python
# 错误！ORM 对象无法直接转换为 Schema
return EventResponse.model_validate(db_event)
# 报错：Input should be a valid dictionary

# 正确：配置 from_attributes = True
class EventResponse(BaseModel):
    model_config = {"from_attributes": True}
    ...
```text

**陷阱：Decimal 转 float 精度丢失**

```python
# 金额计算使用 Decimal
total_amount = Decimal(str(event.price)) * data.quantity

# 存数据库时转为 float（但计算过程保持 Decimal 精度）
order.total_amount = float(total_amount)
```

### 14.5 FastAPI 相关

**陷阱：Depends 的 B008 警告**

Ruff 的 B008 规则会警告函数调用在默认参数中：

```python
# B008 警告：Depends() 在函数定义时执行
async def endpoint(session: AsyncSession = Depends(get_db)):

# 虽然触发警告，但在 FastAPI 中这是标准用法，可以安全使用
# 只需添加 noqa 注释忽略
async def endpoint(session: AsyncSession = Depends(get_db)):  # noqa: B008
```text

### 14.6 数据库相关

**陷阱：MySQL 时区问题**

```python
# 所有时间字段使用 DateTime(timezone=True)
# 查询时使用 UTC 比较
from datetime import datetime, timezone
now = datetime.now(timezone.utc).replace(tzinfo=None)
```

**陷阱：LIKE 查询的 SQL 注入风险**

```python
# 用户输入的关键词可能包含 % 和 _ 通配符
keyword = "test%_"
# 必须转义！
safe_keyword = keyword.replace("%", "\\%").replace("_", "\\_")
conditions.append(Event.title.like(f"%{safe_keyword}%"))
```text

### 14.7 Docker 相关

**陷阱：Docker 容器重启循环**

```bash
# 查看容器日志定位问题
docker compose logs mysql
docker compose logs redis

# 检查健康检查状态
docker inspect ticket-mysql --format '{{json .State.Health}}'
```

**陷阱：端口冲突**

```bash
# 检查端口被哪个进程占用
netstat -ano | findstr :8000
netstat -ano | findstr :3306
netstat -ano | findstr :6379

# 结束占用进程
taskkill /PID <进程ID> /F
```text

### 14.8 测试相关

**陷阱：Mock 未正确模拟异步上下文管理器**

```python
# inventory_service.py 中使用 async with pipeline()
# 测试时必须正确模拟 __aenter__
mock_pipeline = AsyncMock()
mock_redis.pipeline.return_value.__aenter__.return_value = mock_pipeline
```

**陷阱：集成测试需要外部服务**

```bash
# 运行集成测试前，确保 MySQL 和 Redis 已启动
docker compose up -d

# 然后运行
pytest --integration -v
```text
