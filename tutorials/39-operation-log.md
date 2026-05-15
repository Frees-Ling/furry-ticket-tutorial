# 操作日志系统

## 概述

操作日志系统记录所有用户关键操作，提供审计追踪和统计分析：

1. **OperationLog 模型** — 记录用户 ID、动作类型、IP、User-Agent、详情、成功状态
2. **自动记录** — 登录/注册/短信登录/管理员操作等重要行为自动记录
3. **分页查询** — 支持按动作类型、日期范围过滤和分页
4. **Tab 筛选** — 登录日志/注册日志/全部日志 切换
5. **日志统计** — 每日登录/注册计数，用于仪表盘展示

---

## 一、OperationLog 模型

### 代码位置

`app/models/operation_log.py`

```python
class OperationLog(Base):
    __tablename__ = "operation_logs"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    user_id: Mapped[int | None] = mapped_column(Integer, nullable=True, index=True)
    username: Mapped[str | None] = mapped_column(String(50), nullable=True)
    action: Mapped[str] = mapped_column(
        String(50),
        comment="动作类型: login, register, logout, sms_login, 封禁, 解封, 角色变更, create_event, update_event, delete_event, admin_action",
    )
    ip_address: Mapped[str] = mapped_column(String(45))
    user_agent: Mapped[str | None] = mapped_column(String(255), nullable=True)
    detail: Mapped[str | None] = mapped_column(String(500), nullable=True)
    success: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `user_id` | Integer? | 操作用户 ID（可为空，如未登录的注册操作） |
| `username` | String(50)? | 用户名（冗余存储，方便直接展示） |
| `action` | String(50) | 动作类型（login/register/封禁/角色变更 等） |
| `ip_address` | String(45) | 客户端 IP（支持 IPv6） |
| `user_agent` | String(255)? | 客户端 User-Agent |
| `detail` | String(500)? | 操作详情 |
| `success` | Boolean | 操作是否成功 |
| `created_at` | DateTime | 操作时间 |

### 常见的 action 类型

| action | 触发场景 |
|--------|----------|
| `login` | 密码登录成功 |
| `register` | 新用户注册 |
| `短信登录` | 短信验证码登录 |
| `封禁` | 管理员封禁用户 |
| `解封` | 管理员解封用户 |
| `角色变更` | 超级管理员修改用户角色 |
| `create_event` | 创建活动 |
| `update_event` | 更新活动 |
| `delete_event` | 删除活动 |
| `admin_action` | 其他管理员操作 |

---

## 二、记录操作日志

### 通用记录方法

`AuthService.create_operation_log` 是统一的日志记录入口：

```python
class AuthService:
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
        log = OperationLog(
            user_id=user_id,
            username=username,
            action=action,
            ip_address=ip_address or "unknown",
            user_agent=user_agent,
            detail=detail,
            success=success,
        )
        self.session.add(log)
        return log
```

### 各场景调用

**登录：**
```python
await service.create_operation_log(
    user_id=user.id, username=user.username,
    action="登录", ip_address=client_ip,
    user_agent=user_agent, detail=f"用户登录: {user.username}", success=True,
)
```

**注册：**
```python
await service.create_operation_log(
    user_id=user.id, username=user.username,
    action="注册", ip_address=client_ip,
    user_agent=user_agent, detail=f"用户注册: {user.username}", success=True,
)
```

**短信登录：**
```python
await service.create_operation_log(
    user_id=user.id, username=user.username,
    action="短信登录", ip_address=client_ip,
    user_agent=user_agent, detail=f"短信登录: {user.username}", success=True,
)
```

**管理员封禁用户（AdminService）：**
```python
log = OperationLog(
    user_id=admin_id, username=admin_username,
    action="封禁", ip_address=ip_address or "unknown",
    detail=f"封禁用户 {user_id}: {reason}", success=True,
)
self.session.add(log)
```

---

## 三、分页查询日志

### 服务层

`AdminService.list_operation_logs` 支持分页 + 动作类型 + 日期范围过滤：

```python
class AdminService:
    async def list_operation_logs(
        self,
        page: int = 1,
        size: int = 20,
        action: str | None = None,      # 动作类型过滤
        date_from: datetime | None = None,  # 起始日期
        date_to: datetime | None = None,    # 结束日期
    ) -> tuple[list[OperationLog], int]:
        page = max(1, page)
        size = min(max(1, size), 100)

        conditions = []
        if action:
            conditions.append(OperationLog.action == action)
        if date_from:
            conditions.append(OperationLog.created_at >= date_from)
        if date_to:
            conditions.append(OperationLog.created_at <= date_to)

        # 查询总数
        count_stmt = select(func.count(OperationLog.id))
        if conditions:
            count_stmt = count_stmt.where(and_(*conditions))
        total = await self.session.scalar(count_stmt) or 0

        # 查询分页数据（按时间倒序）
        stmt = (
            select(OperationLog)
            .order_by(OperationLog.created_at.desc())
            .offset((page - 1) * size)
            .limit(size)
        )
        if conditions:
            stmt = stmt.where(and_(*conditions))
        result = await self.session.execute(stmt)
        logs = list(result.scalars().all())

        return logs, total
