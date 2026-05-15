# 第 34 章：核销系统 — 票证验证与入场管理

> 本章介绍票务系统的核销（Verification/Redeem）子系统，涵盖验证码生成、二维码生成、票证核销流程、超级管理员系统以及核销统计。

## 目录

1. [核销系统概述](#1-核销系统概述)
2. [核销码生成原理](#2-核销码生成原理)
3. [二维码生成原理](#3-二维码生成原理)
4. [核销流程](#4-核销流程)
5. [超级管理员系统](#5-超级管理员系统)
6. [API 端点参考](#6-api-端点参考)
7. [核销统计](#7-核销统计)
8. [批量核销](#8-批量核销)
9. [安全性考虑](#9-安全性考虑)

---

## 1. 核销系统概述

核销系统是票务平台的核心线下环节，负责在活动入场时验证票证的真实性和有效性。

### 1.1 系统流程图

```text
购票支付成功
    │
    ▼
自动生成核销码 + 二维码 ──→ 用户查看电子票
    │
    ▼
到达活动现场
    │
    ├── 方式一：出示二维码 → 工作人员扫码
    │                    → 系统验证
    │                    → 核销成功 ✅
    │
    ├── 方式二：提供核销码 → 工作人员手动输入
    │                    → 系统验证
    │                    → 核销成功 ✅
    │
    ▼
入场完成（verified_at 记录时间）
```

### 1.2 核心设计原则

| 原则 | 说明 |
| ------ | ------ |
| 唯一性 | 每张票证有唯一的核销码，不可重复 |
| 防伪造 | 18 位随机码，不可猜测 |
| 防重放 | 已核销的票证不可再次核销 |
| 可追溯 | 记录核销操作人和核销时间 |
| 双通道 | 同时支持扫码和手动输入 |

---

## 2. 核销码生成原理

### 2.1 编码格式

核销码采用以下格式：

```text
TC + YYMMDD + 14 位随机十六进制字符（大写）

示例：TC250513A3F8K2M9X1
```

| 部分 | 长度 | 说明 |
| ------ | ------ | ------ |
| `TC` | 2 | Ticket Code 前缀，便于识别 |
| `250513` | 6 | 生成日期（YYMMDD），2025-05-13 |
| `A3F8K2M9X1` | 14 | 随机部分（7 字节 hex，大写） |

**总长度：** 18 字符

### 2.2 为什么使用这种格式

1. **可读性**：前缀 `TC` 明确标识为票证核销码
2. **时间维度**：日期部分便于按天统计和排查
3. **唯一性**：14 位随机字符提供 `16^14 ≈ 7.2 × 10^16` 种组合
4. **可手动输入**：仅包含字母和数字，没有特殊字符
5. **固定长度**：便于输入框格式校验

### 2.3 碰撞处理

生成代码时执行数据库查询确保唯一性：

```python
async def generate_verification_code(self, ticket: Ticket) -> str:
    date_part = datetime.now(timezone.utc).strftime("%y%m%d")
    random_part = secrets.token_hex(7).upper()
    code = f"TC{date_part}{random_part}"

    # 确保唯一性（碰撞概率极低，但以防万一）
    existing = await self.session.execute(
        select(Ticket).where(Ticket.verification_code == code)
    )
    if existing.scalar_one_or_none():
        return await self.generate_verification_code(ticket)

    ticket.verification_code = code
    return code
```text

使用 `secrets.token_hex()` 而非 `random.choices()`，因为 `secrets` 模块提供**密码学安全**的随机数，防止攻击者预测核销码。

---

## 3. 二维码生成原理

### 3.1 qrcode 库使用

使用 `qrcode[pil]` 库生成二维码：

```python
import qrcode
import io
import base64

def generate_qr_base64(verification_code: str) -> str:
    qr = qrcode.QRCode(box_size=10, border=2)
    qr.add_data(verification_code)
    qr.make(fit=True)
    img = qr.make_image(fill_color="black", back_color="white")

    buffer = io.BytesIO()
    img.save(buffer, format="PNG")

    return "data:image/png;base64," + base64.b64encode(buffer.getvalue()).decode()
```

### 3.2 Base64 编码用于 Web 展示

将二维码图片编码为 Base64 字符串并嵌入 `data:image/png;base64,...` URL，可以直接在 HTML/CSS 中显示，无需存储图片文件：

```html
<img src="data:image/png;base64,iVBORw0KGgo..." />
```text

**优点：**

- 无需文件存储系统
- 无需 CDN 分发
- 随 API 响应一起返回，减少网络请求
- 适合临时展示（电子票页面）

### 3.3 QR 纠错等级

`qrcode` 库支持 4 种纠错等级：

| 等级 | 纠错能力 | 适用场景 |
| ------ | --------- | --------- |
| ERROR_CORRECT_L | ~7% | 一般场景 |
| ERROR_CORRECT_M | ~15% | 常用（默认） |
| ERROR_CORRECT_Q | ~25% | 需要部分遮挡时 |
| ERROR_CORRECT_H | ~30% | 最高保护 |

**推荐使用 `ERROR_CORRECT_M`**（默认值），在二维码密度和纠错能力之间取得平衡。如果票证打印在纸质上且可能被折叠或污损，可升级到 `ERROR_CORRECT_Q`。

---

## 4. 核销流程

### 4.1 扫码或手动输入

核销支持两种输入方式：

1. **扫码**：工作人员使用扫码枪或手机扫描用户出示的二维码
2. **手动输入**：用户或工作人员手动输入 18 位核销码

```python
async def verify_ticket(self, code: str, verifier_id: int) -> Ticket:
    # 1. 查找票证
    result = await self.session.execute(
        select(Ticket).where(Ticket.verification_code == code)
    )
    ticket = result.scalar_one_or_none()
    if not ticket:
        raise NotFoundError("票证不存在或验证码无效")

    # 2. 检查是否已核销（防重复）
    if ticket.verified_at:
        raise ConflictError(f"票证已于 {ticket.verified_at} 核销")

    # 3. 检查订单状态
    order = await self.session.get(Order, ticket.order_id)
    if not order or order.status != "paid":
        raise ConflictError("票证对应的订单尚未支付")

    # 4. 执行核销
    ticket.verified_at = datetime.now(timezone.utc)
    ticket.verified_by = verifier_id

    return ticket
```

### 4.2 状态检查

核销前执行三道检查：

```text
请求核销
  │
  ├─ 检查 1: 票证存在性
  │   └─ 验证码在 tickets 表中存在
  │
  ├─ 检查 2: 重复核销防护
  │   └─ verified_at IS NULL
  │
  ├─ 检查 3: 支付状态
  │   └─ orders.status == "paid"
  │
  ▼
核销成功 ✅
```

### 4.3 双重核销预防

系统通过两个机制防止同一张票被重复核销：

1. **数据库层**：检查 `verified_at` 字段是否为 NULL
2. **应用层**：`ConflictError` 异常明确提示已核销

## 5. 超级管理员系统

### 5.1 为什么 ID=1 保留给超级管理员

ID=1 作为系统预设的超级管理员账号，有以下设计考量：

1. **引导性**：新部署的系统默认存在管理员账号
2. **测试便利**：开发和测试环境中默认存在可用于测试的账号
3. **可升级性**：如果注册的第一个用户是普通用户，后续可通过 seed 逻辑升级为管理员

### 5.2 自动种子化

在应用启动时执行种子数据初始化：

```python
# app/main.py
@app.on_event("startup")
async def startup():
    # ... 其他初始化代码 ...
    async with async_session() as session:
        from app.services.seed_service import ensure_super_admin
        await ensure_super_admin(session)
        await session.commit()
```text

`ensure_super_admin` 函数的逻辑：

```python
async def ensure_super_admin(session: AsyncSession) -> User:
    admin = await session.get(User, 1)
    if admin:
        if admin.role != "admin":
            admin.role = "admin"  # 升级现有用户
        return admin

    # 创建新的超级管理员
    admin = User(
        id=1,
        username="admin",
        email="admin@ticket-sell.com",
        password=hash_password("admin123"),
        role="admin",
        ...
    )
    session.add(admin)
    return admin
```

### 5.3 测试账号

| 字段 | 值 |
| ------ | ----- |
| ID | 1 |
| 用户名 | admin |
| 密码 | admin123 |
| 角色 | admin |

---

## 6. API 端点参考

### POST `/api/v1/verification/verify`

核销单张票证（需要 organizer 或 admin 角色）。

**请求体：**

```json
{
  "code": "TC250513A3F8K2M9X1"
}
```text

**成功响应 (200 OK)：**

```json
{
  "id": 42,
  "order_id": 10,
  "event_id": 1,
  "user_id": 5,
  "ticket_no": "TKT20260513a1b-1",
  "price": 680.00,
  "verification_code": "TC250513A3F8K2M9X1",
  "verified_at": "2026-05-13T18:30:00Z",
  "verified_by": 1,
  "created_at": "2026-05-13T10:00:00Z"
}
```

**错误响应：**

| 状态码 | 场景 |
| -------- | ------ |
| 404 | 票证不存在或验证码无效 |
| 409 | 票证已核销 或 订单未支付 |
| 403 | 非组织者/管理员角色 |

---

### POST `/api/v1/verification/batch-verify`

批量核销票证（需要 organizer 或 admin 角色）。

**请求体：**

```json
{
  "codes": [
    "TC250513A3F8K2M9X1",
    "TC250513B7C4D1E2F3"
  ]
}
```text

**成功响应 (200 OK)：**

```json
{
  "success": [...],
  "failed": [
    {"code": "TC250513B7C4D1E2F3", "reason": "票证已于 ... 核销"}
  ],
  "success_count": 1,
  "failed_count": 1
}
```

---

### GET `/api/v1/verification/ticket/{code}`

通过验证码查询票证信息（需要 organizer 或 admin 角色）。

**成功响应 (200 OK)：** 返回 `TicketResponse`。

---

### GET `/api/v1/events/{event_id}/verification/tickets`

获取活动票证列表（核销管理）。

**查询参数：**

| 参数 | 类型 | 默认值 | 说明 |
| ------ | ------ | -------- | ------ |
| page | integer | 1 | 页码 |
| size | integer | 20 | 每页数量 |
| verified | boolean | null | 过滤已核销/未核销 |

---

### GET `/api/v1/events/{event_id}/verification/stats`

获取活动核销统计。

**成功响应 (200 OK)：**

```json
{
  "event_id": 1,
  "total_tickets": 500,
  "verified_count": 320,
  "unverified_count": 180,
  "verification_rate": 64.0
}
```text

---

## 7. 核销统计

系统提供实时核销统计数据，帮助组织者掌握入场进展：

### 统计指标

| 指标 | 说明 |
| ------ | ------ |
| total_tickets | 活动票证总数 |
| verified_count | 已核销数量 |
| unverified_count | 未核销数量 |
| verification_rate | 核销率 (%) |

### 应用场景

1. **入场监控**：实时查看已入场人数，判断是否需要限流
2. **核销进度**：在活动大屏上显示核销进度条
3. **异常检测**：核销率异常低可能表示入口拥堵或核销设备故障

---

## 8. 批量核销

### 使用场景

1. **团队入场**：一个组织者同时核销一组票（如旅行团）
2. **VIP 通道**：VIP 客户一次性核销多张票
3. **离线恢复**：扫码枪断网重连后批量提交

### 失败处理

批量核销采用"部分成功"策略——不会因为一张票失败而全部回滚：

```python
async def batch_verify(self, codes: list[str], verifier_id: int) -> dict:
    success = []
    failed = []
    for code in codes:
        try:
            ticket = await self.verify_ticket(code, verifier_id)
            success.append(ticket)
        except (NotFoundError, ConflictError, ValidationError) as e:
            failed.append({"code": code, "reason": str(e)})

    return {"success": success, "failed": failed}
```

---

## 9. 安全性考虑

### 9.1 验证码防伪造

| 威胁 | 防护措施 |
| ------ | --------- |
| 猜测验证码 | 14 位密码学安全随机数，不可预测 |
| 暴力枚举 | 验证码 18 位长度，枚举空间为 `36^18` |
| 泄露验证码 | 每张票独立验证码，泄露不影响其他票 |

### 9.2 防重放攻击

- 已核销票证的 `verified_at` 不为 NULL
- 数据库唯一索引确保 `verification_code` 不重复
- 即使验证码泄露，重复使用会返回 409 Conflict

### 9.3 组织者授权

核销操作需要 `organizer` 或 `admin` 角色：

```python
@router.post("/verification/verify", response_model=TicketResponse)
async def verify_ticket(
    data: TicketVerifyRequest,
    current_user: User = Depends(require_organizer),  # 角色检查
    ...
):
```text

普通用户无法调用核销接口。

### 9.4 操作审计

| 审计字段 | 来源 | 用途 |
| --------- | ------ | ------ |
| verified_by | JWT token 中的用户 ID | 追踪谁执行了核销 |
| verified_at | 服务器 UTC 时间 | 记录核销时间 |
| verification_code | 自动生成 | 定位具体票证 |

---

## 本章总结

| 知识点 | 说明 |
| -------- | ------ |
| 核销码 | TC + 日期 + 14 位随机码，18 位唯一标识 |
| 二维码 | qrcode[pil] 生成，Base64 编码用于 Web 展示 |
| 核销流程 | 存在检查 → 重复检查 → 支付状态检查 → 标记核销 |
| 批量核销 | 支持批量操作，部分失败不回滚 |
| 超级管理员 | ID=1 默认 admin/admin123，自动种子化 |
| 权限控制 | 仅 organizer/admin 可执行核销操作 |
| 防重复 | verified_at 字段 + 唯一索引双重保障 |

## 相关章节

- [第 16 章：支付宝支付集成](16-alipay-integration.md) — 支付成功触发核销码生成
- [第 07 章：JWT + OAuth2 认证](07-jwt-oauth2-auth.md) — 权限控制基础
- [第 03 章：SQLAlchemy + Alembic ORM](03-sqlalchemy-alembic-orm.md) — 数据库迁移
