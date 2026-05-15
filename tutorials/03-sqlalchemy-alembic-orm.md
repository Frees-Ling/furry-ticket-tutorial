# 第3章：SQLAlchemy 2.0 + Alembic — Python ORM 层

## 本章学习目标

本章将学习 SQLAlchemy 2.0 的全部核心知识，从 Core 到 ORM，从同步到异步，最终使用 Alembic 完成数据库迁移。

**学习路径：**

1. ✅ SQLAlchemy 概述与安装
2. ✅ SQLAlchemy Core — 表达式语言与引擎
3. ✅ Declarative Models — ORM 模型定义
4. ✅ Relationships — 关系映射（一对多/多对多）
5. ✅ Async Sessions — 异步会话完全教程
6. ✅ select() 2.0 风格查询构建器
7. ✅ Alembic — 数据库迁移管理
8. ✅ 项目：ORM 模型定义 + 初始迁移 + 种子数据

---

## 1. SQLAlchemy 概述

### 1.1 什么是 ORM？

ORM（Object-Relational Mapping，对象关系映射）是一种将关系型数据库的表结构映射到面向对象编程语言的技术。

```text
┌─────────────┐          ┌─────────────┐          ┌─────────────┐
│  Python     │          │  SQLAlchemy  │          │  MySQL      │
│  对象       │  ───→   │  ORM 层      │  ───→   │  表         │
│             │          │              │          │             │
│ User(id=1,  │          │ SELECT *     │          │  users 表   │
│  name="张三")│  ←───   │ FROM users   │  ←───   │  id=1,      │
│             │          │ WHERE id=1   │          │  name='张三'│
└─────────────┘          └─────────────┘          └─────────────┘
```

**ORM 的优势：**

- 用 Python 对象操作数据库，不用写 SQL
- 自动处理连接管理、事务边界
- 防止 SQL 注入（参数化查询）
- 跨数据库兼容

### 1.2 SQLAlchemy 架构

SQLAlchemy 由两层构成：

```text
┌─────────────────────────────────────┐
│  SQLAlchemy ORM                      │
│  ┌─────────────────────────────────┐ │
│  │  ORM 层 (declarative/session)   │ │
│  │  - 模型定义                      │ │
│  │  - 关系映射                      │ │
│  │  - 工作单元 (Unit of Work)       │ │
│  └──────────┬──────────────────────┘ │
│             │                        │
│  ┌──────────▼──────────────────────┐ │
│  │  Core 层 (SQL Expression)       │ │
│  │  - Schema / Types               │ │
│  │  - SQL 表达式语言               │ │
│  │  - Engine / Connection          │ │
│  │  - Dialect (方言: asyncmy)      │ │
│  └─────────────────────────────────┘ │
└─────────────────────────────────────┘
```

**为什么分层设计？**

- Core 层：直接使用 SQL 表达式，与数据库无关
- ORM 层：在 Core 基础上的对象映射，方便业务代码
- 可以在同一项目中混合使用两层

### 1.3 安装

```bash
# SQLAlchemy + asyncmy 驱动 + Alembic
pip install sqlalchemy[asyncio] asyncmy alembic

# 验证安装
python -c "import sqlalchemy; print(sqlalchemy.__version__)"
# 输出：2.0.x
```text

**驱动选择：**

| 驱动 | 同步/异步 | 特点 |
| ------ | ----------- | ------ |
| pymysql | 同步 | 纯 Python，较慢 |
| mysqlclient | 同步 | C 扩展，最快 |
| **asyncmy** | 异步 | 异步 MySQL 驱动，推荐 |
| aiomysql | 异步 | 基于 pymysql 异步化 |

---

## 2. SQLAlchemy Core（表达式语言）

### 2.1 Engine — 数据库引擎

```python
from sqlalchemy import create_engine

# 同步引擎（非项目使用，仅演示）
sync_engine = create_engine(
    "mysql+pymysql://root:root123@localhost:3306/ticket_dev?charset=utf8mb4",
    echo=True,           # 打印 SQL 语句
    pool_size=5,         # 连接池大小
    max_overflow=10,     # 最大溢出连接数
)
```

**连接池（QueuePool）工作原理：**

```text
初始化：
┌─ 连接池 ─────────────────────┐
│  [conn1] [conn2] [conn3]     │  ← pool_size=3
│  [conn4] [conn5] [conn6]     │  ← max_overflow=3（溢出连接）
└──────────────────────────────┘

获取连接：
1. 从池中取空闲连接（复用）
2. 若无空闲且未达 pool_size，创建新连接
3. 若达 pool_size，等待超时（pool_timeout=30）
4. 超出 max_overflow 且无空闲 → 抛出 TimeoutError

归还连接：
1. 放回池中 → 可供下次复用
2. 连接池保持 pool_size 个连接
3. 超出 pool_size 的连接在归还后关闭

连接有效性检查：
- pool_pre_ping=True：每次获取连接前执行 SELECT 1 测试
- 无效连接自动丢弃重建
```

### 2.2 Table 与 MetaData

```python
from sqlalchemy import Table, MetaData, Column, Integer, String, DateTime, func

# MetaData 是表定义的容器
metadata_obj = MetaData()

# 定义表（Core 风格）
users_table = Table(
    "users",
    metadata_obj,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("username", String(50), nullable=False),
    Column("email", String(255), nullable=False),
    Column("created_at", DateTime, server_default=func.now()),
)

# 创建所有表
metadata_obj.create_all(engine)
```text

### 2.3 插入数据（Core）

```python
from sqlalchemy import insert

# 单行插入
stmt = insert(users_table).values(
    username="张三",
    email="zhangsan@example.com",
)
with engine.connect() as conn:
    result = conn.execute(stmt)
    conn.commit()
    print(result.inserted_primary_key)  # 返回自增ID

# 多行插入
stmt = insert(users_table).values([
    {"username": "李四", "email": "lisi@example.com"},
    {"username": "王五", "email": "wangwu@example.com"},
])
with engine.connect() as conn:
    conn.execute(stmt)
    conn.commit()
```

### 2.4 查询数据（Core）

