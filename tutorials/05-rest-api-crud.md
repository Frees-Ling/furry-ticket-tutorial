# 第5章：REST API 设计 + 活动管理 CRUD

## 本章学习目标

本章将学习 REST API 设计原理和最佳实践，实现活动管理的完整 CRUD 功能，掌握 Service 层和 Repository 模式。

**学习路径：**

1. ✅ REST 六大约束与 HTTP 方法幂等性证明
2. ✅ HTTP 状态码完整指南与 RFC 7807 Problem Details
3. ✅ 分页/排序/过滤算法与 API 版本化策略
4. ✅ Service 层与 Repository 设计模式
5. ✅ 项目：活动 CRUD 完整实现

---

## 1. REST 架构风格

### 1.1 什么是 REST？

REST（Representational State Transfer，表述性状态转移）是 Roy Fielding 在 2000 年博士论文中提出的一种**架构风格**，不是标准或协议。

**核心思想：** 将 Web 应用视为资源的集合，客户端通过对资源的表述（Representation）进行状态转移（State Transfer）。

### 1.2 REST 六大约束

| 约束 | 含义 | 违反后果 |
| ------ | ------ | --------- |
| **1. 客户端-服务器** | 分离关注点：UI 与数据存储分离 | 耦合，无法独立演进 |
| **2. 无状态** | 每个请求包含全部信息，服务器不存会话状态 | 无法水平扩展 |
| **3. 可缓存** | 响应应明确标即可缓存/不可缓存 | 性能浪费 |
| **4. 统一接口** | 所有资源使用统一的接口（HTTP 方法） | API 混乱 |
| **5. 分层系统** | 中间组件（代理、网关、负载均衡）透明 | 无法伸缩 |
| **6. 按需代码（可选）** | 服务器可向客户端传输可执行代码 | 不常见（如 JS） |

**无状态约束的数学证明：**

```text
假设服务器在请求之间维护会话状态 S。

服务器有 N 个实例，每个实例有 S₁, S₂, ..., Sₙ 不同的会话状态。

客户端请求可能被路由到 Sᵢ 或 Sⱼ（负载均衡），如果请求包含会话标识符，
但会话数据在另一个实例上 → 请求失败。

解决方案：要么维护共享会话存储（引入单点故障），
要么每个请求携带全部状态（REST 方式）。

REST 的无状态：每个请求包含所有必要信息（认证头、参数、数据）。
服务器不需要跨请求记忆任何状态。
→ 任何请求可以发往任何服务器实例 → 水平扩展成为可能。
```

### 1.3 RESTful API 设计规范

```python
# ========== 资源 URI 设计 ==========
# 资源用名词复数
GET    /events                    # 活动列表
POST   /events                    # 创建活动
GET    /events/{id}               # 获取单个活动
PUT    /events/{id}               # 更新活动（全量）
PATCH  /events/{id}               # 更新活动（部分）
DELETE /events/{id}               # 删除活动

# 子资源
GET    /events/{id}/tickets       # 活动的票列表
GET    /users/{id}/orders         # 用户的订单列表

# 动作通过 POST + 子资源表示（不是 RPC 风格的 /doSomething）
POST   /orders/{id}/cancel        # 取消订单（不是 GET /cancelOrder）
POST   /orders/{id}/pay           # 支付订单（不是 GET /payOrder）

# ========== 反模式（不要这样做）==========
GET    /getEventList              # ❌ 动词在 URI 中
POST   /createEvent               # ❌ 动作在 URI 中
GET    /events/list               # ❌ 不必要的嵌套
POST   /events/{id}/update        # ❌ 更新用 PUT/PATCH
DELETE /events/{id}/delete        # ❌ 删除用 DELETE
```text

### 1.4 HTTP 方法幂等性证明

**幂等性定义：** 一个操作执行一次和执行多次的效果相同（服务器状态不变）。

| 方法 | 幂等 | 安全（不修改资源） | 说明 |
| ------ | ------ | ------------------- | ------ |
| GET | ✅ | ✅ | 读取资源 |
| HEAD | ✅ | ✅ | 读取响应头 |
| OPTIONS | ✅ | ✅ | 查询支持的方法 |
| PUT | ✅ | ❌ | 全量替换（多次 PUT 结果相同） |
| DELETE | ✅ | ❌ | 删除（删除不存在的资源也返回相同结果） |
| POST | ❌ | ❌ | 创建新资源（每次创建不同） |
| PATCH | ❌ | ❌ | 部分更新（不保证幂等） |

