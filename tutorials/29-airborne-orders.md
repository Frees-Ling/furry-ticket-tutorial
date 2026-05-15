# 第 29 章：空降订单 — 管理员创建定向订单

## 本章学习目标

- 理解空降订单（Airborne Order）的业务场景和设计原理
- 掌握管理员创建定向订单的完整实现
- 学会处理指定用户支付的流程和边界情况

**难度等级：** 中高级

**前置知识：** 第 7 章（JWT 认证）、第 5 章（REST API）、第 16 章（支付宝支付）

**学习目标：**

1. 掌握空降订单的核心业务流程
2. 理解指定用户标识的确权机制
3. 实现管理员创建订单、用户完成支付的完整链路
4. 掌握空降订单的库存扣减策略

---

## 1. 空降订单业务场景

### 1.1 什么是空降订单？

空降订单指**由管理员创建、指定某个用户支付**的订单。与普通订单不同，它不需要用户自己下单，而是由运营人员代建。

```text
普通订单流程：
用户 → 选票 → 创建订单 → 用户支付 → 完成

空降订单流程：
管理员 → 选择用户 → 创建定向订单 → 该用户登录 → 支付订单 → 完成
```

### 1.2 适用场景

| 场景 | 说明 | 示例 |
| ------ | ------ | ------ |
| **客服代下单** | 用户无法自行下单 | 用户反馈系统故障，客服代为创建 |
| **活动赠票** | 定向赠送特定用户 | 赞助商赠送 VIP 票 |
| **线下转线上** | 线下购票后系统录入 | 现场售票处录入订单 |
| **异常补单** | 支付成功但订单未生成 | 系统故障时手动补单 |

### 1.3 核心挑战

```python
"""
空降订单的核心设计挑战：

1. 库存扣减时机
   - 管理员创建时扣减？→ 防止超卖
   - 用户支付时扣减？→ 可能支付时已无库存

2. 用户支付限制
   - 只有指定用户能支付
   - 防止其他人看到链接后支付

3. 价格策略
   - 与普通订单价格一致
   - 或支持自定义价格（赠票场景）

4. 权限控制
   - 只有管理员可以创建空降订单
   - 普通用户不能绕过下单流程
"""
```text

---

## 2. 数据库设计

### 2.1 订单表扩展

在现有 `orders` 表基础上扩展：

```sql
-- 订单表增加空降订单相关字段
ALTER TABLE orders
    ADD COLUMN order_type VARCHAR(20) NOT NULL DEFAULT 'normal'
        COMMENT '订单类型：normal=普通, airborne=空降',
    ADD COLUMN designated_user_id BIGINT DEFAULT NULL
        COMMENT '空降订单指定用户ID（空降订单专用）',
    ADD COLUMN created_by_admin_id BIGINT DEFAULT NULL
        COMMENT '创建管理员ID（空降订单专用）',
    ADD COLUMN remark VARCHAR(500) DEFAULT NULL
        COMMENT '订单备注（管理员填写）';

-- 索引
ALTER TABLE orders
    ADD INDEX idx_order_type (order_type),
    ADD INDEX idx_designated_user (designated_user_id);
```

### 2.2 Schema 设计

```python
# app/schemas/order.py — 新增 schema

from typing import Optional
from pydantic import BaseModel, Field


class AirborneOrderCreate(BaseModel):
    """管理员创建空降订单请求"""
    event_id: int = Field(..., gt=0, description="活动ID")
    quantity: int = Field(..., gt=0, le=10, description="购买数量")
    designated_user_id: int = Field(..., gt=0, description="指定支付的用户ID")
    remark: Optional[str] = Field(None, max_length=500, description="订单备注")


class AirborneOrderResponse(BaseModel):
    """空降订单响应"""
    model_config = {"from_attributes": True}

    id: int
    order_no: str
    user_id: int
    event_id: int
    quantity: int
    total_amount: float
    status: str
    order_type: str
    designated_user_id: Optional[int]
    created_by_admin_id: Optional[int]
    remark: Optional[str]
    created_at: str
```text

---

## 3. 业务逻辑实现

### 3.1 空降订单服务

```

空降订单核心逻辑：
    管理员
       │
       │ POST /api/v1/admin/airborne-orders
       │ {"event_id": 1, "quantity": 2, "designated_user_id": 5}
       ▼
   ┌─────────────────────────────────────┐
   │ 1. 权限校验：是否为 admin           │
   │ 2. 参数校验：验证指定用户是否存在   │
   │ 3. 活动校验：活动状态、库存          │
   │ 4. 库存扣减（Redis + MySQL 乐观锁）  │
   │ 5. 创建订单（order_type=airborne）   │
   │ 6. 记录管理员操作日志               │
   └─────────────────────────────────────┘
       │
       ▼
   返回订单信息 → 通知指定用户 → 用户支付

```text

