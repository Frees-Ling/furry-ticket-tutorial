# 商业级架构升级：CMS / 排队 / 会员 / 抽签 / 风控

## 概述

从基础票务系统升级为商业级平台：支持内容管理（CMS）、排队系统、会员积分、资源抽签分配、四域风控、单活动身份模式。

## 新增模块总览

```
CMS              → ContentPage + ContentBlock + ContentPageVersion（宣发页面管理）
Queue            → Redis ZSET 公平排队（先到先得 + TTL 自动释放）
Membership       → PointsLog + 会员等级 + 积分制度
Referral         → 邀请码 + 邀请关系 + 首单奖励
Allocation       → 抽签/预约/排队/混合资源分配
RiskControl      → 身份/设备/行为/支付四域风控
UserEventProfile → 单活动模式下用户唯一身份
SSR (Jinja2)     → SEO 友好的服务端渲染公开页面
```

## 架构原则

### 原则一：职责彻底解耦

```
TicketType    → 只定义"卖什么"（商品属性）
Allocation    → 只定义"怎么分配资源"（抽签/预约/排队/候补）
Order/Item    → 只记录"交易与审计"（价格/权益快照）
Inventory     → 保证"库存一致性"（原子扣减/幂等/回滚）
RiskControl   → 输出"风险评分与决策"（四域统一）
UserEventProfile → 记录"用户在活动中的身份"
```

### 原则二：全系统状态机化

所有核心实体持有显式状态字段，只能通过 Service 层方法推进，禁止直接 UPDATE status。

### 原则三：全链路可追踪

每个关键操作携带三个标识：
- `reservation_id` — 库存预占
- `idempotency_key` — 请求幂等
- `operation_trace_id` — 链路追踪

### 原则四：配置驱动

所有业务规则（抽签/风控/退款/展示）通过配置中心管理，禁止写死。

---

## 1. CMS 内容管理系统

### 1.1 模型

**ContentPage** — 宣发页面（状态机：draft / published / scheduled）

```python
class ContentPage(Base):
    __tablename__ = "content_pages"
    id, event_id, slug, title, seo_title, seo_description
    og_image, canonical_url, status, published_at, scheduled_at
    version, created_by, created_at, updated_at
```

**ContentBlock** — 区块（16 种动态类型，JSON 数据）

```python
class ContentBlock(Base):
    __tablename__ = "content_blocks"
    id, page_id, type, data_json, sort_order, is_visible
    # type: hero/text/image/gallery/faq/cta/button/notice/
    #       divider/ticket_pricing/schedule/guests/map/
    #       sponsor/timeline/embedded_html
```

**ContentPageVersion** — 版本快照（发布时保存完整页面）

```python
class ContentPageVersion(Base):
    __tablename__ = "content_page_versions"
    id, page_id, version, snapshot_json, created_by, created_at
```

### 1.2 状态流转

```
draft ──→ published ←──→ scheduled ──→ published
   ↑          │
   └──←──── unpublish
```

方法：`publish_page()` / `unpublish_page()` / `schedule_publish_page()` / `rollback_to_version()`

### 1.3 API 端点

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/pages/{slug}` | 公开页面（SSR + JSON） |
| POST | `/admin/pages` | 创建页面 |
| GET | `/admin/pages` | 页面列表 |
| PUT | `/admin/pages/{id}` | 更新页面 |
| DELETE | `/admin/pages/{id}` | 删除页面 |
| POST | `/admin/pages/{id}/publish` | 发布（创建版本快照） |
| POST | `/admin/pages/{id}/unpublish` | 下架 |
| POST | `/admin/pages/{id}/schedule-publish` | 定时发布 |
| POST | `/admin/pages/{id}/preview-token` | 生成预览 token |
| POST | `/admin/pages/{id}/blocks` | 添加区块 |
| PUT | `/admin/blocks/{id}` | 更新区块 |
| DELETE | `/admin/blocks/{id}` | 删除区块 |
| PUT | `/admin/pages/{id}/blocks/reorder` | 重排区块 |
| GET | `/admin/pages/{id}/versions` | 版本列表 |
| GET | `/admin/pages/{id}/versions/{vid}` | 版本详情 |
| POST | `/admin/pages/{id}/rollback/{vid}` | 回滚版本 |
| GET | `/admin/pages/{id}/diff?v1=X&v2=Y` | 版本差异 |

### 1.4 预览机制

预览 token 通过 Redis 存储（默认 1h 过期），携带 token 访问可查看未发布内容。

---

## 2. 排队系统

### 2.1 原理

基于 Redis ZSET（有序集合）实现的公平排队：
- `ZADD queue:{event_id}:{ticket_type_id} timestamp user_id`
- `ZPOPMIN` 批量取出最早排队用户
- 支持 TTL 超时自动释放队列位置

### 2.2 状态机

```
waiting → selected → processing → completed
    │                    │
    └→ expired └→ cancelled
