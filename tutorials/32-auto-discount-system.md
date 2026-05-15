# 32. 自动折扣系统

> **目标读者**: 开发者 / 系统管理员
> **前置知识**: ORM 模型 / FastAPI 路由 / 服务层模式 / 订单流程

---

## 目录

1. [什么是自动折扣](#1-什么是自动折扣)
2. [架构设计](#2-架构设计)
3. [数据库设计](#3-数据库设计)
4. [规则匹配算法](#4-规则匹配算法)
5. [与订单流程的集成](#5-与订单流程的集成)
6. [API 端点](#6-api-端点)
7. [管理端使用指南](#7-管理端使用指南)
8. [代码示例](#8-代码示例)
9. [常见问题](#9-常见问题)

---

## 1. 什么是自动折扣

### 1.1 自动折扣 vs 优惠券

| 特性 | 自动折扣 | 优惠券 |
| ------ | --------- | -------- |
| 用户操作 | 无需操作，结账时自动应用 | 需手动领取 + 输入编码 |
| 创建方式 | 管理员创建规则 | 管理员创建 + 用户领取 |
| 匹配逻辑 | 按优先级 + 最优金额自动匹配 | 用户自主选择 |
| 适用场景 | 限时促销 / 满减 / 全场折扣 | 定向发放 / 用户专属优惠 |
| 叠加规则 | 取最优的一条规则 | 可与自动折扣叠加 |

### 1.2 核心能力

- **自动匹配**: 用户无需任何操作，下单时系统自动计算最优折扣
- **优先级控制**: 管理员可为规则设置优先级，高优先级规则优先匹配
- **活动限定**: 规则可限定到特定活动，也可设置为全场通用
- **时间窗口**: 支持设置生效时间范围，过期自动失效
- **金额门槛**: 可设置最低订单金额，低于门槛不触发

---

## 2. 架构设计

### 2.1 分层架构

```text
┌─────────────────────────────────────────────────────────────┐
│                      路由层 (Routers)                        │
│  POST /admin/discount-rules  → 创建规则                     │
│  PUT  /admin/discount-rules/{id} → 更新规则                 │
│  GET  /discount-rules/available → 查询可用折扣              │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                     服务层 (Services)                        │
│  DiscountService:                                            │
│  ├─ create_rule / update_rule / delete_rule  (CRUD)         │
│  ├─ get_best_discount()    → 最优规则匹配算法               │
│  └─ auto_apply_discount()  → 自动应用到订单金额             │
│                                                              │
│  OrderService:                                               │
│  └─ create_order() → 调用 DiscountService.auto_apply        │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                    模型层 (Models)                           │
│  DiscountRule: 折扣规则配置                                  │
│  Order:        auto_discount_id, auto_discount_amount         │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 请求流程

```text
用户点击"下单"
    │
    ▼
OrderService.create_order()
    │
    ├─ 1. 查询活动 + 参数校验
    │
    ├─ 2. 计算基础金额 (price × quantity)
    │
    ├─ 3. DiscountService.auto_apply_discount()
    │      │
    │      ├─ 查询所有活跃规则（时间窗口内 + is_active）
    │      ├─ 过滤适用于该活动的规则
    │      ├─ 计算每个规则的折扣金额
    │      ├─ 按优先级 + 折扣金额排序
    │      └─ 返回 (折后金额, 折扣信息)
    │
    ├─ 4. 优惠券处理（可选，在自动折扣基础上叠加）
    │
    ├─ 5. Redis 原子扣减（防超卖）
    │
    └─ 6. 创建订单（含 auto_discount_id）
```

---

## 3. 数据库设计

### 3.1 discount_rules 表

```sql
CREATE TABLE discount_rules (
    id              INT             PRIMARY KEY AUTO_INCREMENT,
    name            VARCHAR(100)    NOT NULL COMMENT '规则名称',
    description     TEXT            COMMENT '规则描述',
    discount_type   VARCHAR(20)     NOT NULL COMMENT 'percentage|fixed',
    discount_value  DECIMAL(10,2)   NOT NULL COMMENT '折扣值',
    min_amount      DECIMAL(10,2)   DEFAULT 0 COMMENT '最低订单金额',
    max_discount    DECIMAL(10,2)   COMMENT '最高优惠金额',
    priority        INT             DEFAULT 0 COMMENT '优先级',
    is_active       TINYINT(1)      DEFAULT 1 COMMENT '是否启用',
    applicable_event_ids JSON       COMMENT '适用活动ID列表',
    start_time      DATETIME        COMMENT '生效开始时间',
    end_time        DATETIME        COMMENT '生效结束时间',
    created_by      BIGINT          NOT NULL,
    created_at      DATETIME        DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME        DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (created_by) REFERENCES users(id)
);
```text

### 3.2 orders 表新增字段

```sql
ALTER TABLE orders ADD COLUMN auto_discount_id     BIGINT       COMMENT '自动折扣规则ID';
ALTER TABLE orders ADD COLUMN auto_discount_amount DECIMAL(10,2) DEFAULT 0 COMMENT '自动折扣金额';
ALTER TABLE orders ADD FOREIGN KEY (auto_discount_id) REFERENCES discount_rules(id);
```

### 3.3 字段说明

| 字段 | 说明 | 示例值 |
| ------ | ------ | -------- |
| `discount_type` | 折扣类型 | `percentage` 或 `fixed` |
| `discount_value` | 折扣值 | 百分比: `15` (15%), 固定: `50.00` (50元) |
| `min_amount` | 最低订单金额 | `100.00` (满100才享折扣) |
| `max_discount` | 最高优惠额 | `30.00` (最多减30) |
| `priority` | 优先级 | `100` 高于 `0` |
| `applicable_event_ids` | 适用活动 | `[1, 3, 5]` 或 `null`(全部) |

---

## 4. 规则匹配算法

### 4.1 算法流程

```text
输入: event_id, quantity, user_id
    │
    ▼
┌─────────────────────────────────────────────┐
│ 1. 查询所有活跃规则                           │
│    WHERE is_active = 1                       │
│    AND (start_time IS NULL OR start_time ≤ now) │
│    AND (end_time IS NULL OR end_time ≥ now)    │
│    ORDER BY priority DESC                    │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│ 2. 过滤适用规则                               │
│    - applicable_event_ids IS NULL (全场)     │
│    - event_id IN applicable_event_ids (指定) │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│ 3. 计算每个规则的折扣金额                     │
│    百分比: discount = amount × value / 100   │
│            capped by max_discount             │
│    固定金额: discount = min(value, amount)    │
│    跳过: amount < min_amount                 │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│ 4. 排序取最优                                 │
│    主排序: priority DESC (优先级高优先)       │
│    次排序: discount_amount DESC (折扣大优先)  │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
            返回: AutoDiscountResult
```

### 4.2 核心代码

```python
async def get_best_discount(
    self, event_id: int, quantity: int, user_id: int | None = None,
) -> AutoDiscountResult | None:
    event = await self.session.get(Event, event_id)
    base_amount = Decimal(str(event.price)) * quantity

    active_rules = await self._get_active_rules()

    candidates: list[tuple[DiscountRule, Decimal]] = []
    for rule in active_rules:
        if not self._is_rule_applicable(rule, event_id):
            continue
        if base_amount < Decimal(str(rule.min_amount)):
            continue
        discount = self._calculate_discount_amount(rule, base_amount)
        if discount > 0:
            candidates.append((rule, discount))

    if not candidates:
        return None

    # 按优先级降序 > 折扣金额降序
    candidates.sort(key=lambda x: (-x[0].priority, -x[1]))
    best_rule, best_discount = candidates[0]

    return AutoDiscountResult(
        rule_id=best_rule.id,
        rule_name=best_rule.name,
        discount_type=best_rule.discount_type,
        discount_value=best_rule.discount_value,
        discount_amount=float(best_discount),
    )
```text

### 4.3 算法特性

- **时间复杂度**: O(n)，n 为活跃规则数（通常 < 100）
- **空间复杂度**: O(n)
- **确定性**: 相同输入始终返回相同结果
- **可扩展性**: 新增规则类型只需扩展 `_calculate_discount_amount`

---

## 5. 与订单流程的集成

### 5.1 集成点

在 `OrderService.create_order()` 中，在计算基础金额后、Redis 扣减前调用自动折扣：

```python
# 2a. 自动折扣（先于优惠券应用，无需用户操作）
discount_svc = DiscountService(self.session, self.redis)
total_amount, discount_info = await discount_svc.auto_apply_discount(
    event_id=data.event_id,
    quantity=data.quantity,
    base_amount=base_amount,
    user_id=user_id,
)
if discount_info:
    auto_discount_id = discount_info["rule_id"]
    auto_discount_amount = Decimal(str(discount_info["discount_amount"]))

# 2b. 优惠券（在自动折扣后的金额上应用）
if data.coupon_code:
    coupon_svc = CouponService(self.session, self.redis)
    coupon, discount = await coupon_svc.validate_and_use_coupon(
        code=data.coupon_code,
        user_id=user_id,
        order_amount=float(base_amount),  # 校验基于原价
    )
    total_amount -= Decimal(str(discount))
    coupon_id = coupon.id
```

### 5.2 折扣叠加规则

| 场景 | 行为 |
| ------ | ------ |
| 只有自动折扣 | 金额减少 auto_discount_amount |
| 只有优惠券 | 金额减少 discount_amount（原有逻辑） |
| 两者都有 | 先减 auto_discount，再减 coupon_discount |
| 两者都无 | 原价支付 |

### 5.3 订单表新增字段

```python
class Order(Base):
    ...
    auto_discount_id: Mapped[int | None] = mapped_column(
        BigInteger, ForeignKey("discount_rules.id"), default=None,
        comment="自动折扣规则ID",
    )
    auto_discount_amount: Mapped[float] = mapped_column(
        Numeric(10, 2), default=0, comment="自动折扣金额",
    )
```text

---

## 6. API 端点

### 6.1 管理端 API

| 方法 | 路径 | 说明 | 认证 |
| ------ | ------ | ------ | ------ |
| POST | `/admin/discount-rules` | 创建规则 | admin |
| PUT | `/admin/discount-rules/{id}` | 更新规则 | admin |
| DELETE | `/admin/discount-rules/{id}` | 删除规则 | admin |
| GET | `/admin/discount-rules` | 规则列表 | admin |
| GET | `/admin/discount-rules/{id}` | 规则详情 | admin |

### 6.2 公共 API

| 方法 | 路径 | 说明 | 认证 |
| ------ | ------ | ------ | ------ |
| GET | `/discount-rules/available?event_id=1&quantity=2` | 查询可用折扣 | 无需 |

### 6.3 请求/响应示例

**POST /admin/discount-rules**

```json
{
    "name": "新用户特惠",
    "discount_type": "percentage",
    "discount_value": 20,
    "min_amount": 100,
    "max_discount": 50,
    "priority": 10,
    "is_active": true,
    "applicable_event_ids": null,
    "start_time": "2026-05-01T00:00:00Z",
    "end_time": "2026-06-01T00:00:00Z"
}
```

**响应:**

```json
{
    "id": 1,
    "name": "新用户特惠",
    "discount_type": "percentage",
    "discount_value": 20.0,
    "min_amount": 100.0,
    "max_discount": 50.0,
    "priority": 10,
    "is_active": true,
    "applicable_event_ids": null,
    "start_time": "2026-05-01T00:00:00Z",
    "end_time": "2026-06-01T00:00:00Z",
    "created_by": 1,
    "created_at": "2026-05-13T10:00:00Z",
    "updated_at": "2026-05-13T10:00:00Z"
}
```text

**GET /discount-rules/available?event_id=1&quantity=3**

```json
[
    {
        "rule_id": 1,
        "rule_name": "新用户特惠",
        "discount_type": "percentage",
        "discount_value": 20.0,
        "discount_amount": 30.0
    }
]
```

---

## 7. 管理端使用指南

### 7.1 创建折扣规则

1. 登录管理员账号
2. 调用 `POST /admin/discount-rules` 创建规则
3. 填写以下关键参数：
   - **规则名称**: 例如"618 大促全场 8 折"
   - **折扣类型**: 百分比折扣或固定金额减免
   - **折扣值**: 百分比数值（如 20 表示 20%）或具体金额
   - **最低订单金额**: 可选，订单金额低于此值时不触发
   - **最高优惠金额**: 百分比折扣时有效，避免大额订单亏损
   - **优先级**: 数值越大越优先匹配
   - **适用活动**: 可选，不填则适用于全部活动
   - **时间窗口**: 确保规则在预期时间内生效

### 7.2 规则优先级管理

- 高优先级规则总是先于低优先级规则匹配
- 同一优先级下，折扣金额更大的规则胜出
- 推荐策略：限时秒杀用高优先级，常规促销用低优先级

### 7.3 最佳实践

- **叠加使用**: 自动折扣与优惠券可叠加，自动折扣优先计算
- **金额门槛**: 合理设置 min_amount 避免小额订单优惠过度
- **上限控制**: 百分比折扣务必设置 max_discount 控制风险
- **时间管理**: 促销结束后规则自动失效，无需手动关闭

---

## 8. 代码示例

### 8.1 创建百分比折扣规则（全场 8 折，最高减 100）

```python
from app.services.discount_service import DiscountService

rule_data = {
    "name": "全场 8 折",
    "discount_type": "percentage",
    "discount_value": 20,  # 20% off
    "min_amount": 0,
    "max_discount": 100,
    "priority": 5,
    "is_active": True,
    "applicable_event_ids": None,  # 全部活动
    "start_time": "2026-05-01T00:00:00Z",
    "end_time": "2026-06-01T00:00:00Z",
}

rule = await service.create_rule(rule_data, admin_id=1)
```text

### 8.2 创建固定金额折扣规则（满 200 减 50）

```python
rule_data = {
    "name": "满 200 减 50",
    "discount_type": "fixed",
    "discount_value": 50,
    "min_amount": 200,
    "max_discount": None,
    "priority": 3,
    "is_active": True,
    "applicable_event_ids": [1, 3, 5],  # 仅限指定活动
    "start_time": None,  # 立即生效
    "end_time": None,    # 永久有效
}

rule = await service.create_rule(rule_data, admin_id=1)
```

### 8.3 查询订单中应用的自动折扣

```python
order = await session.get(Order, order_id)
if order.auto_discount_id:
    rule = await session.get(DiscountRule, order.auto_discount_id)
    print(f"自动折扣: {rule.name}")
    print(f"减免金额: {order.auto_discount_amount}")
```text

---

## 9. 常见问题

### 9.1 自动折扣和优惠券的区别是什么？

自动折扣无需用户操作，在下单时自动计算并应用。优惠券需要用户手动领取并在下单时输入编码。两者可以叠加使用：先计算自动折扣，再应用优惠券。

### 9.2 多个规则同时满足时如何处理？

系统按优先级（priority 字段，数值越大越优先）排序，同一优先级按折扣金额大小排序，取最优的一条。不会叠加多条规则。

### 9.3 如何让某个活动不参与自动折扣？

不在该活动上创建任何规则，或在规则的 `applicable_event_ids` 中排除该活动。如果规则设置 `applicable_event_ids = null`（全场通用），则需要创建一条新的规则专门覆盖需要排除的活动（设置更高的优先级但 `discount_value = 0`），或者直接不创建全场规则。

### 9.4 自动折扣会影响退款吗？

退款时按实际支付金额退款（已扣除自动折扣）。`auto_discount_amount` 存储在订单中，方便对账时追溯优惠来源。

### 9.5 可以创建多少条规则？

没有数量限制。但由于每次下单都需要遍历所有活跃规则进行计算，建议保持活跃规则在 100 条以内。历史规则建议设为 `is_active = false` 而非删除。