**PUT 幂等性证明：**

```

PUT /events/1 HTTP/1.1
{"title": "演唱会", "price": 680}

第一次执行：
服务器状态：events(1) = {"title": "演唱会", "price": 680}

第二次执行（相同请求体）：
服务器状态：events(1) = {"title": "演唱会", "price": 680}（不变）

第 N 次执行：
服务器状态：events(1) = {"title": "演唱会", "price": 680}（不变）

结论：PUT 是幂等的。

```text

**POST 非幂等性证明：**

```

POST /events
{"title": "新活动"}

第一次执行：
服务器创建 events(1) = {"title": "新活动", "id": 1}

第二次执行（相同请求体）：
服务器创建 events(2) = {"title": "新活动", "id": 2}（不同资源！）

结论：POST 不是幂等的（每次创建新资源）。

```text

**PATCH 的非幂等性：**

```python
# PATCH 可能非幂等（取决于更新操作类型）

# 幂等的 PATCH（设置值）：
PATCH /events/1
{"status": "published"}
# 多次执行结果相同

# 非幂等的 PATCH（自增）：
PATCH /events/1
{"sold_count": "+1"}
# 每次执行 sold_count 都增加

# 解决方案：PATCH 使用绝对赋值（非相对操作）
```

---

## 2. HTTP 状态码完整指南

### 2.1 状态码分类

| 分类 | 范围 | 含义 |
| ------ | ------ | ------ |
| 1xx | 100-199 | 信息性响应 |
| 2xx | 200-299 | 成功 |
| 3xx | 300-399 | 重定向 |
| 4xx | 400-499 | 客户端错误 |
| 5xx | 500-599 | 服务器错误 |

### 2.2 REST API 常用状态码

```python
# ========== 2xx 成功 ==========
200 OK          # GET 成功、PUT 成功、PATCH 成功
201 Created     # POST 创建成功
202 Accepted    # 请求已接受但未完成（异步任务）
204 No Content  # DELETE 成功（无响应体）

# ========== 3xx 重定向 ==========
301 Moved Permanently   # 资源永久移动
302 Found               # 临时重定向
304 Not Modified        # 资源未修改（条件请求）

# ========== 4xx 客户端错误 ==========
400 Bad Request         # 请求格式错误
401 Unauthorized        # 未认证（没登录）
403 Forbidden           # 已认证但无权限
404 Not Found           # 资源不存在
405 Method Not Allowed  # HTTP 方法不支持
409 Conflict            # 资源冲突（如重复创建）
422 Unprocessable Entity # 请求语义错误（验证失败）
429 Too Many Requests   # 请求频率过高（限流）

# ========== 5xx 服务器错误 ==========
500 Internal Server Error  # 服务器内部错误（未知）
502 Bad Gateway            # 上游服务不可达
503 Service Unavailable    # 服务暂时不可用（过载/维护）
504 Gateway Timeout        # 上游服务超时
```text

**示例：状态码使用**

```python
from fastapi import FastAPI, HTTPException, status

app = FastAPI()

@app.post("/events", status_code=status.HTTP_201_CREATED)
async def create_event():
    """创建活动 → 201 Created"""
    return {"id": 1, "title": "新活动"}

@app.delete("/events/{id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_event(id: int):
    """删除活动 → 204 No Content（无响应体）"""
    pass  # 正常返回 None，FastAPI 自动转为 204

@app.get("/events/{id}")
async def get_event(id: int):
    if id <= 0:
        raise HTTPException(status_code=400, detail="ID 必须大于 0")
    event = find_event(id)
    if event is None:
        raise HTTPException(status_code=404, detail="活动不存在")
    return event

@app.post("/orders")
async def create_order(data: dict):
    if data["quantity"] > data["available"]:
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail="库存不足",
        )
    # ...
```

### 2.3 RFC 7807 Problem Details

RFC 7807 定义了标准化的错误响应格式：

