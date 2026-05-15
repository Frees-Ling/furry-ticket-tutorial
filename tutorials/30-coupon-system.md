# 第 30 章：优惠券系统完整教程

> **难度等级：** 中级
> **前置知识：** 第 4 章 FastAPI、第 6 章 Redis、第 7 章 JWT 认证
> **学习目标：** 掌握优惠券系统从设计到实现的全过程，包括创建、领取、使用、折扣计算

---

## 目录

1. [系统设计概览](#1-系统设计概览)
2. [数据模型设计](#2-数据模型设计)
3. [优惠券类型与折扣算法](#3-优惠券类型与折扣算法)
4. [优惠券管理（管理员）](#4-优惠券管理管理员)
5. [用户领取优惠券](#5-用户领取优惠券)
6. [下单使用优惠券](#6-下单使用优惠券)
7. [原理深入](#7-原理深入)

---

## 1. 系统设计概览

### 1.1 功能需求

```text
管理员：
  - 创建优惠券活动（满减券、折扣券、立减券）
  - 设置发放数量、每人限领、有效期
  - 查看优惠券使用统计

用户：
  - 浏览可领取优惠券
  - 领取优惠券到个人账户
  - 下单时选择可用优惠券
  - 查看已领取和已使用的优惠券

系统：
  - 核验优惠券有效性（未过期、未使用、满足条件）
  - 计算折扣后金额
  - 防止超发（同防超卖机制）
```

### 1.2 系统架构

```text
优惠券系统与订单系统的交互：

┌──────────────┐     ┌────────────────┐     ┌──────────────┐
│  优惠券管理   │     │   优惠券服务     │     │   订单服务    │
│  (管理员)     │ ──→ │  CouponService  │ ←── │  OrderService │
└──────────────┘     └───────┬────────┘     └──────────────┘
                             │
                    ┌────────┴────────┐
                    │    Redis 缓存    │
                    │  (防超发+计数)  │
                    └─────────────────┘
```

---

## 2. 数据模型设计

### 2.1 优惠券活动表（coupon_campaigns）

```python
# app/models/coupon.py

from datetime import datetime
from decimal import Decimal
from typing import Optional
from sqlalchemy import String, Integer, Numeric, DateTime, Text, Boolean, BigInteger
from sqlalchemy.orm import Mapped, mapped_column
from app.models.base import Base


class CouponCampaign(Base):
    """优惠券活动（定义优惠券的发放规则）"""
    __tablename__ = "coupon_campaigns"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
    title: Mapped[str] = mapped_column(String(100), comment="优惠券标题")
    description: Mapped[Optional[str]] = mapped_column(Text, comment="优惠券描述")

    # 优惠券类型
    coupon_type: Mapped[str] = mapped_column(
        String(20), default="fixed",
        comment="优惠券类型: fixed=立减, discount=折扣,满减=满减"
    )

    # 优惠金额/折扣配置
    discount_value: Mapped[Decimal] = mapped_column(
        Numeric(10, 2), comment="优惠值: 立减/满减时为金额, 折扣时为折扣率(如0.85)"
    )
    min_amount: Mapped[Optional[Decimal]] = mapped_column(
        Numeric(10, 2), default=None, comment="最低使用金额(满减门槛)"
    )
    max_discount: Mapped[Optional[Decimal]] = mapped_column(
        Numeric(10, 2), default=None, comment="最大优惠金额(折扣券封顶)"
    )

    # 发放控制
    total_quantity: Mapped[int] = mapped_column(Integer, comment="总发放数量")
    per_user_limit: Mapped[int] = mapped_column(Integer, default=1, comment="每人限领")
    remain_quantity: Mapped[int] = mapped_column(Integer, comment="剩余数量")

    # 有效期
    valid_from: Mapped[datetime] = mapped_column(DateTime, comment="开始时间")
    valid_to: Mapped[datetime] = mapped_column(DateTime, comment="结束时间")

    # 状态
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

    # 乐观锁版本
    version: Mapped[int] = mapped_column(Integer, default=1)
```text

### 2.2 用户优惠券表（user_coupons）

```python
class UserCoupon(Base):
    """用户已领取的优惠券"""
    __tablename__ = "user_coupons"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
    user_id: Mapped[int] = mapped_column(BigInteger, ForeignKey("users.id"), index=True)
    campaign_id: Mapped[int] = mapped_column(
        BigInteger, ForeignKey("coupon_campaigns.id")
    )

    # 优惠券快照（领券时保存的优惠信息，后续不随活动变化）
    coupon_type: Mapped[str] = mapped_column(String(20))
    discount_value: Mapped[Decimal] = mapped_column(Numeric(10, 2))
    min_amount: Mapped[Optional[Decimal]] = mapped_column(Numeric(10, 2), nullable=True)
    max_discount: Mapped[Optional[Decimal]] = mapped_column(Numeric(10, 2), nullable=True)

    # 使用状态
    status: Mapped[str] = mapped_column(
        String(20), default="unused",
        comment="状态: unused=未使用, used=已使用, expired=已过期"
    )
    used_at: Mapped[Optional[datetime]] = mapped_column(DateTime, nullable=True)
    order_id: Mapped[Optional[int]] = mapped_column(
        BigInteger, ForeignKey("orders.id"), nullable=True
    )

    # 有效期（从活动复制）
    valid_from: Mapped[datetime] = mapped_column(DateTime)
    valid_to: Mapped[datetime] = mapped_column(DateTime)

    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
```

### 2.3 关联订单

```sql
-- 在 orders 表中增加 coupon 相关字段
ALTER TABLE orders
    ADD COLUMN coupon_id BIGINT NULL,
    ADD COLUMN discount_amount DECIMAL(10,2) DEFAULT 0.00,
    ADD COLUMN final_amount DECIMAL(10,2);
```text

---

## 3. 优惠券类型与折扣算法

### 3.1 三种优惠券类型

```python
class CouponType:
    """优惠券类型常量"""
    FIXED = "fixed"        # 立减券：直接减固定金额
    DISCOUNT = "discount"  # 折扣券：按折扣率计算
    FULL_REDUCTION = "满减"  # 满减券：满足最低金额后减固定金额


def calculate_discount(
    coupon_type: str,
    discount_value: Decimal,
    min_amount: Decimal | None,
    max_discount: Decimal | None,
    order_amount: Decimal,
) -> Decimal:
    """
    计算优惠金额

    参数:
        coupon_type: 优惠券类型
        discount_value: 优惠值（立减/满减=金额, 折扣=折扣率如0.85）
        min_amount: 最低使用金额
        max_discount: 最大优惠金额（折扣券封顶）
        order_amount: 订单原金额

    返回:
        Decimal: 优惠金额（>= 0）

    异常:
        ValueError: 不满足使用条件
    """

    # 统一校验：最低金额门槛
    if min_amount and order_amount < min_amount:
        raise ValueError(f"订单金额不足，最低需 {min_amount} 元")

    if coupon_type == CouponType.FIXED:
        # 立减券：直接减固定金额（不超过订单金额）
        discount = min(discount_value, order_amount)

    elif coupon_type == CouponType.DISCOUNT:
        # 折扣券：按折扣率计算
        # discount_value = 0.85 表示打85折
        raw_discount = order_amount * (Decimal("1") - discount_value)
        # 有封顶限制
        if max_discount:
            discount = min(raw_discount, max_discount)
        else:
            discount = raw_discount

    elif coupon_type == CouponType.FULL_REDUCTION:
        # 满减券：满 X 元减 Y 元
        discount = min(discount_value, order_amount)

    else:
        raise ValueError(f"未知优惠券类型: {coupon_type}")

    # 优惠金额不能为负，不能大于订单金额
    discount = max(Decimal("0"), discount)
    return discount
```

### 3.2 折扣计算示例

```python
# 示例：立减券
# 优惠券：立减 50 元
# 订单金额：199 元
# → 优惠金额：50 元
# → 实付：149 元
assert calculate_discount("fixed", Decimal("50"), None, None, Decimal("199")) == Decimal("50")

# 示例：折扣券
# 优惠券：85折，封顶 30 元
# 订单金额：500 元
# → 原始折扣：500 * 0.15 = 75 元
# → 封顶后：30 元
# → 实付：470 元
assert calculate_discount("discount", Decimal("0.85"), None, Decimal("30"), Decimal("500")) == Decimal("30")

# 示例：满减券
# 优惠券：满 200 减 50
# 订单金额：150 元（不满足条件）
# → 抛出 ValueError
try:
    calculate_discount("满减", Decimal("50"), Decimal("200"), None, Decimal("150"))
except ValueError as e:
    print(str(e))  # "订单金额不足，最低需 200 元"
```text

---

## 4. 优惠券管理（管理员）

### 4.1 创建优惠券活动

```python
class CouponService:
    """优惠券服务"""

    def __init__(self, session: AsyncSession, redis: AsyncRedis):
        self.session = session
        self.redis = redis

    async def create_campaign(self, data: CouponCampaignCreate, admin_id: int) -> CouponCampaign:
        """
        创建优惠券活动

        安全要点：
        - 验证时间范围：valid_to > valid_from
        - 验证数量：total_quantity > 0
        - 验证优惠值：立减>0, 折扣在0-1之间

        创建后自动初始化 Redis 剩余数量缓存，用于防超发。
        """
        if data.valid_to <= data.valid_from:
            raise ValueError("结束时间必须大于开始时间")
        if data.total_quantity <= 0:
            raise ValueError("发放数量必须大于0")

        if data.coupon_type == "discount":
            if not (Decimal("0") < data.discount_value < Decimal("1")):
                raise ValueError("折扣券的折扣率必须在0到1之间")
        elif data.coupon_type in ("fixed", "满减"):
            if data.discount_value <= 0:
                raise ValueError("优惠金额必须大于0")

        campaign = CouponCampaign(
            title=data.title,
            description=data.description,
            coupon_type=data.coupon_type,
            discount_value=data.discount_value,
            min_amount=data.min_amount,
            max_discount=data.max_discount,
            total_quantity=data.total_quantity,
            per_user_limit=data.per_user_limit,
            remain_quantity=data.total_quantity,
            valid_from=data.valid_from,
            valid_to=data.valid_to,
        )
        self.session.add(campaign)
        await self.session.flush()

        # 初始化 Redis 缓存
        await self.redis.set(
            f"coupon:remain:{campaign.id}", data.total_quantity
        )

        return campaign
```

### 4.2 Pydantic Schema

```python
# app/schemas/coupon.py

class CouponCampaignCreate(BaseModel):
    """创建优惠券活动"""
    title: str = Field(..., min_length=1, max_length=100)
    description: str | None = None
    coupon_type: str = Field(..., pattern=r"^(fixed|discount|满减)$")
    discount_value: Decimal = Field(..., gt=0)
    min_amount: Decimal | None = Field(None, ge=0)
    max_discount: Decimal | None = Field(None, ge=0)
    total_quantity: int = Field(..., gt=0)
    per_user_limit: int = Field(1, ge=1)
    valid_from: datetime
    valid_to: datetime


class CouponCampaignResponse(BaseModel):
    model_config = {"from_attributes": True}
    id: int
    title: str
    coupon_type: str
    discount_value: Decimal
    min_amount: Decimal | None
    max_discount: Decimal | None
    total_quantity: int
    remain_quantity: int
    per_user_limit: int
    is_active: bool
    valid_from: datetime
    valid_to: datetime


class UserCouponResponse(BaseModel):
    model_config = {"from_attributes": True}
    id: int
    campaign_id: int
    coupon_type: str
    discount_value: Decimal
    min_amount: Decimal | None
    status: str
    valid_from: datetime
    valid_to: datetime
```text

---

## 5. 用户领取优惠券

### 5.1 领取逻辑（防超发）

```python
async def claim_coupon(
    self, campaign_id: int, user_id: int
) -> UserCoupon:
    """
    用户领取优惠券

    防超发机制（与防超卖类似）：
    1. Redis DECR 原子扣减剩余数量（快速路径）
    2. 数据库乐观锁校验（慢速路径兜底）
    3. 每人限领检查（Redis + DB 双重校验）
    """
    # ===== 第一步：校验活动有效性 =====
    campaign = await self.session.get(CouponCampaign, campaign_id)
    if not campaign or not campaign.is_active:
        raise ValueError("优惠券活动不存在或已下架")

    now = datetime.utcnow()
    if now < campaign.valid_from or now > campaign.valid_to:
        raise ValueError("优惠券不在领取时间范围内")

    # ===== 第二步：每人限领检查 =====
    user_key = f"coupon:user:{campaign_id}:{user_id}"
    user_count = await self.redis.get(user_key)
    if user_count and int(user_count) >= campaign.per_user_limit:
        raise ValueError(f"已达到每人限领 {campaign.per_user_limit} 张")

    # ===== 第三步：Redis 原子扣减（防超发快速路径） =====
    remain_key = f"coupon:remain:{campaign_id}"
    remain = await self.redis.decr(remain_key)
    if remain < 0:
        # 超发了！恢复库存
        await self.redis.incr(remain_key)
        raise ValueError("优惠券已被领完")

    try:
        # ===== 第四步：数据库层面扣减（乐观锁） =====
        result = await self.session.execute(
            update(CouponCampaign)
            .where(
                CouponCampaign.id == campaign_id,
                CouponCampaign.remain_quantity > 0,
                CouponCampaign.version == campaign.version,
            )
            .values(
                remain_quantity=CouponCampaign.remain_quantity - 1,
                version=CouponCampaign.version + 1,
            )
        )

        if result.rowcount == 0:
            raise ValueError("优惠券已被领完")

        # ===== 第五步：创建用户优惠券记录 =====
        user_coupon = UserCoupon(
            user_id=user_id,
            campaign_id=campaign_id,
            coupon_type=campaign.coupon_type,
            discount_value=campaign.discount_value,
            min_amount=campaign.min_amount,
            max_discount=campaign.max_discount,
            status="unused",
            valid_from=campaign.valid_from,
            valid_to=campaign.valid_to,
        )
        self.session.add(user_coupon)

        # 更新 Redis 限领计数
        await self.redis.incr(user_key)
        await self.redis.expire(user_key, 86400 * 30)  # 30天过期

        await self.session.flush()
        return user_coupon

    except Exception:
        # 数据库失败，回滚 Redis 计数
        await self.redis.incr(remain_key)
        raise
```

### 5.2 领取流程图

```text
用户点击"领取优惠券"
    │
    ▼
检查优惠券活动是否有效（时间范围/是否下架）
    │
    ▼
检查每人限领（Redis 计数）
    │  ├─ 已达上限 → 返回错误
    ▼
Redis DECR 原子扣减剩余数量
    │  ├─ 剩余 < 0 → 回滚 INCR，返回"已领完"
    ▼
MySQL 乐观锁更新 remain_quantity
    │  ├─ 更新 0 行 → 回滚 Redis，返回"已领完"
    ▼
创建 UserCoupon 记录
    │
    ▼
更新 Redis 每人限领计数
    │
    ▼
领取成功 ✓
```

---

## 6. 下单使用优惠券

### 6.1 校验优惠券有效性

```python
async def validate_coupon(
    self, user_coupon_id: int, user_id: int, order_amount: Decimal
) -> tuple[UserCoupon, Decimal]:
    """
    校验优惠券是否可用

    返回:
        (user_coupon, discount_amount): 优惠券对象和优惠金额

    校验项目：
    - 优惠券属于当前用户
    - 优惠券状态为 unused
    - 在有效期内
    - 满足最低金额条件
    """
    user_coupon = await self.session.get(UserCoupon, user_coupon_id)

    if not user_coupon or user_coupon.user_id != user_id:
        raise ValueError("优惠券不存在")

    if user_coupon.status != "unused":
        raise ValueError("优惠券已使用或已过期")

    now = datetime.utcnow()
    if now < user_coupon.valid_from or now > user_coupon.valid_to:
        raise ValueError("优惠券已过期")

    # 计算优惠金额
    discount = calculate_discount(
        coupon_type=user_coupon.coupon_type,
        discount_value=user_coupon.discount_value,
        min_amount=user_coupon.min_amount,
        max_discount=user_coupon.max_discount,
        order_amount=order_amount,
    )

    return user_coupon, discount
```text

### 6.2 下单时使用优惠券

```python
async def create_order_with_coupon(
    self,
    event_id: int,
    user_id: int,
    quantity: int,
    coupon_id: int | None = None,
) -> Order:
    """
    创建订单（支持优惠券）

    完整流程：
    1. 防超卖库存扣减（Redis Lua + MySQL 乐观锁）
    2. 计算原价
    3. 如果有优惠券，校验并使用
    4. 计算最终金额
    5. 创建订单
    6. 标记优惠券为已使用
    """
    # ===== 第1步：防超卖扣减 =====
    inventory_service = InventoryService(self.redis)
    result = await inventory_service.deduct_stock(event_id, user_id, quantity, max_per_user=4)
    if result <= 0:
        raise HTTPException(status_code=409, detail="库存不足或超过限购")

    try:
        event = await self.session.get(Event, event_id)
        original_amount = event.price * quantity

        # ===== 第2步：优惠券处理 =====
        discount_amount = Decimal("0")
        if coupon_id:
            user_coupon, discount_amount = await self.validate_coupon(
                coupon_id, user_id, original_amount
            )
            # 标记优惠券为使用中（暂态，防止并发重复使用）
            user_coupon.status = "used"
            user_coupon.used_at = datetime.utcnow()

        final_amount = original_amount - discount_amount

        # ===== 第3步：创建订单 =====
        order = Order(
            order_no=self._generate_order_no(),
            user_id=user_id,
            event_id=event_id,
            quantity=quantity,
            total_amount=original_amount,
            discount_amount=discount_amount,
            final_amount=final_amount,
            status="pending_payment",
            coupon_id=coupon_id,
        )
        self.session.add(order)
        await self.session.flush()

        # ===== 第4步：关联优惠券 =====
        if coupon_id:
            user_coupon.order_id = order.id

        return order

    except Exception:
        # 回滚 Redis 库存
        await inventory_service.restore_stock(event_id, quantity)
        raise
```

### 6.3 优惠券使用后的数量同步

```python
# 异步任务：每天凌晨统计优惠券使用情况
async def sync_coupon_stats(self):
    """同步优惠券统计数据（T+1 更新）"""
    campaigns = await self.session.execute(
        select(CouponCampaign).where(CouponCampaign.is_active == True)
    )
    for campaign in campaigns.scalars():
        # 统计已领取数量
        result = await self.session.execute(
            select(func.count()).select_from(UserCoupon).where(
                UserCoupon.campaign_id == campaign.id
            )
        )
        claimed_count = result.scalar() or 0

        # 统计已使用数量
        result = await self.session.execute(
            select(func.count()).select_from(UserCoupon).where(
                UserCoupon.campaign_id == campaign.id,
                UserCoupon.status == "used",
            )
        )
        used_count = result.scalar() or 0
```text

---

## 7. 原理深入

### 7.1 防超发数学证明

```

问题：总发放 N 张，可能超发的关键路径

定义：
  R = Redis 中剩余数量计数器
  D = 数据库（CouponCampaign.remain_quantity）
  预期：R = D = N - 已领取数

超发风险：

  1. Redis 未初始化 → 剩余为 None → DECR 返回 -1
     → 防御：创建时 SET，领取时检查 DECR 返回值 < 0
  2. Redis 和 DB 双写不一致
     → 防御：Redis 快速拒绝，DB 乐观锁兜底
  3. 并发领取时两边的竞态条件
     → 防御：Redis DECR 是原子的，MySQL UPDATE WHERE remain > 0 是原子的

定理：本优惠券领取系统保证不超发。

证明：
  假设共发放 N 张，有 M 个并发领取请求 (M > N)。

  第1层（Redis DECR）：
    对于每个请求，执行 DECR(remain_key)。
    由于 DECR 是原子的，DECR 后的返回值严格递减。
    第 N+1 个请求的 DECR 返回值为 -1（假设从 N 开始减）。
    由于代码检查 remain < 0 时回滚并报错，
    最多 N 个请求通过第1层。

  第2层（MySQL 乐观锁）：
    通过第1层的 ≤ N 个请求到达 MySQL。
    UPDATE coupon_campaigns
    SET remain_quantity = remain_quantity - 1
    WHERE id = :id AND remain_quantity > 0 AND version = :ver

    即使第1层有误差（如 Redis 数据丢失），
    MySQL 的 WHERE remain_quantity > 0 保证不会扣减为负数。
    最多 N 行被更新。

  结论：两层防线各自独立保证不超发。
  Q.E.D.

```text

### 7.2 优惠券状态机

```

用户领取 → unused（未使用）
                │
          用户下单使用
                │
           ┌────┴────┐
           ▼         ▼
        used       expired
      （已使用）    （已过期）

状态转换：
  unused → used     下单支付流程中标记
  unused → expired  系统定期任务（或查询时实时判断）

过期策略：

- 数据库字段 valid_to 与当前时间对比（查询时实时判断）
- 后台定时任务批量标记过期
- 用户查询时同步更新过期状态

```text

### 7.3 Redis 防超发键设计

```

# 剩余数量（用于防超发快速路径）

coupon:remain:{campaign_id}  →  int（DECR 原子操作）

# 每人限领计数（防止重复领取）

coupon:user:{campaign_id}:{user_id}  →  int（已领取数量）

# 优惠券活动信息缓存（减少 DB 查询）

coupon:campaign:{campaign_id}  →  JSON（活动详情）

```text

### 7.4 优惠券叠加与互斥

```

本期不支持多张优惠券叠加使用，但可在订单上扩展：

互斥规则：

- 同一订单只能使用一张优惠券
- 优惠券不能与活动特价同时使用
- 退款时优惠券不退还（视业务规则而定）

扩展方向的提示：
  如需多券叠加，可定义优惠券的「优惠组」概念：
  组内互斥、组间可叠加。
  折扣计算顺序：先折扣券 → 再满减券 → 再立减券（折扣基数为折扣后金额）。

```text

### 7.5 性能优化

```

1. 热门优惠券的缓存预热：
   活动开始前将 coupon:remain 和 coupon:campaign 预热到 Redis

2. 限领计数合并写入：
   Redis INCR + 异步批量回写到 DB（容忍少量不一致）

3. 过期优惠券批量清理：
   UPDATE user_coupons SET status = 'expired'
   WHERE status = 'unused' AND valid_to < NOW()
   LIMIT 1000;

4. 用户优惠券列表分页：
   SELECT * FROM user_coupons
   WHERE user_id = :uid
   ORDER BY
     CASE WHEN status = 'unused' THEN 0 ELSE 1 END,
     valid_to ASC
   LIMIT 20 OFFSET 0;
   将未使用且即将过期的排在最前。

```text

---

## 项目代码参考

| 文件 | 说明 |
| ------ | ------ |
| `app/models/coupon.py` | CouponCampaign + UserCoupon 模型定义 |
| `app/schemas/coupon.py` | 优惠券相关 Pydantic schema |
| `app/services/coupon_service.py` | 优惠券业务逻辑（创建/领取/校验/使用） |
| `app/routers/coupon.py` | 优惠券 API 路由 |
| `app/routers/admin.py` | 管理员优惠券管理路由 |

## 本章总结

✅ **已掌握：**

- 优惠券系统数据库设计（活动表 + 用户领取表 + 订单关联）
- 三种优惠券类型及折扣算法实现
- 双层防超发机制（Redis DECR + MySQL 乐观锁）
- 优惠券领取、校验、使用的完整业务流程
- 原理深入：防超发证明、状态机、性能优化

✅ **项目里程碑：**

- [ ] 管理员可创建优惠券活动
- [ ] 用户可领取优惠券（不超发）
- [ ] 下单时可选择并应用优惠券
- [ ] 折扣金额计算正确
- [ ] 防超发通过并发测试