```python
from sqlalchemy import select, text

# 基本查询
stmt = select(users_table).where(users_table.c.username == "张三")
with engine.connect() as conn:
    result = conn.execute(stmt)
    for row in result:
        print(row.id, row.username, row.email)

# 选择特定列
stmt = select(users_table.c.id, users_table.c.username)
result = conn.execute(stmt)
rows = result.all()  # 返回所有行

# 聚合
from sqlalchemy import func
stmt = select(
    users_table.c.role,
    func.count(users_table.c.id).label("count"),
).group_by(users_table.c.role)

# JOIN
from sqlalchemy import join
j = users_table.join(orders_table, users_table.c.id == orders_table.c.user_id)
stmt = select(users_table, orders_table).select_from(j)

# 子查询
subq = select(orders_table.c.user_id, func.count().label("order_count")).group_by(orders_table.c.user_id).subquery()
stmt = select(users_table, subq.c.order_count).join(subq, users_table.c.id == subq.c.user_id)

# 原生 SQL
stmt = text("SELECT * FROM users WHERE username = :name")
result = conn.execute(stmt, {"name": "张三"})
```text

### 2.5 TypeDecorator — 自定义类型

```python
import json
from sqlalchemy.types import TypeDecorator, TEXT

class JSONType(TypeDecorator):
    """自定义 JSON 类型：Python dict/list ↔ MySQL TEXT 序列化"""
    impl = TEXT

    def process_bind_param(self, value, dialect):
        """Python → 数据库"""
        if value is not None:
            return json.dumps(value, ensure_ascii=False)
        return None

    def process_result_value(self, value, dialect):
        """数据库 → Python"""
        if value is not None:
            return json.loads(value)
        return None

    # 复制类型（SQLAlchemy 要求）
    def copy(self, **kw):
        return JSONType()

# 使用
# extra_info = Column(JSONType, nullable=True)
```

---

## 3. Declarative Models（声明式模型）

### 3.1 基础模型定义

SQLAlchemy 2.0 推荐使用声明式映射（Declarative Mapping）：

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    """所有 ORM 模型的基类"""
    pass

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    username: Mapped[str] = mapped_column(String(50), unique=True)
    email: Mapped[str] = mapped_column(String(255), unique=True)

    def __repr__(self) -> str:
        return f"User(id={self.id}, username={self.username})"
```text

**Mapped 与 mapped_column 详解：**

```python
from typing import Optional
from datetime import datetime

class User(Base):
    __tablename__ = "users"

    # Mapped[类型] 告诉 SQLAlchemy 这是映射列
    # mapped_column 定义列属性

    # 主键
    id: Mapped[int] = mapped_column(primary_key=True)

    # 带字符串长度的字符串列
    username: Mapped[str] = mapped_column(String(50), unique=True)

    # 可为空的列：使用 Optional
    phone: Mapped[Optional[str]] = mapped_column(String(20))

    # 有默认值的列
    is_active: Mapped[bool] = mapped_column(default=True)

    # 服务器默认值
    created_at: Mapped[datetime] = mapped_column(
        DateTime,
        server_default=func.now()     # MySQL 端的默认值
    )

    # Python 端默认值 + 服务器默认值
    updated_at: Mapped[datetime] = mapped_column(
        DateTime,
        default=func.now(),           # Python 端
        server_default=func.now(),    # MySQL 端
        onupdate=func.now(),          # 更新时
    )

    # 带注释
    email: Mapped[str] = mapped_column(String(255), comment="用户邮箱")

    # 带索引
    role: Mapped[str] = mapped_column(String(20), index=True)

    # 枚举
    status: Mapped[str] = mapped_column(
        String(20),
        default="active",
        comment="状态: active/inactive"
    )

    # 不映射到数据库的属性（Python 端使用）
    _password: Mapped[str] = mapped_column("password_hash", String(255))
    # 列名在数据库中叫 password_hash，但 Python 中叫 _password
```

### 3.2 **table_args** — 表级参数

```python
class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True)
    order_no: Mapped[str] = mapped_column(String(32))
    status: Mapped[str] = mapped_column(String(20))

    __table_args__ = (
        # 联合唯一约束
        UniqueConstraint("user_id", "event_id", name="uk_user_event"),
        # 复合索引
        Index("idx_status_date", "status", "created_at"),
        # 表注释
        {"comment": "订单表", "mysql_engine": "InnoDB"},
    )
```text

### 3.3 全部列类型映射

| MySQL 类型 | Python 类型 | mapped_column 参数 |
| ----------- | ------------ | ------------------- |
| BIGINT | int | `BigInteger` 或 `Integer` |
| INT | int | `Integer` |
| SMALLINT | int | `SmallInteger` |
| TINYINT | bool/int | `Boolean` 或 `SmallInteger` |
| VARCHAR | str | `String(N)` |
| CHAR | str | `String(N)` |
| TEXT | str | `Text` |
| DECIMAL(p,s) | Decimal | `Numeric(p, s)` 或 `DECIMAL` |
| FLOAT | float | `Float` |
| DOUBLE | float | `Double` |
| DATE | datetime.date | `Date` |
| DATETIME | datetime.datetime | `DateTime` |
| TIMESTAMP | datetime.datetime | `TIMESTAMP` |
| TIME | datetime.time | `Time` |
| JSON | dict/list | `JSON` |
| BLOB | bytes | `LargeBinary` |
| ENUM | str | `Enum('a', 'b')` |
| UUID | uuid.UUID | `Uuid` |

### 3.4 SQLAlchemy 1.x vs 2.0 语法对照

| 功能 | 1.x 旧语法（不推荐） | 2.0 新语法（推荐） |
| ------ | -------------------- | ------------------ |
| 列定义 | `name = Column(String(50))` | `name: Mapped[str] = mapped_column(String(50))` |
| 查询 | `session.query(User).filter(...)` | `select(User).where(...)` |
| 执行 | `query.all()` | `session.execute(stmt).scalars().all()` |
| 连接 | `query.join(User.addr)` | `select(User).join(Address)` |
| ORM 基类 | `declarative_base()` | `class Base(DeclarativeBase)` |
| 更新 | `query.update({...})` | `update(User).where(...).values(...)` |

---

## 4. Relationships — 关系映射

### 4.1 一对多 / 多对一

```python
from sqlalchemy.orm import Mapped, mapped_column, relationship
from typing import List, Optional

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(String(50))

    # 一对多：一个用户有多个订单
    # back_populates 指定反向引用对方的关系名
    orders: Mapped[List["Order"]] = relationship(
        back_populates="user",
        cascade="all, delete-orphan",  # 级联删除
    )