```json
{
    "type": "https://example.com/errors/insufficient-stock",
    "title": "库存不足",
    "status": 409,
    "detail": "请求数量 10 超过可用库存 5",
    "instance": "/api/v1/orders",
    "available": 5,
    "requested": 10
}
```text

**在 FastAPI 中实现：**

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

class ProblemDetail(Exception):
    """RFC 7807 Problem Details 异常"""
    def __init__(
        self,
        title: str,
        detail: str,
        status: int = 400,
        type_: str = "about:blank",
        instance: str | None = None,
        **extra,
    ):
        self.type = type_
        self.title = title
        self.status = status
        self.detail = detail
        self.instance = instance
        self.extra = extra

@app.exception_handler(ProblemDetail)
async def problem_detail_handler(request: Request, exc: ProblemDetail):
    return JSONResponse(
        status_code=exc.status,
        content={
            "type": exc.type,
            "title": exc.title,
            "status": exc.status,
            "detail": exc.detail,
            "instance": str(request.url.path),
            **exc.extra,
        },
        headers={"Content-Type": "application/problem+json"},
    )

# 使用
@app.get("/orders/{id}")
async def get_order(id: int):
    raise ProblemDetail(
        title="订单不存在",
        detail=f"订单 {id} 未找到",
        status=404,
    )
```

---

## 3. 分页/排序/过滤

### 3.1 分页算法

#### 3.1.1 Offset/Limit 分页（传统）

```python
@app.get("/events")
async def list_events(
    page: int = Query(1, ge=1),
    size: int = Query(20, ge=1, le=100),
):
    offset = (page - 1) * size
    stmt = select(Event).offset(offset).limit(size)
    result = await session.execute(stmt)
    events = result.scalars().all()

    # 获取总数
    count_stmt = select(func.count(Event.id))
    total = await session.scalar(count_stmt)

    return {
        "items": events,
        "total": total,
        "page": page,
        "size": size,
        "total_pages": (total + size - 1) // size,  # 向上取整
    }
```text

**Offset 分页的性能问题数学证明：**

```

SELECT * FROM events ORDER BY id LIMIT 20 OFFSET 100000;

数据库执行步骤：

1. 扫描到第 100020 行（从索引开始）
2. 丢弃前 100000 行
3. 返回最后 20 行

扫描行数 = OFFSET + LIMIT，即使只需要 20 行！

假设 offset=100000, limit=20：

- 索引扫描 100020 次
- 回表 100020 次（如果不是覆盖索引）
- 实际返回 20 行

时间复杂度：O(offset + limit)

```text

#### 3.1.2 游标分页（Keyset Pagination / Seek Method）

```python
@app.get("/events")
async def list_events(
    cursor: int | None = Query(None, description="上一页最后一条的 id"),
    size: int = Query(20, ge=1, le=100),
):
    if cursor:
        stmt = select(Event).where(Event.id > cursor).order_by(Event.id).limit(size)
    else:
        stmt = select(Event).order_by(Event.id).limit(size)

    result = await session.execute(stmt)
    events = result.scalars().all()

    next_cursor = events[-1].id if len(events) == size else None

    return {
        "items": events,
        "next_cursor": next_cursor,
        "has_more": len(events) == size,
    }
```

**游标分页的效率证明：**

```text
SELECT * FROM events WHERE id > 100000 ORDER BY id LIMIT 20;

数据库执行步骤：
1. 从索引中找到 id=100000 的位置
2. 继续扫描 20 行
3. 返回 20 行

扫描行数 = LIMIT，和 OFFSET 无关！

时间复杂度：O(limit)  与总数据量无关！

B+Tree 定位 id=100000 的开销：
- 3 层 B+Tree → 3 次索引查找
- 然后只需要扫描 20 个叶子节点
- 总操作 ≈ 常数级
```

**游标分页的并发安全证明：**

```text
Offset 分页的并发问题：
用户 A 浏览第 1 页时，新活动插入 → 第 2 页出现重复数据

游标分页的并发安全：
WHERE id > last_seen_id → 按 id 递增，新插入的数据 id 更大
→ 不会影响当前页的查询结果
→ 新增数据出现在下一页，不会出现重复
→ 删除数据导致 id 断层，但不影响连续性

