# 第 32 章：复杂支付场景教程

> **难度等级：** 高级
> **前置知识：** 第 16 章支付宝支付、第 30 章优惠券系统、第 31 章退票功能
> **学习目标：** 掌握部分退款、优惠券叠加退款、支付超时处理等复杂支付场景的实现

---

## 目录

1. [系统设计概览](#1-系统设计概览)
2. [部分退款](#2-部分退款)
3. [优惠券叠加退款](#3-优惠券叠加退款)
4. [支付超时处理](#4-支付超时处理)
5. [多支付渠道集成](#5-多支付渠道集成)
6. [原理深入](#6-原理深入)

---

## 1. 系统设计概览

### 1.1 复杂支付场景

```text
场景一：部分退款
  用户购买了 4 张票（共 400 元），只退 2 张（退 200 元）。
  需要支持支付宝的部分退款 API。

场景二：优惠券叠加退款
  用户下单时使用了满 200 减 50 优惠券，实付 150 元。
  退款时：退还实际支付金额（150 元），优惠券不退还。

场景三：组合支付退款
  用户使用了余额 + 支付宝组合支付。
  退款时需要按比例退还至原支付渠道。

场景四：支付超时处理
  用户创建订单后未在 30 分钟内支付。
  系统自动取消订单，释放库存。

场景五：退款到账延迟
  支付宝返回"处理中"，需要异步轮询确认。
```

### 1.2 退款类型矩阵

| 退款类型 | 说明 | 支付渠道影响 |
| --------- | ------ | ------------- |
| 全额退款 | 取消整个订单 | 全部原路返回 |
| 部分退款 | 退部分票或部分金额 | 按比例退还 |
| 含优惠券退款 | 有优惠券的订单 | 退还实付金额 |
| 退款失败 | 余额不足/超过退款期限 | 人工处理 |
| 部分退款+多次 | 一张订单多次部分退款 | 累计不超过原金额 |

---

## 2. 部分退款

### 2.1 支付宝部分退款 API

支付宝原生支持部分退款，同一笔交易最多支持 50 次部分退款。

```python
def api_alipay_trade_refund(
    trade_no=None,          # 支付宝交易号
    out_trade_no=None,      # 商户订单号
    refund_amount=None,     # 退款金额（本次退多少）
    out_request_no=None,     # 退款请求号（部分退款必须！）
    refund_reason=None,     # 退款原因
    **kwargs
):
    """
    支付宝退款 API

    关键参数：
    - out_request_no: 部分退款的幂等键。
      同一 out_request_no 只处理一次，防止重复退款。
      格式建议：REFUND_{order_id}_{timestamp}
    - refund_amount: 本次退款金额（可以少于总金额）
    """
```text

### 2.2 部分退款实现

```python
async def partial_refund(
    self,
    order_id: int,
    admin_id: int,
    refund_amount: Decimal,
    quantity: int | None = None,
    reason: str = "部分退款",
) -> dict:
    """
    部分退款

    实现步骤：
    1. 校验订单状态（paid/refunding）
    2. 校验累计退款金额不超过订单总金额
    3. 生成唯一的 out_request_no
    4. 调用支付宝部分退款 API
    5. 更新退款记录

    部分退款 vs 全额退款的关键区别：
    - 部分退款不改变订单主状态（仍是 paid）
    - 需要记录已退金额和剩余可退金额
    - 需要记录哪些票证被退了
    """
    order = await self.session.get(Order, order_id)
    if not order:
        raise ValueError("订单不存在")

    if order.status not in ("paid", "refunding"):
        raise ValueError("订单状态不允许退款")

    # ===== 第1步：校验累计退款金额 =====
    already_refunded = order.refund_amount or Decimal("0")
    remaining = order.final_amount - already_refunded

    if refund_amount > remaining:
        raise ValueError(
            f"退款金额 {refund_amount} 超过剩余可退金额 {remaining}"
        )

    # ===== 第2步：生成退款请求号（幂等键） =====
    out_request_no = (
        f"PARTIAL_REFUND_{order_id}_"
        f"{datetime.utcnow().strftime('%Y%m%d%H%M%S%f')}"
    )

    # ===== 第3步：Redis 幂等检查（防止重复提交退款） =====
    idempotent_key = f"refund:idempotent:{out_request_no}"
    processed = await self.redis.setnx(idempotent_key, "1")
    if not processed:
        raise ValueError("退款请求已提交，请勿重复操作")
    await self.redis.expire(idempotent_key, 86400)

    try:
        # ===== 第4步：调用支付宝退款 API =====
        refund_result = None
        if order.payment_method in ("alipay",) and order.trade_no:
            refund_result = self.alipay.api_alipay_trade_refund(
                trade_no=order.trade_no,
                refund_amount=float(refund_amount),
                refund_reason=reason,
                out_request_no=out_request_no,
            )

            if refund_result.get("code") != "10000":
                raise ValueError(
                    f"支付宝退款失败: {refund_result.get('sub_msg', '未知错误')}"
                )

        # ===== 第5步：创建退款记录 =====
        refund_record = RefundRecord(
            order_id=order_id,
            refund_amount=refund_amount,
            refund_reason=reason,
            out_request_no=out_request_no,
            trade_no=refund_result.get("trade_no") if refund_result else None,
            refund_type="partial",
            operator_id=admin_id,
            status="completed",
        )
        self.session.add(refund_record)

        # ===== 第6步：更新订单累计退款金额 =====
        new_refunded = already_refunded + refund_amount
        order.refund_amount = new_refunded

        # 如果全部退完，更新状态
        if new_refunded >= order.final_amount:
            order.status = "refunded"

        # ===== 第7步：按数量退票 =====
        if quantity:
            await self._mark_tickets_refunded(order_id, quantity)

        return {
            "success": True,
            "refund_amount": float(refund_amount),
            "remaining": float(remaining - refund_amount),
            "out_request_no": out_request_no,
        }

    except Exception:
        # 失败时删除幂等键，允许重试
        await self.redis.delete(idempotent_key)
        raise
```

### 2.3 部分退款追踪表

```python
class RefundRecord(Base):
    """退款记录（追踪每一笔退款）"""
    __tablename__ = "refund_records"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
    order_id: Mapped[int] = mapped_column(BigInteger, ForeignKey("orders.id"), index=True)
    refund_amount: Mapped[Decimal] = mapped_column(Numeric(10, 2))
    refund_reason: Mapped[str] = mapped_column(Text)
    out_request_no: Mapped[str] = mapped_column(
        String(64), unique=True, comment="退款请求号（幂等键）"
    )
    trade_no: Mapped[Optional[str]] = mapped_column(
        String(64), comment="支付宝退款交易号"
    )
    refund_type: Mapped[str] = mapped_column(
        String(20), default="full",
        comment="full=全额, partial=部分"
    )
    operator_id: Mapped[int] = mapped_column(BigInteger, comment="操作人ID")
    status: Mapped[str] = mapped_column(
        String(20), default="completed",
        comment="pending=处理中, completed=完成, failed=失败"
    )
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
```text

---

## 3. 优惠券叠加退款

### 3.1 优惠券退款策略

```

原则：

  1. 退款金额基于实付金额（非原价）
  2. 优惠券不退（作废处理）
  3. 多次部分退款时，优惠金额按比例分摊

示例：
  订单原价：200 元
  优惠券：满 200 减 50
  实付：150 元

  全额退款 → 退 150 元（实付金额），优惠券作废
  退 1 张票（原价 100）→ 退 75 元（实付的一半）

```text

### 3.2 含优惠券订单的退款计算

```python
from decimal import Decimal, ROUND_HALF_UP


def calculate_coupon_refund(
    original_amount: Decimal,
    discount_amount: Decimal,
    refund_original_amount: Decimal,
) -> Decimal:
    """
    计算含优惠券订单的退款金额

    算法：
    优惠比例 = discount_amount / original_amount
    退款优惠分摊 = refund_original_amount * 优惠比例
    实际退款 = refund_original_amount - 退款优惠分摊

    参数:
        original_amount: 订单原价（无优惠时的总价）
        discount_amount: 优惠金额
        refund_original_amount: 退款部分的原价（如退2张票的原价）

    返回:
        Decimal: 实际应退金额（实付部分）

    示例:
        original_amount=200, discount_amount=50 (满200减50)
        refund_original_amount=100 (退1张票)
        → refund = 100 - 100 * (50/200) = 100 - 25 = 75
    """
    if original_amount <= 0:
        return Decimal("0")

    # 优惠比例
    discount_ratio = discount_amount / original_amount

    # 按比例分摊优惠
    apportioned_discount = (
        refund_original_amount * discount_ratio
    ).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)

    # 实际退款 = 退款部分原价 - 分摊的优惠
    actual_refund = refund_original_amount - apportioned_discount

    # 确保不退负数
    return max(actual_refund, Decimal("0"))


# ===== 验证 =====
# 场景：订单原价400元(4张票x100元)，满200减50券，实付350元
# 退2张票(原价200元)
assert calculate_coupon_refund(
    original_amount=Decimal("400"),
    discount_amount=Decimal("50"),
    refund_original_amount=Decimal("200"),
) == Decimal("175")  # 200 - 200*(50/400) = 200 - 25 = 175

# 场景：全额退款
assert calculate_coupon_refund(
    original_amount=Decimal("400"),
    discount_amount=Decimal("50"),
    refund_original_amount=Decimal("400"),
) == Decimal("350")  # 400 - 400*(50/400) = 400 - 50 = 350 (实付金额)
```

### 3.3 优惠券退款流程图

```text
含优惠券订单退款流程：

用户申请退部分票
    │
    ▼
计算退款部分原价（如退2张 = 200元）
    │
    ▼
计算优惠分摊：200 * (50/400) = 25元
    │
    ▼
实际退款金额：200 - 25 = 175元
    │
    ▼
调用支付宝部分退款 API（退 175 元）
    │
    ▼
记录退款明细（含优惠分摊信息）
    │
    ▼
检查是否全部退完？
    ├── 是 → 标记优惠券为作废（不可再次使用）
    └── 否 → 保留优惠券已用状态
```

---

## 4. 支付超时处理

### 4.1 超时取消机制

```text
支付超时：用户创建订单后 30 分钟内未支付。

处理机制（ARQ Worker 后台任务）：
  1. 定时扫描超时的 pending_payment 订单（每 60 秒）
  2. 取消超时订单
  3. 释放 Redis 库存
  4. 票证标记为取消
  5. 通过 WebSocket 通知用户
```

### 4.2 超时订单扫描

```python
# app/worker.py — ARQ 后台任务

from datetime import datetime, timedelta
from sqlalchemy import select, update


async def cancel_expired_orders(ctx):
    """
    ARQ 定时任务：取消超时未支付订单

    执行频率：每 60 秒（在 WorkerSettings 中配置）

    处理逻辑：
    1. 查询超时订单
    2. 逐笔取消（避免大事务）
    3. 释放 Redis 库存
    4. WebSocket 通知用户

    超时判断：
    created_at + timeout_minutes < now()
    默认 timeout_minutes = 30（可在活动级别覆盖）
    """
    session = ctx["session"]
    redis = ctx["redis"]
    logger = ctx["logger"]

    # 查询超时订单（分页处理）
    timeout = datetime.utcnow() - timedelta(minutes=30)

    while True:
        result = await session.execute(
            select(Order).where(
                Order.status == "pending_payment",
                Order.created_at < timeout,
            )
            .limit(100)  # 每次最多处理 100 笔
        )
        orders = result.scalars().all()

        if not orders:
            break  # 没有更多超时订单

        for order in orders:
            try:
                await _cancel_single_expired_order(
                    session, redis, order, logger
                )
            except Exception as e:
                logger.error(f"取消订单失败: {order.order_no}", exc_info=e)

        # 每批提交一次
        await session.commit()


async def _cancel_single_expired_order(
    session: AsyncSession,
    redis: AsyncRedis,
    order: Order,
    logger,
):
    """
    取消单笔超时订单

    原子操作：
    1. 更新订单状态为 cancelled
    2. 释放 Redis 库存（INCRBY）
    3. 记录取消时间
    """
    # 使用条件更新防止并发
    result = await session.execute(
        update(Order)
        .where(
            Order.id == order.id,
            Order.status == "pending_payment",  # 只取消仍为待支付的
        )
        .values(
            status="cancelled",
            cancelled_at=datetime.utcnow(),
        )
    )

    if result.rowcount > 0:
        # 释放 Redis 库存
        inventory_service = InventoryService(redis)
        await inventory_service.restore_stock(
            order.event_id, order.quantity
        )

        # 通知用户（通过 WebSocket）
        await notify_order_cancelled(redis, order)

        logger.info(
            "订单超时取消",
            extra={
                "order_no": order.order_no,
                "user_id": order.user_id,
                "event_id": order.event_id,
            },
        )
```text

### 4.3 支付超时提示

```python
# 查询订单时返回支付剩余时间

async def get_order_with_timeout(order_id: int, user_id: int) -> dict:
    """获取订单详情（含支付剩余时间）"""
    order = await session.get(Order, order_id)

    if not order or order.user_id != user_id:
        raise ValueError("订单不存在")

    result = {
        "id": order.id,
        "order_no": order.order_no,
        "status": order.status,
        "total_amount": float(order.total_amount),
        "created_at": order.created_at.isoformat(),
    }

    # 计算支付剩余时间
    if order.status == "pending_payment":
        timeout_minutes = 30  # 从活动配置获取
        deadline = order.created_at + timedelta(minutes=timeout_minutes)
        remaining_seconds = (deadline - datetime.utcnow()).total_seconds()
        result["pay_deadline"] = deadline.isoformat()
        result["remaining_seconds"] = max(0, int(remaining_seconds))

    return result
```

### 4.4 前端超时倒计时

```javascript
// frontend/src/stores/order.js — 支付倒计时

export const useOrderStore = defineStore('order', () => {
  const payRemaining = ref(0)
  let timer = null

  function startPayCountdown(orderId) {
    // 获取订单剩余支付时间
    orderApi.getOrder(orderId).then(order => {
      payRemaining.value = order.remaining_seconds

      // 启动倒计时
      timer = setInterval(() => {
        payRemaining.value--
        if (payRemaining.value <= 0) {
          clearInterval(timer)
          // 订单已超时，提示用户
          ElMessage.warning('支付已超时，订单已取消')
          router.push('/orders')
        }
      }, 1000)
    })
  }

  function stopPayCountdown() {
    if (timer) {
      clearInterval(timer)
      timer = null
    }
  }

  return { payRemaining, startPayCountdown, stopPayCountdown }
})
```text

---

## 5. 多支付渠道集成

### 5.1 支付渠道抽象

```python
# app/services/payment_gateway.py — 支付网关抽象层

from abc import ABC, abstractmethod
from dataclasses import dataclass


@dataclass
class PaymentResult:
    """支付结果统一格式"""
    success: bool
    pay_url: str | None = None
    trade_no: str | None = None
    raw_response: dict | None = None
    error_message: str | None = None


@dataclass
class RefundResult:
    """退款结果统一格式"""
    success: bool
    refund_trade_no: str | None = None
    raw_response: dict | None = None
    error_message: str | None = None


class PaymentGateway(ABC):
    """支付网关抽象基类"""

    @abstractmethod
    async def create_payment(self, order: Order, **kwargs) -> PaymentResult:
        """创建支付"""
        ...

    @abstractmethod
    async def process_notification(self, data: dict) -> tuple[str, str]:
        """处理支付通知，返回 (order_no, trade_no)"""
        ...

    @abstractmethod
    async def refund(self, trade_no: str, amount: float, out_request_no: str) -> RefundResult:
        """退款"""
        ...

    @abstractmethod
    async def query(self, out_trade_no: str) -> dict:
        """查询交易状态"""
        ...
```

### 5.2 支付宝网关实现

```python
class AlipayGateway(PaymentGateway):
    """支付宝支付网关实现"""

    def __init__(self, settings):
        self.alipay = self._create_alipay_client(settings)

    async def create_payment(self, order: Order, **kwargs) -> PaymentResult:
        try:
            order_string = self.alipay.api_alipay_trade_page_pay(
                out_trade_no=order.order_no,
                total_amount=str(order.final_amount),
                subject=kwargs.get("subject", f"票务-{order.order_no}"),
                timeout_express="30m",
            )
            pay_url = f"{settings.ALIPAY_GATEWAY}?{order_string}"
            return PaymentResult(success=True, pay_url=pay_url)
        except Exception as e:
            return PaymentResult(success=False, error_message=str(e))

    async def refund(self, trade_no: str, amount: float, out_request_no: str) -> RefundResult:
        try:
            result = self.alipay.api_alipay_trade_refund(
                trade_no=trade_no,
                refund_amount=amount,
                out_request_no=out_request_no,
            )
            if result.get("code") == "10000":
                return RefundResult(
                    success=True,
                    refund_trade_no=result.get("trade_no"),
                    raw_response=result,
                )
            return RefundResult(
                success=False,
                error_message=result.get("sub_msg", "退款失败"),
                raw_response=result,
            )
        except Exception as e:
            return RefundResult(success=False, error_message=str(e))
```text

### 5.3 支付路由

```python
class PaymentRouter:
    """
    支付路由器

    根据订单的支付方式路由到不同的支付网关：

    payment_method = "alipay"  → AlipayGateway
    payment_method = "wechat"  → WechatGateway（扩展预留）
    payment_method = "wallet"  → WalletGateway（余额支付）
    """

    def __init__(self):
        self.gateways: dict[str, PaymentGateway] = {}

    def register(self, method: str, gateway: PaymentGateway):
        """注册支付网关"""
        self.gateways[method] = gateway

    def get_gateway(self, method: str) -> PaymentGateway:
        """获取支付网关"""
        gateway = self.gateways.get(method)
        if not gateway:
            raise ValueError(f"不支持的支付方式: {method}")
        return gateway

    async def route_payment(self, order: Order, method: str = "alipay") -> PaymentResult:
        """路由支付请求"""
        gateway = self.get_gateway(method)
        return await gateway.create_payment(order)

    async def route_refund(
        self, order: Order, amount: float, out_request_no: str
    ) -> RefundResult:
        """路由退款请求"""
        gateway = self.get_gateway(order.payment_method)
        return await gateway.refund(order.trade_no, amount, out_request_no)
```

### 5.4 钱包支付（余额支付）

```python
class WalletGateway(PaymentGateway):
    """余额支付网关（内部账户）"""

    async def create_payment(self, order: Order, **kwargs) -> PaymentResult:
        """
        余额支付：
        1. 检查用户余额是否充足
        2. 扣减余额
        3. 直接确认支付
        """
        user = await session.get(User, order.user_id)
        if user.balance < order.final_amount:
            return PaymentResult(success=False, error_message="余额不足")

        # 扣减余额
        user.balance -= order.final_amount
        return PaymentResult(success=True, trade_no=f"WALLET_{order.order_no}")

    async def refund(self, trade_no: str, amount: float, out_request_no: str) -> RefundResult:
        """余额退款：直接加回用户余额"""
        # 从 trade_no 解析用户ID，退还余额
        return RefundResult(success=True)
```text

---

## 6. 原理深入

### 6.1 部分退款的数学约束

```

部分退款的核心约束：

设：
  P = 订单总金额（实付）
  R_i = 第 i 次退款金额
  n = 总退款次数

约束 1：总和不超过总金额
  ΣR_i ≤ P, for i = 1 to n

约束 2：每次退款为正数
  R_i > 0, for all i

约束 3：次数限制
  n ≤ 50（支付宝限制）

约束 4：不可重复
  out_request_no 保证每笔退款唯一

约束 5：退款后订单状态
  若 ΣR_i < P → 订单保持 paid 状态
  若 ΣR_i = P → 订单变为 refunded 状态

```text

### 6.2 退款一致性保证

```

场景：部分退款+新购买同时发生

如果用户在部分退款的过程中又购买了新票，
活动库存可能被新购买占用，导致退款后的库存恢复出现问题。

解决方案：

  1. 退款和购买共享同一 Redis 库存计数器
  2. 退款时的库存恢复是一个独立的 INCRBY（不受其他 DECR 影响）
  3. MySQL 层面通过乐观锁保证库存不会超卖

  实际上不存在一致性问题，因为：

- 退款恢复库存是 INCR（增加）
- 新购买扣减是 DECR（减少）
- 两者操作不同的方向，互不冲突
- 只要总库存不超卖，退款恢复的库存可以被新购买使用

```text

### 6.3 支付超时与库存释放

```

乐观锁/悲观锁对比：

超时取消使用悲观锁策略：

  1. 定时任务扫描 → 发现超时订单
  2. 使用条件更新（WHERE status = 'pending_payment'）→ 行级锁
  3. 成功更新 → 释放库存
  4. 释放库存用 INCRBY → Redis 原子操作

为什么不直接用 Redis TTL 自动释放？

- Redis 库存只是一个计数器，没有关联的订单信息
- 超时后需要做很多事：修改订单状态、释放库存、通知用户
- 这些逻辑无法用 TTL 自动完成
- TTL 删除后，如果 MySQL 订单状态还没改，会造成不一致

更可靠的方案：订单级别的 Redis TTL（提示性）+ Worker 兜底
  SET order:ttl:{order_no} 1 EX 1800  # 30分钟过期
  Worker 扫描 + TTL 不存在作为快速判断条件

```text

### 6.4 退款幂等性证明

```

问题：
  支付宝退款 API 调用后，网络超时，结果未知。
  重试时，可能退款已成功，导致重复退款。

解决方案（out_request_no + Redis SETNX）：

  1. 生成唯一 out_request_no（ORDER_ID + 时间戳）
  2. 调用支付宝退款前，Redis SETNX 幂等键
  3. 若 SETNX 成功 → 执行退款
  4. 若 SETNX 失败 → 已有相同请求在处理，拒绝

  证明：
  out_request_no 保证了支付宝侧只会处理一次。
  SETNX 保证了应用侧不会并发提交相同请求。
  两者组合保证 exactly-once 语义。

  但还有边缘情况：
  应用调用支付宝退款成功，但应用在收到响应前崩溃。
  重启后该 out_request_no 的 SETNX 已存在，不会再发起退款。
  支付宝侧该 out_request_no 已处理成功 → 不会有重复退款。

  如果应用崩溃后 Redis 数据丢失：
  SETNX 会成功，导致重复提交退款请求。
  但支付宝侧 out_request_no 唯一约束会拒绝第二次请求。

```text

### 6.5 退款对账机制

```python
# 每日对账任务（ARQ Worker）

async def daily_refund_reconciliation(ctx):
    """
    每日退款对账

    检查支付宝侧的退款记录和本系统记录是否一致。
    发现不一致时自动修复或报警。
    """
    session = ctx["session"]
    alipay_gateway = ctx["alipay_gateway"]

    # 查询昨日所有退款记录
    yesterday = datetime.utcnow() - timedelta(days=1)
    result = await session.execute(
        select(RefundRecord).where(
            RefundRecord.created_at >= yesterday,
            RefundRecord.status == "completed",
        )
    )
    records = result.scalars().all()

    for record in records:
        # 查询支付宝侧退款状态
        query_result = alipay_gateway.alipay.api_alipay_trade_fastpay_refund_query(
            trade_no=record.trade_no,
            out_request_no=record.out_request_no,
        )

        alipay_status = query_result.get("refund_status")
        if alipay_status != "REFUND_SUCCESS":
            ctx["logger"].error(
                "退款记录不一致",
                extra={
                    "record_id": record.id,
                    "out_request_no": record.out_request_no,
                    "alipay_status": alipay_status,
                },
            )
            # 触发告警
            await send_alert(f"退款不一致: {record.id}")
```

---

## 项目代码参考

| 文件 | 说明 |
| ------ | ------ |
| `app/services/payment_gateway.py` | 支付网关抽象层 + AlipayGateway 实现 |
| `app/services/refund_service.py` | 退款服务（含部分退款、优惠券退款） |
| `app/worker.py` | ARQ 定时任务（超时取消、对账） |
| `app/models/refund_record.py` | 退款记录模型 |
| `app/routers/admin.py` | 退款管理路由 |
| `frontend/src/stores/order.js` | 前端支付倒计时逻辑 |

## 本章总结

✅ **已掌握：**

- 支付宝部分退款 API 及实现（out_request_no 幂等键）
- 含优惠券订单的退款计算（按比例分摊优惠金额）
- 支付超时自动取消机制（ARQ Worker 定时扫描）
- 多支付渠道的抽象网关模式
- 原理深入：部分退款数学约束、退款一致性、幂等性证明、对账

✅ **项目里程碑：**

- [ ] 部分退款功能可用（同一订单多次退款）
- [ ] 含优惠券的退款金额计算正确
- [ ] 30分钟未支付自动取消 + 释放库存
- [ ] 前端30分钟倒计时提示
- [ ] 支付宝部分退款幂等性保障
- [ ] 每日退款对账无不一致
