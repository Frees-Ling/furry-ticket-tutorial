# 超级管理员与角色管理

## 概述

系统实现三级 RBAC 角色体系：`user → organizer → admin → super_admin`

1. **角色层级** — 每种角色包含下级所有权限
2. **super_admin** — 最高权限角色，可管理其他管理员
3. **角色编辑** — 超级管理员可修改任意用户角色
4. **IP 历史** — 查看用户登录 IP 记录
5. **数据库浏览器** — 超级管理员专用 DB 管理工具

---

## 一、角色层级体系

### 角色定义

| 角色 | 权限范围 | 说明 |
|------|----------|------|
| `user` | 基础功能 | 购票、查看活动、个人中心 |
| `organizer` | 活动管理 | 创建/编辑/删除自己的活动 |
| `admin` | 系统管理 | 用户管理、订单管理、查看日志 |
| `super_admin` | 完全控制 | 角色管理、IP 历史、DB 浏览、系统配置 |

### RoleChecker 实现

`app/middleware/auth.py`

```python
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
        # super_admin 不受封禁限制
        if current_user.role != "super_admin" and current_user.is_banned is True:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="账户已被封禁",
            )
        return current_user


# 预定义的角色检查器
require_admin = RoleChecker(["admin", "super_admin"])
require_super_admin = RoleChecker(["super_admin"])
require_organizer = RoleChecker(["admin", "super_admin", "organizer"])
```

**关键行为：**

- `require_admin` — 允许 admin 和 super_admin
- `require_super_admin` — 仅允许 super_admin
- `require_organizer` — 允许 organizer、admin、super_admin
- super_admin **不受封禁限制**（即使被封禁也能操作）

### 使用示例

```python
@router.put("/users/{user_id}/role")
async def set_user_role(
    ...,
    admin: User = Depends(require_super_admin),  # 仅超级管理员
):
    ...

@router.get("/users")
async def list_users(
    ...,
    _admin: User = Depends(require_admin),  # admin 及以上
):
    ...
```

---

## 二、超级管理员功能

### 1. 设置用户角色

**路由：** `PUT /admin/users/{user_id}/role`

**权限：** `require_super_admin`

```python
@router.put("/users/{user_id}/role")
async def set_user_role(
    user_id: int,
    data: SetRoleRequest,
    request: Request,
    session: AsyncSession = Depends(get_db),
    redis: Redis = Depends(get_redis),
    admin: User = Depends(require_super_admin),
) -> dict:
    """设置用户角色（仅超级管理员可操作）"""
    service = AdminService(session, redis)
    try:
        await service.set_user_role(
            user_id=user_id,
            role=data.role,
            current_user_id=admin.id,
            ip_address=client_ip,
        )
        await session.commit()
        return {"message": f"用户 {user_id} 角色已设置为 {data.role}", "role": data.role}
    except (ValueError, AppError) as e:
        await session.rollback()
        raise HTTPException(status_code=400, detail=str(e))
```

**服务层：**

```python
class AdminService:
    async def set_user_role(
        self, user_id: int, role: str, current_user_id: int,
        ip_address: str | None = None,
    ) -> User:
        # 不能修改自己的角色
        if user_id == current_user_id:
            raise ValidationError("不能修改自己的角色")

        # 校验角色值
        if role not in ("super_admin", "admin", "organizer", "user"):
            raise ValidationError(f"无效的角色: {role}")

        user = await self.session.get(User, user_id)
        if not user:
            raise NotFoundError("用户不存在")

        old_role = user.role
        if old_role == role:
            return user  # 未变化，直接返回

        user.role = role
        await self.session.flush()

        # 记录操作日志
        log = OperationLog(
            user_id=current_user_id,
            action="角色变更",
            ip_address=ip_address or "unknown",
            detail=f"用户 {user_id} 角色变更: {old_role} -> {role}",
            success=True,
        )
        self.session.add(log)
        return user
```

**安全限制：**

- 不能修改自己的角色（防止误操作导致所有 super_admin 丢失）
- 角色值白名单校验（`super_admin` / `admin` / `organizer` / `user`）
- 每次变更记录 OperationLog（action="角色变更"）

### 2. 登录锁定豁免

超级管理员不受账户锁定限制：

```python
class AuthService:
    async def login(self, username, password, ...):
        user = await self.session.execute(
            select(User).where(User.username == username)
        ).scalar_one_or_none()

        # 超级管理员不受登录锁定限制
        if user and user.role == "super_admin":
            await self.lockout.reset_attempts(username)
        else:
            if await self.lockout.is_locked(username):
                raise ValueError(f"账户已锁定，请 {ttl} 秒后再试")
        ...
```

### 3. IP 历史查询

**路由：** `GET /admin/users/{user_id}/ip-history`

**权限：** `require_super_admin`

```python
@router.get("/users/{user_id}/ip-history")
async def get_user_ip_history(
    user_id: int,
    page: int = Query(1, ge=1),
    size: int = Query(20, ge=1, le=100),
    session: AsyncSession = Depends(get_db),
    redis: Redis = Depends(get_redis),
    _admin: User = Depends(require_super_admin),  # 仅超级管理员
) -> dict:
    """获取用户IP历史（仅超级管理员可查看）"""
    service = AdminService(session, redis)
    history, total = await service.get_user_ip_history(user_id, page=page, size=size)
    ...
```

**响应示例：**

```json
{
    "items": [
        {
            "ip_address": "192.168.1.100",
            "user_agent": "Mozilla/5.0 ...",
            "login_at": "2026-05-14T10:30:00",
            "success": true
        }
    ],
    "total": 50,
    "page": 1,
    "size": 20,
    "total_pages": 3
}
```