结论：游标分页在并发写入下是安全的（无重复、无遗漏）。
```

**两种分页的对比：**

| 特性 | Offset/Limit | 游标分页 |
| ------ | ------------- | --------- |
| 实现复杂度 | 简单 | 中等 |
| 大偏移量性能 | ❌ 极差 | ✅ 优秀 |
| 随机跳页 | ✅ 支持 | ❌ 不支持 |
| 并发安全 | ❌ 有重复风险 | ✅ 安全 |
| 适用场景 | 管理后台、小数据量 | API、大数据量 |

**混合策略：** 前台 API 用游标分页，管理后台用 Offset 分页。

### 3.2 排序

```python
@app.get("/events")
async def list_events(
    sort_by: str = Query("created_at", pattern=r"^(created_at|price|start_date)$"),
    sort_order: str = Query("desc", pattern=r"^(asc|desc)$"),
    cursor: int | None = None,
    size: int = Query(20, ge=1, le=100),
):
    # 构建 ORDER BY 子句
    order_column = getattr(Event, sort_by)
    order_expr = order_column.desc() if sort_order == "desc" else order_column.asc()

    stmt = select(Event).order_by(order_expr).limit(size)
    if cursor:
        if sort_order == "desc":
            stmt = stmt.where(order_column < cursor)
        else:
            stmt = stmt.where(order_column > cursor)

    result = await session.execute(stmt)
    events = result.scalars().all()

    return {"items": events, "next_cursor": getattr(events[-1], sort_by) if events else None}
```text

### 3.3 过滤

```python
from sqlalchemy import and_, or_

@app.get("/events")
async def list_events(
    # 精确过滤
    category: str | None = None,
    status: str | None = None,

    # 范围过滤
    min_price: float | None = None,
    max_price: float | None = None,

    # 文本搜索
    keyword: str | None = None,

    # 日期范围
    start_date_from: datetime | None = None,
    start_date_to: datetime | None = None,

    # 分页
    page: int = Query(1, ge=1),
    size: int = Query(20, ge=1, le=100),
):
    # 构建 WHERE 条件
    conditions = []

    if category:
        conditions.append(Event.category == category)
    if status:
        conditions.append(Event.status == status)
    if min_price is not None:
        conditions.append(Event.price >= min_price)
    if max_price is not None:
        conditions.append(Event.price <= max_price)
    if keyword:
        conditions.append(Event.title.like(f"%{keyword}%"))
    if start_date_from:
        conditions.append(Event.start_date >= start_date_from)
    if start_date_to:
        conditions.append(Event.start_date <= start_date_to)

    # 应用条件
    stmt = select(Event)
    if conditions:
        stmt = stmt.where(and_(*conditions))

    # 排序 + 分页
    stmt = stmt.order_by(Event.created_at.desc())
    stmt = stmt.offset((page - 1) * size).limit(size)

    result = await session.execute(stmt)
    events = result.scalars().all()

    return {"items": events, "page": page, "size": size}
```

### 3.4 API 版本化策略

```python
# 策略一：URL 路径版本化（最常用）
# /api/v1/events
# /api/v2/events

from fastapi import APIRouter, FastAPI

app = FastAPI()
v1_router = APIRouter(prefix="/api/v1")
v2_router = APIRouter(prefix="/api/v2")

app.include_router(v1_router)
app.include_router(v2_router)

# 策略二：请求头版本化
# Accept: application/vnd.ticket.v1+json

from fastapi import Header

@app.get("/events")
async def list_events(
    accept: str = Header("application/vnd.ticket.v1+json"),
):
    version = accept.split(".")[-2] if "vnd.ticket" in accept else "v1"
    # 根据版本返回不同格式

# 策略三：查询参数版本化（不推荐）
# /events?version=1
```text

---

## 4. 依赖注入模式

### 4.1 Service 层

Service 层封装业务逻辑，位于路由处理器和数据库之间。

```

┌─────────┐     ┌───────────┐     ┌───────────┐     ┌─────────┐
│ 路由    │ →   │ Pydantic  │ →   │ Service   │ →   │ Model   │
│ (request)│    │ Schema    │     │ (业务逻辑) │     │ (ORM)   │
└─────────┘     └───────────┘     └───────────┘     └─────────┘

