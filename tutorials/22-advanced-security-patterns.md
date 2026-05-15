# 第 22 章：进阶安全模式与防滥用

> 本章在前一章安全体系的基础上，进一步深入实现用户管理安全、防滥用系统、实时订单推送、支付超时处理和前端用户体验优化。这些内容共同构成了票务系统的第二道安全防线——业务层面的安全防护。

## 目录

1. [用户管理安全](#22.1)
2. [防滥用系统](#22.2)
3. [实时订单推送](#22.3)
4. [支付超时处理](#22.4)
5. [前端用户体验优化](#22.5)
6. [综合防护架构](#22.6)
7. [代码示例](#22.7)
8. [生产部署安全清单更新](#22.8)

---

## 22.1 用户管理安全 {#22.1}

在第 7 章的认证授权系统基础上，本系统需要更精细的用户管理能力，包括封号、角色管理和登录审计。

### 22.1.1 封号系统设计

封号系统用于对违规用户进行临时或永久封禁，是维护平台秩序的最后手段。

#### 模型字段设计

在 `User` 模型中新增以下字段：

```python
# app/models/user.py — 用户模型封号字段

class User(Base):
    """用户模型"""
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(unique=True, index=True)
    email: Mapped[str] = mapped_column(unique=True, index=True)
    password_hash: Mapped[str]
    role: Mapped[str] = mapped_column(default="user")  # user / organizer / admin
    is_active: Mapped[bool] = mapped_column(default=True)

    # === 封号字段 ===
    is_banned: Mapped[bool] = mapped_column(default=False)           # 封号标记
    ban_reason: Mapped[str | None] = mapped_column(nullable=True)    # 封号原因
    banned_at: Mapped[datetime | None] = mapped_column(nullable=True)  # 封号时间
    banned_by: Mapped[int | None] = mapped_column(nullable=True)     # 操作管理员 ID
```text

**字段设计说明：**

| 字段 | 类型 | 用途 |
| ------ | ------ | ------ |
| `is_banned` | bool | 快速筛选被封禁用户，建立索引 |
| `ban_reason` | varchar | 记录封禁原因，供用户申诉时查看 |
| `banned_at` | datetime | 封禁时间，区分永久封禁和临时封禁 |
| `banned_by` | int | 追责——哪个管理员执行的封禁操作 |

#### 封号检查流程

```

用户请求 API
  ↓
JWT 解析 → 获取 user_id
  ↓
数据库查询用户
  ↓
is_banned = True？
  ├── 是 → 返回 403 "账户已被封禁，原因：{ban_reason}"
  └── 否 → 继续处理

```text

在 `get_current_user` 依赖中集成封号检查：

```python
# app/middleware/auth.py — 集成封号检查（⚠️ 实际文件路径）

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    session: AsyncSession = Depends(get_db),
) -> User:
    """获取当前用户（集成封号检查）"""
    payload = decode_token(token)
    if not payload:
        raise HTTPException(status_code=401, detail="无效的访问令牌")

    user_id = int(payload["sub"])
    user = await session.get(User, user_id)

    if not user:
        raise HTTPException(status_code=401, detail="用户不存在")

    if not user.is_active:
        raise HTTPException(status_code=403, detail="账户已被禁用")

    # 封号检查
    if user.is_banned:
        raise HTTPException(
            status_code=403,
            detail=f"账户已被封禁。原因：{user.ban_reason or '违反平台规定'}",
        )

    return user
```

#### 管理员封号 API

```python
# app/routers/admin.py — 管理员封号接口

from pydantic import BaseModel, Field

class BanUserRequest(BaseModel):
    """封号请求"""
    reason: str = Field(..., min_length=1, max_length=500, description="封号原因")
    duration_hours: int | None = Field(
        None, ge=1, le=8760,
        description="封禁时长（小时），不传则为永久封禁"
    )

class UnbanUserRequest(BaseModel):
    """解封请求"""
    reason: str = Field(..., min_length=1, max_length=500, description="解封原因")


class AdminService:
    """管理员服务"""

    def __init__(self, session: AsyncSession):
        self.session = session

    async def ban_user(
        self,
        user_id: int,
        admin_id: int,
        reason: str,
        duration_hours: int | None = None,
    ) -> User:
        """封禁用户"""
        user = await self.session.get(User, user_id)
        if not user:
            raise HTTPException(status_code=404, detail="用户不存在")

        if user.is_banned:
            raise HTTPException(status_code=409, detail="用户已被封禁")

        if user.role == "admin":
            raise HTTPException(status_code=403, detail="无法封禁管理员账号")

        user.is_banned = True
        user.ban_reason = reason
        user.banned_at = datetime.utcnow()
        user.banned_by = admin_id

        # 如果设置了临时封禁，记录到期时间（实际可在定时任务中自动解封）
        if duration_hours:
            user.ban_expires_at = datetime.utcnow() + timedelta(hours=duration_hours)

        await self.session.flush()
        return user

    async def unban_user(
        self,
        user_id: int,
        admin_id: int,
        reason: str,
    ) -> User:
        """解封用户"""
        user = await self.session.get(User, user_id)
        if not user:
            raise HTTPException(status_code=404, detail="用户不存在")

        if not user.is_banned:
            raise HTTPException(status_code=409, detail="用户未被封禁")

        user.is_banned = False
        user.ban_reason = None
        user.banned_at = None
        user.banned_by = None

        await self.session.flush()
        return user
```text

### 22.1.2 角色管理

本系统采用 **user / organizer / admin** 三层角色权限体系：

```

角色层级（从低到高）：
  user       → 基本用户：购票、查看订单、管理个人信息
  organizer  → 主办方：创建活动、管理座位、查看统计数据
  admin      → 管理员：所有权限 + 用户管理 + 封号/解封

权限继承：
  admin 拥有 organizer 和 user 的所有权限
  organizer 拥有 user 的所有权限
  user 仅有基本权限

```text

```python
# app/utils/permissions.py — 权限常量定义

from enum import Enum

class Role(str, Enum):
    USER = "user"
    ORGANIZER = "organizer"
    ADMIN = "admin"

# 角色优先级（数字越大权限越高）
ROLE_PRIORITY = {
    Role.USER: 1,
    Role.ORGANIZER: 2,
    Role.ADMIN: 3,
}

def role_ge(required: Role, current: str) -> bool:
    """检查当前角色是否 >= 要求的角色"""
    return ROLE_PRIORITY.get(current, 0) >= ROLE_PRIORITY[required]


# 路由权限装饰器
def require_role(minimum_role: Role):
    """要求最低角色"""
    async def role_checker(current_user: User = Depends(get_current_user)):
        if not role_ge(minimum_role, current_user.role):
            raise HTTPException(
                status_code=403,
                detail=f"需要 {minimum_role.value} 及以上角色",
            )
        return current_user
    return role_checker


# 使用
OrganizerDep = Annotated[User, Depends(require_role(Role.ORGANIZER))]
AdminDep = Annotated[User, Depends(require_role(Role.ADMIN))]


@router.post("/admin/users/{user_id}/ban", dependencies=[Depends(AdminDep)])
async def ban_user(user_id: int, req: BanUserRequest, session: DbSession):
    """封禁用户（仅管理员）"""
    service = AdminService(session)
    user = await service.ban_user(user_id, req.reason, req.duration_hours)
    return {"message": "用户已封禁", "user_id": user.id}
```

### 22.1.3 登录历史审计

记录每次登录尝试，包括 IP 地址、User-Agent、成功/失败状态：

```python
# app/models/login_log.py — 登录日志模型

class LoginLog(Base):
    """登录日志"""
    __tablename__ = "login_logs"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int | None] = mapped_column(nullable=True)  # 登录失败时可能为 NULL
    username: Mapped[str]                              # 尝试登录的用户名
    ip_address: Mapped[str]                            # IP 地址
    user_agent: Mapped[str | None] = mapped_column(nullable=True)
    success: Mapped[bool]                              # 是否成功
    fail_reason: Mapped[str | None] = mapped_column(nullable=True)
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
```text

```python
# app/services/auth_service.py — 登录审计

class LoginAuditService:
    """登录审计服务"""

    def __init__(self, session: AsyncSession):
        self.session = session

    async def record_login(
        self,
        username: str,
        ip_address: str,
        user_agent: str | None,
        success: bool,
        user_id: int | None = None,
        fail_reason: str | None = None,
    ):
        """记录登录尝试"""
        log = LoginLog(
            user_id=user_id,
            username=username,
            ip_address=ip_address,
            user_agent=user_agent,
            success=success,
            fail_reason=fail_reason,
        )
        self.session.add(log)
        await self.session.flush()

    async def get_user_login_history(
        self,
        user_id: int,
        limit: int = 20,
    ) -> list[LoginLog]:
        """获取用户最近登录记录"""
        result = await self.session.execute(
            select(LoginLog)
            .where(LoginLog.user_id == user_id)
            .order_by(LoginLog.created_at.desc())
            .limit(limit)
        )
        return result.scalars().all()
```

**审计数据可用场景：**

- **异常检测**：同一用户短时间内在多个不同 IP 登录 → 账户可能被盗
- **用户申诉**：用户反馈"账号被盗"时，可提供登录记录供其核对
- **合规要求**：票务平台通常需要保留登录日志 180 天以上

### 22.1.4 批量用户操作

管理员需要对用户进行批量操作，如批量封号（黄牛检测后）、批量角色变更：

```python
class AdminService:
    # ... 接前面的代码 ...

    async def batch_ban_users(
        self,
        user_ids: list[int],
        admin_id: int,
        reason: str,
    ) -> dict:
        """批量封禁用户"""
        result = await self.session.execute(
            select(User).where(User.id.in_(user_ids))
        )
        users = result.scalars().all()

        banned_count = 0
        skipped_count = 0
        for user in users:
            if user.role == "admin":
                skipped_count += 1
                continue
            user.is_banned = True
            user.ban_reason = reason
            user.banned_at = datetime.utcnow()
            user.banned_by = admin_id
            banned_count += 1

        await self.session.flush()
        return {
            "total": len(user_ids),
            "banned": banned_count,
            "skipped": skipped_count,
        }
```text

---

## 22.2 防滥用系统 {#22.2}

防滥用系统旨在防止机器人和恶意用户对平台进行攻击，包括刷票、占座、黄牛操作等。

### 22.2.1 下单频率限制（滑动窗口算法）

使用 Redis Sorted Set 实现滑动窗口算法，精确统计用户在时间窗口内的操作次数：

```python
# app/services/behavior_service.py — 行为检测服务

import time
from redis.asyncio import Redis

class BehaviorService:
    """行为检测服务"""

    def __init__(self, redis: Redis):
        self.redis = redis

    async def check_order_frequency(
        self,
        user_id: int,
        window_seconds: int = 600,    # 时间窗口：10 分钟
        max_orders: int = 5,           # 最大订单数
    ) -> tuple[bool, int]:
        """
        检查用户下单频率

        使用 Redis Sorted Set 实现滑动窗口：
        - member: 时间戳（每个请求唯一）
        - score: 时间戳
        - 窗口内请求数 = ZCARD

        返回：
        - (allowed, current_count) 是否允许继续 + 当前窗口内请求数
        """
        key = f"behavior:order_freq:{user_id}"
        now = time.time()
        window_start = now - window_seconds

        # 移除窗口外的记录
        await self.redis.zremrangebyscore(key, 0, window_start)

        # 统计窗口内记录数
        count = await self.redis.zcard(key)

        if count >= max_orders:
            return False, count

        # 添加当前记录
        await self.redis.zadd(key, {str(now): now})

        # 设置过期时间（窗口 + 60 秒缓冲）
        await self.redis.expire(key, window_seconds + 60)

        return True, count + 1
```

**滑动窗口算法图解：**

```text
时间轴（10 分钟窗口）：
                    窗口开始                   窗口结束
                      │                          │
                      ▼                          ▼
    ──┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───
      │   │   │   │   │   │   │   │   │   │   │   │   │   │   │
      ↑       ↑   ↑       ↑                               ↑
    旧记录     ←─── 窗口内的记录（计入当前频率）───→      当前请求
    （已过期）
```

**与固定窗口算法的对比：**

| 特性 | 固定窗口（每分钟重置） | 滑动窗口（Sorted Set） |
| ------ | ---------------------- | ---------------------- |
| 精度 | 低（边界突发可能翻倍） | 高（精确到秒） |
| Redis 内存 | O(1)（一个计数器） | O(N)（N 为窗口内请求数） |
| 实现复杂度 | 低 | 中 |
| 边界效应 | 存在 | 无 |

### 22.2.2 并发订单限制

限制用户同时持有的未支付订单数量，防止恶意占座：

```python
class BehaviorService:
    # ... 接前面的代码 ...

    async def check_concurrent_orders(self, user_id: int, max_pending: int = 3) -> bool:
        """
        检查用户并发未支付订单数

        ⚠️ 实际项目使用 Redis 计数器（INCR/DECR 模式），
        而非查询数据库。计数器在 record_order_attempt 中递增，
        在 release_pending_order 中递减。

        Returns:
            True 表示允许下单，False 表示已超过并发限制
        """
        key = f"behavior:pending:{user_id}"
        current = await self.redis.get(key)
        count = int(current) if current else 0

        if count >= max_pending:
            return False

        return True
```text

**为什么需要这个限制？**

```

场景：
  正常用户：一次购买 1-2 张票，创建 1 笔订单
  黄牛用户：使用脚本创建 100 笔订单，每笔占 1 张票
            ↓ 锁定 100 张库存
            ↓ 慢慢付钱或取消，导致正常用户无票可买

并发限制作用：
  限制每用户最多 3 笔未支付订单
  → 黄牛最多占 3 张，大幅降低囤票效率

```text

### 22.2.3 IP 维度限制

在用户维度限制之外增加 IP 维度限制，防止同一 IP 下多个账号的集体刷票行为：

```python
class BehaviorService:
    # ... 接前面的代码 ...

    async def check_ip_order_frequency(
        self,
        ip_address: str,
        window_seconds: int = 3600,     # 时间窗口：1 小时
        max_orders: int = 20,           # 最大订单数
    ) -> tuple[bool, int]:
        """
        检查 IP 下单频率

        防止同一 IP 下多个账号同时刷票。
        使用与用户频率检查相同的滑动窗口算法。
        """
        key = f"behavior:ip_order_freq:{ip_address}"
        now = time.time()
        window_start = now - window_seconds

        await self.redis.zremrangebyscore(key, 0, window_start)
        count = await self.redis.zcard(key)

        if count >= max_orders:
            return False, count

        await self.redis.zadd(key, {str(now): now})
        await self.redis.expire(key, window_seconds + 60)

        return True, count + 1


    async def check_ip_ban(
        self,
        ip_address: str,
    ) -> bool:
        """检查 IP 是否被临时封禁"""
        key = f"behavior:ip_banned:{ip_address}"
        return await self.redis.exists(key) == 1


    async def temporary_ban_ip(
        self,
        ip_address: str,
        duration_seconds: int = 1800,   # 临时封禁 30 分钟
    ):
        """临时封禁 IP"""
        key = f"behavior:ip_banned:{ip_address}"
        await self.redis.setex(key, duration_seconds, "1")
```

### 22.2.4 可疑行为检测

通过分析用户行为模式，识别黄牛和恶意用户：

```python
class BehaviorService:
    # ... 接前面的代码 ...

    async def record_order_attempt(self, user_id: int, ip: str):
        """记录下单行为（下单成功后调用）

        更新三条 Redis 数据：
        1. 用户下单频率窗口（Sorted Set）
        2. IP 下单频率窗口（Sorted Set）
        3. 用户并发订单计数器（INCR）
        """
        now = time.time()

        freq_key = f"behavior:order_freq:{user_id}"
        await self.redis.zadd(freq_key, {str(now): now})
        await self.redis.expire(freq_key, 600)

        ip_key = f"behavior:ip_rate:{ip}"
        await self.redis.zadd(ip_key, {str(now): now})
        await self.redis.expire(ip_key, 3600)

        pending_key = f"behavior:pending:{user_id}"
        await self.redis.incr(pending_key)
        await self.redis.expire(pending_key, 86400)

    async def release_pending_order(self, user_id: int):
        """释放一笔并发订单（支付成功或取消时调用）"""
        key = f"behavior:pending:{user_id}"
        current = await self.redis.get(key)
        if current and int(current) > 0:
            await self.redis.decr(key)

    async def is_suspicious(self, user_id: int) -> Tuple[bool, str]:
        """
        检测可疑行为（基于 Redis，无 DB 查询）

        检查项：
        - 短时间内大量下单未支付（恶意占座）
        - 大量高频操作

        Returns:
            (is_suspicious, reason)
        """
        freq_key = f"behavior:order_freq:{user_id}"
        now = time.time()
        await self.redis.zremrangebyscore(freq_key, 0, now - 600)
        freq_count = await self.redis.zcard(freq_key)

        if freq_count >= 5:
            return True, "短时间内订单频率异常"

        pending_key = f"behavior:pending:{user_id}"
        pending_count = int(await self.redis.get(pending_key) or 0)
        if pending_count >= 5:
            return True, f"有 {pending_count} 笔未支付订单"

        return False, ""
```text

### 22.2.5 分级惩罚

根据违规严重程度，实施分级惩罚策略：

```

惩罚等级金字塔：

                    ┌─────────────────────┐
                    │   Level 3: 永久封号   │ ← 严重违规
                    │   多次恶意刷票       │
                    │   支付欺诈           │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Level 2: 临时封禁   │ ← 中度违规
                    │   24-72 小时         │
                    │   可疑行为检测触发    │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Level 1: 操作限制   │ ← 轻微违规
                    │   限流 + 警告        │
                    │   超过频率阈值触发    │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Level 0: 正常操作   │
                    │   无限制             │
                    └─────────────────────┘

```text

```python
class BehaviorService:
    # ... 接前面的代码 ...

    PENALTY_CONFIG = {
        "warning": {
            "description": "警告提示",
            "action": "return_warning",
        },
        "rate_limit": {
            "description": "限流 30 分钟",
            "action": "throttle",
            "duration": 1800,
        },
        "temp_ban": {
            "description": "临时封禁 24 小时",
            "action": "ban_user",
            "duration": 86400,
        },
        "perm_ban": {
            "description": "永久封禁",
            "action": "ban_user_permanent",
        },
    }

    async def apply_penalty(
        self,
        user_id: int,
        level: int,
        db_session: AsyncSession,
    ) -> dict:
        """根据违规等级实施惩罚"""
        if level == 0:
            return {"penalty": "none", "message": "正常"}

        if level == 1:
            # 限流：Redis 中设置用户限流标记
            key = f"behavior:throttled:{user_id}"
            await self.redis.setex(key, 1800, "1")  # 30 分钟
            return {
                "penalty": "rate_limit",
                "message": "您的操作过于频繁，请 30 分钟后再试",
            }

        if level == 2:
            # 临时封禁
            key = f"behavior:banned:{user_id}"
            await self.redis.setex(key, 86400, "1")  # 24 小时
            return {
                "penalty": "temp_ban",
                "message": "因检测到可疑行为，账号已被临时冻结 24 小时",
            }

        if level == 3:
            # 永久封号
            async with db_session.begin():
                result = await db_session.execute(
                    select(User).where(User.id == user_id)
                )
                user = result.scalar_one_or_none()
                if user:
                    user.is_banned = True
                    user.ban_reason = "多次违规操作，永久封禁"
                    user.banned_at = datetime.utcnow()
            return {
                "penalty": "perm_ban",
                "message": "账号已被永久封禁",
            }

        return {"penalty": "unknown"}
```

**分级惩罚决策树：**

```text
请求到达
  ↓
频率检查 ← 超过阈值？ → Level 1: 限流 + 警告
  ↓ 正常
并发检查 ← 超过限制？ → Level 1: 拒绝 + 提示
  ↓ 正常
IP 检查 ← 异常？     → Level 1: IP 限流
  ↓ 正常
可疑行为检测 ← 可疑？ → Level 2: 临时封禁
  ↓ 正常
重复违规？          → Level 3: 永久封号
  ↓ 正常
放行
```

---

## 22.3 实时订单推送 {#22.3}

当订单状态发生变化（支付成功、超时取消、手动取消）时，系统需要实时通知用户。本系统使用用户级 WebSocket 房间实现这一功能。

### 22.3.1 用户级 WebSocket 房间

与第 8 章中座位更新的"事件房间"不同，订单推送使用"用户房间"：

```text
Room 命名规范：
  座位更新: seats:{event_id}        — 所有查看该活动座位图的用户
  订单状态: user:{user_id}          — 仅该用户

特点：
  seats 房间：多个用户共享，广播所有用户座位选择
  user 房间：一对一关系，只有用户自己接收自己的订单通知
```

```python
# app/routers/ws_user.py — 用户订单 WebSocket 路由

from fastapi import APIRouter, WebSocket, WebSocketDisconnect, Query
from app.services.connection_manager import manager
from app.utils.security import decode_token

router = APIRouter()


@router.websocket("/ws/user/orders")
async def user_orders_websocket(
    websocket: WebSocket,
    token: str = Query(...),           # 通过查询参数传递 JWT
):
    """用户订单状态 WebSocket 端点

    用户连接后订阅自己的订单状态变更：
    - 支付成功 → 收到 paid 通知
    - 超时取消 → 收到 expired 通知
    - 手动取消 → 收到 cancelled 通知

    客户端连接示例：
    new WebSocket("ws://localhost:8000/ws/user/orders?token=xxx")
    """
    # 1. 验证 JWT 令牌
    payload = decode_token(token)
    if not payload or payload.get("type") != "access":
        await websocket.close(code=4001, reason="Invalid token")
        return

    user_id = int(payload["sub"])
    room = f"user:{user_id}"

    # 2. 连接到用户房间
    await manager.connect(websocket, room)

    try:
        # 发送初始连接确认
        await websocket.send_json({
            "type": "connected",
            "message": "订单状态推送已连接",
        })

        # 保持连接，持续接收消息（实际上订单推送是单向的服务器→客户端）
        # 但需要保持循环以处理断开事件
        while True:
            data = await websocket.receive_text()
            # 客户端可以发送 ping 保持连接
            if data == "ping":
                await websocket.send_json({"type": "pong"})

    except WebSocketDisconnect:
        manager.disconnect(websocket, room)
    except Exception:
        manager.disconnect(websocket, room)
```text

### 22.3.2 订单状态变更自动推送

在订单状态变更的业务逻辑中，调用 WebSocket 广播：

```python
# app/services/connection_manager.py — ConnectionManager（广播器）
# ⚠️ 注意：实际项目中不存在 OrderNotificationService 类。
# 订单状态广播直接使用 ConnectionManager.broadcast_order_status() 方法。

class ConnectionManager:
    """WebSocket 连接管理器：支持房间级广播"""

    def __init__(self):
        self._rooms: Dict[str, Set[WebSocket]] = {}

    async def broadcast(self, room: str, message: dict):
        """向房间内所有客户端广播消息"""
        if room not in self._rooms:
            return
        stale = set()
        for ws in self._rooms[room]:
            try:
                await ws.send_json(message)
            except Exception:
                stale.add(ws)
        for ws in stale:
            self._rooms[room].discard(ws)

    async def broadcast_order_status(
        self, user_id: int, order_id: int, status: str, order_no: str
    ):
        """向特定用户广播订单状态变更"""
        room = f"user:{user_id}"
        await self.broadcast(room, {
            "type": "order_status",
            "order_id": order_id,
            "status": status,
            "order_no": order_no,
        })


# 全局单例（整个应用共享同一个 ConnectionManager）
manager = ConnectionManager()
```

在 `order_service.py` 中实际调用方式如下：

```python
# app/services/order_service.py — 订单服务中调用广播
from app.services.connection_manager import manager

# 取消订单时广播
await manager.broadcast_order_status(
    user_id=order.user_id,
    order_id=order.id,
    status="cancelled",
    order_no=order.order_no,
)

# 支付成功时广播（在 payment_service.py 中）
await manager.broadcast_order_status(
    user_id=order.user_id,
    order_id=order.id,
    status="paid",
    order_no=order.order_no,
)
```text

### 22.3.3 前端状态自动更新

前端 JavaScript 监听 WebSocket 消息，收到订单状态变更后自动更新 UI：

```javascript
// frontend/src/utils/orderSocket.js — 订单 WebSocket 客户端

class OrderWebSocket {
    constructor(token) {
        this.token = token;
        this.ws = null;
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = 5;
        this.listeners = {};
    }

    connect() {
        this.ws = new WebSocket(
            `ws://localhost:8000/ws/user/orders?token=${this.token}`
        );

        this.ws.onopen = () => {
            console.log("订单推送连接已建立");
            this.reconnectAttempts = 0;
            this._emit("connected");
        };

        this.ws.onmessage = (event) => {
            const data = JSON.parse(event.data);
            if (data.type === "order_status") {
                // 触发订单状态更新事件
                this._emit("order_status", data);
            } else if (data.type === "connected") {
                console.log("已连接到订单推送服务");
            } else if (data.type === "pong") {
                // 心跳响应，无需处理
            }
        };

        this.ws.onclose = (event) => {
            console.log("订单推送连接关闭:", event.code);
            this._emit("disconnected");
            this._reconnect();
        };

        this.ws.onerror = (error) => {
            console.error("订单推送连接错误:", error);
        };
    }

    // 发送心跳保持连接
    startHeartbeat() {
        setInterval(() => {
            if (this.ws && this.ws.readyState === WebSocket.OPEN) {
                this.ws.send("ping");
            }
        }, 30000);  // 每 30 秒发送一次 ping
    }

    // 监听事件
    on(event, callback) {
        if (!this.listeners[event]) {
            this.listeners[event] = [];
        }
        this.listeners[event].push(callback);
    }

    _emit(event, data) {
        const callbacks = this.listeners[event] || [];
        callbacks.forEach(cb => cb(data));
    }

    // 自动重连
    _reconnect() {
        if (this.reconnectAttempts >= this.maxReconnectAttempts) {
            console.error("重连次数已达上限");
            return;
        }
        this.reconnectAttempts++;
        const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
        console.log(`将在 ${delay}ms 后重连...`);
        setTimeout(() => this.connect(), delay);
    }

    disconnect() {
        if (this.ws) {
            this.ws.close(1000, "用户主动断开");
        }
    }
}
---

## 22.4 支付超时处理 {#22.4}

支付超时处理是票务系统的核心可靠性机制，确保未支付的订单不会永久占用库存。

### 22.4.1 15 分钟支付超时

创建订单时设置超时时间：

```python
# app/services/order_service.py — 订单超时字段

class OrderService:
    """订单服务"""

    async def create_order(
        self,
        user_id: int,
        event_id: int,
        seat_ids: list[str],
        session: AsyncSession,
    ) -> Order:
        """创建订单（设置超时时间）"""
        # ... 库存扣减、价格计算等逻辑 ...

        expires_at = datetime.utcnow() + timedelta(minutes=15)

        order = Order(
            order_no=generate_order_no(),
            user_id=user_id,
            event_id=event_id,
            total_amount=total_amount,
            status="pending_payment",
            expires_at=expires_at,          # 支付超时时间
            created_at=datetime.utcnow(),
        )
        session.add(order)
        await session.flush()

        # 调度 ARQ 超时任务
        await self.schedule_timeout_check(order)

        return order
```

### 22.4.2 ARQ Worker 定时取消

ARQ Worker 每 60 秒轮询一次，批量取消超时未支付的订单：

```python
# app/worker.py — 超时订单取消任务

from datetime import datetime, timedelta
from sqlalchemy import select, update
from arq import cron


async def cancel_expired_orders(ctx):
    """
    ARQ 定时任务：取消超时未支付订单

    每 60 秒执行一次：
    1. 查询所有超时的 pending_payment 订单
    2. 批量取消（更新状态）
    3. 归还库存（Redis INCRBY）
    4. 广播超时通知给用户
    5. 记录日志

    返回取消的订单数量。
    """
    redis = ctx["redis"]
    session = ctx["session"]

    # 1. 查询超时订单
    now = datetime.utcnow()
    result = await session.execute(
        select(Order).where(
            Order.status == "pending_payment",
            Order.expires_at < now,
        )
    )
    expired_orders = result.scalars().all()

    if not expired_orders:
        return {"cancelled": 0, "message": "无超时订单"}

    # 2. 批量取消
    order_ids = [o.id for o in expired_orders]
    await session.execute(
        update(Order)
        .where(Order.id.in_(order_ids))
        .values(
            status="expired",
            updated_at=now,
        )
    )

    # 3. 归还库存 + 广播通知
    from app.services.connection_manager import manager

    for order in expired_orders:
        # 归还库存
        stock_key = f"stock:{order.event_id}"
        await redis.incrby(stock_key, order.quantity)

        # 广播过期通知到活动房间（座位更新）
        await manager.broadcast(f"seats:{order.event_id}", {
            "type": "seat_released",
            "data": {"seats": order.seat_ids},
            "reason": "payment_timeout",
        })

        # 推送超时通知到用户房间（直接使用 broadcast_order_status）
        await manager.broadcast_order_status(
            user_id=order.user_id,
            order_id=order.id,
            status="cancelled",
            order_no=order.order_no,
        )

    await session.commit()

    return {
        "cancelled": len(expired_orders),
        "order_ids": order_ids,
        "message": f"已取消 {len(expired_orders)} 笔超时订单",
    }


# WorkerSettings 中注册
class WorkerSettings:
    functions = [
        cancel_expired_orders,
        # ... 其他任务函数
    ]
    cron_jobs = [
        cron(
            cancel_expired_orders,
            second=0,               # 每分钟的第 0 秒执行
            description="取消超时未支付订单",
        ),
    ]
    redis_settings = RedisSettings(...)
```text

### 22.4.3 库存自动归还时序图

```

订单创建                                     15 分钟后
  │                                             │
  ▼                                             ▼
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│  创建订单     │     │  等待支付         │     │   ARQ 取消   │
│  扣减库存     │ ──→ │  expires_at 倒计时 │ ──→ │  归还库存     │
│  expires_at  │     │  用户可在此期内支付 │     │  广播通知     │
│  = now+15min │     │                   │     │  状态→expired│
└──────────────┘     └──────────────────┘     └──────────────┘
                             │
                             │ 用户支付成功？
                             ├── 是 → 订单 → paid，库存确认扣减
                             └── 否 → ARQ 取消 → 归还库存

```text

### 22.4.4 前端倒计时展示

```vue
<!-- frontend/src/components/PaymentCountdown.vue -->
<template>
  <div class="payment-countdown" :class="{ urgent: isUrgent }">
    <div class="countdown-label">支付剩余时间</div>
    <div class="countdown-timer">{{ displayTime }}</div>
    <div v-if="isUrgent" class="urgent-warning">
      ⚠ 即将超时，请尽快完成支付
    </div>
    <div class="progress-bar">
      <div
        class="progress-fill"
        :style="{ width: progressPercent + '%' }"
        :class="{ 'progress-danger': isUrgent }"
      ></div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from "vue";

const props = defineProps({
  expiresAt: {
    type: String,   // ISO 8601 格式
    required: true,
  },
});

const emit = defineEmits(["expired"]);

const now = ref(Date.now());
let timer = null;

const remainingMs = computed(() => {
  const expires = new Date(props.expiresAt).getTime();
  return Math.max(0, expires - now.value);
});

const displayTime = computed(() => {
  const totalSeconds = Math.floor(remainingMs.value / 1000);
  const minutes = Math.floor(totalSeconds / 60);
  const seconds = totalSeconds % 60;
  return `${String(minutes).padStart(2, "0")}:${String(seconds).padStart(2, "0")}`;
});

const progressPercent = computed(() => {
  const total = 15 * 60 * 1000;  // 15 分钟总毫秒数
  return (remainingMs.value / total) * 100;
});

const isUrgent = computed(() => {
  return remainingMs.value < totalTime.value * 0.1;  // 剩余 < 10%
});

const totalTime = computed(() => {
  const expires = new Date(props.expiresAt).getTime();
  const created = expires - 15 * 60 * 1000;
  return expires - created;
});

onMounted(() => {
  timer = setInterval(() => {
    now.value = Date.now();
    if (remainingMs.value <= 0) {
      clearInterval(timer);
      emit("expired");
    }
  }, 1000);
});

onUnmounted(() => {
  if (timer) clearInterval(timer);
});
</script>
```

---

## 22.5 前端用户体验优化 {#22.5}

### 22.5.1 全局 Toast 通知系统

```vue
<!-- frontend/src/components/ToastNotification.vue -->
<template>
  <div class="toast-container">
    <transition-group name="toast">
      <div
        v-for="toast in toasts"
        :key="toast.id"
        :class="['toast-item', `toast-${toast.type}`]"
      >
        <div class="toast-icon">
          <span v-if="toast.type === 'success'">✓</span>
          <span v-else-if="toast.type === 'error'">✕</span>
          <span v-else-if="toast.type === 'warning'">⚠</span>
          <span v-else>ℹ</span>
        </div>
        <div class="toast-content">
          <div class="toast-title">{{ toast.title }}</div>
          <div v-if="toast.message" class="toast-message">
            {{ toast.message }}
          </div>
        </div>
        <button class="toast-close" @click="removeToast(toast.id)">×</button>
      </div>
    </transition-group>
  </div>
</template>

<script setup>
import { ref } from "vue";

const toasts = ref([]);
let toastId = 0;

function showToast(type, title, message = "", duration = 4000) {
  const id = ++toastId;
  toasts.value.push({ id, type, title, message });

  if (duration > 0) {
    setTimeout(() => removeToast(id), duration);
  }
}

function removeToast(id) {
  const index = toasts.value.findIndex(t => t.id === id);
  if (index !== -1) {
    toasts.value.splice(index, 1);
  }
}

// 全局暴露
defineExpose({ showToast });
</script>

<style scoped>
.toast-container {
  position: fixed;
  top: 20px;
  right: 20px;
  z-index: 9999;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.toast-item {
  display: flex;
  align-items: flex-start;
  padding: 12px 16px;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  min-width: 300px;
  max-width: 420px;
  animation: slideIn 0.3s ease;
}

.toast-success { background: #f0fdf4; border-left: 4px solid #22c55e; }
.toast-error   { background: #fef2f2; border-left: 4px solid #ef4444; }
.toast-warning { background: #fffbeb; border-left: 4px solid #f59e0b; }
.toast-info    { background: #eff6ff; border-left: 4px solid #3b82f6; }

@keyframes slideIn {
  from { transform: translateX(100%); opacity: 0; }
  to   { transform: translateX(0); opacity: 1; }
}
</style>
```text

### 22.5.2 支付倒计时组件

倒计时组件已在 22.4.4 节中详细展示，此处不再重复。

### 22.5.3 骨架屏加载状态

```vue
<!-- frontend/src/components/SkeletonLoader.vue -->
<template>
  <div class="skeleton-loader" :class="`skeleton-${type}`">
    <div v-for="i in rows" :key="i" class="skeleton-row">
      <div class="skeleton-block" :style="randomWidth()"></div>
    </div>
  </div>
</template>

<script setup>
const props = defineProps({
  type: {
    type: String,
    default: "text",     // text / card / table
  },
  rows: {
    type: Number,
    default: 3,
  },
});

function randomWidth() {
  const widths = ["60%", "75%", "85%", "90%", "70%", "80%"];
  return { width: widths[Math.floor(Math.random() * widths.length)] };
}
</script>

<style scoped>
.skeleton-block {
  height: 16px;
  border-radius: 4px;
  background: linear-gradient(
    90deg,
    #e5e7eb 25%,
    #f3f4f6 50%,
    #e5e7eb 75%
  );
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
  margin-bottom: 12px;
}

@keyframes shimmer {
  0%   { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}

.skeleton-card .skeleton-block {
  height: 200px;
  width: 100% !important;
}

.skeleton-table .skeleton-block {
  height: 40px;
  width: 100% !important;
}
</style>
```

### 22.5.4 订单状态徽章

```vue
<!-- frontend/src/components/OrderStatusBadge.vue -->
<template>
  <span :class="['badge', `badge-${status}`]">
    {{ statusText }}
  </span>
</template>

<script setup>
const props = defineProps({
  status: {
    type: String,
    required: true,
    validator: (v) => [
      "pending_payment", "paid", "cancelled", "expired", "refunded",
    ].includes(v),
  },
});

const statusMap = {
  pending_payment: { text: "待支付",   class: "badge-warning" },
  paid:            { text: "已支付",   class: "badge-success" },
  cancelled:       { text: "已取消",   class: "badge-secondary" },
  expired:         { text: "已超时",   class: "badge-danger" },
  refunded:        { text: "已退款",   class: "badge-info" },
};

const statusText = computed(() => statusMap[props.status]?.text || props.status);
</script>

<style scoped>
.badge {
  display: inline-block;
  padding: 4px 12px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 500;
  line-height: 1.5;
}

.badge-warning { background: #fef3c7; color: #92400e; }
.badge-success { background: #d1fae5; color: #065f46; }
.badge-secondary { background: #e5e7eb; color: #374151; }
.badge-danger { background: #fee2e2; color: #991b1b; }
.badge-info { background: #dbeafe; color: #1e40af; }
</style>
```text

### 22.5.5 表单实时校验反馈

```vue
<!-- frontend/src/components/ValidatedInput.vue -->
<template>
  <div class="validated-input">
    <label :for="name" class="input-label">
      {{ label }}
      <span v-if="required" class="required-mark">*</span>
    </label>

    <input
      :id="name"
      :type="type"
      :value="modelValue"
      :placeholder="placeholder"
      :class="['input-field', { 'input-error': error, 'input-success': valid }]"
      @input="handleInput"
      @blur="handleBlur"
    />

    <div v-if="error" class="error-message">{{ error }}</div>
    <div v-else-if="hint && !valid" class="hint-message">{{ hint }}</div>
  </div>
</template>

<script setup>
import { ref, watch } from "vue";

const props = defineProps({
  modelValue: { type: String, default: "" },
  name: { type: String, required: true },
  label: { type: String, required: true },
  type: { type: String, default: "text" },
  placeholder: { type: String, default: "" },
  required: { type: Boolean, default: false },
  rules: { type: Array, default: () => [] },    // 校验规则数组
  hint: { type: String, default: "" },
});

const emit = defineEmits(["update:modelValue"]);

const error = ref("");
const valid = ref(false);
const touched = ref(false);

function validate(value) {
  if (props.required && !value) {
    return `${props.label}不能为空`;
  }
  for (const rule of props.rules) {
    const result = rule(value);
    if (result !== true) {
      return result;
    }
  }
  return null;
}

function handleInput(e) {
  const value = e.target.value;
  emit("update:modelValue", value);
  if (touched.value) {
    error.value = validate(value);
    valid.value = !error.value;
  }
}

function handleBlur() {
  touched.value = true;
  error.value = validate(props.modelValue);
  valid.value = !error.value;
}

watch(() => props.modelValue, (newVal) => {
  if (touched.value) {
    error.value = validate(newVal);
    valid.value = !error.value;
  }
});
</script>

<style scoped>
.input-field {
  width: 100%;
  padding: 10px 12px;
  border: 2px solid #d1d5db;
  border-radius: 6px;
  font-size: 14px;
  transition: border-color 0.2s, box-shadow 0.2s;
}

.input-field:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.input-error {
  border-color: #ef4444;
}

.input-success {
  border-color: #22c55e;
}

.error-message {
  color: #ef4444;
  font-size: 12px;
  margin-top: 4px;
}

.hint-message {
  color: #6b7280;
  font-size: 12px;
  margin-top: 4px;
}
</style>
```

---

## 22.6 综合防护架构 {#22.6}

### 22.6.1 用户请求管道

```text
┌──────────────────────────────────────────────────────────────────────────┐
│                          用户请求管道                                     │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐ │
│  │   L1     │  │   L2     │  │   L3     │  │   L4     │  │    L5     │ │
│  │  JWT     │  │  IP      │  │  行为频率  │  │  并发检查  │  │ WebSocket │ │
│  │  认证    │  │  限流    │  │  检查     │  │  (订单)   │  │   推送    │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └─────┬─────┘ │
│       │             │             │             │              │       │
│       ▼             ▼             ▼             ▼              ▼       │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    拒绝服务 / 返回错误 / 限制操作                  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  每层防护说明：                                                            │
│  L1 - JWT 认证：验证用户身份，拦截未授权请求                              │
│  L2 - IP 限流：基于 Nginx + Redis 限制 IP 请求频率                       │
│  L3 - 行为频率：基于 Sorted Set 滑动窗口检测异常行为                      │
│  L4 - 并发检查：限制未支付订单数量，防止占座                             │
│  L5 - WebSocket 推送：实时通知订单状态变更                               │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 22.6.2 防护层次对比

| 防护层级 | 技术实现 | 响应时间 | 拦截对象 |
| --------- | --------- | --------- | --------- |
| L1: JWT 认证 | HS256 签名 + 令牌黑名单 | < 1ms | 未授权/过期令牌 |
| L2: IP 限流 | Nginx limit_req + Redis 计数 | < 2ms | 超频 IP |
| L3: 行为频率 | Redis Sorted Set 滑动窗口 | < 5ms | 超频用户 |
| L4: 并发检查 | SQL COUNT 查询 | < 10ms | 占座用户 |
| L5: WebSocket 推送 | ConnectionManager 广播 | < 50ms | —（通知层） |

### 22.6.3 数据流向

```text
                     ┌─────────────────────────────────────┐
                     │            前端 (Vue 3)              │
                     │  ┌─────────┐  ┌───────────────────┐  │
                     │  │ 用户页面  │  │ WebSocket 客户端   │  │
                     │  └────┬────┘  └────────┬──────────┘  │
                     │       │                │              │
                     └───────┼────────────────┼──────────────┘
                             │                │
                    HTTP API │                │ WebSocket
                             │                │
                     ┌───────▼────────────────▼──────────────┐
                     │          Nginx 反向代理                │
                     │    limit_req + 缓存 + SSL 终止         │
                     └───────────────────┬───────────────────┘
                                         │
                     ┌───────────────────▼───────────────────┐
                     │         FastAPI 应用服务器              │
                     │                                       │
                     │  ┌─────────────────────────────────┐  │
                     │  │   L1: JWT 认证检查               │  │
                     │  │   L2: IP 限流检查                 │  │
                     │  │   L3: BehaviorService 行为检查    │  │
                     │  │   L4: OrderService 并发检查       │  │
                     │  └─────────────────────────────────┘  │
                     │                │                      │
                     ├────────┬───────┼───────┬──────────────┤
                     │        │       │       │              │
                     ▼        ▼       ▼       ▼              │
                 ┌────────┐ ┌────────┐ ┌────────┐ ┌──────┐  │
                 │ MySQL  │ │ Redis  │ │ 支付宝  │ │ ARQ  │  │
                 │ 数据库  │ │ 缓存   │ │ 支付    │ │ Worker│  │
                 └────────┘ └────────┘ └────────┘ └──────┘  │
                     │        │                              │
                     │        └─────────── L5: WebSocket ────┘
                     │                   ConnectionManager
                     └────────────────── 推送通知 ──────────┘
```

---

## 22.7 代码示例 {#22.7}

### 22.7.1 AdminService.ban_user() 完整实现

```python
class AdminService:
    """管理员服务"""

    def __init__(
        self,
        session: AsyncSession,
        redis: Redis | None = None,
    ):
        self.session = session
        self.redis = redis

    async def ban_user(
        self,
        user_id: int,
        admin_id: int,
        reason: str,
        duration_hours: int | None = None,
    ) -> User:
        """封禁用户——完整的实现，含审计和通知"""
        user = await self.session.get(User, user_id)
        if not user:
            raise HTTPException(status_code=404, detail="用户不存在")

        if user.is_banned:
            raise HTTPException(status_code=409, detail="用户已被封禁")

        if user.role == "admin":
            raise HTTPException(status_code=403, detail="无法封禁管理员账号")

        # 封禁用户
        user.is_banned = True
        user.ban_reason = reason
        user.banned_at = datetime.utcnow()
        user.banned_by = admin_id

        # 记录审计日志
        audit_log = AuditLog(
            action="user_banned",
            actor_id=admin_id,
            target_id=user_id,
            details={
                "reason": reason,
                "duration_hours": duration_hours,
                "username": user.username,
            },
        )
        self.session.add(audit_log)

        # 如果设置了 Redis，主动使该用户的会话失效
        if self.redis:
            token_version_key = f"user_token_version:{user_id}"
            await self.redis.incr(token_version_key)

        await self.session.flush()
        return user
```text

### 22.7.2 BehaviorService.check_order_frequency() 完整实现

```python
class BehaviorService:
    """行为检测服务"""

    def __init__(self, redis: Redis):
        self.redis = redis

    async def check_order_frequency(
        self,
        user_id: int,
        window_seconds: int = 600,
        max_orders: int = 5,
    ) -> tuple[bool, int, int | None]:
        """
        检查用户下单频率——完整的滑动窗口实现

        参数：
            user_id: 用户 ID
            window_seconds: 时间窗口（秒）
            max_orders: 窗口内允许的最大订单数

        返回：
            (allowed, current_count, retry_after)
            - allowed: 是否允许下单
            - current_count: 当前窗口内订单数
            - retry_after: 建议重试时间（秒），None 表示不需要等待
        """
        key = f"behavior:order_freq:{user_id}"
        now = time.time()
        window_start = now - window_seconds

        # 使用 Lua 脚本保证原子性
        lua_script = """
            local key = KEYS[1]
            local now = tonumber(ARGV[1])
            local window_start = tonumber(ARGV[2])
            local max_orders = tonumber(ARGV[3])
            local window_seconds = tonumber(ARGV[4])

            -- 移除过期记录
            redis.call('ZREMRANGEBYSCORE', key, 0, window_start)

            -- 统计当前窗口内数量
            local count = redis.call('ZCARD', key)

            if count >= max_orders then
                -- 获取最早的记录时间，计算需要等待的时间
                local earliest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
                local retry_after = math.ceil(tonumber(earliest[2]) + window_seconds - now)
                return {0, count, retry_after}
            end

            -- 添加当前记录
            redis.call('ZADD', key, now, now)
            redis.call('EXPIRE', key, window_seconds + 60)

            return {1, count + 1, 0}
        """

        result = await self.redis.eval(
            lua_script,
            1,  # keys 数量
            key,          # KEYS[1]
            now,          # ARGV[1]
            window_start, # ARGV[2]
            max_orders,   # ARGV[3]
            window_seconds, # ARGV[4]
        )

        allowed = bool(result[0])
        count = result[1]
        retry_after = result[2] if not allowed else None

        return allowed, count, retry_after
```

### 22.7.3 ConnectionManager.broadcast_order_status()

```python
# 扩展 ConnectionManager，增加便捷方法

class ConnectionManager:
    """WebSocket 连接管理器"""

    # ... 原有方法（connect, disconnect, broadcast）...

    async def broadcast_order_status(
        self,
        user_id: int,
        order_no: str,
        status: str,
        extra: dict | None = None,
    ):
        """向用户广播订单状态变更"""
        message = {
            "type": "order_status",
            "order_no": order_no,
            "status": status,
            "timestamp": datetime.utcnow().isoformat(),
        }
        if extra:
            message.update(extra)

        await self.broadcast(f"user:{user_id}", message)

    async def broadcast_seat_update(self, event_id: int, data: dict):
        """向活动房间广播座位变更"""
        await self.broadcast(f"seats:{event_id}", {
            "type": "seat_update",
            "data": data,
        })
```text

### 22.7.4 综合判断入口

在订单创建路由中集成所有防护检查：

```python
@router.post("/orders")
async def create_order(
    req: OrderCreate,
    current_user: Annotated[User, Depends(get_current_user)],
    session: DbSession,
    redis: RedisDep,
    request: Request,
):
    """创建订单（含全部防滥用检查）"""
    behavior = BehaviorService(redis)
    user_id = current_user.id
    ip_address = request.client.host

    # 1. 检查用户是否被封禁（已在 get_current_user 中检查，此处作为二次确认）
    if current_user.is_banned:
        raise HTTPException(status_code=403, detail="账户已被封禁")

    # 2. 检查 IP 是否被封禁
    ip_banned = await behavior.check_ip_ban(ip_address)
    if ip_banned:
        raise HTTPException(status_code=429, detail="IP 已被临时封禁")

    # 3. 检查用户下单频率
    allowed, wait = await behavior.check_order_frequency(user_id)
    if not allowed:
        raise HTTPException(
            status_code=429,
            detail=f"操作过于频繁，请 {wait} 秒后再试",
        )

    # 4. 检查并发订单数
    concurrent_allowed = await behavior.check_concurrent_orders(user_id)
    if not concurrent_allowed:
        raise HTTPException(
            status_code=429,
            detail="未支付订单过多，请先完成支付或取消订单",
        )

    # 5. 检查 IP 下单频率
    ip_allowed = await behavior.check_ip_order_rate(ip_address)
    if not ip_allowed:
        raise HTTPException(status_code=429, detail="IP 下单过于频繁，请稍后再试")

    # 6. 可疑行为检测（轻量级，非阻塞）
    suspicious, reason = await behavior.is_suspicious(user_id)
    if suspicious:
        raise HTTPException(
            status_code=429,
            detail=f"检测到可疑操作：{reason}，账号已被临时冻结",
        )

    # 全部检查通过 → 创建订单
    order_service = OrderService(session, redis)
    order = await order_service.create_order(user_id, req)

    # 下单成功后记录行为数据
    await behavior.record_order_attempt(user_id, ip_address)
    await session.commit()

    return order
```

### 22.7.5 Toast 通知系统在 Vue 中的使用

```javascript
// frontend/src/stores/toast.js — Pinia 状态管理
import { defineStore } from "pinia";

export const useToastStore = defineStore("toast", {
    state: () => ({
        toasts: [],
        _nextId: 0,
    }),

    actions: {
        show(type, title, message = "", duration = 4000) {
            const id = ++this._nextId;
            this.toasts.push({ id, type, title, message });

            if (duration > 0) {
                setTimeout(() => this.dismiss(id), duration);
            }
            return id;
        },

        success(title, message) {
            return this.show("success", title, message);
        },

        error(title, message) {
            return this.show("error", title, message);
        },

        warning(title, message) {
            return this.show("warning", title, message);
        },

        info(title, message) {
            return this.show("info", title, message);
        },

        dismiss(id) {
            const index = this.toasts.findIndex(t => t.id === id);
            if (index !== -1) {
                this.toasts.splice(index, 1);
            }
        },

        clear() {
            this.toasts = [];
        },
    },
});


// 在 Vue 组件中使用
// <script setup>
// import { useToastStore } from "@/stores/toast";
// const toast = useToastStore();
//
// // 支付成功
// toast.success("支付成功", `订单 ${orderNo} 已完成支付`);
//
// // 支付失败
// toast.error("支付失败", "请检查账户余额后重试");
//
// // 检测到可疑行为
// toast.warning("操作受限", "您的操作频率过高，请稍后再试");
//
// // 订单状态变更（通过 WebSocket 接收后）
// socket.on("order_status", (data) => {
//     if (data.status === "paid") {
//         toast.success("订单已支付", `订单 ${data.order_no} 支付成功`);
//     } else if (data.status === "expired") {
//         toast.error("订单已超时", `订单 ${data.order_no} 因超时已自动取消`);
//     } else if (data.status === "cancelled") {
//         toast.info("订单已取消", `订单 ${data.order_no} 已取消`);
//     }
// });
// </script>
```text

---

## 22.8 生产部署安全清单更新 {#22.8}

在第 15 章和第 21 章的上线清单基础上，补充以下安全相关检查项：

### 22.8.1 封号与权限系统

- [ ] **封号 API 仅限管理员访问**：`ban_user` 和 `batch_ban_users` 路由添加了 `AdminDep` 权限检查
- [ ] **封号操作审计日志**：每次封号/解封操作都记录 `AuditLog`
- [ ] **角色提升审批流程**：普通用户→主办方→管理员需要后台审批，不可自助变更
- [ ] **最小权限原则**：API 路由使用 `require_role` 装饰器，不依赖前端权限控制
- [ ] **token 版本号**：封禁用户后递增其 token 版本号，强制使其所有会话即时失效

### 22.8.2 行为限制系统

- [ ] **行为限制阈值已根据业务调整**：
  - 下单频率：5 单/10 分钟（正常用户极少超过）
  - 并发未支付：3 笔上限
  - IP 下单频率：20 单/小时
- [ ] **Redis 滑动窗口内存监控**：Sorted Set 大小与窗口内请求数成正比，需监控 Redis 内存
- [ ] **分级惩罚配置确认**：Level 1 限流 / Level 2 临时封禁 / Level 3 永久封号
- [ ] **误判回退机制**：用户可通过客服申诉解除误封

### 22.8.3 实时推送系统

- [ ] **ARQ Worker 定时任务已启动**：

  ```bash
  # 启动 Worker（生产建议使用 supervisor 或 systemd 管理）
  arq app.worker.WorkerSettings
  ```

- [ ] **Worker 自动重连**：确保 Worker 进程崩溃后自动重启（Docker restart policy 或 systemd）
- [ ] **WebSocket 心跳检测配置完毕**：
  - 客户端每 30 秒发送 ping
  - 服务器 60 秒无响应后断开连接
  - 前端自动重连（指数退避，最多 5 次）
- [ ] **WebSocket 连接数限制**：Nginx 配置 `worker_connections` 足够容纳预期并发连接数

### 22.8.4 全量检查清单

| # | 检查项 | 验证方法 | 状态 |
| --- | -------- | --------- | ------ |
| 1 | 封号 API 权限 | 非管理员调用 `/admin/users/{id}/ban` 返回 403 | ☐ |
| 2 | 封号后即时生效 | 被封禁用户的下一个请求返回 403 | ☐ |
| 3 | 封号审计日志 | 操作后查询 audit_logs 表有对应记录 | ☐ |
| 4 | 下单频率限制 | 10 分钟内下 6 单，第 6 单被拒绝 | ☐ |
| 5 | 并发订单限制 | 有 3 笔未支付订单时，第 4 笔被拒绝 | ☐ |
| 6 | IP 限流 | 同一 IP 下多个账号高频下单被限流 | ☐ |
| 7 | ARQ Worker 运行 | `arq` Worker 日志显示定时轮询 | ☐ |
| 8 | 超时订单取消 | 创建超时订单，15 分钟后被自动取消 | ☐ |
| 9 | 超时后库存归还 | 取消后 Redis `stock:{event_id}` 正确增加 | ☐ |
| 10 | WebSocket 推送 | 订单支付后用户收到实时通知 | ☐ |
| 11 | 前端倒计时 | 支付页倒计时显示正确，超时后触发跳转 | ☐ |
| 12 | Toast 通知 | 订单状态变更时正确弹出通知 | ☐ |
| 13 | Worker 自动恢复 | 手动 kill Worker 后自动重启 | ☐ |

---

## 本章小结

本章在第 21 章安全体系基础上，进一步实现了业务层面的综合安全防护：

1. **用户管理安全**：封号系统（模型字段 + 检查流程 + 管理员 API）、三级角色管理、登录审计、批量操作
2. **防滥用系统**：基于 Redis Sorted Set 的滑动窗口频率限制、并发订单限制、IP 维度防护、可疑行为检测、分级惩罚
3. **实时订单推送**：用户级 WebSocket 房间、订单状态自动推送、前端自动更新 UI
4. **支付超时处理**：15 分钟超时设置、ARQ Worker 定时轮询取消、库存自动归还、前端倒计时展示
5. **前端用户体验优化**：全局 Toast 通知、骨架屏加载、订单状态徽章、表单实时校验
6. **综合防护架构**：L1-L5 五层用户请求管道，从 JWT 认证到 WebSocket 推送的全链路防护

**关键设计决策：**

- 使用 Sorted Set 而非固定窗口实现频率限制，消除边界突发效应
- 用户房间与事件房间分离，确保订单通知的私密性
- 封号同时递增 token 版本号，实现即时会话失效
- 分级惩罚策略，误判时可回退

**下一章预告：** 第 23 章将回顾开发过程中遇到的实际 Bug 与修复记录，包括 func.now() 比较陷阱、浮点数精度丢失、CSWSH 漏洞、Token 刷新竞态条件等经典问题。