```

**过滤逻辑：**

- `action` 精确匹配（如 `"登录"`、`"注册"`、`"封禁"`）
- `date_from` + `date_to` 范围过滤（`created_at >= date_from AND created_at <= date_to`）
- 空条件时返回全量
- 按 `created_at DESC` 排序

### 路由层

`app/routers/admin.py` 提供三个日志查询端点：

**全量日志：** `GET /admin/logs`

```python
@router.get("/logs", response_model=OperationLogListResponse)
async def list_operation_logs(
    page: int = Query(1, ge=1),
    size: int = Query(20, ge=1, le=100),
    action: str | None = Query(None, description="动作类型过滤"),
    date_from: datetime | None = Query(None, description="起始日期"),
    date_to: datetime | None = Query(None, description="结束日期"),
    session: AsyncSession = Depends(get_db),
    redis: Redis = Depends(get_redis),
    _admin: User = Depends(require_admin),  # 需要管理员权限
) -> OperationLogListResponse:
    """查看操作日志（分页，支持动作和日期范围过滤）"""
    service = AdminService(session, redis)
    logs, total = await service.list_operation_logs(
        page=page, size=size, action=action,
        date_from=date_from, date_to=date_to,
    )
    total_pages = (total + size - 1) // size if total > 0 else 1
    items = [OperationLogResponse.model_validate(log) for log in logs]
    return OperationLogListResponse(
        items=items, total=total, page=page, size=size, total_pages=total_pages,
    )
```

### Tab 筛选

系统提供三个日志端点实现 Tab 切换：

| Tab | 端点 | action 过滤 |
|-----|------|-------------|
| 全部日志 | `GET /admin/logs` | 无过滤 |
| 登录日志 | `GET /admin/logs/login` | `action="登录"` |
| 注册日志 | `GET /admin/logs/registration` | `action="注册"` |

**登录日志：**
```python
@router.get("/logs/login", response_model=OperationLogListResponse)
async def list_login_logs(...):
    service = AdminService(session, redis)
    logs, total = await service.list_operation_logs(
        page=page, size=size, action="登录",  # 仅过滤登录
        date_from=date_from, date_to=date_to,
    )
    ...
```

**注册日志：**
```python
@router.get("/logs/registration", response_model=OperationLogListResponse)
async def list_registration_logs(...):
    service = AdminService(session, redis)
    logs, total = await service.list_operation_logs(
        page=page, size=size, action="注册",  # 仅过滤注册
        date_from=date_from, date_to=date_to,
    )
    ...
```

### 响应格式

```json
{
    "items": [
        {
            "id": 1,
            "user_id": 1,
            "username": "admin",
            "action": "登录",
            "ip_address": "127.0.0.1",
            "user_agent": "Mozilla/5.0 ...",
            "detail": "用户登录: admin",
            "success": true,
            "created_at": "2026-05-14T10:30:00+08:00"
        }
    ],
    "total": 100,
    "page": 1,
    "size": 20,
    "total_pages": 5
}
```

---

## 四、日志统计

### 代码位置

`AdminService.get_log_statistics`

```python
async def get_log_statistics(self) -> dict:
    """获取日志统计数据"""
    today_start = datetime.now(UTC).replace(hour=0, minute=0, second=0, microsecond=0)

    # 总用户数
    result = await self.session.execute(select(func.count(User.id)))
    total_users = result.scalar() or 0

    # 今日新增用户
    result = await self.session.execute(
        select(func.count(User.id)).where(User.created_at >= today_start)
    )
    new_users_today = result.scalar() or 0

    # 今日登录次数（从操作日志）
    result = await self.session.execute(
        select(func.count(OperationLog.id)).where(
            OperationLog.action == "login",
            OperationLog.created_at >= today_start,
        )
    )
    login_count_today = result.scalar() or 0

    # 今日注册次数（从操作日志）
    result = await self.session.execute(
        select(func.count(OperationLog.id)).where(
            OperationLog.action == "register",
            OperationLog.created_at >= today_start,
        )
    )
    register_count_today = result.scalar() or 0

    return {
        "total_users": total_users,
        "new_users_today": new_users_today,
        "total_orders": total_orders,
        "total_revenue": total_revenue,
        "login_count_today": login_count_today,
        "register_count_today": register_count_today,
    }
```

**API：** `GET /admin/logs/statistics`（需要 `require_super_admin` 权限）

---

## 五、API 端点参考

| 方法 | 路径 | 说明 | 权限 |
|------|------|------|------|
| GET | `/api/v1/admin/logs` | 操作日志分页（支持 action/date 过滤） | admin |
| GET | `/api/v1/admin/logs/login` | 登录日志 | admin |
| GET | `/api/v1/admin/logs/registration` | 注册日志 | admin |
| GET | `/api/v1/admin/logs/statistics` | 日志统计数据 | super_admin |

**通用查询参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `page` | int | 页码（默认 1） |
| `size` | int | 每页数量（默认 20，最大 100） |
| `action` | str | 动作类型过滤（仅全量端点） |
| `date_from` | datetime | 起始日期 |
| `date_to` | datetime | 结束日期 |

---

## 六、安全要点

1. **只读接口**：日志系统只提供查询接口，不提供删除/修改接口，保证审计完整性
2. **权限控制**：日志查询需要 admin 权限，统计需要 super_admin 权限
3. **自动记录**：关键操作自动记录，无需手动调用
4. **不可否认性**：记录 IP + User-Agent + 时间戳，提供审计追踪依据
5. **冗余存储**：username 冗余存储，即使将来用户被删除，日志仍可追溯