```text

**app/services/**init**.py：**

```python
```

**app/services/base.py：**

```python
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, func
from typing import TypeVar, Generic, Type, List, Optional

from app.models import Base

ModelType = TypeVar("ModelType", bound=Base)


class BaseService(Generic[ModelType]):
    """服务基类：提供通用的 CRUD 操作"""

    def __init__(self, session: AsyncSession, model: Type[ModelType]):
        self.session = session
        self.model = model

    async def get_by_id(self, id: int) -> Optional[ModelType]:
        """根据 ID 获取记录"""
        return await self.session.get(self.model, id)

    async def list(
        self,
        page: int = 1,
        size: int = 20,
        order_by: str = "created_at",
        order_desc: bool = True,
    ) -> tuple[List[ModelType], int]:
        """分页查询"""
        # 排序
        order_col = getattr(self.model, order_by)
        order_expr = order_col.desc() if order_desc else order_col.asc()

        # 查询
        stmt = (
            select(self.model)
            .order_by(order_expr)
            .offset((page - 1) * size)
            .limit(size)
        )
        result = await self.session.execute(stmt)
        items = list(result.scalars().all())

        # 总数
        count_stmt = select(func.count()).select_from(self.model)
        total = await self.session.scalar(count_stmt) or 0

        return items, total
```text

**app/services/event_service.py：**

```python
from typing import List, Optional
from sqlalchemy import select, func, and_
from sqlalchemy.ext.asyncio import AsyncSession

from app.models import Event
from app.schemas.event import EventCreate, EventUpdate


class EventService:
    """活动服务：封装活动相关的业务逻辑"""

    def __init__(self, session: AsyncSession):
        self.session = session

    async def list_events(
        self,
        page: int = 1,
        size: int = 20,
        category: Optional[str] = None,
        status: str = "published",
        keyword: Optional[str] = None,
        min_price: Optional[float] = None,
        max_price: Optional[float] = None,
    ) -> tuple[List[Event], int]:
        """获取活动列表（支持分页、过滤、搜索）"""
        conditions = [Event.status == status]

        if category:
            conditions.append(Event.category == category)
        if keyword:
            conditions.append(Event.title.like(f"%{keyword}%"))
        if min_price is not None:
            conditions.append(Event.price >= min_price)
        if max_price is not None:
            conditions.append(Event.price <= max_price)

        # 总数
        count_stmt = select(func.count(Event.id)).where(and_(*conditions))
        total = await self.session.scalar(count_stmt) or 0

        # 分页查询
        stmt = (
            select(Event)
            .where(and_(*conditions))
            .order_by(Event.start_date.asc())
            .offset((page - 1) * size)
            .limit(size)
        )
        result = await self.session.execute(stmt)
        events = list(result.scalars().all())

        return events, total

    async def get_event(self, event_id: int) -> Optional[Event]:
        """获取活动详情"""
        return await self.session.get(Event, event_id)

    async def create_event(self, data: EventCreate, user_id: int) -> Event:
        """创建活动"""
        event = Event(
            title=data.title,
            description=data.description,
            venue=data.venue,
            start_date=data.start_date,
            end_date=data.end_date,
            category=data.category,
            price=data.price,
            total_stock=data.total_stock,
            status="draft",
            created_by=user_id,
        )
        self.session.add(event)
        await self.session.flush()
        return event

    async def update_event(self, event_id: int, data: EventUpdate) -> Optional[Event]:
        """更新活动"""
        event = await self.session.get(Event, event_id)
        if not event:
            return None

        update_data = data.model_dump(exclude_unset=True)
        for field, value in update_data.items():
            setattr(event, field, value)

        await self.session.flush()
        return event

    async def delete_event(self, event_id: int) -> bool:
        """删除活动"""
        event = await self.session.get(Event, event_id)
        if not event:
            return False

        await self.session.delete(event)
        await self.session.flush()
        return True
```

### 4.2 Repository 模式（进阶）

Repository 模式在 Service 和 Model 之间增加一层抽象，适合复杂查询：

```python
from abc import ABC, abstractmethod
from typing import Generic, TypeVar, Type, List, Optional, Any
from sqlalchemy import select, func, and_, Select
from sqlalchemy.ext.asyncio import AsyncSession

