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
15. [管理后台前端开发 (Admin Dashboard Frontend)](#15-管理后台前端开发-admin-dashboard-frontend)
16. [短信验证码登录实现](#16-短信验证码登录实现)
17. [用户注册验证与密码策略](#17-用户注册验证与密码策略)
18. [操作日志系统](#18-操作日志系统)
19. [超级管理员与角色管理](#19-超级管理员与角色管理)
20. [实名认证系统](#20-实名认证系统)
21. [多票种系统 (Multi-Ticket System)](#21-多票种系统-multi-ticket-system)

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
│   │   ├── admin.py         # 管理员：用户管理/封号/数据库浏览
│   │   ├── upload.py        # 文件上传：图片上传
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
| auth | `/api/v1/auth` | `/register`, `/login`, `/refresh`, `/me`, `/real-name/verify`, `/real-name/status` |
| events | `/api/v1/events` | `/`, `/{id}`, `/{id}/stock` |
| orders | `/api/v1/orders` | `/`, `/{id}`, `/{id}/cancel` |
| payments | `/api/v1/payments` | `/alipay`, `/alipay/notify`, `/refund` |
| admin | `/api/v1/admin` | `/users`, `/users/{id}/ban`, `/db/tables`, `/db/tables/{name}` |
| upload | `/api/v1/upload` | `/image` |
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

### 7.4 图片上传

图片上传使用 `UploadFile` 接收文件，校验扩展名和大小后保存到媒体目录，返回可访问的 URL。

**上传端点**：

```python
# app/routers/upload.py
@router.post("/image")
async def upload_image(
    file: UploadFile = File(...),
    _admin: User = Depends(require_admin),
):
    # 1. 校验文件扩展名（仅允许 jpg/png/gif/webp）
    ext = os.path.splitext(file.filename)[1].lower()
    if ext not in settings.ALLOWED_EXTENSIONS:
        raise HTTPException(status_code=400, detail=f"不支持的文件格式: {ext}")
    # 2. 读取内容并校验大小（默认最大 5MB）
    contents = await file.read()
    if len(contents) > settings.MAX_UPLOAD_SIZE:
        raise HTTPException(status_code=400, detail="文件大小超过限制")
    # 3. 用 UUID 重命名保存，避免文件名冲突
    os.makedirs(settings.MEDIA_DIR, exist_ok=True)
    filename = f"{uuid.uuid4().hex}{ext}"
    filepath = os.path.join(settings.MEDIA_DIR, filename)
    with open(filepath, "wb") as f:
        f.write(contents)
    return {"url": f"{settings.MEDIA_URL}/{filename}"}
```

**静态文件服务**（在 `app/main.py` 中配置）：

```python
os.makedirs(settings.MEDIA_DIR, exist_ok=True)
app.mount(settings.MEDIA_URL, StaticFiles(directory=settings.MEDIA_DIR), name="media")
```text

**设计要点**：

- 仅管理员可上传图片（`require_admin` 权限控制）
- 使用 UUID 重命名文件，防止路径遍历攻击和文件名冲突
- 扩展名白名单校验，防止上传恶意文件
- 上传返回的 URL 可以直接填入活动（Event）的 `cover_image` 字段

**活动封面图**：Event 模型中的 `cover_image` 字段存储上传返回的 URL 路径。前端 EventCard 和 EventDetail 会根据该字段渲染封面图片，无封面时显示默认渐变色+图标。支持在活动编辑页面直接上传和设置封面。

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

### 8.4 JWT 认证中间件与被封禁用户处理

认证中间件（`app/middleware/auth.py`）提供 `get_current_user()`、`RoleChecker` 和 `require_not_banned()` 三个核心依赖：

```python
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    session: AsyncSession = Depends(get_db),
) -> User:
    """获取当前登录用户（不检查封禁状态，允许被封禁用户查看个人信息）"""
    payload = decode_access_token(token)
    if payload is None:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)
    user = await session.get(User, int(payload.get("sub")))
    if user is None or not user.is_active:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)
    return user

async def require_not_banned(
    current_user: User = Depends(get_current_user),
) -> User:
    """要求用户未被封禁（用于非管理员写操作）"""
    if current_user.is_banned is True:
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="账户已被封禁")
    return current_user

class RoleChecker:
    """角色检查依赖（含封禁检查，但 super_admin 豁免）"""
    def __init__(self, allowed_roles: list[str]) -> None:
        self.allowed_roles = allowed_roles

    async def __call__(self, current_user: User = Depends(get_current_user)) -> User:
        if current_user.role not in self.allowed_roles:
            raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, ...)
        if current_user.role != "super_admin" and current_user.is_banned is True:
            raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="账户已被封禁")
        return current_user
```

**为什么这样设计？**

被封禁用户需要能：

1. **查看个人页面** — 了解被封禁的原因和时间
2. **看到封禁提示** — 登录后页面显示"你的账户已被封禁"及原因
3. **联系客服/申诉** — 通过个人页面获取帮助信息

但被封禁用户**不能**：

1. 创建新订单
2. 创建活动
3. 执行任何写操作

所以 `get_current_user()` 不检查 `is_banned`，而是：

- `require_not_banned()` 用于普通用户的写操作端点
- `RoleChecker` 自动检查封禁状态（但 `super_admin` 豁免，确保超管即使被封禁也能管理系统）

---

## 9. Async/Await 异步编程模式

### 9.1 为什么使用异步？

本项目的所有 I/O 操作（数据库查询、Redis 操作、HTTP 请求）都是异步的。异步编程允许在等待 I/O 时切换执行其他任务，提高并发处理能力。

**同步 vs 异步**：

```text
同步：请求1 → DB查询(等1秒) → 响应1 → 请求2 → DB查询(等1秒) → 响应2   (总耗时：2+秒)
异步：请求1 → DB查询(等1秒) → (等待时处理请求2) → DB查询(等1秒) → ... (总耗时：~1秒)
```

### 9.2 async/await 语法

```python
# 定义异步函数
async def get_user(user_id: int) -> User:
    # await 等待一个异步操作完成
    user = await session.get(User, user_id)
    return user

# 调用异步函数
user = await get_user(1)
```text

### 9.3 异步上下文管理器

```python
# 数据库会话使用异步上下文管理器
async with async_session_factory() as session:
    user = await session.get(User, 1)

# Redis pipeline 也类似
async with self.redis.pipeline(transaction=True) as pipe:
    await pipe.watch(key)
    # ...
```

### 9.4 并发执行任务

```python
import asyncio

# 并发执行多个独立的异步任务
results = await asyncio.gather(
    get_user(1),
    get_user(2),
    get_user(3),
)
```text

### 9.5 常见模式和陷阱

**陷阱 1：在同步函数中调用异步函数**

```python
# 错误！不能在同步函数中直接 await
def get_user_sync(user_id: int):
    return await session.get(User, user_id)  # SyntaxError!

# 正确：必须定义为 async
async def get_user_sync(user_id: int):
    return await session.get(User, user_id)
```

**陷阱 2：忘记 await**

```python
# 错误！没有 await，得到一个 coroutine 对象
result = session.execute(stmt)
# result 是 coroutine，不是查询结果

# 正确
result = await session.execute(stmt)
```text

**陷阱 3：在 FastAPI 路由中混用同步和异步**

FastAPI 支持同步和异步路由函数。但如果路由是异步的，其中的 I/O 操作也必须是异步的：

```python
@router.get("/events")
async def list_events():  # 异步路由
    events = await get_all_events()  # 必须 await
    return events
```

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
```text

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
```

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
```text

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
```

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
```text

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
```

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
```text

- `from_attributes = True` — 关键配置！允许 `UserResponse.model_validate(db_user)` 直接从 ORM 对象创建

### 11.4 model_validate 与 model_dump

```python
# 从 ORM 对象创建 Schema 实例（替换 v1 的 from_orm）
response = EventResponse.model_validate(db_event)

# Schema 实例转字典（替换 v1 的 dict()）
data = response.model_dump()

# 排除未设置字段的字典
update_data = data.model_dump(exclude_unset=True)
```

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
```text

在 `app/models/__init__.py` 中注册：

```python
from .favorite import Favorite
__all__ = ["Base", "Event", "LoginHistory", "Order", "Ticket", "User", "Favorite"]
```

### 12.2 第二步：创建 Alembic 迁移

```bash
alembic revision --autogenerate -m "add favorites table"
```text

检查生成的迁移文件，然后应用：

```bash
alembic upgrade head
```

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
```text

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
```

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
```text

### 12.6 第六步：注册路由

在 `app/routers/__init__.py` 中添加：

```python
from app.routers import favorites
api_router.include_router(favorites.router, prefix="/favorites", tags=["收藏"])
```

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
```text

### 12.8 第八步：测试并提交

```bash
# 运行新功能的测试
pytest tests/test_favorites.py -v

# 运行全部测试确保没有回归
pytest

# 代码检查
ruff check .
pyright .
```

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
```text

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
```

**Mock Redis**：

```python
mock_redis = AsyncMock()
mock_redis.zcard = AsyncMock(return_value=3)  # 模拟 3 次请求
mock_redis.get = AsyncMock(return_value=b"2")  # 模拟 2 笔并发订单
```text

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
```

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
```text

**原则 2：每个测试只测一个功能**

```python
# 好的做法 — 分开测试
def test_create_order_success(): ...
def test_create_order_insufficient_stock(): ...
def test_create_order_event_not_found(): ...

# 不好的做法 — 一个测试测多个功能
def test_create_order(): ...  # 又测成功又测失败
```

**原则 3：使用有意义的断言消息**

```python
assert success_count <= total_stock, (
    f"超卖！成功 {success_count} 单，但库存只有 {total_stock}"
)
```text

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
```

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
```text

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
```

**陷阱：提交后对象过期**

```python
# 如果在 expire_on_commit=True 时访问属性会报错
# session.commit() 后，对象属性需要重新查询

# 本项目的 session 设置为 expire_on_commit=False 避免了此问题
```text

**陷阱：server_default 字段在 commit 后需要 refresh**

```python
# 使用 server_default=func.now() 的字段（如 created_at、updated_at）
# 在 flush 后不会被 Python 赋值，必须 refresh 才能读取

# 错误！Pydantic 序列化时 created_at 为 None
event = Event(title="test")
session.add(event)
await session.commit()
return EventResponse.model_validate(event)  # created_at 缺失！

# 正确：commit 后 refresh 一次
event = Event(title="test")
session.add(event)
await session.commit()
await session.refresh(event)  # 加载所有 server_default 字段
return EventResponse.model_validate(event)
```

这同样适用于其他 `server_default` 字段，如数据库自动生成的默认值、计算列等。

**陷阱：AsyncSession 惰性加载（Lazy Loading）失败**

```python
# 在 SQLAlchemy 异步模式下，通过 relationship 访问关联对象会导致错误：
# "greenlet_spawn has not been called"

# 错误！order.event 会触发惰性加载，在 async 上下文中失败
order = await session.get(Order, 1)
print(order.event.title)  # ❌ greenlet_spawn 错误！

# 解法 1：使用 selectinload 预加载
from sqlalchemy.orm import selectinload
stmt = select(Order).options(selectinload(Order.event)).where(Order.id == 1)
result = await session.execute(stmt)
order = result.scalar_one_or_none()
print(order.event.title)  # ✅ 已预加载

# 解法 2：显式查询关联对象（适用于只需要少量关联数据时）
order = await session.get(Order, 1)
event = await session.get(Event, order.event_id)
print(event.title)  # ✅ 显式查询，不依赖惰性加载

# 在列表查询中，必须使用解法 1（selectinload），否则 N+1 问题会导致大量查询
```text

**为什么 AsyncSession 不支持惰性加载？** SQLAlchemy 的惰性加载需要在数据库连接线程中执行额外查询，但 AsyncSession 将查询任务交给了 asyncio 事件循环，无法在同步上下文中发起异步查询。因此必须预加载（eager loading）所有需要的关联数据。

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
```

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
```text

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
```

**陷阱：Decimal 转 float 精度丢失**

```python
# 金额计算使用 Decimal
total_amount = Decimal(str(event.price)) * data.quantity

# 存数据库时转为 float（但计算过程保持 Decimal 精度）
order.total_amount = float(total_amount)
```text

### 14.5 FastAPI 相关

**陷阱：Depends 的 B008 警告**

Ruff 的 B008 规则会警告函数调用在默认参数中：

```python
# B008 警告：Depends() 在函数定义时执行
async def endpoint(session: AsyncSession = Depends(get_db)):

# 虽然触发警告，但在 FastAPI 中这是标准用法，可以安全使用
# 只需添加 noqa 注释忽略
async def endpoint(session: AsyncSession = Depends(get_db)):  # noqa: B008
```

### 14.6 数据库相关

**陷阱：MySQL 时区问题**

```python
# 所有时间字段使用 DateTime(timezone=True)
# 查询时使用 UTC 比较
from datetime import datetime, timezone
now = datetime.now(timezone.utc).replace(tzinfo=None)
```text

**陷阱：LIKE 查询的 SQL 注入风险**

```python
# 用户输入的关键词可能包含 % 和 _ 通配符
keyword = "test%_"
# 必须转义！
safe_keyword = keyword.replace("%", "\\%").replace("_", "\\_")
conditions.append(Event.title.like(f"%{safe_keyword}%"))
```

### 14.7 Docker 相关

**陷阱：Docker 容器重启循环**

```bash
# 查看容器日志定位问题
docker compose logs mysql
docker compose logs redis

# 检查健康检查状态
docker inspect ticket-mysql --format '{{json .State.Health}}'
```text

**陷阱：端口冲突**

```bash
# 检查端口被哪个进程占用
netstat -ano | findstr :8000
netstat -ano | findstr :3306
netstat -ano | findstr :6379

# 结束占用进程
taskkill /PID <进程ID> /F
```

### 14.8 测试相关

**陷阱：Mock 未正确模拟异步上下文管理器**

```python
# inventory_service.py 中使用 async with pipeline()
# 测试时必须正确模拟 __aenter__
mock_pipeline = AsyncMock()
mock_redis.pipeline.return_value.__aenter__.return_value = mock_pipeline
```text

**陷阱：集成测试需要外部服务**

```bash
# 运行集成测试前，确保 MySQL 和 Redis 已启动
docker compose up -d

# 然后运行
pytest --integration -v
```

---

## 15. 管理后台前端开发 (Admin Dashboard Frontend)

> **前置知识**: 第 18 章（Vue 3 基础）、第 20 章（前后端集成）
> **对应代码**: `frontend/src/views/admin/`、`frontend/src/stores/admin.js`、`frontend/src/router/index.js`

### 15.1 功能概述

管理后台是面向管理员的操作面板，提供 8 个核心管理页面：

| 页面 | 功能 | 文件 |
| ------ | ------ | ------ |
| 仪表盘 | 统计概览、收入趋势、活动销售排行、超管每日概览+最近操作 | `DashboardView.vue` |
| 用户管理 | 用户搜索/筛选/编辑/封禁/解封、登录历史、IP历史（超管）、角色编辑（超管） | `UserManageView.vue` |
| 活动管理 | 活动CRUD、封面图片上传、状态管理、购票须知编辑 | `EventManageView.vue` |
| 订单管理 | 订单筛选/详情/退款审核（含批量）、空降订单创建（含自定义金额） | `OrderManageView.vue` |
| 票证管理 | 票证搜索/筛选/使用状态查看 | `TicketManageView.vue` |
| 日志管理 | 操作日志查看（Tabs分页、多维度筛选、点击查看详情弹窗） | `LogManageView.vue` |
| 数据库管理 | 超管专享的只读数据库表浏览器，可查看表结构和数据（分页） | `DatabaseManageView.vue` |

### 15.2 项目结构

管理后台的所有代码集中在以下目录：

```text
frontend/src/
├── views/admin/
│   ├── AdminLayout.vue      # 侧边栏布局容器（含超管专属菜单）
│   ├── DashboardView.vue    # 仪表盘（超管含每日概览+最近操作）
│   ├── UserManageView.vue   # 用户管理（超管可改角色+查看IP历史）
│   ├── EventManageView.vue  # 活动管理（含图片上传+状态编辑）
│   ├── OrderManageView.vue  # 订单管理（含自定义金额空降订单）
│   ├── TicketManageView.vue # 票证管理
│   ├── LogManageView.vue    # 日志管理（超管专属，含详情弹窗）
│   └── DatabaseManageView.vue # 数据库管理（超管专属只读表浏览器）
├── stores/
│   └── admin.js             # 管理后台 Pinia Store
└── router/
    └── index.js             # 路由配置（含 admin 嵌套路由）
```

### 15.3 AdminLayout.vue — 侧边栏导航布局

管理后台使用**嵌套路由**模式：`AdminLayout.vue` 作为父组件，提供一个左侧导航栏 + `<router-view />` 占位，所有子页面在内容区渲染。

```vue
<!-- AdminLayout.vue (简化) -->
<template>
  <div class="admin-layout">
    <aside class="admin-sidebar">
      <div class="admin-sidebar-title">管理面板</div>

      <router-link to="/admin" class="admin-sidebar-link"
        :class="{ active: $route.path === '/admin' }">
        <span class="admin-sidebar-icon">📊</span>
        <span>仪表盘</span>
      </router-link>

      <router-link to="/admin/users" class="admin-sidebar-link"
        :class="{ active: $route.path.startsWith('/admin/users') }">
        <span class="admin-sidebar-icon">👥</span>
        <span>用户管理</span>
      </router-link>

      <router-link to="/admin/events" class="admin-sidebar-link">
        <span class="admin-sidebar-icon">🎫</span>
        <span>活动管理</span>
      </router-link>

      <router-link to="/admin/orders" class="admin-sidebar-link">
        <span class="admin-sidebar-icon">📦</span>
        <span>订单管理</span>
      </router-link>

      <!-- 超管专属菜单 -->
      <template v-if="currentUserRole === 'super_admin'">
        <div class="super-admin-section">超级管理</div>
        <router-link to="/admin/logs" class="admin-sidebar-link">
          <span>📋</span><span>日志管理</span>
        </router-link>
        <router-link to="/admin/database" class="admin-sidebar-link">
          <span>🗄️</span><span>数据库管理</span>
        </router-link>
      </template>
    </aside>

    <main class="admin-content">
      <router-view />  <!-- 子页面在此渲染 -->
    </main>
  </div>
</template>
```text

**设计要点**：

- 使用 `$route.path` 判断当前激活的导航项，高亮显示
- 底部提供"返回前台"链接，方便管理员在前后台之间切换
- 侧边栏样式固定宽度 240px，内容区自适应填充剩余空间

### 15.4 路由配置与管理员守卫

管理后台路由在 `frontend/src/router/index.js` 中作为**嵌套路由**配置：

```javascript
// router/index.js
const routes = [
  // ... 前台页面路由 ...

  // Admin routes - 需要认证 + 管理员角色
  {
    path: '/admin',
    component: () => import('../views/admin/AdminLayout.vue'),
    meta: { requiresAuth: true, requiresAdmin: true },
    children: [
      { path: '', name: 'admin-dashboard',
        component: () => import('../views/admin/DashboardView.vue') },
      { path: 'users', name: 'admin-users',
        component: () => import('../views/admin/UserManageView.vue') },
      { path: 'events', name: 'admin-events',
        component: () => import('../views/admin/EventManageView.vue') },
      { path: 'orders', name: 'admin-orders',
        component: () => import('../views/admin/OrderManageView.vue') },
      { path: 'tickets', name: 'admin-tickets',
        component: () => import('../views/admin/TicketManageView.vue') },
      { path: 'logs', name: 'admin-logs',
        component: () => import('../views/admin/LogManageView.vue'),
        meta: { requiresSuperAdmin: true } },
      { path: 'database', name: 'admin-database',
        component: () => import('../views/admin/DatabaseManageView.vue'),
        meta: { requiresSuperAdmin: true } },
    ],
  },
]
```

**关键点**：

- `meta.requiresAdmin: true` — 自定义元字段，标识需要管理员权限
- `meta.requiresSuperAdmin: true` — 日志管理和数据库管理页面仅超级管理员可见
- **嵌套路由** — 父路由使用 `AdminLayout`，子路由自动在 `<router-view />` 中渲染
- **路由懒加载** — `() => import(...)` 实现代码分割，访问时才加载对应组件

**管理员路由守卫**：

```javascript
router.beforeEach((to, from, next) => {
  const token = localStorage.getItem('token')

  // 1. 检查登录状态
  if (to.meta.requiresAuth && !token) {
    next({ name: 'login', query: { redirect: to.fullPath } })
    return
  }

  // 2. 检查管理员角色（从 localStorage 读取缓存的用户信息）
  if (to.meta.requiresAdmin && token) {
    const stored = localStorage.getItem('user')
    if (stored) {
      try {
        const user = JSON.parse(stored)
        if (!['admin', 'super_admin', 'organizer'].includes(user.role)) {
          next({ name: 'home' })  // 非管理员重定向到首页
          return
        }
      } catch {}
    }
  }

  // 3. 检查超级管理员角色（日志管理、数据库管理等页面需要）
  if (to.meta.requiresSuperAdmin && token) {
    const stored = localStorage.getItem('user')
    if (stored) {
      try {
        const user = JSON.parse(stored)
        if (user.role !== 'super_admin') {
          next({ path: '/admin' })  // 普通管理员跳转管理后台首页
          return
        }
      } catch {}
    }
  }

  next()
})
```text

**为什么从 localStorage 读取角色而不是调用 API？**

- 导航守卫是同步的，而 API 调用是异步的
- 用户信息在登录时已经被缓存到 `localStorage`（见 `auth.js` store 的 `fetchUser()`）
- 如果未缓存用户信息，API 返回 403 时，页面内部会自行处理错误
- 这是一种 **快路径检查 + 服务端强制校验** 的双层防护模式

### 15.5 DashboardView.vue — 仪表盘

仪表盘页面包含三个核心区域：统计卡片、收入趋势图（CSS 柱状图）、活动销售排行。

**统计卡片**：

```html
<div class="stats-grid" v-if="stats">
  <div class="stat-card">
    <span class="stat-card-label">用户总数</span>
    <span class="stat-card-value">{{ stats.total_users }}</span>
  </div>
  <div class="stat-card">
    <span class="stat-card-label">活动总数</span>
    <span class="stat-card-value">{{ stats.total_events }}</span>
  </div>
  <!-- 订单总数、总营收、已完成订单、待处理退款 -->
</div>
```

6 张统计卡片来自一个 API 调用 `GET /api/v1/admin/dashboard/stats`，返回 `total_users`、`total_events`、`total_orders`、`total_revenue`、`paid_orders`、`pending_refunds`。

**CSS 柱状图（收入趋势）**：

```html
<div class="bar-chart">
  <div class="bar-item" v-for="(rev, i) in trendData.revenues" :key="i">
    <div class="bar" :style="{ height: barHeight(rev, trendData.revenues) + '%' }"></div>
    <span class="bar-label">{{ trendData.dates[i].slice(5) }}</span>
  </div>
</div>
```text

```javascript
function barHeight(val, arr) {
  const max = Math.max(...arr, 1)
  return Math.max((val / max) * 100, 4)  // 最低 4% 保证可见
}
```

**为什么选择 CSS 柱状图而不是 ECharts/Chart.js？**

| 对比项 | CSS 柱状图 | ECharts/Chart.js |
| -------- | ----------- | ----------------- |
| 包体积 | 0 KB（纯 CSS） | ~100-500 KB |
| 学习成本 | 低 | 中高 |
| 交互性 | 低 | 高 |
| 适用场景 | 简单数据展示 | 复杂图表需求 |

对于展示近 14 天的收入趋势这种简单场景，CSS 柱状图完全够用，且避免了引入大型依赖库。

**活动销售排行**：

```html
<tr v-for="item in rankData.items" :key="item.event_id">
  <td>
    <span v-if="item.rank <= 3"
      :style="{ color: ['#ff4d4f','#faad14','#52c41a'][item.rank-1], fontWeight: 'bold' }">
      #{{ item.rank }}
    </span>
    <span v-else>#{{ item.rank }}</span>
  </td>
  <td>{{ item.title }}</td>
  <!-- 售罄率进度条 -->
  <td>
    <div class="progress-bar">
      <div :style="{ width: item.sell_rate + '%',
            background: item.sell_rate > 80 ? '#52c41a' : '#1677ff' }"></div>
    </div>
    <span>{{ item.sell_rate }}%</span>
  </td>
</tr>
```text

- 前三名使用不同的颜色（金/银/铜色）
- 售罄率使用进度条可视化，>80% 时变绿色
- 使用 `toLocaleString('zh-CN')` 格式化金额，显示千分位分隔符

### 15.6 Pinia Admin Store

管理后台的所有 API 调用集中在 `frontend/src/stores/admin.js` 中。这是典型的"一个 Store 管理多个资源"模式。

```javascript
// stores/admin.js
import { defineStore } from 'pinia'
import { ref } from 'vue'
import { api } from '../utils/api'

export const useAdminStore = defineStore('admin', () => {
  const stats = ref(null)
  const loading = ref(false)

  // ===== Dashboard =====
  async function fetchStats() {
    stats.value = await api.get('/admin/dashboard/stats')
    return stats.value
  }

  async function fetchRevenueTrends(days = 7) {
    return api.get('/admin/dashboard/revenue-trends', { params: { days } })
  }

  async function fetchEventRankings(limit = 10) {
    return api.get('/admin/dashboard/event-rankings', { params: { limit } })
  }

  // ===== Users =====
  async function fetchUsers(params = {}) {
    return api.get('/admin/users', { params })
  }

  async function banUser(userId, reason) {
    return api.post(`/admin/users/${userId}/ban`, { reason })
  }

  async function unbanUser(userId) {
    return api.post(`/admin/users/${userId}/unban`)
  }

  // ===== Events =====
  async function fetchAdminEvents(params = {}) {
    return api.get('/events/', { params })
  }

  async function createEvent(data) {
    return api.post('/events/', data)
  }

  // ===== Orders =====
  async function fetchAdminOrders(params = {}) {
    return api.get('/admin/orders', { params })
  }

  async function reviewRefund(orderId, approve) {
    return api.post(`/admin/refunds/${orderId}/review`, { approve })
  }

  async function batchReviewRefunds(orderIds, approve) {
    return api.post('/admin/refunds/batch-review', { order_ids: orderIds, approve })
  }

  // ===== Events =====
  async function updateEvent(eventId, data) {
    return api.put(`/events/${eventId}`, data)
  }

  async function uploadImage(formData) {
    return api.post('/upload/image', formData, {
      headers: { 'Content-Type': 'multipart/form-data' },
    })
  }

  // ===== Database =====
  async function fetchDbTables() {
    return api.get('/admin/db/tables')
  }

  async function fetchDbTableData(tableName, page = 1, size = 20) {
    return api.get(`/admin/db/tables/${tableName}`, { params: { page, size } })
  }

  // ===== Role Management =====
  async function setUserRole(userId, role) {
    return api.put(`/admin/users/${userId}/role`, { role })
  }

  return {
    stats, loading,
    fetchStats, fetchRevenueTrends, fetchEventRankings,
    fetchUsers, banUser, unbanUser,
    fetchAdminEvents, createEvent, updateEvent, uploadImage,
    fetchAdminOrders, reviewRefund, batchReviewRefunds,
    fetchAdminTickets,
    fetchDbTables, fetchDbTableData,
    setUserRole,
  }
})
```

**设计模式分析**：

1. **一个 Store 多个模块** — 虽然包含仪表盘/用户/活动/订单/票证 5 组功能，但都在一个 Store 中。原因：这些功能同属"管理后台"上下文，且只在管理后台使用，不会与前台 Store（如 `useAuthStore`、`useEventStore`）产生命名冲突
2. **函数分组注释** — 使用 `===== Dashboard =====` 等注释分隔不同功能组，便于维护
3. **API 调用封装** — 每个函数封装一个 API 端点，页面组件只需调用 `adminStore.fetchUsers(...)` 而无需拼接 URL 和处理请求参数
4. **状态缓存** — `stats` 保留在 Store 中，切换页面时不需要重新加载（但页面刷新后会丢失，需要重新获取）
5. **不缓存列表数据** — 用户/活动/订单/票证列表每次进入页面时重新获取，保证数据实时性

### 15.7 公共模式：数据表格

所有 5 个管理页面都使用了相同的数据表格模式，包含 4 个核心部分：

**1. 搜索/筛选栏（Filter Bar）**

```html
<div class="filter-bar">
  <input v-model="keyword" placeholder="搜索..." @keyup.enter="search" />
  <select v-model="roleFilter" @change="search">
    <option value="">全部角色</option>
    <option value="user">普通用户</option>
  </select>
  <button class="btn btn-primary" @click="search">搜索</button>
</div>
```text

- 输入框使用 `@keyup.enter` 触发搜索
- 下拉框使用 `@change` 立即触发搜索
- 每次搜索都会重置到第 1 页

**2. 数据表格（Data Table）**

```html
<table class="data-table">
  <thead>
    <tr>
      <th>ID</th><th>用户名</th><th>邮箱</th><!-- ... -->
    </tr>
  </thead>
  <tbody>
    <tr v-if="loading"><td colspan="8" class="loading-spinner">加载中...</td></tr>
    <tr v-else-if="!items.length"><td colspan="8" class="empty-row">暂无数据</td></tr>
    <tr v-for="item in items" :key="item.id">
      <!-- 数据行 -->
    </tr>
  </tbody>
</table>
```

- 统一的加载状态（`loading-spinner`）和空数据提示（`empty-row`）
- `colspan` 跨列合并，确保提示信息占满整行

**3. 分页（Pagination）**

```javascript
const totalPages = computed(() => Math.ceil(total.value / size.value) || 1)
const pages = computed(() => {
  const p = []
  const tp = totalPages.value
  const cur = page.value
  for (let i = Math.max(1, cur - 2); i <= Math.min(tp, cur + 2); i++) p.push(i)
  return p
})
```text

- 显示当前页周围 5 个页码按钮（前后各 2 个）
- 上一页/下一页按钮在边界时禁用
- 显示总记录数

**4. 模态弹窗（Modal）**

```html
<div class="modal-overlay" v-if="detailItem" @click.self="detailItem=null">
  <div class="modal-content">
    <div class="modal-title">详情</div>
    <!-- 表单内容 -->
    <div class="modal-actions">
      <button class="btn btn-default" @click="close">关闭</button>
      <button class="btn btn-primary" @click="save">保存</button>
    </div>
  </div>
</div>
```

- `@click.self="close"` — 点击遮罩层背景关闭弹窗
- `modal-overlay` 和 `modal-content` 两个层级实现居中弹窗
- 通过 `v-if` 控制显隐，关闭时销毁 DOM 避免表单残留

### 15.8 各页面特色功能

**UserManageView.vue — 用户管理**

- **角色标签**：`admin-role-badge` CSS 类配合 `role-{role}` 动态类名，不同角色不同颜色
- **封禁/解封**：封禁需要输入原因（弹窗表单），解封需要二次确认（`confirm()`）
- **用户详情弹窗**：包含编辑表单 + 登录历史列表，一次获取所有数据
- **分页参数保留**：切换页码时保留搜索条件
- **超管角色编辑**：超级管理员可通过行内下拉框直接修改用户角色（user/organizer/admin/super_admin），变更会调用 `adminStore.setUserRole(userId, role)` 并记录操作日志
- **角色变更确认弹窗**：修改角色时弹出二次确认对话框，显示用户名和即将变更的角色，防止误操作

**EventManageView.vue — 活动管理**

- **创建/编辑共用表单**：`showCreate` 控制创建模式，`editTarget` 控制编辑模式；点击"创建活动"按钮时 `editTarget=null`，点击行内"编辑"按钮时填充 `editTarget`
- **售罄率进度条**：`Math.round((sold_count || 0) / total_stock * 100)` 计算，`>= 100%` 时标红警告
- **删除确认**：使用 `confirm()` 双重确认，文案提示"此操作不可撤销"
- **封面图片上传**：编辑表单中包含图片上传区域，支持选择文件、预览、上传。上传后返回的 URL 自动填入 `cover_image` 字段
- **状态管理**：活动状态下拉框（draft/published），管理员可直接切换活动上下架状态
- **购票须知编辑**：文本域支持编辑活动的 `purchase_notes` 字段，用户购票时能看到相关说明

**OrderManageView.vue — 订单管理**

- **退款审核工作流**：待审核订单行背景高亮（`#fffbe6`），支持单个审核（通过/拒绝）和批量操作
- **多选框逻辑**：`selectedOrders` 数组记录选中的订单 ID，`toggleAll` 全选/取消全选，`allSelected` 计算属性判断是否全部选中
- **空降订单**：管理员直接为用户创建订单，支持自定义总金额（`total_amount` 可选字段），用于线下收款后的补录。不指定金额时自动按 `price * quantity` 计算
- **筛选条件组合**：状态 + 退款状态 + 用户 ID + 活动 ID 4 个维度的组合筛选

**TicketManageView.vue — 票证管理**

- **只读页面**：票证是支付后的产物，只能查看不能修改，因此没有编辑/删除操作按钮
- **多维搜索**：活动 ID + 用户 ID + 订单 ID + 使用状态 4 个维度的精确查询

**LogManageView.vue — 日志管理（超管专属）**

- **多维度筛选**：按操作类型（Tabs 分类）、用户 ID、日期范围组合筛选
- **分页展示**：标准分页组件，每页 20 条，显示总记录数
- **详情弹窗**：点击任意日志行弹出详情模态框，展示：
  - 日志 ID、操作类型标签（彩色徽章）
  - 操作者信息（用户名、用户 ID）
  - 目标用户信息（如果是封禁/改角色操作）
  - IP 地址和 User-Agent
  - 操作详情文本
  - 操作时间（精确到秒）
- **Tabs 分类过滤**：全部 / 登录日志 / 注册日志 / 操作日志

**DatabaseManageView.vue — 数据库管理（超管专属）**

- **表列表**：以网格卡片形式展示数据库中所有表，每张卡片显示表名
- **表数据浏览**：点击表名进入数据视图，支持分页（每页 20 条）
- **表结构查看**：通过"结构"标签页查看表的字段名、类型、默认值、约束等信息
- **只读模式**：仅支持查看，不支持修改/删除数据，保障数据安全
- **后端 API**：`GET /api/v1/admin/db/tables`（列出表）、`GET /api/v1/admin/db/tables/{name}?page=1&size=20`（查询表数据+结构）

### 15.9 前后端交互流程

以"管理员审核退款"为例，展示完整的前后端交互：

```text
管理员点击"通过"按钮
    │
    ▼
adminStore.reviewRefund(orderId, approve=true)
    │  POST /api/v1/admin/refunds/:orderId/review
    │  Body: { "approve": true }
    ▼
后端 AdminService 处理退款逻辑
    │  1. 检查订单状态（必须是 pending_payment）
    │  2. 更新 refund_status = approved
    │  3. 执行退款操作（调用支付宝退款 API）
    │  4. 更新订单状态为 refunded
    ▼
返回成功响应
    │
    ▼
前端重新获取订单列表（刷新页面数据）
    fetchOrders()
```

### 15.10 样式方案

管理后台没有使用 UI 组件库（如 Element Plus、Ant Design Vue），而是采用**纯 CSS 自定义样式**：

```css
/* 全局样式示例 */
.data-table {
  width: 100%;
  border-collapse: collapse;
  background: #fff;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 1px 4px rgba(0,0,0,0.08);
}

.data-table th {
  background: #fafafa;
  padding: 12px 16px;
  font-weight: 600;
  font-size: 13px;
  color: #555;
  text-align: left;
  border-bottom: 1px solid #f0f0f0;
}

.data-table td {
  padding: 12px 16px;
  font-size: 14px;
  border-bottom: 1px solid #f0f0f0;
}

.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.4);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-content {
  background: #fff;
  border-radius: 12px;
  padding: 24px;
  min-width: 480px;
  max-width: 90vw;
  max-height: 80vh;
  overflow-y: auto;
}
```text

**为什么不用 UI 组件库？**

1. **减少依赖** — 避免引入 Element Plus (~2MB) 等大型库，加快首屏加载
2. **灵活控制** — 自定义样式可以精确匹配设计需求
3. **学习价值** — 纯 CSS 实现让读者理解布局、弹窗、表格等常见模式的原生实现方法
4. **管理后台页面数量少** — 只有 7 个页面，UI 组件库的收益不高

---

## 16. 短信验证码登录实现

> **对应代码**: `app/routers/auth.py`、`app/services/auth_service.py`、`app/schemas/user.py`

### 16.1 功能概述

除了传统的密码登录，本系统还提供了短信验证码登录方式。用户输入已注册的手机号，接收短信验证码，输入验证码即可完成登录，无需输入密码。

### 16.2 Schema 定义

```python
# app/schemas/user.py
class SMSLoginRequest(BaseModel):
    """短信验证码登录请求"""
    phone: str = Field(..., pattern=r"^1\d{10}$", description="手机号")
    sms_code: str = Field(..., min_length=6, max_length=6, description="短信验证码")

class SendSMSRequest(BaseModel):
    """发送短信验证码请求"""
    phone: str = Field(..., pattern=r"^1\d{10}$", description="手机号")
```

### 16.3 服务层实现

```python
# app/services/auth_service.py
async def login_by_sms(
    self,
    phone: str,
    sms_code: str,
    ip_address: str | None = None,
    user_agent: str | None = None,
) -> tuple[User, str, str]:
    """短信验证码登录"""
    from app.services.sms_service import SMSService

    # 1. 验证短信验证码
    sms_svc = SMSService(self.redis)
    if not await sms_svc.verify_code(phone, sms_code):
        raise ValueError("短信验证码错误")

    # 2. 查找用户
    result = await self.session.execute(
        select(User).where(User.phone == phone)
    )
    user = result.scalar_one_or_none()
    if not user:
        raise ValueError("手机号未注册")
    if not user.is_active:
        raise ValueError("用户已被禁用")

    # 3. 更新最后登录信息
    now = datetime.now(timezone.utc).replace(tzinfo=None)
    user.last_login_at = now
    user.last_login_ip = ip_address

    # 4. 记录登录历史
    await self._record_login_history(
        user_id=user.id, ip_address=ip_address,
        user_agent=user_agent, success=True,
    )

    # 5. 生成令牌
    token_data = {"sub": str(user.id)}
    access_token = create_access_token(token_data)
    refresh_token = create_refresh_token(token_data)
    return user, access_token, refresh_token
```text

**设计要点**：

- 登录不需要密码，改为验证码验证，适用于忘记密码或便捷登录场景
- 验证码校验通过 `SMSService.verify_code()` 完成，该服务管理验证码的生成、存储（Redis）和过期
- 登录成功后会更新 `last_login_at` 和 `last_login_ip`
- 通过操作日志记录 `sms_login` 动作，与普通登录区分

### 16.4 路由层

```python
# app/routers/auth.py
@router.post("/sms-login", response_model=TokenResponse)
async def sms_login(
    data: SMSLoginRequest,
    request: Request,
    session: Annotated[AsyncSession, Depends(get_db)],
    redis: Annotated[Redis, Depends(get_redis)],
):
    """手机短信验证码登录"""
    # 提取客户端 IP 和 User-Agent
    forwarded = request.headers.get("X-Forwarded-For")
    client_ip = forwarded.split(",")[0].strip() if forwarded else (
        request.client.host if request.client else None
    )
    user_agent = request.headers.get("User-Agent")

    service = AuthService(session, redis)
    try:
        user, access_token, refresh_token = await service.login_by_sms(
            phone=data.phone, sms_code=data.sms_code,
            ip_address=client_ip, user_agent=user_agent,
        )
        # 记录操作日志
        await service.create_operation_log(
            user_id=user.id, username=user.username,
            action="sms_login", ip_address=client_ip,
            user_agent=user_agent,
            detail=f"短信登录: {user.username}", success=True,
        )
        await session.commit()
        return TokenResponse(access_token=access_token, refresh_token=refresh_token)
    except ValueError as e:
        await session.rollback()
        # 区分不同类型的错误，返回中文提示
        if "未注册" in str(e):
            raise HTTPException(status_code=404, detail="手机号未注册")
        if "验证码错误" in str(e):
            raise HTTPException(status_code=400, detail="短信验证码错误")
        raise HTTPException(status_code=400, detail=str(e))
```

### 16.5 前端 3-Tab 登录页

```vue
<!-- LoginView.vue — 三标签布局 -->
<div class="tab-bar">
  <button v-for="tab in tabs" :key="tab.key"
    class="tab-btn" :class="{ active: activeTab === tab.key }"
    @click="switchTab(tab.key)">
    {{ tab.label }}
  </button>
</div>

<!-- Tab 1: 密码登录 -->
<form v-show="activeTab === 'password'" @submit.prevent="handlePasswordLogin">
  <!-- 用户名 + 密码 -->
</form>

<!-- Tab 2: 短信登录 -->
<form v-show="activeTab === 'sms'" @submit.prevent="handleSmsLogin">
  <!-- 手机号 + 短信验证码（含发送按钮） -->
</form>

<!-- Tab 3: 注册 -->
<form v-show="activeTab === 'register'" @submit.prevent="handleRegister">
  <!-- 用户名 + 密码 + 确认密码 + 邮箱 + 手机号 + 短信验证码 + 防机器人验证 -->
</form>
```text

**设计要点**：

- 使用 `v-show` 切换标签页，保持各表单的 DOM 状态不被销毁
- 视觉错误消息使用中文（如"手机号未注册"），不显示原始 HTTP 错误码
- 短信验证码发送按钮带 60 秒倒计时，防止频繁发送
- 密码字段支持显示/隐藏切换

---

## 17. 用户注册验证与密码策略

> **对应代码**: `app/services/auth_service.py`（PasswordPolicy 类）、`app/schemas/user.py`（RegisterRequest）

### 17.1 密码策略（PasswordPolicy）

本系统实现了严格的密码策略校验，规则可配置：

```python
# app/services/auth_service.py
class PasswordPolicy:
    """密码策略校验."""
    MIN_LENGTH = settings.PASSWORD_MIN_LENGTH
    REQUIRE_UPPER = settings.PASSWORD_REQUIRE_UPPER
    REQUIRE_LOWER = settings.PASSWORD_REQUIRE_LOWER
    REQUIRE_DIGIT = settings.PASSWORD_REQUIRE_DIGIT
    REQUIRE_SPECIAL = settings.PASSWORD_REQUIRE_SPECIAL

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

**配置项**（在 `.env` 中定义）：

```text
PASSWORD_MIN_LENGTH=8
PASSWORD_REQUIRE_UPPER=true
PASSWORD_REQUIRE_LOWER=true
PASSWORD_REQUIRE_DIGIT=true
PASSWORD_REQUIRE_SPECIAL=true
```

**为什么需要特殊字符？** 增加密码的熵值。一个8位纯小写字母密码的熵约为 38 位，而包含大小写字母+数字+特殊字符的8位密码熵约为 53 位，暴力破解难度提高 3 万倍。

### 17.2 增强注册流程

```python
# app/schemas/user.py
class RegisterRequest(BaseModel):
    """增强注册请求"""
    username: str = Field(..., min_length=3, max_length=50, pattern=r"^[a-zA-Z0-9_]+$")
    password: str = Field(..., min_length=8, max_length=128)
    password_confirm: str = Field(..., min_length=8, max_length=128)  # 确认密码
    email: EmailStr
    phone: str = Field(..., pattern=r"^1\d{10}$")           # 手机号
    sms_code: str = Field(..., min_length=6, max_length=6)   # 短信验证码
    captcha_token: str                                       # 防机器人验证
    captcha_answer: str                                      # 验证答案
```text

注册流程多重验证：

```

1. 密码一致性校验（password == password_confirm）
2. 密码强度校验（PasswordPolicy.validate）
3. 防机器人验证（CaptchaService 数学题验证）
4. 手机短信验证码验证（SMSService.verify_code）
5. 用户名唯一性 + 邮箱唯一性

```text

### 17.3 防机器人验证（CAPTCHA）

```python
# app/services/captcha_service.py（示意）
class CaptchaService:
    """防机器人验证（简单数学题）"""
    async def generate_public(self) -> dict:
        """生成验证题"""
        a, b = random.randint(1, 20), random.randint(1, 20)
        token = str(uuid4())
        # 将答案存入 Redis，5 分钟后过期
        await self.redis.setex(f"captcha:{token}", 300, str(a + b))
        return {
            "captcha_token": token,
            "question": f"{a} + {b} = ?",
            "expired_at": int(time.time()) + 300,
        }

    async def verify(self, token: str, answer: str) -> bool:
        """验证答案"""
        key = f"captcha:{token}"
        stored = await self.redis.get(key)
        await self.redis.delete(key)  # 一次性使用
        return stored is not None and stored == answer
```

---

## 18. 操作日志系统

> **对应代码**: `app/models/operation_log.py`、`app/schemas/operation_log.py`、`app/routers/admin.py`

### 18.1 为什么需要操作日志？

操作日志记录了系统中所有重要操作的"谁、什么时间、做了什么"，是安全审计和问题排查的基础。与 LoginHistory 的区别：

- **LoginHistory**：只记录登录尝试（成功/失败），侧重于认证安全
- **OperationLog**：记录所有操作（登录、注册、封禁、改角色、创建活动等），侧重于操作审计

### 18.2 数据模型

```python
# app/models/operation_log.py
class OperationLog(Base):
    __tablename__ = "operation_logs"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int | None] = mapped_column(Integer, nullable=True, index=True)
    username: Mapped[str | None] = mapped_column(String(50), nullable=True)
    action: Mapped[str] = mapped_column(
        String(50),
        comment="动作类型: login, register, logout, sms_login, 封禁, 解封, 角色变更, create_event, update_event, delete_event, admin_action",
                "delete_event, admin_action",
    )
    ip_address: Mapped[str] = mapped_column(String(45))
    user_agent: Mapped[str | None] = mapped_column(String(255), nullable=True)
    detail: Mapped[str | None] = mapped_column(String(500), nullable=True)
    success: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
```text

**设计权衡**：

- `user_id` 可为 NULL（允许匿名操作记录，如未登录的注册尝试）
- `action` 使用字符串而非枚举，便于扩展新操作类型
- `detail` 字段记录操作的附加信息，如"用户注册: admin"、"封禁: 恶意刷票"

### 18.3 日志记录方式

操作日志在服务层中记录。以 AuthService 为例：

```python
# app/services/auth_service.py
async def create_operation_log(
    self,
    user_id: int | None,
    username: str | None,
    action: str,
    ip_address: str | None,
    user_agent: str | None = None,
    detail: str | None = None,
    success: bool = True,
) -> OperationLog:
    """记录操作日志."""
    log = OperationLog(
        user_id=user_id, username=username, action=action,
        ip_address=ip_address or "unknown", user_agent=user_agent,
        detail=detail, success=success,
    )
    self.session.add(log)
    return log
```

**日志记录的关键操作**：

| 操作 | action 值 | 触发位置 |
| ------ | ----------- | ---------- |
| 密码登录 | `登录` | `POST /auth/login` |
| 短信登录 | `短信登录` | `POST /auth/sms-login` |
| 注册 | `注册` | `POST /auth/register` |
| 封禁用户 | `封禁` | `POST /admin/users/{id}/ban` |
| 解封用户 | `解封` | `POST /admin/users/{id}/unban` |
| 修改角色 | `角色变更` | `PUT /admin/users/{id}/role` |
| 创建活动 | `create_event` | `POST /events/` |
| 更新活动 | `update_event` | `PUT /events/{id}` |

### 18.4 API 端点

```python
# app/routers/admin.py
# 所有日志端点都需要 admin 角色

# 通用日志查询（可按操作类型、日期范围过滤）
GET /api/v1/admin/logs?page=1&size=20&action=login&date_from=2026-01-01&date_to=2026-12-31

# 登录日志（等价于 action=login 的快捷方式）
GET /api/v1/admin/logs/login

# 注册日志（等价于 action=register 的快捷方式）
GET /api/v1/admin/logs/registration

# 统计数据（超管仪表盘使用）
GET /api/v1/admin/logs/statistics
```text

**为什么设计为分页 API？** 操作日志可能快速增长，单次返回所有数据会占用大量内存和带宽。分页查询（默认每页 20 条，最大 100 条）确保接口的响应速度和稳定性。

### 18.5 Schema

```python
# app/schemas/operation_log.py
class OperationLogResponse(BaseModel):
    model_config = {"from_attributes": True}
    id: int
    user_id: int | None = None
    username: str | None = None
    action: str
    ip_address: str
    user_agent: str | None = None
    detail: str | None = None
    success: bool = True
    created_at: datetime

class OperationLogListResponse(BaseModel):
    items: list[OperationLogResponse]
    total: int
    page: int
    size: int
    total_pages: int
```

---

## 19. 超级管理员与角色管理

> **对应代码**: `app/middleware/auth.py`、`app/routers/admin.py`、`app/schemas/admin.py`、`frontend/src/views/admin/`

### 19.1 角色体系

本系统实现了四级角色体系：

```text
super_admin（超级管理员） — 最高权限，可管理所有资源
    ↑
admin（普通管理员）       — 可管理用户/订单/活动，但不能查看日志/修改角色
    ↑
organizer（组织者）       — 可编辑自己管理的活动页面（标题、内容、封面图、购票须知）
    ↑
user（普通用户）          — 基本操作权限
```

### 19.2 require_super_admin 依赖

```python
# app/middleware/auth.py

class RoleChecker:
    """角色检查依赖"""
    def __init__(self, allowed_roles: list[str]) -> None:
        self.allowed_roles = allowed_roles

    async def __call__(self, current_user: User = Depends(get_current_user)) -> User:
        if current_user.role not in self.allowed_roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"需要 {'/'.join(self.allowed_roles)} 角色",
            )
        return current_user

# 预定义的检查器
require_admin = RoleChecker(["admin", "super_admin"])        # admin 或 super_admin
require_super_admin = RoleChecker(["super_admin"])            # 仅 super_admin
require_organizer = RoleChecker(["admin", "super_admin", "organizer"])
```text

**为什么引入 organizer 角色？** 组织者是一个中间角色，既可以编辑活动内容但又不是管理员。这适用于活动主办方——他们需要能修改活动介绍、封面图、购票须知等页面内容，但不应该看到订单管理、用户管理等管理后台功能。

**为什么 require_admin 允许 super_admin？** 这是角色层级的体现。super_admin 拥有 admin 的所有权限，因此 admin 接口也应该对 super_admin 开放。需要严格限制的接口（如修改角色、查看日志）才使用 `require_super_admin`。

### 19.3 超管专属 API 端点

```python
# 修改用户角色（仅 super_admin）
PUT /api/v1/admin/users/{id}/role
Body: { "role": "admin" }  # 可选值: super_admin, admin, organizer, user

# 查看用户 IP 历史（仅 super_admin）
GET /api/v1/admin/users/{id}/ip-history
```

每次角色变更都会记录操作日志，包括：操作者 ID、目标用户 ID、原角色、新角色、IP 地址和时间。前端在角色变更时会弹出确认弹窗，防止误操作。

### 19.3.1 前端角色管理

在 `UserManageView.vue` 中，超级管理员可以看到每行用户数据的**角色下拉框**：

```vue
<!-- 仅超管可见的角色编辑下拉框 -->
<select v-if="isSuperAdmin" v-model="u.role"
  class="role-select" @change="confirmRoleChange(u)">
  <option value="user">普通用户</option>
  <option value="organizer">组织者</option>
  <option value="admin">管理员</option>
  <option value="super_admin">超级管理员</option>
</select>
```text

变更角色时弹出二次确认对话框："确定将用户 xxx 的角色修改为 xxx？"，确认后调用 `adminStore.setUserRole(userId, role)` 发送请求。后端验证当前操作者是否是 super_admin，确保安全。

### 19.4 前端角色感知布局

**AdminLayout.vue — 超管专用菜单**：

```vue
<!-- 普通管理员看到的侧边栏：没有日志管理入口 -->
<router-link to="/admin/users">用户管理</router-link>
<router-link to="/admin/orders">订单管理</router-link>
<!-- ... -->

<!-- 超级管理员额外看到：日志管理入口 -->
<template v-if="currentUserRole === 'super_admin'">
  <div class="super-admin-section">超级管理</div>
  <router-link to="/admin/logs">日志管理</router-link>
</template>
```

**路由守卫 — 双重权限检查**：

```javascript
// frontend/src/router/index.js
// admin 路由需要 requiresAdmin: true
// 日志管理路由额外需要 requiresSuperAdmin: true
{
  path: 'logs',
  name: 'admin-logs',
  component: () => import('../views/admin/LogManageView.vue'),
  meta: { requiresAdmin: true, requiresSuperAdmin: true, title: '日志管理' },
}

// 路由守卫检查
router.beforeEach((to, from, next) => {
  // 1. 检查登录
  if (to.meta.requiresAuth && !token) { /* 跳转登录 */ }

  // 2. 检查 admin 角色
  if (to.meta.requiresAdmin && token) {
    const user = JSON.parse(localStorage.getItem('user') || '{}')
    if (!['admin', 'super_admin'].includes(user.role)) {
      return next({ name: 'home' })  // 非管理员跳转首页
    }
  }

  // 3. 检查 super_admin 角色（日志管理页面特有）
  if (to.meta.requiresSuperAdmin && token) {
    const user = JSON.parse(localStorage.getItem('user') || '{}')
    if (user.role !== 'super_admin') {
      return next({ path: '/admin' })  // 普通管理员跳转管理后台首页
    }
  }
  next()
})
```text

### 19.5 UserManageView 增强

用户管理页面增加了三个超管专属功能：

1. **IP 地址列** — 显示用户的 `last_login_ip`，点击直接弹出 IP 历史模态框
2. **行内角色编辑** — 超管可以在表格中直接通过下拉框修改用户角色
3. **IP 历史模态框** — 展示该用户所有登录记录的 IP 地址、User-Agent、时间和状态

```vue
<!-- IP 地址列（可点击） -->
<td>
  <span v-if="u.last_login_ip" class="ip-address-link"
    @click="showIPHistory(u)">{{ u.last_login_ip }}</span>
</td>

<!-- 行内角色编辑（仅超管可见） -->
<select v-if="isSuperAdmin" v-model="u.role"
  class="role-select" @change="changeUserRole(u)">
  <option value="user">普通用户</option>
  <option value="organizer">组织者</option>
  <option value="admin">管理员</option>
  <option value="super_admin">超级管理员</option>
</select>

<!-- IP 历史模态框 -->
<div class="modal-overlay" v-if="ipHistoryTarget">
  <table class="ip-history-table">
    <thead>
      <tr><th>IP地址</th><th>用户代理</th><th>登录时间</th><th>状态</th></tr>
    </thead>
    <tbody>
      <tr v-for="h in ipHistoryList" :key="h.id">
        <td>{{ h.ip_address }}</td>
        <td>{{ h.user_agent }}</td>
        <td>{{ formatDate(h.login_at) }}</td>
        <td>{{ h.success ? '成功' : '失败' }}</td>
      </tr>
    </tbody>
  </table>
</div>
```

### 19.6 DashboardView 增强

超级管理员在仪表盘页面可以看到额外内容：

1. **今日概览** — 今日注册数、今日登录数、总用户数、总收入
2. **最近操作** — 显示最近的 10 条操作日志，包括操作类型标签、用户、IP 和时间

```vue
<!-- 今日概览（仅超管） -->
<div v-if="dailyStats && isSuperAdmin">
  <div class="stats-grid">
    <div class="stat-card">
      <span>今日注册数</span>
      <span>{{ dailyStats.today_registrations }}</span>
    </div>
    <div class="stat-card">
      <span>今日登录数</span>
      <span>{{ dailyStats.today_logins }}</span>
    </div>
    <!-- ... -->
  </div>
</div>

<!-- 最近操作（仅超管） -->
<div v-if="recentOps.length && isSuperAdmin">
  <ul>
    <li v-for="op in recentOps" :key="op.id">
      <span class="log-badge" :class="logBadgeClass(op.action)">{{ logActionLabel(op.action) }}</span>
      <span>{{ op.username }} - {{ op.details }}</span>
      <span>{{ op.ip_address }}</span>
      <span>{{ formatDate(op.created_at) }}</span>
    </li>
  </ul>
</div>
```text

### 19.7 测试用户

启动时，SeedService 会自动创建以下测试用户：

| ID | 用户名 | 密码 | 角色 | 手机号 | 实名 | 用途 |
| ---- | -------- | ------ | ------ | -------- | ------ | ------ |
| 1 | admin | Admin@12345 | super_admin | — | — | 超级管理员，全部权限 |
| 2 | ceshiyonghu | Test@123456 | user | 13800000002 | ✅ 张三 | 已实名普通用户 |
| 3 | youke | Youke@123456 | user | 13800000003 | ❌ | 游客（仅手机验证） |
| 4 | guanliyuan | Admin@123456 | admin | 13800000004 | — | 一般管理员（无日志权限） |

**为什么 ID 固定？** 这简化了开发测试流程，团队所有成员都使用相同的测试账号。生产环境应禁用 SeedService。

### 19.8 安全最佳实践

1. **最小权限原则**：默认创建的用户角色为 `user`，需要时才提升为 `admin` 或 `super_admin`
2. **超管操作审计**：所有角色修改操作都应记录到 OperationLog，便于追溯
3. **前端不暴露敏感信息**：普通管理员看不到 IP 历史和日志管理入口
4. **服务端双重校验**：前端只是隐藏了入口，真正的权限控制在后端 RoleChecker 中完成

---

## 20. 实名认证系统

> **对应代码**: `app/routers/auth.py`（real_name 端点）、`app/schemas/user.py`（RealNameVerifyRequest）、`app/models/user.py`（id_verified 字段）、`frontend/src/views/RealNameVerifyView.vue`

### 20.1 功能概述

实名认证用于满足票务系统的合规要求。用户需要输入真实姓名和身份证号完成认证，认证成功后才能在实名制活动中购票。

### 20.2 Schema 定义

```python
# app/schemas/user.py
class RealNameVerifyRequest(BaseModel):
    """实名认证请求"""
    real_name: str = Field(..., min_length=2, max_length=50, description="真实姓名")
    id_number: str = Field(
        ..., min_length=18, max_length=18,
        pattern=r"^\d{17}[\dXx]$", description="身份证号（18位）",
    )

class RealNameStatusResponse(BaseModel):
    """实名认证状态响应"""
    model_config = {"from_attributes": True}
    id_verified: bool
    real_name: str | None = Field(None, description="真实姓名（已脱敏）")
    id_number_mask: str | None = Field(None, description="身份证号（已脱敏）")
    id_verified_at: datetime | None = Field(None, description="实名认证时间")
```

**身份证号校验规则**：

- 必须恰好 18 位（`min_length=18, max_length=18`）
- 前 17 位为数字（`^\d{17}`）
- 第 18 位为数字或字母 X/x（`[\dXx]$`）
- 脱敏显示：`110***********1234`（仅显示前3后4）

### 20.3 API 端点

```python
# app/routers/auth.py

# POST /api/v1/auth/real-name/verify
@router.post("/real-name/verify", response_model=RealNameStatusResponse)
async def verify_real_name(
    data: RealNameVerifyRequest,
    session: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    """提交实名认证"""
    service = AuthService(session, None)
    try:
        user = await service.verify_real_name(
            user_id=current_user.id,
            real_name=data.real_name,
            id_number=data.id_number,
        )
        await session.commit()
        return user
    except ValueError as e:
        await session.rollback()
        raise HTTPException(status_code=400, detail=str(e))

# GET /api/v1/auth/real-name/status
@router.get("/real-name/status", response_model=RealNameStatusResponse)
async def get_real_name_status(
    current_user: User = Depends(get_current_user),
):
    """查询实名认证状态"""
    return current_user
```text

### 20.4 服务层实现

```python
# app/services/auth_service.py
async def verify_real_name(
    self, user_id: int, real_name: str, id_number: str,
) -> User:
    """实名认证"""
    result = await self.session.execute(select(User).where(User.id == user_id))
    user = result.scalar_one_or_none()
    if not user:
        raise ValueError("用户不存在")
    if user.id_verified:
        raise ValueError("已完成实名认证，无需重复认证")

    # 保存实名信息（脱敏存储，身份证号只保留前3后4用于展示）
    user.real_name = real_name
    masked = id_number[:3] + "***********" + id_number[-4:]
    user.id_number = masked
    user.id_verified = True
    user.id_verified_at = datetime.now(timezone.utc).replace(tzinfo=None)

    # 记录操作日志
    await self.create_operation_log(
        user_id=user_id, username=user.username,
        action="verify_real_name", ip_address="",
        detail=f"实名认证: {real_name}", success=True,
    )
    return user
```

### 20.5 前端页面 (`RealNameVerifyView.vue`)

实名认证页面设计为独立路由 `/real-name-verify`，在导航栏中有独立入口：

```vue
<template>
  <div class="profile-container">
    <!-- 用户信息卡片 -->
    <div class="profile-card">
      <div class="profile-header">
        <div class="profile-avatar">
          {{ user.username.charAt(0).toUpperCase() }}
        </div>
        <div>
          <h2>{{ user.username }}</h2>
          <p class="profile-meta">{{ user.email }} · {{ user.phone || '未绑定手机' }}</p>
        </div>
      </div>
    </div>

    <!-- 实名认证区域 -->
    <div class="profile-card">
      <h3>实名认证</h3>

      <!-- 已认证状态 -->
      <div v-if="status.id_verified" class="verified-status">
        <div class="verified-badge">✅ 已实名认证</div>
        <p>姓名：{{ status.real_name || '—' }}</p>
        <p>身份证：{{ status.id_number_mask || '—' }}</p>
        <p>认证时间：{{ formatDate(status.id_verified_at) }}</p>
      </div>

      <!-- 认证表单（未认证时显示） -->
      <form v-else @submit.prevent="handleVerify">
        <div class="form-group">
          <label>真实姓名</label>
          <input v-model="form.real_name" placeholder="请输入真实姓名" required />
        </div>
        <div class="form-group">
          <label>身份证号码</label>
          <input v-model="form.id_number" placeholder="请输入18位身份证号"
            maxlength="18" required />
        </div>
        <button type="submit" class="btn btn-primary"
          :disabled="verifying">
          {{ verifying ? '提交中...' : '提交认证' }}
        </button>
        <p v-if="error" class="error-message">{{ error }}</p>
      </form>
    </div>
  </div>
</template>
```text

**页面状态处理**：

1. **加载中**：显示骨架屏/加载提示
2. **已认证**：显示认证信息（脱敏姓名+脱敏身份证号+认证时间），隐藏表单
3. **未认证**：显示认证表单，包含实时输入校验
4. **提交中**：按钮显示"提交中..."并禁用，防止重复提交
5. **提交失败**：显示错误消息（如"身份证号格式不正确"）

### 20.6 路由配置

```javascript
// frontend/src/router/index.js
{
  path: '/real-name-verify',
  name: 'real-name-verify',
  component: () => import('../views/RealNameVerifyView.vue'),
  meta: { requiresAuth: true },
}
```

需要登录才能访问（`requiresAuth: true`），因为实名认证是面向已登录用户的功能。

### 20.7 实名认证与购票流程的关系

实名认证是某些活动的购票前提：

1. 活动创建时可设置 `requires_real_name` 标记（默认 true）
2. 用户下单时，后端 `order_service.py` 检查：

   ```python
   if event.requires_real_name and not current_user.id_verified:
       raise ValueError("该活动需要实名认证后才能购票")
   ```text

3. 前端在购票前通过 `GET /api/v1/auth/real-name/status` 检查认证状态

4. 未认证用户会被引导到 `/real-name-verify` 页面
5. `PurchaseNotesResponse` 中包含 `requires_real_name` 字段供前端展示

---

## 21. 多票种系统 (Multi-Ticket System)

### 21.1 概述

多票种系统允许一个活动设置多种票价和库存（如 VIP、普通票、早鸟票），用户在购票页面可选择每种票种的数量组合下单。系统支持：

- 一个活动多个票种（TicketType），每个票种独立定价、独立库存
- 多票种组合下单：一次订单可包含不同票种的不同数量
- 传统单票种兼容：没有配置票种的活动自动获得一个默认票种（标准票）
- 双层超卖防护：Redis Lua 原子扣减 + MySQL 乐观锁

### 21.2 数据模型

#### TicketType（票种）

```python
# app/models/ticket_type.py
class TicketType(Base):
    __tablename__ = "ticket_types"
    
    id = Column(BigInteger, primary_key=True, autoincrement=True)
    event_id = Column(BigInteger, ForeignKey("events.id", ondelete="CASCADE"), nullable=False, index=True)
    name = Column(String(100), nullable=False, comment="票种名称（如 VIP/普通/早鸟）")
    description = Column(String(500), comment="票种描述")
    price = Column(Numeric(10, 2), nullable=False, comment="该票种单价")
    total_stock = Column(Integer, nullable=False, comment="该票种总库存")
    sold_count = Column(Integer, default=0, comment="已售数量")
    max_per_order = Column(Integer, default=10, comment="每单限购数量")
    sort_order = Column(Integer, default=0, comment="排序号")
    is_active = Column(Boolean, default=True)
    version = Column(Integer, default=1, comment="乐观锁版本号")
```

- `event_id` FK 指向 events 表，CASCADE 删除
- `version` 字段用于乐观锁并发控制
- `sold_count` 实时跟踪已售数量

#### OrderItem（订单明细）

```python
# app/models/order_item.py
class OrderItem(Base):
    __tablename__ = "order_items"
    
    id = Column(BigInteger, primary_key=True, autoincrement=True)
    order_id = Column(BigInteger, ForeignKey("orders.id", ondelete="CASCADE"), nullable=False)
    ticket_type_id = Column(BigInteger, ForeignKey("ticket_types.id", ondelete="SET NULL"), nullable=True)
    ticket_type_name = Column(String(100), comment="票种名称（下单时快照）")
    quantity = Column(Integer, nullable=False, comment="购买数量")
    unit_price = Column(Numeric(10, 2), nullable=False, comment="单价（快照）")
    subtotal = Column(Numeric(10, 2), nullable=False, comment="小计")
```text

- `ticket_type_name` 和 `unit_price` 是下单时的快照，防止后续修改影响历史订单
- `ticket_type_id` 可为 NULL（SET NULL），删除票种不影响历史订单

### 21.3 API 端点

#### 票种 CRUD

| 方法 | 路径 | 权限 | 说明 |
| ------ | ------ | ------ | ------ |
| GET | `/api/v1/events/{event_id}/ticket-types` | 公开 | 获取活动的所有票种 |
| POST | `/api/v1/events/{event_id}/ticket-types` | 管理员/组织者 | 创建票种 |
| PUT | `/api/v1/events/{event_id}/ticket-types/{id}` | 管理员/组织者 | 更新票种 |
| DELETE | `/api/v1/events/{event_id}/ticket-types/{id}` | 管理员/组织者 | 删除票种 |

#### 活动创建时指定票种

```json
POST /api/v1/events/
{
  "title": "演唱会",
  "price": 399,
  "total_stock": 1000,
  "ticket_types": [
    {"name": "VIP", "price": 888, "total_stock": 100, "max_per_order": 4},
    {"name": "普通票", "price": 399, "total_stock": 600, "max_per_order": 10},
    {"name": "早鸟票", "price": 199, "total_stock": 300, "max_per_order": 2}
  ]
}
```

不传 ticket_types 则自动创建默认票种（使用 event.price 和 event.total_stock）。

#### 多票种下单

```json
POST /api/v1/orders/
{
  "event_id": 1,
  "items": [
    {"ticket_type_id": 1, "quantity": 2},
    {"ticket_type_id": 2, "quantity": 1}
  ]
}
```text

传统格式仍然兼容：

```json
POST /api/v1/orders/
{
  "event_id": 1,
  "quantity": 2
}
```

订单响应包含 items 数组：

```json
{
  "id": 17,
  "items": [
    {"id": 1, "ticket_type_name": "VIP", "quantity": 2, "unit_price": 888, "subtotal": 1776},
    {"id": 2, "ticket_type_name": "普通票", "quantity": 1, "unit_price": 399, "subtotal": 399}
  ],
  "total_amount": 2175
}
```text

### 21.4 双层超卖防护

下单时执行两层防护：

**第一层：Redis Lua 原子扣减（快速路径）**

```lua
-- 批量检查并扣减多个票种的库存
for i, item in ipairs(items) do
  local key = "stock:" .. event_id .. ":" .. item.ticket_type_id
  local stock = redis.call("GET", key)
  if not stock or tonumber(stock) < item.quantity then
    return {err = "库存不足", index = i}
  end
end
for i, item in ipairs(items) do
  local key = "stock:" .. event_id .. ":" .. item.ticket_type_id
  redis.call("DECRBY", key, item.quantity)
end
return {ok = "success"}
```

**第二层：MySQL 乐观锁（慢速路径）**

```sql
UPDATE ticket_types 
SET sold_count = sold_count + :qty, version = version + 1
WHERE id = :tt_id AND sold_count + :qty <= total_stock AND version = :ver
```text

任一失败触发全部回滚：Redis 恢复库存 + 所有已更新的 TicketType 回退 sold_count。

### 21.5 前端实现

#### EventDetail.vue - 多票种 Stepper 选择器

```vue
<div v-for="tt in ticketTypes" :key="tt.id" class="ticket-type-item">
  <div class="tt-info">
    <div class="tt-name">{{ tt.name }}</div>
    <div class="tt-price">¥{{ formatPrice(tt.price) }}</div>
    <div class="tt-stock">剩余 {{ Math.max(0, tt.total_stock - tt.sold_count) }} 张</div>
  </div>
  <div class="quantity-control">
    <button @click="decrement(tt)">−</button>
    <input v-model.number="selections[tt.id]" type="number" />
    <button @click="increment(tt)">+</button>
  </div>
</div>
```

- `selections` 是 reactive 对象，key 为 ticket_type_id
- `totalQty` 和 `totalAmount` 为 computed 属性，自动汇总
- `buy()` 函数构建 items 数组提交

#### Checkout.vue - 订单明细展示

```vue
<div v-if="order.items?.length" class="order-items-breakdown">
  <div v-for="item in order.items" :key="item.id" class="price-row">
    <span>{{ item.ticket_type_name }} x{{ item.quantity }}</span>
    <span>¥{{ formatMoney(item.subtotal) }}</span>
  </div>
</div>
```text

#### EventManageView.vue - 后台票种管理

后台创建/编辑活动时可添加多个票种，每行包含：名称、价格、库存、限购数量。

### 21.6 取消订单的库存归还

取消订单时自动归还各票种库存：

```python
# 遍历订单的所有 OrderItem，逐个归还
for item in order.items:
    # Redis 恢复
    await redis.decrby(f"stock:{event.id}:{item.ticket_type_id}", -item.quantity)
    # MySQL 乐观锁更新
    await session.execute(
        update(TicketType)
        .where(TicketType.id == item.ticket_type_id)
        .where(TicketType.version == tt.version)
        .values(sold_count=TicketType.sold_count - item.quantity, version=TicketType.version + 1)
    )
```

### 21.7 数据库迁移

`0011_add_ticket_types_order_items.py` 完成：

1. 创建 `ticket_types` 表
2. 创建 `order_items` 表
3. 为 events 表添加 `banner_images` JSON 字段
4. 为 tickets 表添加 `ticket_type_id` 和 `ticket_type_name` 字段
5. 数据迁移：为已有活动创建默认"标准票"票种
