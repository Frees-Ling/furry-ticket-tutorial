# 第 31 章：空降订单 + 退票功能教程

> **难度等级：** 中级
> **前置知识：** 第 5 章 REST API、第 7 章 JWT/RBAC、第 16 章支付集成
> **学习目标：** 掌握管理员代客下单（空降订单）、用户申请退票、管理员审核退票的完整流程

---

## 目录

1. [系统设计概览](#1-系统设计概览)
2. [数据模型扩展](#2-数据模型扩展)
3. [管理员空降订单](#3-管理员空降订单)
4. [用户申请退票](#4-用户申请退票)
5. [管理员审核退票](#5-管理员审核退票)
6. [退款金额计算](#6-退款金额计算)
7. [原理深入](#7-原理深入)

---

## 1. 系统设计概览

### 1.1 功能需求

```text
空降订单（管理员代客下单）：
  管理员为用户创建订单并直接标记为已支付
  场景：线下收款后手动录入、客服代操作、活动方批量赠票

退票流程（用户申请 → 管理员审核）：
  用户提交退票申请（关联订单）
  管理员审核通过 → 执行退款（支付宝/余额）
  管理员审核拒绝 → 退票关闭
  场景：用户无法观演、活动改期、重复购票

退款类型：
  全额退款：活动取消或未举办
  部分退款：根据退票时间阶梯扣费（如开赛前7天退90%，3天退50%）
  不支持退款：特价票/赠品票
```

### 1.2 业务流程

```text
空降订单流程：
  管理员 → 选择活动+用户+数量 → 系统扣库存 → 创建订单(paid) → 生成票证

退票流程：
  用户提交申请 → 系统锁定票证 → 管理员审核
    ├── 审核通过 → 释放库存 → 执行退款 → 状态完成
    └── 审核拒绝 → 解锁票证 → 申请关闭
```

### 1.3 状态机扩展

```text
订单状态机（扩展后）：

          ┌──────────────────┐
          │  pending_payment │ ← 用户普通下单
          └────────┬─────────┘
                   │
     ┌─────────────┼──────────────┐
     ▼             ▼              ▼
  paid        cancelled       refunding
（已支付）    （已取消）      （退款中）
     │                          │
     │              ┌───────────┼───────────┐
     ▼              ▼           ▼           ▼
  refunded      refund_approved refund_rejected  refunding
（已退款）      （审核通过）    （审核拒绝）   （处理中）

新增状态：
  refunding        → 用户申请退票，待审核
  refund_approved  → 审核通过，等待支付宝退款
  refund_rejected  → 审核拒绝
```

---

## 2. 数据模型扩展

### 2.1 退票申请表（refund_requests）

```python
# app/models/refund.py

from datetime import datetime
from typing import Optional
from sqlalchemy import (
    String, Integer, Numeric, DateTime, Text, BigInteger, ForeignKey
)
from sqlalchemy.orm import Mapped, mapped_column
from app.models.base import Base


class RefundRequest(Base):
    """退票申请"""
    __tablename__ = "refund_requests"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
    order_id: Mapped[int] = mapped_column(BigInteger, ForeignKey("orders.id"), index=True)
    user_id: Mapped[int] = mapped_column(BigInteger, ForeignKey("users.id"), index=True)

    # 退款信息
    refund_amount: Mapped[Decimal] = mapped_column(
        Numeric(10, 2), comment="申请退款金额"
    )
    refund_reason: Mapped[str] = mapped_column(Text, comment="退票原因")
    refund_type: Mapped[str] = mapped_column(
        String(20), default="full",
        comment="退款类型: full=全额, partial=部分"
    )

    # 审核信息
    status: Mapped[str] = mapped_column(
        String(30), default="pending",
        comment="状态: pending=待审核, approved=审核通过, rejected=审核拒绝, completed=已完成"
    )
    reviewer_id: Mapped[Optional[int]] = mapped_column(
        BigInteger, ForeignKey("users.id"), nullable=True, comment="审核人ID"
    )
    review_remark: Mapped[Optional[str]] = mapped_column(
        Text, nullable=True, comment="审核备注"
    )
    reviewed_at: Mapped[Optional[datetime]] = mapped_column(
        DateTime, nullable=True, comment="审核时间"
    )

    # 支付宝关联
    trade_no: Mapped[Optional[str]] = mapped_column(
        String(64), nullable=True, comment="支付宝退款交易号"
    )

    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
    completed_at: Mapped[Optional[datetime]] = mapped_column(
        DateTime, nullable=True, comment="退款完成时间"
    )
```text

### 2.2 订单模型扩展

```sql
-- 在 orders 表新增退票关联字段
ALTER TABLE orders
    ADD COLUMN refund_status VARCHAR(20) DEFAULT NULL COMMENT '退款状态',
    ADD COLUMN refund_request_id BIGINT DEFAULT NULL COMMENT '当前退票申请ID',
    ADD COLUMN refund_amount DECIMAL(10,2) DEFAULT 0.00 COMMENT '已退款金额';
```

---

## 3. 管理员空降订单

### 3.1 管理员下单服务

```python
class AdminOrderService:
    """
    管理员订单服务
    处理空降订单、线下收款录入等管理员专属操作
    """

    def __init__(self, session: AsyncSession, redis: AsyncRedis):
        self.session = session
        self.redis = redis
        self.inventory_service = InventoryService(redis)
        self.order_service = OrderService(session, redis)

    async def create_admin_order(
        self,
        event_id: int,
        user_id: int,
        quantity: int,
        admin_id: int,
        remark: str | None = None,
        payment_method: str = "manual",
    ) -> Order:
        """
        管理员代客下单（空降订单）

        与普通下单的区别：
        1. 不经过行为检查（频率限制/IP限制）
        2. 不受每人限购约束（管理员特权）
        3. 创建后直接标记为 paid（跳过支付环节）
        4. 记录操作管理员 ID

        参数:
            event_id: 活动ID
            user_id: 目标用户ID（为空降用户创建）
            quantity: 数量
            admin_id: 操作管理员ID
            remark: 备注
            payment_method: 支付方式（manual=手工录入, gift=赠票, compensate=补偿）

        返回:
            Order: 已支付订单
        """
        # ===== 第1步：查询活动信息 =====
        event = await self.session.get(Event, event_id)
        if not event:
            raise ValueError("活动不存在")

        # ===== 第2步：Redis 库存扣减（仍然需要防超卖） =====
        result = await self.inventory_service.deduct_stock(
            event_id=event_id,
            user_id=user_id,
            quantity=quantity,
            max_per_user=0,  # 管理员不受限购限制
        )
        if result <= 0:
            raise ValueError("库存不足")

        try:
            # ===== 第3步：MySQL 乐观锁扣减 + 创建订单 =====
            # 读取活动当前版本
            event = await self.session.get(Event, event_id)

            # 校验库存（MySQL 层面）
            if event.sold_count + quantity > event.total_stock:
                raise ValueError("库存不足")

            # 生成订单号（带 admin 前缀标识）
            order_no = self._generate_admin_order_no(admin_id)

            order = Order(
                order_no=order_no,
                user_id=user_id,
                event_id=event_id,
                quantity=quantity,
                total_amount=event.price * quantity,
                final_amount=event.price * quantity,
                discount_amount=0,
                status="paid",  # 直接标记为已支付！(跳过支付环节)
                payment_method=payment_method,
                paid_at=datetime.utcnow(),
                remark=remark or f"管理员{admin_id}代客下单",
                admin_id=admin_id,  # 记录操作人
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
                await self.session.rollback()
                raise ValueError("库存扣减失败，请重试")

            # ===== 第4步：生成票证 =====
            for i in range(quantity):
                ticket = Ticket(
                    order_id=order.id,
                    event_id=event_id,
                    user_id=user_id,
                    ticket_no=f"{order_no}-{i+1}",
                    price=event.price,
                    status="active",
                )
                self.session.add(ticket)

            await self.session.flush()
            return order

        except Exception:
            # 任何错误时回滚 Redis 库存
            await self.inventory_service.restore_stock(event_id, quantity)
            raise

    def _generate_admin_order_no(self, admin_id: int) -> str:
        """生成管理员订单号（ADM + 时间戳 + 管理员ID）"""
        now = datetime.now()
        return f"ADM{now.strftime('%Y%m%d%H%M%S')}{admin_id:04d}{random.randint(1000, 9999)}"
```text

### 3.2 管理员路由

```python
# app/routers/admin.py

from app.middleware.auth import require_admin
from app.services.admin_order_service import AdminOrderService


@router.post("/orders", response_model=OrderResponse)
async def create_admin_order(
    data: AdminOrderCreate,
    session: AsyncSession = Depends(get_db),
    redis: Redis = Depends(get_redis),
    current_user: User = Depends(require_admin),
):
    """
    管理员代客下单

    POST /api/v1/admin/orders

    请求体:
    {
        "event_id": 1,
        "user_id": 42,
        "quantity": 2,
        "remark": "线下收款录入",
        "payment_method": "manual"
    }

    权限: admin
    """
    service = AdminOrderService(session, redis)

    try:
        order = await service.create_admin_order(
            event_id=data.event_id,
            user_id=data.user_id,
            quantity=data.quantity,
            admin_id=current_user.id,
            remark=data.remark,
            payment_method=data.payment_method or "manual",
        )

        await session.commit()
        return order

    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
```

---

## 4. 用户申请退票

### 4.1 退票规则配置

```python
# app/config.py — 退票规则

class RefundPolicy:
    """退票规则配置"""

    # 阶梯退票费率
    # key: 距活动开始时间（天）, value: 退款比例
    TIERED_RATES: list[tuple[int, float]] = [
        (30, 1.0),    # 30天以上：全额退款
        (14, 0.9),    # 14-30天：退90%
        (7, 0.7),     # 7-14天：退70%
        (3, 0.5),     # 3-7天：退50%
        (1, 0.0),     # 1-3天：不退
    ]

    # 不可退票类型
    NON_REFUNDABLE_TYPES = ["gift", "compensate", "special"]

    # 距离活动开始不足 N 天不可退
    MIN_DAYS_BEFORE_EVENT = 1

    @classmethod
    def calculate_refund_amount(
        cls, total_amount: Decimal, event_start_time: datetime
    ) -> tuple[Decimal, str]:
        """
        根据距活动开始时间计算退款金额

        返回:
            (refund_amount, refund_type)
            refund_amount: 退款金额
            refund_type: full=全额, partial=部分
        """
        now = datetime.utcnow()
        days_until_event = (event_start_time - now).days

        for min_days, rate in cls.TIERED_RATES:
            if days_until_event >= min_days:
                refund_amount = total_amount * Decimal(str(rate))
                refund_type = "full" if rate >= 1.0 else "partial"
                return refund_amount, refund_type

        # 不在任何区间内：不可退款
        return Decimal("0"), "none"
```text

### 4.2 退票申请服务

```python
class RefundService:
    """退票服务"""

    def __init__(self, session: AsyncSession, redis: AsyncRedis):
        self.session = session
        self.redis = redis
        self.payment_service = PaymentService(session, redis)

    async def apply_refund(
        self, order_id: int, user_id: int, reason: str
    ) -> RefundRequest:
        """
        用户申请退票

        流程：
        1. 校验订单属于用户
        2. 校验订单状态（必须是 paid）
        3. 校验是否已申请过退票
        4. 计算退款金额（阶梯费率）
        5. 创建退票申请
        6. 更新订单状态为 refunding
        """
        # ===== 第1步：查询订单 =====
        order = await self.session.get(Order, order_id)
        if not order or order.user_id != user_id:
            raise ValueError("订单不存在")

        # ===== 第2步：校验订单状态 =====
        if order.status != "paid":
            raise ValueError("当前订单状态不允许退票")

        # ===== 第3步：校验重复申请 =====
        existing = await self.session.execute(
            select(RefundRequest).where(
                RefundRequest.order_id == order_id,
                RefundRequest.status.in_(["pending", "approved"]),
            )
        )
        if existing.scalar_one_or_none():
            raise ValueError("已有退票申请正在处理中")

        # ===== 第4步：查询活动时间 =====
        event = await self.session.get(Event, order.event_id)
        if not event:
            raise ValueError("活动不存在")

        # ===== 第5步：计算退款金额 =====
        refund_amount, refund_type = RefundPolicy.calculate_refund_amount(
            order.total_amount, event.start_time
        )
        if refund_amount <= 0:
            raise ValueError("该订单不在可退票时间范围内")

        # ===== 第6步：创建退票申请 =====
        refund_request = RefundRequest(
            order_id=order_id,
            user_id=user_id,
            refund_amount=refund_amount,
            refund_reason=reason,
            refund_type=refund_type,
            status="pending",
        )
        self.session.add(refund_request)
        await self.session.flush()

        # ===== 第7步：更新订单状态 =====
        order.status = "refunding"
        order.refund_request_id = refund_request.id

        return refund_request
```

---

## 5. 管理员审核退票

### 5.1 审核逻辑

```python
async def review_refund(
    self,
    refund_request_id: int,
    admin_id: int,
    action: str,        # "approve" 或 "reject"
    remark: str | None = None,
) -> RefundRequest:
    """
    管理员审核退票

    参数:
        refund_request_id: 退票申请ID
        admin_id: 审核管理员ID
        action: approve=通过, reject=拒绝
        remark: 审核备注

    流程:
        approve:
        1. 扣减 Redis 幂等键（防止重复处理）
        2. 更新订单状态 → refund_approved
        3. 调用支付宝退款 API
        4. 释放 Redis 库存
        5. 更新退票状态 → completed
        6. 更新订单状态 → refunded

        reject:
        1. 更新退票状态 → rejected
        2. 更新订单状态 → paid（恢复为已支付状态）
    """
    refund_request = await self.session.get(RefundRequest, refund_request_id)
    if not refund_request or refund_request.status != "pending":
        raise ValueError("退票申请不存在或已处理")

    if action == "reject":
        # ===== 拒绝退票 =====
        refund_request.status = "rejected"
        refund_request.reviewer_id = admin_id
        refund_request.review_remark = remark
        refund_request.reviewed_at = datetime.utcnow()

        # 恢复订单状态
        order = await self.session.get(Order, refund_request.order_id)
        if order:
            order.status = "paid"
            order.refund_request_id = None

        return refund_request

    elif action == "approve":
        # ===== 审核通过 =====
        order = await self.session.get(Order, refund_request.order_id)
        if not order:
            raise ValueError("关联订单不存在")

        # 更新退票申请状态
        refund_request.status = "approved"
        refund_request.reviewer_id = admin_id
        refund_request.review_remark = remark
        refund_request.reviewed_at = datetime.utcnow()

        # 更新订单状态
        order.status = "refund_approved"

        # 执行支付宝退款
        try:
            if order.payment_method == "alipay" and order.trade_no:
                refund_result = await self.payment_service.refund(
                    order_id=order.id,
                    amount=float(refund_request.refund_amount),
                    reason=refund_request.refund_reason,
                )
                refund_request.trade_no = refund_result.get("trade_no")

            # 释放库存
            await self._restore_ticket_inventory(order)

            # 标记退票完成
            refund_request.status = "completed"
            refund_request.completed_at = datetime.utcnow()
            order.status = "refunded"
            order.refund_amount = refund_request.refund_amount

        except Exception as e:
            # 退款失败，回滚状态
            refund_request.status = "pending"
            order.status = "refunding"
            raise ValueError(f"退款执行失败: {e}")

        return refund_request

    else:
        raise ValueError(f"未知审核操作: {action}")


async def _restore_ticket_inventory(self, order: Order):
    """
    退票后释放库存

    逻辑：
    - 将 Redis 库存恢复（INCRBY）
    - MySQL 活动库存回退
    - 将票证状态标记为 refunded
    """
    # Redis 库存恢复
    await self.inventory_service.restore_stock(order.event_id, order.quantity)

    # MySQL 库存回退（乐观锁）
    event = await self.session.get(Event, order.event_id)
    await self.session.execute(
        update(Event)
        .where(
            Event.id == order.event_id,
            Event.version == event.version,
        )
        .values(
            sold_count=Event.sold_count - order.quantity,
            version=Event.version + 1,
        )
    )

    # 标记票证
    await self.session.execute(
        update(Ticket)
        .where(Ticket.order_id == order.id)
        .values(status="refunded")
    )
```text

### 5.2 退票审核路由

```python
# app/routers/admin.py


@router.post("/refund-requests", response_model=RefundRequestResponse)
async def apply_refund(
    data: RefundApplyRequest,
    session: AsyncSession = Depends(get_db),
    redis: Redis = Depends(get_redis),
    current_user: User = Depends(get_current_user),
):
    """
    用户申请退票

    POST /api/v1/refund-requests

    请求体:
    {
        "order_id": 123,
        "reason": "因个人原因无法观演"
    }
    """
    service = RefundService(session, redis)
    try:
        refund_request = await service.apply_refund(
            order_id=data.order_id,
            user_id=current_user.id,
            reason=data.reason,
        )
        await session.commit()
        return refund_request
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.put("/refund-requests/{request_id}/review")
async def review_refund(
    request_id: int,
    data: RefundReviewRequest,
    session: AsyncSession = Depends(get_db),
    redis: Redis = Depends(get_redis),
    current_user: User = Depends(require_admin),
):
    """
    管理员审核退票

    PUT /api/v1/admin/refund-requests/{id}/review

    请求体:
    {
        "action": "approve",      # approve 或 reject
        "remark": "审核通过"
    }
    """
    service = RefundService(session, redis)
    try:
        refund_request = await service.review_refund(
            refund_request_id=request_id,
            admin_id=current_user.id,
            action=data.action,
            remark=data.remark,
        )
        await session.commit()
        return refund_request
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
```

---

## 6. 退款金额计算

### 6.1 阶梯退票费率

```python
# app/services/refund_service.py — 退款计算完整实现

from decimal import Decimal, ROUND_HALF_UP


def calculate_refund(
    total_amount: Decimal,
    event_start_time: datetime,
    order_type: str = "normal",
    policy: dict | None = None,
) -> RefundCalculation:
    """
    计算退款金额

    支持多种策略：
    - 阶梯退票（按距活动天数递减）
    - 固定比例退票（统一退 X%）
    - 不可退票（赠票、特价票）

    参数:
        total_amount: 订单总金额
        event_start_time: 活动开始时间
        order_type: normal/gift/compensate/special
        policy: 自定义退票策略

    返回:
        RefundCalculation: 退款计算结果
    """

    # 不可退票类型
    non_refundable_types = {"gift", "compensate", "special"}

    if order_type in non_refundable_types:
        return RefundCalculation(
            refund_amount=Decimal("0"),
            refund_type="none",
            policy_name="不可退票",
            message="该类型订单不支持退款",
        )

    # 使用自定义策略或默认策略
    if policy:
        tiers = policy.get("tiers", [])
    else:
        tiers = [
            (30, Decimal("1.00")),
            (14, Decimal("0.90")),
            (7, Decimal("0.70")),
            (3, Decimal("0.50")),
            (1, Decimal("0.00")),
        ]

    now = datetime.utcnow()
    days_until_event = (event_start_time - now).days

    for min_days, rate in tiers:
        if days_until_event >= min_days:
            refund_amount = (total_amount * rate).quantize(
                Decimal("0.01"), rounding=ROUND_HALF_UP
            )

            # 最低退票金额限制（如手续费不低于 5 元）
            min_refund = Decimal("5.00")
            if refund_amount > 0 and refund_amount < min_refund:
                refund_amount = min_refund

            return RefundCalculation(
                refund_amount=refund_amount,
                refund_type="full" if rate >= 1 else "partial",
                policy_name=f"距活动{min_days}天以上({int(rate*100)}%)",
                message=f"可退款 {refund_amount} 元（{int(rate*100)}%）",
            )

    return RefundCalculation(
        refund_amount=Decimal("0"),
        refund_type="none",
        policy_name="超期不可退",
        message="距活动开始不足1天，不支持退款",
    )
```text

---

## 7. 原理深入

### 7.1 退款幂等性设计

```

问题：
  支付宝退款接口调用可能因网络超时而结果不确定。
  重试时若已退款成功，再次调用会报错。

解决方案 - 退款幂等键（out_request_no）：

  out_request_no = f"REFUND_{order_id}_{timestamp}"

  每次退款请求使用唯一 out_request_no。
  支付宝保证同一 out_request_no 只处理一次。
  应用层使用 Redis SETNX 防止重复提交：

  key = f"refund:processing:{order_id}"
  SETNX → 成功：开始处理退款
  SETNX → 失败：已有退款在处理中

```text

### 7.2 库存恢复的原子性

```

退票审核通过后需要原子地完成：

  1. 释放 Redis 库存（INCRBY）
  2. 回退 MySQL 活动库存（乐观锁）
  3. 标记票证为 refunded
  4. 更新订单状态

如果 1 成功但 2 失败 → 库存不一致
如果 2 成功但 3 失败 → 票证仍为 active

解决方案：使用 Python 的 try/except + 日志补偿
  真正的原子性需要分布式事务（如 Seata/TCC），
  本系统采用最终一致性方案：

  1. 先扣减 Redis（可回滚）
  2. 再更新 MySQL
  3. 再标记票证
  4. 任一失败 → 启动补偿任务（ARQ Worker 异步恢复）

  补偿任务（补偿队列）：
  refund_compensate_queue → 检查不一致 → 自动修复

```text

### 7.3 退票并发控制

```

同一订单同时被多个管理员审核的风险：

  审核退票操作 =
    SELECT refund_request WHERE id = :id AND status = 'pending'
    + UPDATE refund_request SET status = 'approved'
    + UPDATE order SET status = 'refunded'
    + 释放库存
    + 调用支付宝退款

  如果两个管理员同时审核同一申请：
  两人都 SELECT 到 status = 'pending'
  两人都会执行退款操作 → 重复退款！

  解决方案：

  1. MySQL 行级锁：SELECT ... FOR UPDATE
  2. 乐观锁：UPDATE ... WHERE status = 'pending'（受影响行数为0则冲突）
  3. Redis 分布式锁：SETNX refund:lock:{request_id}

  推荐方法2（最简单）：
  result = await session.execute(
      update(RefundRequest)
      .where(
          RefundRequest.id == request_id,
          RefundRequest.status == "pending",  -- 关键条件！
      )
      .values(status="approved", ...)
  )
  if result.rowcount == 0:
      raise ValueError("退票申请已被处理")
  -- 只有一行被更新，后续操作不会重复执行

```text

### 7.4 退款超时处理

```

场景：
  调用支付宝退款接口后，支付宝返回"处理中"（code=20000），
  退款最终结果不确定。

处理方案 - 退款轮询：

  async def poll_refund_result(refund_request_id: int, max_retries: int = 10):
      """轮询支付宝退款结果"""
      for i in range(max_retries):
          result = alipay.api_alipay_trade_fastpay_refund_query(
              trade_no=order.trade_no,
              out_request_no=refund_request.out_request_no,
          )
          if result.get("refund_status") == "REFUND_SUCCESS":
              # 退款成功
              return True
          elif result.get("code") == "40004":
              # 退款不存在（尚未处理）
              await asyncio.sleep(2 ** i)  # 指数退避：1s, 2s, 4s, 8s...
          else:
              # 其他错误
              raise Exception(f"退款查询异常: {result}")

      # 超时，需要人工介入
      raise Exception("退款超时，请人工核查")

```text

---

## 项目代码参考

| 文件 | 说明 |
| ------ | ------ |
| `app/models/refund.py` | 退票申请模型 |
| `app/services/refund_service.py` | 退票业务逻辑 |
| `app/services/admin_order_service.py` | 管理员下单服务 |
| `app/routers/admin.py` | 管理路由（含退款审核） |
| `app/schemas/refund.py` | 退票相关 Schema |
| `app/config.py` | RefundPolicy 退票规则配置 |

## 本章总结

✅ **已掌握：**

- 管理员空降订单的业务流程和实现
- 用户退票申请的完整流程（阶梯费率计算）
- 管理员审核退票（通过/拒绝）的实现
- 退款金额计算策略
- 原理深入：退款幂等性、库存恢复原子性、并发控制、超时处理

✅ **项目里程碑：**

- [ ] 管理员可代客下单（跳过支付，直接生成已支付订单+票证）
- [ ] 用户可查看自己订单并申请退票
- [ ] 管理员可审核退票（通过/拒绝）
- [ ] 审核通过后自动退款 + 释放库存
- [ ] 退款金额按阶梯费率正确计算
- [ ] 退票并发控制可靠（不重复退款）