from app.models import Base

ModelType = TypeVar("ModelType", bound=Base)


class BaseRepository(ABC, Generic[ModelType]):
    """仓储基类（抽象接口）"""

    @abstractmethod
    async def get(self, id: Any) -> Optional[ModelType]:
        ...

    @abstractmethod
    async def list(self, page: int, size: int) -> tuple[List[ModelType], int]:
        ...

    @abstractmethod
    async def add(self, instance: ModelType) -> ModelType:
        ...

    @abstractmethod
    async def update(self, instance: ModelType) -> ModelType:
        ...

    @abstractmethod
    async def delete(self, instance: ModelType) -> None:
        ...


class SQLAlchemyRepository(BaseRepository[ModelType]):
    """SQLAlchemy 仓储实现"""

    def __init__(self, session: AsyncSession, model: Type[ModelType]):
        self.session = session
        self.model = model

    async def get(self, id: Any) -> Optional[ModelType]:
        return await self.session.get(self.model, id)

    async def list(self, page: int = 1, size: int = 20, **filters) -> tuple[List[ModelType], int]:
        stmt = select(self.model)
        if filters:
            conditions = [getattr(self.model, k) == v for k, v in filters.items()]
            stmt = stmt.where(and_(*conditions))

        count_stmt = select(func.count()).select_from(self.model)
        if filters:
            count_stmt = count_stmt.where(and_(*conditions))
        total = await self.session.scalar(count_stmt) or 0

        stmt = stmt.offset((page - 1) * size).limit(size)
        result = await self.session.execute(stmt)
        items = list(result.scalars().all())

        return items, total

    async def add(self, instance: ModelType) -> ModelType:
        self.session.add(instance)
        await self.session.flush()
        return instance

    async def update(self, instance: ModelType) -> ModelType:
        await self.session.merge(instance)
        await self.session.flush()
        return instance

    async def delete(self, instance: ModelType) -> None:
        await self.session.delete(instance)
        await self.session.flush()
```text

### 4.3 Service 与 Repository 的对比

| 方面 | Service 层 | Repository 层 |
| ------ | ----------- | --------------- |
| 职责 | 业务逻辑（验证、权限、事务） | 数据访问（查询、持久化） |
| 抽象程度 | 面向业务 | 面向数据 |
| 复杂度 | 复杂，包含业务规则 | 简单，CRUD 操作 |
| 测试 | 需要 mock Repository | 需要 mock Database |
| 复用性 | 跨模块复用 | 跨数据源复用 |

**小型项目**可以直接使用 Service 层，
**大型项目**可以 Service + Repository 两层都用。

---

## 5. 项目：活动管理 CRUD 完整实现

### 5.1 Pydantic Schema

**app/schemas/event.py：**

```python
from datetime import datetime
from typing import Optional, List
from pydantic import BaseModel, Field, field_validator


class EventCreate(BaseModel):
    """创建活动请求"""
    title: str = Field(..., min_length=1, max_length=200, description="活动标题")
    description: Optional[str] = Field(None, max_length=2000, description="活动描述")
    venue: str = Field(..., min_length=1, max_length=200, description="场馆地址")
    start_date: datetime = Field(..., description="开始时间")
    end_date: datetime = Field(..., description="结束时间")
    category: str = Field("other", description="活动分类")
    price: float = Field(..., gt=0, le=99999.99, description="票价")
    total_stock: int = Field(..., ge=1, le=100000, description="总库存")

    @field_validator("end_date")
    @classmethod
    def check_dates(cls, v, info):
        """验证结束时间大于开始时间"""
        if "start_date" in info.data and v <= info.data["start_date"]:
            raise ValueError("结束时间必须大于开始时间")
        return v

    @field_validator("category")
    @classmethod
    def check_category(cls, v):
        allowed = ["concert", "sports", "theater", "other"]
        if v not in allowed:
            raise ValueError(f"分类必须是 {allowed} 之一")
        return v