```

### 2.3 API 端点

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/queue/register` | 加入排队 |
| GET | `/queue/status` | 查看排队状态 |
| POST | `/queue/cancel` | 取消排队 |
| POST | `/queue/complete` | 完成处理 |

---

## 3. 会员与积分系统

### 3.1 模型

**User 扩展字段**：
```python
points, total_points_earned, lifetime_value
invite_code, referred_by, membership_level
```

**MembershipLevel** — 会员等级配置：
```python
level: str  # bronze/silver/gold/platinum
name: str   # 显示名称
min_lifetime_value, min_points  # 升级条件
benefits: JSON  # 权益配置
```

**PointsLog** — 积分流水（每次变动记录）：
```python
user_id, action, points, balance_after
reference_type, reference_id, remark
```

**Referral** — 邀请关系：
```python
referrer_id, invite_code, referred_user_id
reward_status: pending/awarded/expired
reward_points: 200
```

### 3.2 会员等级自动升级

下单支付成功后，`update_lifetime_value()` 被调用，触发 `_check_level_upgrade()` 根据 lifetime_value + total_points_earned 自动升级。

### 3.3 邀请流程

1. 用户生成唯一邀请码（8 位 hex）
2. 新用户注册时携带邀请码 → `process_referral()` 建立关系
3. 被邀请人完成首单 → `award_referral_points()` 奖励邀请人积分

### 3.4 API 端点

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/membership/my` | 我的会员信息 |
| GET | `/membership/points-log` | 积分流水 |
| GET | `/referral/my-code` | 我的邀请码 |
| GET | `/referral/statistics` | 邀请统计 |
| GET | `/admin/membership/levels` | 会员等级列表 |
| POST | `/admin/membership/levels` | 创建会员等级 |
| PUT | `/admin/membership/levels/{id}` | 更新会员等级 |

---

## 4. 资源分配系统（Allocation）

### 4.1 四种分配策略

| resource_type | 适用场景 | 算法 |
|-------------|---------|------|
| lottery | 限量高阶票/酒店/赞助 | Fisher-Yates shuffle |
| reservation | 前排座位/限量礼包 | 先到先得 + 候补 |
| queue | 等待名单/现场补位 | FIFO 队列 |
| hybrid | 先到先得 + 部分抽签 | 混合比例配置 |

### 4.2 Allocation 状态机

```
pending → registration → processing → result → payment → completed
                  ↑                        ↓
                  └── cancelled ←─ expired ┘
```

### 4.3 AllocationRegistration 状态机

```
registered → won → paid → confirmed
    │          │       └→ expired → waitlist
    │          └→ lost
    └→ cancelled
```

### 4.4 可复现抽签

- Fisher-Yates shuffle + 保存随机种子
- `draw_seed` 字段记录种子哈希
- 给定 seed 可验证中签结果

---

## 5. 单活动模式（Single Event）

### 5.1 概念

两种活动模式通过 `event.mode` 字段切换：
- `multi_event`（默认）— 用户可买多个票种/参与多个活动
- `single_event` — 每个用户一个身份（如 furry 展会）

### 5.2 UserEventProfile

```python
class UserEventProfile(Base):
    user_id, event_id
    identity_tier: str  # 普通/赞助/VIP/工作人员
    badge_name, id_number_hash, device_fingerprint
    benefits_snapshot: JSON  # 下单时权益快照
    status: pending → approved → confirmed → checked_in
            → rejected → cancelled

    # 唯一约束
    UNIQUE(user_id, event_id)          # 一人一个身份
    UNIQUE(event_id, id_number_hash)   # 一身份证一次