class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))

    # 多对一：多个订单属于一个用户
    user: Mapped["User"] = relationship(back_populates="orders")
```

**关系背后的原理：**

```text
SQL 层面：
- 通过外键 (user_id) 实现表关联
- 查询时生成 JOIN 语句

ORM 层面：
- user.orders → SELECT * FROM orders WHERE user_id = ?
- order.user  → SELECT * FROM users WHERE id = ?
```

### 4.2 一对一

```python
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    profile: Mapped["Profile"] = relationship(
        back_populates="user",
        uselist=False,  # 关键：uselist=False 表示一对一
    )

class Profile(Base):
    __tablename__ = "profiles"
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), unique=True)
    bio: Mapped[str] = mapped_column(Text)
    user: Mapped["User"] = relationship(back_populates="profile")
```text

### 4.3 多对多

```python
# 关联表（不映射为 ORM 模型）
user_event_association = Table(
    "user_event_favorites",
    Base.metadata,
    Column("user_id", ForeignKey("users.id"), primary_key=True),
    Column("event_id", ForeignKey("events.id"), primary_key=True),
)

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    favorite_events: Mapped[List["Event"]] = relationship(
        secondary=user_event_association,  # 指定关联表
        back_populates="favorited_by",
    )

class Event(Base):
    __tablename__ = "events"
    id: Mapped[int] = mapped_column(primary_key=True)
    favorited_by: Mapped[List["User"]] = relationship(
        secondary=user_event_association,
        back_populates="favorite_events",
    )
```

### 4.4 lazy 加载选项

```python
# relationship 的 lazy 参数控制关联数据的加载时机

class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))

    # lazy='select'（默认）：延迟加载，访问时才查询
    # SELECT * FROM users WHERE id = ?
    user: Mapped["User"] = relationship(lazy="select")

    # lazy='joined'：使用 JOIN 立即加载
    # SELECT * FROM orders JOIN users ON ...（LEFT OUTER JOIN）
    user: Mapped["User"] = relationship(lazy="joined")

    # lazy='subquery'：使用子查询立即加载
    # SELECT * FROM orders; SELECT * FROM users WHERE id IN (SELECT user_id FROM orders)
    user: Mapped["User"] = relationship(lazy="subquery")

    # lazy='selectin'：使用 IN 子句加载（推荐！）
    # SELECT * FROM orders; SELECT * FROM users WHERE id IN (1, 2, 3...)
    user: Mapped["User"] = relationship(lazy="selectin")

    # lazy='dynamic'：返回 Query 对象（1.x 旧语法，2.0 不推荐）
```text

**lazy 加载选项对比：**

| 策略 | 查询次数 | N+1 问题 | 适用场景 |
| ------ | --------- | ---------- | --------- |
| select（默认） | 1 + N | ✅ 有风险 | 关联数据不常访问 |
| joined | 1 | ❌ 无 | 总是需要关联数据 |
| selectin | 1 + 1 | ❌ 无（批量） | **推荐，平衡性能** |
| subquery | 1 + 1 | ❌ 无 | 大数据量分页时 |

### 4.5 N+1 查询问题详解

```python
# ========== 错误示例：N+1 问题 ==========
# 查询所有订单，同时打印每个订单的用户名

# 如果不加任何优化：
orders = session.execute(select(Order)).scalars().all()
# 1 次查询：SELECT * FROM orders

for order in orders:
    print(order.user.username)  # 每次访问都查询一次用户！
    # N 次查询：SELECT * FROM users WHERE id = ?
# 总共 1 + N 次查询！

# ========== 解决方案 ==========

# 方案一：selectinload（推荐）
from sqlalchemy.orm import selectinload

stmt = select(Order).options(selectinload(Order.user))
orders = session.execute(stmt).scalars().all()
# 第1次：SELECT * FROM orders
# 第2次：SELECT * FROM users WHERE id IN (1, 2, 3, ...)

# 方案二：joinedload
from sqlalchemy.orm import joinedload

stmt = select(Order).options(joinedload(Order.user))
orders = session.execute(stmt).scalars().all()
# 1次：SELECT orders.*, users.* FROM orders LEFT JOIN users ON ...

# 方案三：在 relationship 中设置 lazy="selectin"
class Order(Base):
    user: Mapped["User"] = relationship(lazy="selectin")
```

### 4.6 cascade 级联操作

```python
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)

    orders: Mapped[List["Order"]] = relationship(
        back_populates="user",
        cascade="all, delete-orphan",  # 级联行为
    )

# cascade 选项详解：
# save-update：      将关联对象添加到 Session（默认）
# merge：            合并关联对象（默认）
# refresh-expire：   刷新时级联（默认）
# expunge：          从 Session 移除时级联
# delete：           删除父对象时级联删除子对象
# delete-orphan：    解除关联时删除子对象（严格模式）
# all：              save-update + merge + refresh-expire + expunge + delete
# all, delete-orphan：最常用的级联模式

# 示例：
user = session.get(User, 1)
session.delete(user)  # 自动删除 user 的所有 orders！
```text

### 4.7 passive_deletes

```sql
-- 更高效的级联：使用数据库级别的 ON DELETE CASCADE
-- 而不是 SQLAlchemy 逐行删除