class EventUpdate(BaseModel):
    """更新活动请求（全部可选）"""
    title: Optional[str] = Field(None, min_length=1, max_length=200)
    description: Optional[str] = Field(None, max_length=2000)
    venue: Optional[str] = Field(None, min_length=1, max_length=200)
    start_date: Optional[datetime] = None
    end_date: Optional[datetime] = None
    category: Optional[str] = None
    price: Optional[float] = Field(None, gt=0, le=99999.99)
    total_stock: Optional[int] = Field(None, ge=1, le=100000)

    @field_validator("end_date")
    @classmethod
    def check_dates(cls, v, info):
        if v is not None and "start_date" in info.data and info.data["start_date"] is not None:
            if v <= info.data["start_date"]:
                raise ValueError("结束时间必须大于开始时间")
        return v


class EventResponse(BaseModel):
    """活动响应"""
    model_config = {"from_attributes": True}

    id: int
    title: str
    description: Optional[str] = None
    venue: str
    start_date: datetime
    end_date: datetime
    category: str
    price: float
    total_stock: int
    sold_count: int
    status: str
    created_by: int
    created_at: datetime
    updated_at: datetime

    # 计算字段
    @property
    def available_stock(self) -> int:
        return self.total_stock - self.sold_count


class EventListResponse(BaseModel):
    """活动列表响应"""
    items: List[EventResponse]
    total: int
    page: int = 1
    size: int = 20
    total_pages: int = 0
```

**app/schemas/**init**.py：**

```python
from .event import EventCreate, EventUpdate, EventResponse, EventListResponse

__all__ = [
    "EventCreate",
    "EventUpdate",
    "EventResponse",
    "EventListResponse",
]
```text

### 5.2 完整路由

**app/routers/events.py（完整版）：**

```python
from typing import Annotated, Optional

from fastapi import APIRouter, Depends, Query, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.schemas.event import EventCreate, EventUpdate, EventResponse, EventListResponse
from app.services.event_service import EventService
from app.dependencies import get_current_admin, get_current_user

router = APIRouter()
DbSession = Annotated[AsyncSession, Depends(get_db)]


@router.get("/", response_model=EventListResponse)
async def list_events(
    session: DbSession,
    page: int = Query(1, ge=1),
    size: int = Query(20, ge=1, le=100),
    category: Optional[str] = None,
    min_price: Optional[float] = Query(None, ge=0),
    max_price: Optional[float] = Query(None, ge=0),
    keyword: Optional[str] = None,
):
    """获取活动列表（支持分页、分类过滤、价格范围、关键词搜索）"""
    service = EventService(session)
    events, total = await service.list_events(
        page=page, size=size,
        category=category,
        keyword=keyword,
        min_price=min_price,
        max_price=max_price,
    )

    return EventListResponse(
        items=events,
        total=total,
        page=page,
        size=size,
        total_pages=(total + size - 1) // size,
    )


@router.get("/{event_id}", response_model=EventResponse)
async def get_event(event_id: int, session: DbSession):
    """获取活动详情"""
    service = EventService(session)
    event = await service.get_event(event_id)
    if not event:
        raise HTTPException(status_code=404, detail="活动不存在")
    return event


@router.post("/", response_model=EventResponse, status_code=status.HTTP_201_CREATED)
async def create_event(
    data: EventCreate,
    session: DbSession,
    current_user = Depends(get_current_admin),
):
    """创建活动（管理员权限）"""
    service = EventService(session)
    return await service.create_event(data, user_id=current_user.id)


@router.put("/{event_id}", response_model=EventResponse)
async def update_event(
    event_id: int,
    data: EventUpdate,
    session: DbSession,
    current_user = Depends(get_current_admin),
):
    """更新活动（管理员权限）"""
    service = EventService(session)
    event = await service.update_event(event_id, data)
    if not event:
        raise HTTPException(status_code=404, detail="活动不存在")
    return event


@router.delete("/{event_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_event(
    event_id: int,
    session: DbSession,
    current_user = Depends(get_current_admin),
):
    """删除活动（管理员权限）"""
    service = EventService(session)
    deleted = await service.delete_event(event_id)
    if not deleted:
        raise HTTPException(status_code=404, detail="活动不存在")
    return None