```

### 5.3 UI 变化

```
multi_event           →  single_event
─────────────────────────────────
"购买门票"            →  "申请身份"
"数量选择器"          →  "身份确认（仅限一个）"
"立即购买"            →  "提交申请"
"订单列表"            →  "我的身份"
```

---

## 6. 四域风控系统

### 6.1 评分模型

| 域 | 检测项 | 分值 |
|---|--------|------|
| IdentityRisk | 多账号/一证多报/实名不一致 | 0-30 |
| DeviceRisk | 同设备多账号/代理IP | 0-25 |
| BehaviorRisk | 下单速度/频率/取消率 | 0-25 |
| PaymentRisk | 支付实名/金额异常 | 0-20 |

### 6.2 策略引擎

| 总分 | 决策 | single_event |
|------|------|-------------|
| 0-30 | allow（放行） | allow |
| 30-60 | captcha（验证码） | captcha |
| 60-85 | review（人工审核） | reject |
| 85+ | reject（拒绝） | reject |

---

## 7. SSR 服务端渲染

### 7.1 技术选型

- Jinja2Templates（FastAPI 内置集成）
- Bootstrap 5（CDN 引入）
- 16 种区块类型各有独立 partial 模板
- mistune 支持区块内容 Markdown 渲染

### 7.2 区块模板通过 `include_block` 渲染

```jinja
{% for block in blocks %}
    {% include_block block %}
{% endfor %}
```

### 7.3 公开页面路由

| 路径 | 说明 |
|------|------|
| `/` | 首页（活动列表）|
| `/events` | 活动列表 |
| `/events/{id}` | 活动详情 |
| `/pages/{slug}` | CMS 宣发页面 |
| `/pages/{slug}?preview_token=X` | 预览模式 |

---

## 8. 数据库表结构（新增）

| 表名 | 说明 | 所属模块 |
|------|------|---------|
| content_pages | CMS 宣发页面 | CMS |
| content_blocks | 区块数据 | CMS |
| content_page_versions | 版本快照 | CMS |
| guests | 嘉宾库 | 日程 |
| event_guests | 活动-嘉宾关联 | 日程 |
| schedules | 活动日程 | 日程 |
| membership_levels | 会员等级配置 | 会员 |
| points_logs | 积分流水 | 会员 |
| referrals | 邀请关系 | 会员 |
| event_subscriptions | 活动订阅 | 通知 |
| allocations | 资源分配 | 分配 |
| allocation_registrations | 分配注册 | 分配 |
| stock_operations | 库存操作日志 | 库存 |
| user_event_profiles | 用户活动身份 | 单活动 |
| device_fingerprints | 设备指纹 | 风控 |

### 8.1 字段类型注意事项

MySQL 8 要求外键列类型与主键完全匹配：
- `events.id` = BIGINT → 引用它的 FKs 必须也是 BIGINT
- `users.id` = BIGINT → 引用它的 FKs 必须也是 BIGINT
- `ticket_types.id` = INTEGER → 引用它的 FKs 必须也是 INTEGER
- `content_pages.id` = INTEGER → 引用它的 FKs 必须也是 INTEGER
- `guests.id` = INTEGER → 引用它的 FKs 必须也是 INTEGER
- `allocations.id` = INTEGER → 引用它的 FKs 必须也是 INTEGER

---

## 9. 后台 Worker 任务

ARQ Worker 新增任务：

| 任务 | 触发 | 说明 |
|------|------|------|
| release_expired_order | 订单超时 | 释放库存 + 标记取消 |
| advance_allocation | 每分钟 | 推进 allocation 状态 |
| draw_lottery | 抽签开始 | 执行 Fisher-Yates 抽签 |
| notify_waitlist | 支付超时 | 通知候补用户 |
| auto_review_profiles | 每 5 分钟 | 自动审核身份申请 |
| compensation_check | 每 10 分钟 | 补偿检查一致性 |

---

## 10. 配置中心

业务规则通过配置中心统一管理（`/admin/config`），配置项：

```json
{
  "sale_rules": {
    "max_per_user_default": 10,
    "min_payment_window_minutes": 15
  },
  "refund_rules": {
    "deadline_before_event_hours": 48,
    "fee_percentage": 0.1
  },
  "risk_thresholds": {
    "allow_max": 30,
    "captcha_max": 60,
    "review_max": 85
  },
  "allocation_rules": {
    "default_waitlist_ratio": 1.0,
    "payment_window_hours": 24
  }
}
```

---

## 相关文件清单

### 模型 (13 个新建)
- `app/models/content.py` — ContentPage, ContentBlock, ContentPageVersion
- `app/models/schedule.py` — Guest, EventGuest, Schedule
- `app/models/allocation.py` — Allocation, AllocationRegistration
- `app/models/membership_level.py` — MembershipLevel
- `app/models/points_log.py` — PointsLog
- `app/models/referral.py` — Referral
- `app/models/event_subscription.py` — EventSubscription
- `app/models/stock_operation.py` — StockOperation
- `app/models/user_event_profile.py` — UserEventProfile
- `app/models/device_fingerprint.py` — DeviceFingerprint

### 服务 (12 个新建)
- `app/services/content_service.py` — CMS 业务逻辑
- `app/services/schedule_service.py` — 日程业务逻辑
- `app/services/jinja_renderer.py` — SSR 渲染引擎
- `app/services/queue_service.py` — 排队系统
- `app/services/membership_service.py` — 会员/积分/邀请
- `app/services/allocation_service.py` — 资源分配 + 抽签
- `app/services/lottery_algo.py` — Fisher-Yates 抽签算法
- `app/services/profile_service.py` — UserEventProfile
- `app/services/risk_service.py` — 风控策略引擎
- `app/services/risk_identity.py` — 身份风控
- `app/services/risk_device.py` — 设备风控
- `app/services/risk_behavior.py` — 行为风控
- `app/services/risk_payment.py` — 支付风控
- `app/services/device_service.py` — 设备指纹服务

### 路由 (6 个新建)
- `app/routers/content.py` — CMS API
- `app/routers/schedule_routes.py` — 日程 API
- `app/routers/queue.py` — 排队 API
- `app/routers/membership.py` — 会员 API
- `app/routers/allocation.py` — 资源分配 API
- `app/routers/event_profiles.py` — 活动身份 API
- `app/routers/admin_config.py` — 配置中心 API
- `app/routers/ssr_pages.py` — SSR 公开页面

### 前端 Vue 组件 (10 个新建)
- `src/views/admin/PageManageView.vue` — CMS 页面管理
- `src/views/admin/PageEditorView.vue` — 页面编辑器
- `src/components/admin/BlockEditor.vue` — 区块编辑器
- `src/components/BlockRenderer.vue` — 区块渲染器
- `src/views/PageView.vue` — 公开页面
- `src/components/TicketCard.vue` — 票种卡片
- `src/components/TicketStatusBadge.vue` — 状态标签
- `src/components/BenefitsTagGroup.vue` — 权益标签
- `src/views/EventApplyView.vue` — 身份申请
- `src/views/MyProfileView.vue` — 我的身份
- `src/views/admin/ProfileReviewView.vue` — 身份审核
- `src/views/admin/ConfigCenterView.vue` — 配置中心
- `src/views/admin/TicketTypeManageView.vue` — 票种管理

### Composable
- `src/composables/useTicketStatus.js` — 统一状态计算
- `src/composables/useTicketFilters.js` — 排序筛选

### 模板
- `app/templates/base.html` — 基础布局
- `app/templates/pages/page.html` — 页面模板
- `app/templates/pages/home.html` — 首页
- `app/templates/pages/event_detail.html` — 活动详情
- `app/templates/blocks/*.html` — 16 种区块 partial

---

## 迁移

新增表通过单一 migration `672c74327b19` 创建，包含所有新表和字段扩展：

```bash
alembic upgrade head
```

验证迁移状态：

```bash
alembic current
# → 672c74327b19 (head)
```