### 4. 数据库浏览器

**路由：** `GET /admin/db/tables` 和 `GET /admin/db/tables/{table_name}`

**权限：** `require_super_admin`

```python
@router.get("/db/tables")
async def list_db_tables(
    session: AsyncSession = Depends(get_db),
    _admin: User = Depends(require_super_admin),
) -> dict:
    """列出所有数据库表（仅超级管理员）"""
    result = await session.execute(text("SHOW TABLES"))
    tables = [r[0] for r in result.all()]
    return {"tables": tables}


@router.get("/db/tables/{table_name}")
async def get_table_data(
    table_name: str,
    page: int = Query(1, ge=1),
    size: int = Query(20, ge=1, le=100),
    session: AsyncSession = Depends(get_db),
    _admin: User = Depends(require_super_admin),
) -> dict:
    """查看表结构和数据（仅超级管理员）"""
    # 获取列信息
    col_result = await session.execute(text(f"SHOW COLUMNS FROM `{table_name}`"))
    columns = [{"field": r[0], "type": r[1], ...} for r in col_result.all()]

    # 获取数据
    data_result = await session.execute(
        text(f"SELECT * FROM `{table_name}` LIMIT {size} OFFSET {offset}")
    )
    rows = [dict(r._mapping) for r in data_result.all()]
    return {"columns": columns, "rows": rows, ...}
```

### 5. 监控指标重置

**路由：** `POST /admin/metrics/reset`

**权限：** `require_super_admin`

```python
@router.post("/metrics/reset")
async def reset_metrics(
    _admin: User = Depends(require_super_admin),
) -> dict:
    """重置所有监控指标计数器（仅超级管理员）"""
    service = MetricsService()
    await service.reset()
    return {"message": "监控指标已重置"}
```

### 6. 日志统计

**路由：** `GET /admin/logs/statistics`

**权限：** `require_super_admin`

```python
@router.get("/logs/statistics", response_model=DashboardStatsResponse)
async def get_log_statistics(
    session: AsyncSession = Depends(get_db),
    redis: Redis = Depends(get_redis),
    _admin: User = Depends(require_super_admin),
) -> DashboardStatsResponse:
    """获取系统统计数据"""
    service = AdminService(session, redis)
    stats = await service.get_log_statistics()
    return DashboardStatsResponse(**stats)
```

---

## 三、超级管理员初始化

### 系统启动时自动创建

`app/services/seed_service.py` — `SeedService.ensure_super_admin`

```python
async def ensure_super_admin(self) -> User:
    """确保超级管理员账号存在（ID=1）

    检查 users 表 ID=1 的用户：
    - 不存在 → 创建超级管理员
    - 存在但角色不是 super_admin → 升级为 super_admin
    """
    user = await self.session.get(User, 1)
    if user is None:
        user = User(
            id=1,  # 固定 ID=1
            username=settings.DEFAULT_ADMIN_USERNAME,
            password_hash=hash_password(settings.DEFAULT_ADMIN_PASSWORD),
            email=settings.DEFAULT_ADMIN_EMAIL,
            role="super_admin",
            is_active=True,
        )
        self.session.add(user)
        await self.session.flush()
        logger.info("超级管理员已创建: username=%s", user.username)
    elif user.role != "super_admin":
        user.role = "super_admin"
        logger.info("用户 ID=1 已升级为超级管理员")
    return user
```

**配置项：**

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `DEFAULT_ADMIN_USERNAME` | `admin` | 超级管理员用户名 |
| `DEFAULT_ADMIN_PASSWORD` | `Admin@12345` | **生产环境必须修改！** |
| `DEFAULT_ADMIN_EMAIL` | `admin@ticket-sell.com` | 管理员邮箱 |

---

## 四、API 端点参考

| 方法 | 路径 | 说明 | 权限 |
|------|------|------|------|
| GET | `/api/v1/admin/users` | 用户列表（过滤/搜索） | admin |
| GET | `/api/v1/admin/users/{id}` | 用户详情（含订单统计） | admin |
| POST | `/api/v1/admin/users/{id}/ban` | 封禁用户 | admin |
| POST | `/api/v1/admin/users/{id}/unban` | 解封用户 | admin |
| PUT | `/api/v1/admin/users/{id}` | 修改用户信息 | admin |
| **PUT** | `/api/v1/admin/users/{id}/role` | **设置角色** | **super_admin** |
| **GET** | `/api/v1/admin/users/{id}/ip-history` | **IP 历史** | **super_admin** |
| **GET** | `/api/v1/admin/db/tables` | **列出数据库表** | **super_admin** |
| **GET** | `/api/v1/admin/db/tables/{name}` | **查看表数据** | **super_admin** |
| **POST** | `/api/v1/admin/metrics/reset` | **重置监控指标** | **super_admin** |
| **GET** | `/api/v1/admin/logs/statistics` | **日志统计** | **super_admin** |
| GET | `/api/v1/admin/logs` | 操作日志 | admin |
| GET | `/api/v1/admin/logs/login` | 登录日志 | admin |
| GET | `/api/v1/admin/logs/registration` | 注册日志 | admin |

---

## 五、安全要点

1. **不可自降**：super_admin 不能修改自己的角色，防止误操作导致系统失去最高管理员
2. **角色白名单**：角色值严格限定在 `["super_admin", "admin", "organizer", "user"]`
3. **操作审计**：角色变更自动记录 OperationLog，包含新旧角色和操作 IP
4. **封禁豁免**：super_admin 不受封禁限制，确保紧急情况下能登录系统
5. **启动保护**：默认密码 `Admin@12345` 必须在生产环境修改
6. **最小权限**：日常管理使用 admin 角色，仅必要时使用 super_admin