```

### 5.3 依赖注入（认证相关）

> ⚠️ **项目实际位置：** 以下代码在 `app/middleware/auth.py` 中实现（而非 `dependencies.py`）。认证中间件包含了完整的 JWT 验证、令牌黑名单检查和 RBAC 角色控制。

**app/middleware/auth.py（简版）：**

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.models import User
from app.utils.jwt import decode_token

security = HTTPBearer()


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    session: AsyncSession = Depends(get_db),
) -> User:
    """获取当前登录用户"""
    token = credentials.credentials
    payload = decode_token(token)
    if payload is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="无效的访问令牌",
        )

    user = await session.get(User, int(payload.get("sub")))
    if user is None:
        raise HTTPException(status_code=404, detail="用户不存在")

    return user


async def get_current_admin(current_user: User = Depends(get_current_user)) -> User:
    """获取当前管理员（需要 admin 角色）"""
    if current_user.role != "admin":
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="需要管理员权限",
        )
    return current_user
```text

### 5.4 测试 API

```bash
# 1. 启动服务
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# 2. 先注册管理员（第7章实现完整认证，当前用种子数据）
# 假设已有用户 admin / admin123

# 3. 获取 token
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login \
    -H "Content-Type: application/json" \
    -d '{"username": "admin", "password": "admin123"}' \
    | python -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# 4. 创建活动
curl -X POST http://localhost:8000/api/v1/events/ \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{
        "title": "周杰伦2024巡回演唱会 - 北京站",
        "venue": "国家体育场（鸟巢）",
        "start_date": "2024-08-15T19:30:00",
        "end_date": "2024-08-15T23:00:00",
        "price": 680.00,
        "total_stock": 1000,
        "category": "concert"
    }'

# 5. 获取活动列表
curl http://localhost:8000/api/v1/events/

# 6. 获取活动详情
curl http://localhost:8000/api/v1/events/1

# 7. 分页测试
curl "http://localhost:8000/api/v1/events/?page=1&size=5"

# 8. 过滤测试
curl "http://localhost:8000/api/v1/events/?category=concert&min_price=100&max_price=1000"

# 9. 更新活动
curl -X PUT http://localhost:8000/api/v1/events/1 \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"price": 780.00, "status": "published"}'

# 10. 删除活动
curl -X DELETE http://localhost:8000/api/v1/events/1 \
    -H "Authorization: Bearer $TOKEN"
```

---

## 6. 常见问题与排错

| 问题 | 原因 | 解决 |
| ------ | ------ | ------ |
| PUT 更新后部分字段变 NULL | 使用了 `exclude_unset=False` | 使用 `exclude_unset=True` |
| 分页越到后面越慢 | Offset 分页问题 | 大数据量改用游标分页 |
| 过滤条件不生效 | SQLAlchemy 条件未正确组合 | 使用 `and_(*conditions)` |
| 外键约束错误 1452 | 引用的父记录不存在 | 先插入父记录 |
| 权限验证 403 | Token 中角色不对 | 检查用户是否为 admin |
| PUT 与 PATCH 混用 | 语义理解不清 | PUT 全量更新，PATCH 部分更新 |

---

## 7. 本章总结

✅ **已掌握：**

- REST 六大约束、HTTP 方法幂等性数学证明
- HTTP 状态码选择（2xx/4xx/5xx）和 RFC 7807 Problem Details
- 分页算法：Offset vs 游标分页及性能证明
- 排序、过滤、API 版本化策略
- Service 层和 Repository 设计模式
- 活动 CRUD 完整实现（路由 + Schema + Service）

✅ **项目里程碑：**

- [ ] GET /api/v1/events — 活动列表（分页+过滤+排序）
- [ ] GET /api/v1/events/{id} — 活动详情
- [ ] POST /api/v1/events — 创建活动（管理员）
- [ ] PUT /api/v1/events/{id} — 更新活动（管理员）
- [ ] DELETE /api/v1/events/{id} — 删除活动（管理员）
- [ ] Swagger 文档显示所有端点

**下一章预告：** 第6章将深入学习 Redis 和 Lua 脚本，实现防超卖核心逻辑——这是票务系统最关键的技术挑战。

**练习：**

1. 完成本章所有代码文件的创建
2. 在 Swagger 页面测试活动 CRUD
3. 实现游标分页版本的活动列表接口
4. 添加自定义排序字段（如按价格、按时间）
5. 实现多条件组合过滤