-- 建表时定义外键 CASCADE
CREATE TABLE orders (
    user_id BIGINT,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

```python
class Order(Base):
    __tablename__ = "orders"
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id", ondelete="CASCADE"))

class User(Base):
    orders: Mapped[List["Order"]] = relationship(
        passive_deletes=True,  # 告诉 SQLAlchemy：数据库自己处理级联，不用你管
    )
```text

### 4.8 Association Proxy

```python
from sqlalchemy.ext.associationproxy import association_proxy

# 场景：通过订单直接访问票号，无需经过 order.tickets
class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(primary_key=True)

    tickets: Mapped[List["Ticket"]] = relationship(back_populates="order")

    # 代理：将 order.ticket_numbers 映射为 ticket.ticket_no 的列表
    ticket_numbers: list[str] = association_proxy("tickets", "ticket_no")

# 使用
order = session.get(Order, 1)
print(order.ticket_numbers)  # ✅ ['T001', 'T002', 'T003']
# 相当于 [t.ticket_no for t in order.tickets]
```

---

## 5. Async Sessions（异步会话）

### 5.1 异步引擎与会话

```python
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)

# 创建异步引擎
engine = create_async_engine(
    settings.DATABASE_URL,  # mysql+asyncmy://root:root123@localhost:3306/ticket_dev?charset=utf8mb4
    echo=settings.DEBUG,
    pool_size=10,
    max_overflow=5,
    pool_pre_ping=True,
    pool_recycle=3600,  # 回收超过 1 小时的连接
)

# 异步会话工厂
async_session_factory = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,  # 提交后不使对象过期（可继续访问）
)
```text

### 5.2 获得会话的两种方式

```python
# 方式一：上下文管理器（推荐）
async with async_session_factory() as session:
    result = await session.execute(select(User))
    user = result.scalar_one_or_none()
    # 退出上下文时自动关闭

# 方式二：依赖注入（FastAPI 使用）
async def get_db() -> AsyncSession:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()
```

### 5.3 增删改查操作

```python
# ========== 插入 ==========
async with async_session_factory() as session:
    user = User(username="张三", email="zhangsan@example.com")
    session.add(user)        # 加入会话（INSERT 延迟到 flush）
    await session.commit()   # 提交事务（真正执行 INSERT）
    print(user.id)           # 提交后能获取自增 ID

# ========== 批量插入 ==========
async with async_session_factory() as session:
    users = [
        User(username="李四", email="lisi@example.com"),
        User(username="王五", email="wangwu@example.com"),
    ]
    session.add_all(users)
    await session.commit()

# ========== 查询 ==========
async with async_session_factory() as session:
    # 获取单行（主键）
    user = await session.get(User, 1)

    # 执行查询
    stmt = select(User).where(User.username == "张三")
    result = await session.execute(stmt)

    # 获取 0 或 1 行
    user = result.scalar_one_or_none()

    # 获取所有行
    users = result.scalars().all()

    # 获取唯一一行（不存在或多余一行会报错）
    user = result.scalar_one()

    # 获取第一行
    user = result.first()

# ========== 更新 ==========
async with async_session_factory() as session:
    user = await session.get(User, 1)
    user.email = "newemail@example.com"  # 修改对象
    await session.commit()               # 自动生成 UPDATE

    # 批量更新
    stmt = (
        update(User)
        .where(User.role == "inactive")
        .values(is_active=False)
    )
    await session.execute(stmt)
    await session.commit()

# ========== 删除 ==========
async with async_session_factory() as session:
    user = await session.get(User, 1)
    await session.delete(user)  # 软删除或硬删除
    await session.commit()
```text

### 5.4 Session 状态管理

```python
from sqlalchemy import inspect

# SQLAlchemy 跟踪对象的四种状态：
# transient（瞬态）：      刚刚创建，未与 session 关联
# pending（待定）：        已 session.add()，未 flush 到数据库
# persistent（持久化）：   已 flush/commit，对象与会话关联
# detached（分离）：       对象曾在 session 中，但 session 已关闭

user = User(username="test")  # transient
session.add(user)              # pending
await session.flush()          # persistent（写入数据库但未提交）
await session.commit()         # persistent
await session.close()          # detached（session 关闭后对象分离）

# 查看对象状态
insp = inspect(user)
print(insp.transient)     # True/False
print(insp.pending)       # True/False
print(insp.persistent)    # True/False
print(insp.detached)      # True/False

# 将分离的对象重新加入新 session
session.add(user)  # 从 detached 变为 persistent（会执行一次 SELECT 检查版本）
session.merge(user)  # 合并（不会先 SELECT，直接覆盖）
```

### 5.5 flush vs commit 的区别

```python
# flush：将会话的修改发送到数据库（生成 SQL 语句），但未提交事务
# commit：提交事务（持久化修改），同时触发 flush

# 需要提前 flush 的场景：获取生成的主键或默认值
async with async_session_factory() as session:
    user = User(username="测试", email="test@example.com")
    session.add(user)
    await session.flush()          # 发送 INSERT，获取 id
    print(user.id)                 # 现在可以获取自增ID
    # ... 可以用 user.id 做其他操作 ...
    await session.commit()         # 提交事务
```text

### 5.6 事务边界管理

```python
# ========== 自动事务管理（推荐）==========
async def create_user(session: AsyncSession, data: dict):
    """一个事务内完成多个操作"""
    user = User(**data)
    session.add(user)
    await session.flush()  # 获取 id

    log = OperationLog(user_id=user.id, action="create")
    session.add(log)
    await session.commit()

    return user

# ========== 嵌套事务 ==========
async with async_session_factory() as session:
    try:
        # 外层事务
        user = User(username="张三")
        session.add(user)
        await session.flush()

        # 保存点（内层事务）
        await session.begin_nested()
        try:
            log = OperationLog(user_id=user.id, action="test")
            session.add(log)
            await session.flush()  # 到保存点
        except Exception:
            await session.rollback()  # 回滚到保存点

        await session.commit()
    except Exception:
        await session.rollback()
```

---

## 6. select() 2.0 查询构建器

### 6.1 基础查询

```python
from sqlalchemy import select, func, and_, or_, desc, asc

# 查询所有行
stmt = select(User)

# 条件过滤
stmt = select(User).where(User.username == "张三")
stmt = select(User).where(User.id > 10)
stmt = select(User).where(
    and_(User.is_active == True, User.role == "admin")
)
stmt = select(User).where(
    or_(User.status == "pending", User.status == "active")
)
stmt = select(User).where(User.username.in_(["张三", "李四", "王五"]))
stmt = select(User).where(User.email.like("%@example.com"))
stmt = select(User).where(User.created_at.between("2024-01-01", "2024-12-31"))
```text

### 6.2 JOIN 查询

```python
# 隐式 JOIN（通过 relationship）
stmt = select(Order).join(Order.user)
stmt = select(Order).join(Order.user).where(User.username == "张三")

# 显式 JOIN（指定 ON 条件）
stmt = select(Order).join(User, Order.user_id == User.id)

# LEFT JOIN
stmt = select(User).outerjoin(Order, User.id == Order.user_id)

# 多表 JOIN
stmt = select(Order, User, Event).join(Order.user).join(Order.event)

# 关联查询
stmt = select(User).join(User.orders).where(Order.status == "paid")
```

### 6.3 聚合查询

```python
# 统计
stmt = select(func.count(User.id)).where(User.is_active == True)

# 分组统计
stmt = select(
    User.role,
    func.count(User.id).label("count"),
    func.max(User.created_at).label("last_created"),
).group_by(User.role)

# HAVING
stmt = select(
    Order.user_id,
    func.sum(Order.total_amount).label("total_spent"),
).group_by(Order.user_id).having(func.sum(Order.total_amount) > 1000)
```text

### 6.4 排序与分页

```python
# 排序
stmt = select(User).order_by(User.created_at.desc())
stmt = select(User).order_by(desc(User.created_at), asc(User.username))

# 分页（传统 OFFSET 分页）
stmt = select(User).order_by(User.id).offset(20).limit(10)

# 游标分页（高效）
last_id = 20  # 上一页的最后 ID
stmt = select(User).where(User.id > last_id).order_by(User.id).limit(10)
```

### 6.5 子查询

```python
# 标量子查询：每个订单关联该用户的订单总数
subq = (
    select(func.count(Order.id))
    .where(Order.user_id == User.id)
    .correlate(User)  # 关联外层查询的 User 表
    .scalar_subquery()
)

stmt = select(User.id, User.username, subq.label("order_count"))

# 表子查询
subq = (
    select(Order.user_id, func.count().label("cnt"))
    .group_by(Order.user_id)
    .subquery()
)

stmt = select(User, subq.c.cnt).join(subq, User.id == subq.c.user_id)
```text

### 6.6 UNION

```python
# UNION（去重）
stmt1 = select(User.username, User.email).where(User.role == "admin")
stmt2 = select(User.username, User.email).where(User.is_active == False)
union_stmt = union(stmt1, stmt2).order_by("username")

# UNION ALL（不去重，更快）
union_all_stmt = union_all(stmt1, stmt2)
```

### 6.7 WITH FOR UPDATE（锁定读）

```python
# 悲观锁
stmt = (
    select(Inventory)
    .where(Inventory.event_id == 1)
    .with_for_update()  # 加排他锁
)
inventory = await session.execute(stmt).scalar_one()

# 跳过锁定行（MySQL 8.0+）
stmt = (
    select(Task)
    .where(Task.status == "pending")
    .order_by(Task.priority.desc())
    .limit(1)
    .with_for_update(skip_locked=True)  # 跳过被其他事务锁定的行
)
```text

### 6.8 select() vs query() 对比

```python
# 1.x 旧语法（不推荐）
session.query(User).filter(User.id == 1).first()
session.query(User).join(Order).filter(Order.status == "paid")
session.query(User).options(joinedload(User.orders)).all()

# 2.0 新语法（推荐）
session.execute(select(User).where(User.id == 1)).scalar_one()
session.execute(
    select(User).join(Order).where(Order.status == "paid")
).scalars().all()
session.execute(
    select(User).options(joinedload(User.orders))
).scalars().all()
```

---

## 7. Alembic 完全教程

### 7.1 安装与初始化

```bash
# 安装
pip install alembic

# 初始化（在项目根目录）
alembic init alembic  # 创建 alembic/ 目录和 alembic.ini
```text

生成的结构：

```

项目根目录/
├── alembic/                  # Alembic 迁移目录
│   ├── versions/             # 迁移版本文件
│   ├── env.py                # 环境配置（连接数据库）
│   ├── script.py.mako        # 迁移文件模板
│   └── README
├── alembic.ini               # Alembic 配置文件（数据库 URL）

```text

### 7.2 配置连接

**alembic.ini：**

```ini
# 修改数据库连接 URL
sqlalchemy.url = mysql+asyncmy://root:root123@localhost:3306/ticket_dev?charset=utf8mb4
```

**alembic/env.py：**

```python
# 关键修改：添加你的模型的 MetaData
from app.database import Base
from app.models import User, Event, Order, Ticket  # 导入所有模型

# 修改 target_metadata
target_metadata = Base.metadata  # 使用 ORM 模型的元数据

# 异步支持
from sqlalchemy.ext.asyncio import create_async_engine

def run_migrations_online():
    connectable = create_async_engine(config.get_main_option("sqlalchemy.url"))

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            compare_type=True,    # 比较类型变更
            compare_server_default=True,  # 比较默认值变更
        )

        with context.begin_transaction():
            context.run_migrations()
```text

### 7.3 创建迁移

```bash
# 自动检测模型变更，生成迁移文件
alembic revision --autogenerate -m "init_tables"

# 查看生成的迁移文件
# alembic/versions/xxx_init_tables.py
```

**自动生成的迁移文件示例：**

```python
"""init_tables

Revision ID: abc123def456
Revises:
Create Date: 2024-01-15 10:30:00.000000
"""
from typing import Sequence, Union
from alembic import op
import sqlalchemy as sa

# revision identifiers
revision: str = "abc123def456"
down_revision: Union[str, None] = None
branch_labels: Union[str, None] = None
depends_on: Union[str, None] = None


def upgrade() -> None:
    """升级：应用迁移"""
    op.create_table(
        "users",
        sa.Column("id", sa.BigInteger(), autoincrement=True, nullable=False),
        sa.Column("username", sa.String(50), nullable=False),
        sa.Column("email", sa.String(255), nullable=False),
        sa.Column("password_hash", sa.String(255), nullable=False),
        sa.Column("role", sa.String(20), nullable=False),
        sa.Column("is_active", sa.Boolean(), nullable=False),
        sa.Column("created_at", sa.DateTime(), nullable=False),
        sa.Column("updated_at", sa.DateTime(), nullable=False),
        sa.PrimaryKeyConstraint("id"),
        sa.UniqueConstraint("username"),
        sa.UniqueConstraint("email"),
    )
    op.create_index("idx_role", "users", ["role"])


def downgrade() -> None:
    """降级：回滚迁移"""
    op.drop_index("idx_role", table_name="users")
    op.drop_table("users")
```text

### 7.4 执行迁移

```bash
# 升级到最新版本
alembic upgrade head

# 升级指定版本
alembic upgrade abc123def456

# 降级（回滚）一个版本
alembic downgrade -1

# 降级到指定版本
alembic downgrade abc123def456

# 查看当前版本
alembic current

# 查看迁移历史
alembic history

# 查看待应用的迁移
alembic heads
```

### 7.5 手动迁移操作

自动迁移不能处理所有的修改（例如重命名列、更改列类型等）。以下是最常用的手动迁移操作：

```python
"""手动迁移示例

"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import mysql


def upgrade() -> None:
    # 添加列
    op.add_column("users", sa.Column("phone", sa.String(20), nullable=True))

    # 删除列
    op.drop_column("users", "avatar")

    # 修改列类型
    op.alter_column(
        "users",
        "username",
        type_=sa.String(100),  # 从 VARCHAR(50) 改为 VARCHAR(100)
        existing_type=sa.String(50),
    )

    # 重命名列
    op.alter_column("users", "username", new_column_name="user_name")

    # 添加唯一约束
    op.create_unique_constraint("uk_phone", "users", ["phone"])

    # 删除唯一约束
    op.drop_constraint("uk_phone", "users", type_="unique")

    # 添加外键
    op.create_foreign_key(
        "fk_orders_user",
        "orders", "users",
        ["user_id"], ["id"],
        ondelete="RESTRICT",
    )

    # 删除外键
    op.drop_constraint("fk_orders_user", "orders", type_="foreignkey")

    # 创建索引
    op.create_index("idx_email", "users", ["email"])

    # 删除索引
    op.drop_index("idx_email", table_name="users")

    # 重命名表
    op.rename_table("users", "members")

    # 执行原生 SQL
    op.execute("UPDATE users SET role = 'user' WHERE role IS NULL")

    # 创建新表
    op.create_table(
        "operation_logs",
        sa.Column("id", sa.BigInteger(), primary_key=True),
        sa.Column("user_id", sa.BigInteger(), nullable=False),
        sa.Column("action", sa.String(50), nullable=False),
        sa.Column("created_at", sa.DateTime(), nullable=False),
    )


def downgrade() -> None:
    # 反向操作（严格镜像）
    op.drop_table("operation_logs")
    op.rename_table("members", "users")
    op.drop_index("idx_email", table_name="users")
    op.drop_constraint("fk_orders_user", "orders", type_="foreignkey")
    op.drop_constraint("uk_phone", "users", type_="unique")
    op.alter_column("users", "user_name", new_column_name="username")
    op.add_column("users", sa.Column("avatar", sa.String(500), nullable=True))
    op.drop_column("users", "phone")
```text

### 7.6 迁移最佳实践

```python
# 1. 总是同时实现 upgrade() 和 downgrade()
# upgrade() 做修改，downgrade() 做反向修改

# 2. 数据迁移与 Schema 迁移分开
def upgrade() -> None:
    # Step 1: 添加新列
    op.add_column("users", sa.Column("phone", sa.String(20)))

    # Step 2: 将旧数据复制到新列
    op.execute("UPDATE users SET phone = contact_phone WHERE contact_phone IS NOT NULL")

    # Step 3: 删除旧列
    op.drop_column("users", "contact_phone")

# 3. 批量处理大表
def upgrade() -> None:
    # 分批次更新，避免锁表太久
    connection = op.get_bind()
    total = 0
    while True:
        result = connection.execute(
            sa.text("UPDATE orders SET status='converted' WHERE status='old' LIMIT 1000")
        )
        total += result.rowcount
        if result.rowcount == 0:
            break
    print(f"Updated {total} rows")

# 4. 每个迁移文件只做一件事
# 错误做法：一个迁移同时修改 users 和 orders
# 正确做法：users 迁移一个文件，orders 迁移另一个
```

---

## 8. 项目：ORM 模型定义 + 初始迁移 + 种子数据

### 8.1 项目目录结构

```text
app/
├── models/
│   ├── __init__.py          # 导出所有模型
│   ├── base.py              # Base 声明式基类
│   ├── user.py              # User 模型
│   ├── event.py             # Event 模型
│   ├── order.py             # Order 模型
│   └── ticket.py            # Ticket 模型
└── scripts/
    └── seed_data.py         # 种子数据脚本
```

### 8.2 创建模型文件

**app/models/base.py：**

```python
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy import MetaData

# MySQL 命名规范约定
convention = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}

# 声明式基类
class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=convention)
```text

**app/models/user.py：**

```python
from datetime import datetime
from typing import Optional, List
from sqlalchemy import String, Boolean, DateTime, Enum as SAEnum, func
from sqlalchemy.orm import Mapped, mapped_column, relationship

from .base import Base


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    username: Mapped[str] = mapped_column(String(50), unique=True)
    password_hash: Mapped[str] = mapped_column(String(255))
    email: Mapped[str] = mapped_column(String(255), unique=True)
    phone: Mapped[Optional[str]] = mapped_column(String(20), default=None)
    avatar: Mapped[Optional[str]] = mapped_column(String(500), default=None)
    role: Mapped[str] = mapped_column(String(20), default="user")
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    last_login_at: Mapped[Optional[datetime]] = mapped_column(DateTime(3), default=None)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(3), server_default=func.now(3)
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(3), server_default=func.now(3), onupdate=func.now(3)
    )

    # 关系
    orders: Mapped[List["Order"]] = relationship(
        back_populates="user",
        cascade="all, delete-orphan",
    )
    tickets: Mapped[List["Ticket"]] = relationship(back_populates="user")
    created_events: Mapped[List["Event"]] = relationship(
        back_populates="creator",
        foreign_keys="Event.created_by",
    )

    def __repr__(self) -> str:
        return f"<User {self.username}({self.role})>"
```

**app/models/event.py：**

```python
from datetime import datetime
from typing import Optional, List
from sqlalchemy import String, Text, DateTime, Integer, Numeric, BigInteger, ForeignKey, func
from sqlalchemy.orm import Mapped, mapped_column, relationship

from .base import Base


class Event(Base):
    __tablename__ = "events"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    title: Mapped[str] = mapped_column(String(200))
    description: Mapped[Optional[str]] = mapped_column(Text, default=None)
    venue: Mapped[str] = mapped_column(String(200))
    start_date: Mapped[datetime] = mapped_column(DateTime(3))
    end_date: Mapped[datetime] = mapped_column(DateTime(3))
    category: Mapped[str] = mapped_column(String(50), default="other")
    cover_image: Mapped[Optional[str]] = mapped_column(String(500), default=None)
    total_stock: Mapped[int] = mapped_column(Integer, default=0)
    sold_count: Mapped[int] = mapped_column(Integer, default=0)
    price: Mapped[float] = mapped_column(Numeric(10, 2))
    status: Mapped[str] = mapped_column(String(20), default="draft")
    created_by: Mapped[int] = mapped_column(BigInteger, ForeignKey("users.id"))
    created_at: Mapped[datetime] = mapped_column(
        DateTime(3), server_default=func.now(3)
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(3), server_default=func.now(3), onupdate=func.now(3)
    )

    # 关系
    creator: Mapped["User"] = relationship(
        back_populates="created_events",
        foreign_keys=[created_by],
    )
    orders: Mapped[List["Order"]] = relationship(back_populates="event")
    tickets: Mapped[List["Ticket"]] = relationship(back_populates="event")

    def __repr__(self) -> str:
        return f"<Event {self.title}>"
```text

**app/models/order.py：**

```python
from datetime import datetime
from typing import Optional, List
from sqlalchemy import String, Integer, Numeric, BigInteger, DateTime, ForeignKey, func
from sqlalchemy.orm import Mapped, mapped_column, relationship

from .base import Base


class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    order_no: Mapped[str] = mapped_column(String(32), unique=True)
    user_id: Mapped[int] = mapped_column(BigInteger, ForeignKey("users.id"))
    event_id: Mapped[int] = mapped_column(BigInteger, ForeignKey("events.id"))
    quantity: Mapped[int] = mapped_column(Integer)
    total_amount: Mapped[float] = mapped_column(Numeric(10, 2))
    status: Mapped[str] = mapped_column(String(20), default="pending_payment")
    payment_method: Mapped[Optional[str]] = mapped_column(String(20), default=None)
    trade_no: Mapped[Optional[str]] = mapped_column(String(64), default=None)
    paid_at: Mapped[Optional[datetime]] = mapped_column(DateTime(3), default=None)
    cancelled_at: Mapped[Optional[datetime]] = mapped_column(DateTime(3), default=None)
    remark: Mapped[Optional[str]] = mapped_column(String(500), default=None)
    version: Mapped[int] = mapped_column(Integer, default=1)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(3), server_default=func.now(3)
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(3), server_default=func.now(3), onupdate=func.now(3)
    )

    # 关系
    user: Mapped["User"] = relationship(back_populates="orders")
    event: Mapped["Event"] = relationship(back_populates="orders")
    tickets: Mapped[List["Ticket"]] = relationship(
        back_populates="order",
        cascade="all, delete-orphan",
    )

    def __repr__(self) -> str:
        return f"<Order {self.order_no} ({self.status})>"
```

**app/models/ticket.py：**

```python
from datetime import datetime
from typing import Optional
from sqlalchemy import String, Boolean, Numeric, BigInteger, DateTime, ForeignKey, func
from sqlalchemy.orm import Mapped, mapped_column, relationship

from .base import Base


class Ticket(Base):
    __tablename__ = "tickets"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    order_id: Mapped[int] = mapped_column(BigInteger, ForeignKey("orders.id"))
    event_id: Mapped[int] = mapped_column(BigInteger, ForeignKey("events.id"))
    user_id: Mapped[int] = mapped_column(BigInteger, ForeignKey("users.id"))
    ticket_no: Mapped[str] = mapped_column(String(32), unique=True)
    seat_info: Mapped[str] = mapped_column(String(50), default="stand")
    price: Mapped[float] = mapped_column(Numeric(10, 2))
    qr_code: Mapped[Optional[str]] = mapped_column(String(255), default=None)
    used: Mapped[bool] = mapped_column(Boolean, default=False)
    used_at: Mapped[Optional[datetime]] = mapped_column(DateTime(3), default=None)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(3), server_default=func.now(3)
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(3), server_default=func.now(3), onupdate=func.now(3)
    )

    # 关系
    order: Mapped["Order"] = relationship(back_populates="tickets")
    event: Mapped["Event"] = relationship(back_populates="tickets")
    user: Mapped["User"] = relationship(back_populates="tickets")

    def __repr__(self) -> str:
        return f"<Ticket {self.ticket_no}>"
```text

**app/models/**init**.py：**

```python
from .base import Base
from .user import User
from .event import Event
from .order import Order
from .ticket import Ticket

__all__ = ["Base", "User", "Event", "Order", "Ticket"]
```

### 8.3 配置 Alembic

```bash
# 在项目根目录初始化
cd /Users/freesling/Develop/hujiu
alembic init alembic
```text

修改 `alembic/env.py`：

```python
import sys
from pathlib import Path

# 添加项目根目录到 Python 路径
sys.path.append(str(Path(__file__).parent.parent))

from app.models import Base
from app.config import settings
from sqlalchemy.ext.asyncio import create_async_engine

# 使用同步 URL（Alembic 的连接不支持异步驱动）
from alembic import context
from sqlalchemy import create_engine

config = context.config

# 设置数据库 URL
config.set_main_option("sqlalchemy.url", settings.DATABASE_URL.replace("+asyncmy", "+pymysql"))

target_metadata = Base.metadata


def run_migrations_online() -> None:
    """在线模式运行迁移"""
    connectable = create_engine(config.get_main_option("sqlalchemy.url"))

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            compare_type=True,
        )
        with context.begin_transaction():
            context.run_migrations()
```

### 8.4 生成并执行初始迁移

```bash
# 生成初始迁移
alembic revision --autogenerate -m "init_tables"

# 查看生成的迁移文件
cat alembic/versions/xxx_init_tables.py

# 执行迁移（创建所有表）
alembic upgrade head

# 验证表已创建
docker exec -it ticket-mysql mysql -uroot -proot123 ticket_dev -e "SHOW TABLES;"
```text

### 8.5 种子数据脚本

**app/scripts/seed_data.py：**

```python
"""
种子数据脚本：插入初始用户和活动数据
用于开发和测试环境。

用法：
    python -m app.scripts.seed_data
"""

import asyncio
from datetime import datetime, timedelta
from sqlalchemy import select

from app.database import async_session_factory
from app.models import User, Event, Order, Ticket
from app.utils.security import get_password_hash  # 第7章会实现


async def seed_users():
    """插入测试用户"""
    async with async_session_factory() as session:
        # 检查是否已有数据
        result = await session.execute(select(User).limit(1))
        if result.scalar_one_or_none():
            print("用户数据已存在，跳过")
            return

        users = [
            User(
                username="admin",
                password_hash=get_password_hash("admin123"),
                email="admin@ticket.com",
                role="admin",
                is_active=True,
            ),
            User(
                username="zhangsan",
                password_hash=get_password_hash("123456"),
                email="zhangsan@example.com",
                role="user",
                is_active=True,
            ),
            User(
                username="lisi",
                password_hash=get_password_hash("123456"),
                email="lisi@example.com",
                role="user",
                is_active=True,
            ),
        ]
        session.add_all(users)
        await session.commit()
        print(f"✅ 已创建 {len(users)} 个用户")


async def seed_events():
    """插入测试活动"""
    async with async_session_factory() as session:
        result = await session.execute(select(Event).limit(1))
        if result.scalar_one_or_none():
            print("活动数据已存在，跳过")
            return

        # 获取管理员用户
        result = await session.execute(
            select(User).where(User.role == "admin")
        )
        admin = result.scalar_one()

        events = [
            Event(
                title="周杰伦2024巡回演唱会 - 北京站",
                description="地表最强，周杰伦2024年巡回演唱会北京站。",
                venue="国家体育场（鸟巢）",
                start_date=datetime(2024, 8, 15, 19, 30),
                end_date=datetime(2024, 8, 15, 23, 0),
                category="concert",
                total_stock=100,
                price=680.00,
                status="published",
                created_by=admin.id,
            ),
            Event(
                title="2024中超联赛：北京国安 vs 上海申花",
                description="中超联赛焦点战，北京国安主场迎战上海申花。",
                venue="工人体育场",
                start_date=datetime(2024, 9, 1, 19, 35),
                end_date=datetime(2024, 9, 1, 21, 30),
                category="sports",
                total_stock=200,
                price=280.00,
                status="published",
                created_by=admin.id,
            ),
            Event(
                title="话剧《茶馆》",
                description="老舍经典作品，人艺倾情演绎。",
                venue="首都剧场",
                start_date=datetime(2024, 10, 1, 19, 30),
                end_date=datetime(2024, 10, 1, 22, 0),
                category="theater",
                total_stock=50,
                price=480.00,
                status="draft",
                created_by=admin.id,
            ),
        ]
        session.add_all(events)
        await session.commit()
        print(f"✅ 已创建 {len(events)} 个活动")


async def main():
    print("🚀 开始插入种子数据...")
    await seed_users()
    await seed_events()
    print("🎉 种子数据插入完成！")


if __name__ == "__main__":
    asyncio.run(main())
```

### 8.6 运行种子数据

```bash
# 确保 Docker MySQL 正在运行
docker start ticket-mysql

# 执行迁移
alembic upgrade head

# 插入种子数据
python -m app.scripts.seed_data
```text

验证种子数据：

```bash
docker exec -it ticket-mysql mysql -uroot -proot123 ticket_dev -e "
    SELECT 'users:' as ''; SELECT id, username, role FROM users;
    SELECT 'events:' as ''; SELECT id, title, status, price FROM events;
"
```

---

## 9. 常见问题与排错

| 问题 | 原因 | 解决 |
| ------ | ------ | ------ |
| `ModuleNotFoundError: No module named 'app'` | Python 路径问题 | 在项目根目录运行，或用 `PYTHONPATH=. python -m ...` |
| `sqlalchemy.exc.InterfaceError: (asyncmy) ...` | 数据库未连接 | 检查 Docker MySQL 是否运行：`docker ps` |
| `sqlalchemy.exc.NoReferencedTableError` | 外键引用的表未创建 | 确保导入所有模型，迁移顺序正确 |
| `Alembic: Target database is not up to date` | 数据库版本落后 | `alembic upgrade head` |
| `DetachedInstanceError` | 会话关闭后访问对象 | 设置 `expire_on_commit=False` |
| `MissingGreenlet: greenlet_spawn has not been called` | 同步代码中使用了异步会话 | 确保在 async 函数中操作 |
| 自动迁移检测不到表名变更 | Alembic 无法检测重命名 | 手动编写 migration |
| `Multiple rows returned for scalar_one()` | 查到了多行但用了 scalar_one | 用 `scalar_one_or_none()` 或 `first()` |

---

---

## 📁 项目代码参考

| 文件 | 说明 |
| ------ | ------ |
| `app/models/base.py` | ORM 声明基类（`DeclarativeBase`） |
| `app/models/user.py` | User 模型：`unique` 约束、`is_banned_or_inactive` 属性 |
| `app/models/event.py` | Event 模型：`version` 乐观锁字段、`sold_count` |
| `app/models/order.py` | Order 模型：`status` 枚举、外键关联、时间戳 |
| `alembic/versions/` | 数据库迁移版本文件（可用 `alembic history` 查看） |
| `alembic.ini` | Alembic 配置 |
| `app/database.py` | `async_engine` / `AsyncSession` 工厂 |

**迁移工作流示例：**

```bash
alembic revision --autogenerate -m "add login_history table"
alembic upgrade head
```text

## 10. 本章总结

✅ **已掌握：**

- SQLAlchemy Core 层：Engine/ConnectionPool、Table/MetaData、表达式语言
- Declarative Models：Mapped/mapped_column 2.0 语法、全部列类型映射
- Relationships：一对多/多对一/一对一/多对多、lazy 加载、cascade
- N+1 查询问题的检测和解决（selectinload/joinedload）
- Async Sessions：安全获取、增删改查、事务管理
- select() 2.0 完整查询构建器：JOIN、聚合、子查询、UNION、FOR UPDATE
- Alembic：初始化、自动迁移、手动迁移、执行管理
- 项目 ORM 模型完整定义和种子数据脚本

✅ **项目里程碑：**

- [ ] User/Event/Order/Ticket ORM 模型定义完成
- [ ] 关系映射（外键 + relationship）配置正确
- [ ] Alembic 初始迁移成功
- [ ] 所有表在 MySQL 中创建
- [ ] 种子数据插入成功

**练习：**

1. 完成所有模型文件的创建
2. 使用 Alembic 生成并执行初始迁移
3. 运行种子数据脚本
4. 编写查询：查某个用户的所有已支付订单
5. 编写查询：查某个活动的可售票数量

**下一章预告：** 第4章将学习 FastAPI 和 Pydantic v2，构建 Web 框架基础，包括路径操作、请求验证、依赖注入、中间件等核心内容。