```python
# app/services/admin_service.py — 空降订单创建逻辑

from decimal import Decimal
from typing import Optional
from sqlalchemy import select, update
from sqlalchemy.ext.asyncio import AsyncSession
from redis.asyncio import Redis

from app.models import Order, Event, User
from app.services.inventory_service import InventoryService
from app.utils.order_no import generate_order_no


class AdminService:
    """管理员服务"""

    def __init__(self, session: AsyncSession, redis: Redis):
        self.session = session
        self.redis = redis
        self.inventory_service = InventoryService(redis)

    async def create_airborne_order(
        self,
        event_id: int,
        quantity: int,
        designated_user_id: int,
        admin_id: int,
        remark: Optional[str] = None,
    ) -> Order:
        """
        创建空降订单（管理员代用户下单）

        流程：
        1. 验证指定用户存在
        2. 验证活动有效
        3. 库存扣减（使用现成的双层防超卖机制）
        4. 创建定向订单
        5. 返回订单信息

        参数：
            event_id: 活动ID
            quantity: 数量
            designated_user_id: 指定支付的用户ID
            admin_id: 管理员ID
            remark: 备注
        """
        # === 1. 验证指定用户存在 ===
        user_result = await self.session.execute(
            select(User).where(
                User.id == designated_user_id,
                User.is_active == True,
            )
        )
        user = user_result.scalar_one_or_none()
        if not user:
            raise ValueError("指定用户不存在或已禁用")

        # === 2. 验证活动 ===
        event_result = await self.session.execute(
            select(Event).where(
                Event.id == event_id,
                Event.status == "published",
            )
        )
        event = event_result.scalar_one_or_none()
        if not event:
            raise ValueError("活动不存在或未发布")

        # === 3. 库存扣减（双层防超卖） ===
        # 第一层：Redis Lua 原子扣减
        deduct_result = await self.inventory_service.deduct_stock(
            event_id=event_id,
            user_id=designated_user_id,
            quantity=quantity,
            max_per_user=0,  # 空降不限制单人购买数量（但受总库存约束）
        )

        if deduct_result <= 0:
            error_map = {
                0: "库存不足",
                -1: "超过限购数量",
                -2: "活动不存在",
            }
            raise ValueError(error_map.get(deduct_result, "库存扣减失败"))

        try:
            # 第二层：MySQL 乐观锁
            total_amount = Decimal(str(event.price)) * quantity
            order = Order(
                order_no=generate_order_no(),
                user_id=designated_user_id,          # 订单归属于指定用户
                event_id=event_id,
                quantity=quantity,
                total_amount=total_amount,
                status="pending_payment",            # 初始状态：待支付
                order_type="airborne",               # 标记为空降订单
                designated_user_id=designated_user_id,
                created_by_admin_id=admin_id,        # 记录创建者
                remark=remark,
                version=1,
            )
            self.session.add(order)
            await self.session.flush()

            # 乐观锁更新库存
            result = await self.session.execute(
                update(Event)
                .where(
                    Event.id == event_id,
                    Event.version == event.version,
                    Event.sold_count + quantity <= Event.total_stock,
                )
                .values(
                    sold_count=Event.sold_count + quantity,
                    version=Event.version + 1,
                )
            )

            if result.rowcount == 0:
                # 乐观锁冲突，回滚
                await self.session.rollback()
                await self.inventory_service.restore_stock(event_id, quantity)
                raise ValueError("库存已更新，请重试")

            return order

        except ValueError:
            # 业务异常，回滚 Redis 库存
            await self.inventory_service.restore_stock(event_id, quantity)
            raise
        except Exception:
            # 系统异常，回滚 Redis 库存
            await self.inventory_service.restore_stock(event_id, quantity)
            raise
```

### 3.2 支付流程

空降订单的支付流程与普通订单**共享支付接口**，但多了一层验证：

```python
# app/services/payment_service.py — 空降订单支付验证增强

class PaymentService:
    async def create_alipay_payment(self, order_id: int, user_id: int) -> dict:
        """生成支付宝支付参数（支持空降订单）"""
        order = await self.session.get(Order, order_id)

        # 通用校验
        if not order:
            raise ValueError("订单不存在")

        # === 关键校验：空降订单只能由指定用户支付 ===
        if order.order_type == "airborne":
            if order.designated_user_id != user_id:
                raise ValueError("该订单由管理员指定其他用户支付，您无权支付")
        else:
            # 普通订单：必须是订单所属用户
            if order.user_id != user_id:
                raise ValueError("无权操作此订单")

        # 状态校验
        if order.status != "pending_payment":
            raise ValueError("订单状态不允许支付")

        # ... 后续处理 ...
```text

### 3.3 管理员 API 路由

```python
# app/routers/admin.py — 空降订单路由

from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession
from redis.asyncio import Redis

from app.database import get_db
from app.redis_client import get_redis
from app.middleware.auth import get_current_user, require_admin
from app.models import User
from app.schemas.order import AirborneOrderCreate, AirborneOrderResponse
from app.services.admin_service import AdminService

router = APIRouter()


@router.post(
    "/airborne-orders",
    response_model=AirborneOrderResponse,
    status_code=status.HTTP_201_CREATED,
)
async def create_airborne_order(
    data: AirborneOrderCreate,
    session: AsyncSession = Depends(get_db),
    redis: Redis = Depends(get_redis),
    current_user: User = Depends(require_admin),  # 要求管理员权限
):
    """
    管理员创建空降订单。

    权限要求：admin 角色
    """
    service = AdminService(session, redis)

    try:
        order = await service.create_airborne_order(
            event_id=data.event_id,
            quantity=data.quantity,
            designated_user_id=data.designated_user_id,
            admin_id=current_user.id,
            remark=data.remark,
        )
        await session.commit()
        return order

    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e),
        )


@router.get("/airborne-orders", response_model=list[AirborneOrderResponse])
async def list_airborne_orders(
    session: AsyncSession = Depends(get_db),
    current_user: User = Depends(require_admin),
):
    """管理员查看空降订单列表"""
    from sqlalchemy import select

    result = await self.session.execute(
        select(Order)
        .where(Order.order_type == "airborne")
        .order_by(Order.created_at.desc())
    )
    orders = result.scalars().all()
    return orders
```

---

## 4. 前端实现

```vue
<!-- frontend/src/views/admin/AirborneOrderCreate.vue -->
<template>
  <div class="airborne-create">
    <h2>创建空降订单</h2>

    <form @submit.prevent="createOrder">
      <div class="form-group">
        <label>选择活动</label>
        <select v-model="form.event_id" required>
          <option v-for="ev in events" :key="ev.id" :value="ev.id">
            {{ ev.title }} - ¥{{ ev.price }}
          </option>
        </select>
      </div>

      <div class="form-group">
        <label>指定用户（输入用户ID或用户名搜索）</label>
        <input
          v-model="searchQuery"
          @input="searchUsers"
          placeholder="搜索用户..."
        />
        <select v-model="form.designated_user_id" size="5" v-if="users.length">
          <option
            v-for="u in users"
            :key="u.id"
            :value="u.id"
          >
            {{ u.username }} (ID: {{ u.id }})
          </option>
        </select>
      </div>

      <div class="form-group">
        <label>数量</label>
        <input type="number" v-model.number="form.quantity" min="1" max="10" />
      </div>

      <div class="form-group">
        <label>备注</label>
        <textarea v-model="form.remark" rows="3"></textarea>
      </div>

      <button type="submit" :disabled="loading">
        {{ loading ? '创建中...' : '创建空降订单' }}
      </button>
    </form>
  </div>
</template>
```text

---

## 5. 安全边界与注意事项

### 5.1 安全边界

```python
"""
空降订单安全边界清单：

1. 权限边界
   ✅ 只有 admin 角色可以创建空降订单
   ✅ 普通用户不能通过伪造请求创建
   ✅ 订单记录 created_by_admin_id

2. 支付边界
   ✅ 空降订单只能由 designated_user_id 支付
   ✅ 其他用户（包括管理员）不能代付
   ✅ 共享支付接口，代码复用

3. 库存边界
   ✅ 与普通订单共享 Redis + MySQL 双层防超卖
   ✅ 创建失败时自动归还 Redis 库存
   ✅ 不预留库存（空降订单也是实时扣减）

4. 审计边界
   ✅ 记录 created_by_admin_id（谁创建的）
   ✅ 订单类型标记为 airborne
   ✅ 可追溯、可审计
"""
```

### 5.2 注意事项

| 问题 | 说明 | 处理方式 |
| ------ | ------ | --------- |
| 用户不存在 | 指定用户已被删除或禁用 | 创建时实时校验用户状态 |
| 支付超时 | 空降订单也有支付时限 | 与普通订单相同，30分钟超时 |
| 重复创建 | 管理员可能重复下单 | 无防重，需要管理员确认 |
| 库存超卖 | 创建后库存被其他人买走 | 双层防超卖，创建时即扣库存 |
| 价格变更 | 创建后活动涨价/降价 | 空降订单价格以创建时为准 |
| 通知用户 | 用户需知道被指定了订单 | WebSocket 推送 + 订单列表可见 |

---

## 本章总结

✅ **已掌握：**

- 空降订单的业务场景：客服代下单、活动赠票、线下转线上、异常补单
- 数据库扩展：新增 order_type、designated_user_id 等字段
- 服务层实现：权限校验、用户验证、库存扣减、订单创建
- 支付集成：空降订单与普通订单共享支付接口，增加指定用户校验
- 前端实现：活动选择、用户搜索、表单提交
- 安全边界：5 层边界确保空降订单安全可控
