# 管理员统一票务管理系统

> 本文档介绍管理员后台的完整功能，包括仪表盘统计、订单管理、票证查看和退票审核等。

---

## 目录

1. [功能概述](#1-功能概述)
2. [仪表盘统计](#2-仪表盘统计)
3. [订单管理](#3-订单管理)
4. [票证管理](#4-票证管理)
5. [退票管理](#5-退票管理)
6. [API 端点汇总](#6-api-端点汇总)
7. [前端集成示例](#7-前端集成示例)
8. [最佳实践](#8-最佳实践)

---

## 1. 功能概述

管理员统一票务管理系统为运营人员提供一站式管理界面，涵盖以下核心功能：

- **仪表盘**：系统运行状态概览，包括用户数、订单数、收入等关键指标
- **用户管理**：查看、搜索、封禁/解封用户，修改用户角色
- **订单管理**：查看所有订单（含筛选），查看订单详情，创建空降订单
- **票证管理**：查看所有票证，支持按活动/用户/订单筛选
- **退票审核**：审核用户的退票申请，查看退票统计

所有管理接口都需要 `admin` 角色权限，通过 JWT 中的角色信息进行鉴权。

### 架构示意

```text
用户请求 → JWT 鉴权 → admin 角色校验 → 路由处理 → 服务层 → 数据库
                                          ↓
                                    响应模型序列化
```

---

## 2. 仪表盘统计

### 2.1 统计指标说明

| 指标 | 说明 | 数据来源 |
| ------ | ------ | ---------- |
| `total_users` | 系统注册用户总数 | `COUNT(users.id)` |
| `total_events` | 活动总数 | `COUNT(events.id)` |
| `total_orders` | 订单总数 | `COUNT(orders.id)` |
| `total_revenue` | 总收入（已支付订单金额总和） | `SUM(orders.total_amount) WHERE status='paid'` |
| `pending_refunds` | 待审核退票数 | `COUNT(orders.id) WHERE refund_status='pending'` |
| `active_events` | 进行中的活动数（已发布） | `COUNT(events.id) WHERE status='published'` |
| `today_orders` | 今日新增订单数 | `COUNT(orders.id) WHERE created_at >= today` |
| `today_revenue` | 今日收入 | `SUM(orders.total_amount) WHERE created_at >= today AND status='paid'` |

### 2.2 接口详情

```text
GET /api/v1/admin/dashboard/stats
Authorization: Bearer <admin_token>
```

**响应示例**：

```json
{
  "total_users": 1250,
  "total_events": 45,
  "total_orders": 8920,
  "total_revenue": 568900.00,
  "pending_refunds": 12,
  "active_events": 23,
  "today_orders": 156,
  "today_revenue": 43200.00
}
```text

### 2.3 实现要点

```python
# 通过 SQLAlchemy aggregate 函数计算
result = await self.session.execute(
    select(func.count(Order.id))
    .where(Order.status == "paid")
)
count = result.scalar() or 0
```

- 所有统计使用单个查询各自计数，互不影响
- 使用 `func.coalesce()` 处理 NULL 值，确保聚合结果为 `0` 而非 `None`
- `today_start` 使用 UTC 时间当天的 0 点

---

## 3. 订单管理

### 3.1 订单筛选

支持多维度的订单筛选，所有参数均为可选的 query parameters：

| 参数 | 类型 | 说明 |
| ------ | ------ | ------ |
| `status` | string | 订单状态：`pending_payment` / `paid` / `cancelled` / `refunding` / `refunded` |
| `order_type` | string | 订单类型：`normal` / `admin_assigned` |
| `refund_status` | string | 退款状态：`pending` / `approved` / `rejected` |
| `user_id` | int | 用户 ID |
| `event_id` | int | 活动 ID |
| `date_from` | datetime | 创建时间起始 |
| `date_to` | datetime | 创建时间截止 |
| `page` | int | 页码（默认 1） |
| `size` | int | 每页数量（默认 20，最大 100） |

### 3.2 接口详情

**订单列表**：

```text
GET /api/v1/admin/orders?status=paid&order_type=normal&page=1&size=20
Authorization: Bearer <admin_token>
```

**响应示例**：

```json
{
  "items": [
    {
      "id": 1,
      "order_no": "20260513123456",
      "user_id": 42,
      "event_id": 7,
      "quantity": 2,
      "total_amount": 1360.00,
      "discount_amount": 0,
      "status": "paid",
      "order_type": "normal",
      "payment_method": "alipay",
      "trade_no": "2026051322001487651234",
      "paid_at": "2026-05-13T12:00:00+00:00",
      "created_at": "2026-05-13T11:55:00+00:00",
      "updated_at": "2026-05-13T12:00:00+00:00"
    }
  ],
  "total": 1,
  "page": 1,
  "size": 20,
  "total_pages": 1
}
```text

**订单详情**（无归属校验）：

```

GET /api/v1/admin/orders/{order_id}
Authorization: Bearer <admin_token>

```text

### 3.3 空降订单

管理员可以为指定用户创建订单（无需用户操作）：

```

POST /api/v1/admin/orders
Authorization: Bearer <admin_token>

{
  "user_id": 42,
  "event_id": 7,
  "quantity": 2,
  "remark": "内部赠票"
}

```text

空降订单的特点是：

- `order_type` 为 `admin_assigned`
- 有效期延长至 24 小时（普通订单为 15 分钟）
- 不经过 Redis 库存扣减，直接插入数据库
- 需要用户自行支付

---

## 4. 票证管理

### 4.1 票证筛选

| 参数 | 类型 | 说明 |
| ------ | ------ | ------ |
| `event_id` | int | 活动 ID |
| `user_id` | int | 用户 ID |
| `order_id` | int | 订单 ID |
| `used` | bool | 使用状态 |
| `page` | int | 页码（默认 1） |
| `size` | int | 每页数量（默认 20，最大 100） |

### 4.2 接口详情

```

GET /api/v1/admin/tickets?event_id=7&used=false&page=1&size=20
Authorization: Bearer <admin_token>

```text

**响应示例**：

```json
{
  "items": [
    {
      "id": 100,
      "order_id": 1,
      "event_id": 7,
      "user_id": 42,
      "ticket_no": "20260513123456-1",
      "seat_info": "stand",
      "price": 680.00,
      "qr_code": "https://example.com/qr/abc123",
      "used": false,
      "created_at": "2026-05-13T12:00:00+00:00",
      "updated_at": "2026-05-13T12:00:00+00:00",
      "event_title": "周杰伦2026演唱会",
      "username": "testuser",
      "order_no": "20260513123456"
    }
  ],
  "total": 100,
  "page": 1,
  "size": 20,
  "total_pages": 5
}
```

### 4.3 实现要点

票证列表通过 SQLAlchemy 的 relationship 懒加载机制获取关联数据：

```python
for ticket in tickets:
    event_name = ticket.event.title if ticket.event else None
    username = ticket.user.username if ticket.user else None
    order_no = ticket.order.order_no if ticket.order else None
```text

注意懒加载可能触发额外的 SQL 查询，在列表数据量大时需要考虑使用 `selectinload` 或 `joinedload` 进行预加载优化。

---

## 5. 退票管理

### 5.1 退票流程

```

用户申请退票 → refund_status: pending
                    ↓
          管理员审核退票
          /            \
      通过            拒绝
    status: refunded  status: paid (恢复)
    refund_status: approved  refund_status: rejected
    归还库存

```text

### 5.2 退票统计

```

GET /api/v1/admin/refunds/stats
Authorization: Bearer <admin_token>

```text

**响应示例**：

```json
{
  "total_refund_requests": 45,
  "pending_count": 12,
  "approved_count": 28,
  "rejected_count": 5,
  "total_refund_amount": 32400.00
}
```

### 5.3 退票审核

```text
POST /api/v1/admin/refunds/{order_id}/review
Authorization: Bearer <admin_token>

// 审核通过
{ "approve": true }

// 审核拒绝
{ "approve": false, "reject_reason": "超过退票期限" }
```

审核通过时会自动归还活动库存（Redis + MySQL 双写），审核拒绝时订单状态恢复为 `paid`。

---

## 6. API 端点汇总

所有管理端点均需 `admin` 角色认证，前缀为 `/api/v1/admin`。

### 用户管理

| 方法 | 路径 | 说明 |
| ------ | ------ | ------ |
| GET | `/users` | 用户列表（支持角色/封禁状态/关键词搜索和分页） |
| GET | `/users/{user_id}` | 用户详情（含订单统计） |
| PUT | `/users/{user_id}` | 修改用户信息 |
| POST | `/users/{user_id}/ban` | 封禁用户 |
| POST | `/users/{user_id}/unban` | 解封用户 |
| GET | `/users/{user_id}/login-history` | 登录历史 |
| PUT | `/users/{user_id}/role` | 修改用户角色（仅 super_admin） |
| GET | `/users/{user_id}/ip-history` | 查看用户 IP 历史（仅 super_admin） |

### 订单管理

| 方法 | 路径 | 说明 |
| ------ | ------ | ------ |
| GET | `/orders` | 订单列表（支持多维筛选和分页） |
| GET | `/orders/{order_id}` | 订单详情（无归属校验） |
| POST | `/orders` | 创建空降订单 |

### 票证管理

| 方法 | 路径 | 说明 |
| ------ | ------ | ------ |
| GET | `/tickets` | 票证列表（含关联信息） |

### 退票管理

| 方法 | 路径 | 说明 |
| ------ | ------ | ------ |
| POST | `/refunds/{order_id}/review` | 审核退票申请 |
| GET | `/refunds/stats` | 退票统计 |

### 仪表盘

| 方法 | 路径 | 说明 |
| ------ | ------ | ------ |
| GET | `/dashboard/stats` | 系统仪表盘统计 |

### 系统日志（仅 super_admin）

| 方法 | 路径 | 说明 |
| ------ | ------ | ------ |
| GET | `/logs` | 操作日志列表（分页，支持动作和日期过滤） |
| GET | `/logs/login` | 登录日志（等价于 action=login） |
| GET | `/logs/registration` | 注册日志（等价于 action=register） |
| GET | `/logs/statistics` | 日志统计数据（今日注册/登录数等） |

---

## 7. Vue 3 前端管理后台实现

本节介绍如何使用 Vue 3 + Pinia 构建完整的 Admin 管理后台。代码位于 `frontend/src/` 目录。

### 7.1 项目结构

```text
frontend/src/
├── stores/
│   └── admin.js          # Admin Pinia Store（所有 admin API 调用）
├── views/admin/
│   ├── AdminLayout.vue    # 侧边栏布局（超管有额外菜单项）
│   ├── DashboardView.vue  # 仪表盘（统计卡片 + 趋势图 + 排行 + 超管概览）
│   ├── UserManageView.vue # 用户管理（搜索/封禁/解封/编辑/IP历史）
│   ├── EventManageView.vue# 活动管理（CRUD + 售罄率）
│   ├── OrderManageView.vue# 订单管理（筛选/退款审核/空降订单）
│   ├── TicketManageView.vue# 票证管理（筛选查看）
│   └── LogManageView.vue  # 日志管理（Tabs + 筛选 + 分页, 仅超管）
├── router/
│   └── index.js          # 路由配置（含 admin 路由守卫）
└── assets/styles/
    └── admin.css          # Admin 专用样式
```

### 7.2 Admin Store（数据层）

`stores/admin.js` 封装所有 admin API 调用，每个方法对应一个后端接口：

```javascript
import { defineStore } from 'pinia'
import { ref } from 'vue'
import { api } from '../utils/api'

export const useAdminStore = defineStore('admin', () => {
  const stats = ref(null)
  const loading = ref(false)

  // ===== Dashboard =====
  async function fetchStats() {
    stats.value = await api.get('/admin/dashboard/stats')
    return stats.value
  }

  async function fetchRevenueTrends(days = 7) {
    return api.get('/admin/dashboard/revenue-trends', { params: { days } })
  }

  // ===== Users =====
  async function fetchUsers(params = {}) {
    return api.get('/admin/users', { params })
  }

  async function banUser(userId, reason) {
    return api.post(`/admin/users/${userId}/ban`, { reason })
  }

  async function unbanUser(userId) {
    return api.post(`/admin/users/${userId}/unban`)
  }

  // ===== Orders =====
  async function fetchAdminOrders(params = {}) {
    return api.get('/admin/orders', { params })
  }

  async function reviewRefund(orderId, approve, rejectReason = null) {
    return api.post(`/admin/refunds/${orderId}/review`, {
      approve, reject_reason: rejectReason
    })
  }

  async function batchReviewRefunds(orderIds, approve, rejectReason = null) {
    return api.post('/admin/refunds/batch-review', {
      order_ids: orderIds, approve, reject_reason: rejectReason
    })
  }

  // ===== Events =====
  async function createEvent(data) {
    return api.post('/events/', data)
  }

  // ===== Tickets =====
  async function fetchAdminTickets(params = {}) {
    return api.get('/admin/tickets', { params })
  }

  return {
    stats, loading,
    fetchStats, fetchRevenueTrends, fetchEventRankings,
    fetchUsers, fetchUserDetail, banUser, unbanUser, updateUser, fetchLoginHistory,
    fetchAdminEvents, createEvent, updateEvent, deleteEvent,
    fetchAdminOrders, fetchOrderDetail, createAdminOrder,
    reviewRefund, batchReviewRefunds, fetchRefundStats,
    fetchAdminTickets,
  }
})
```text

**要点解析**：

- 使用 Pinia 的 `defineStore` 组合式 API（类似 Vue 3 的 `setup()`）
- 每个方法返回 `api.get/post` 的结果，`api` 是配置好的 axios 实例（自动注入 JWT token，自动处理 401 刷新）
- `fetchStats` 将结果存入 `stats.value`，其他方法直接返回 Promise 让组件处理

### 7.3 Admin 路由与权限守卫

`router/index.js` 中 admin 路由配置：

```javascript
// Admin 子路由（嵌套在 AdminLayout 中）
{
  path: '/admin',
  component: () => import('../views/admin/AdminLayout.vue'),
  meta: { requiresAuth: true, requiresAdmin: true },
  children: [
    { path: '', name: 'admin-dashboard', component: () => import('../views/admin/DashboardView.vue') },
    { path: 'users', name: 'admin-users', component: () => import('../views/admin/UserManageView.vue') },
    { path: 'events', name: 'admin-events', component: () => import('../views/admin/EventManageView.vue') },
    { path: 'orders', name: 'admin-orders', component: () => import('../views/admin/OrderManageView.vue') },
    { path: 'tickets', name: 'admin-tickets', component: () => import('../views/admin/TicketManageView.vue') },
  ],
}
```

路由守卫（`router.beforeEach`）：

```javascript
router.beforeEach((to, from, next) => {
  const token = localStorage.getItem('token')

  // 需要认证但未登录 → 跳转登录页
  if (to.meta.requiresAuth && !token) {
    next({ name: 'login', query: { redirect: to.fullPath } })
    return
  }

  // 需要 admin 角色 → 检查 localStorage 中的用户信息
  if (to.meta.requiresAdmin && token) {
    const stored = localStorage.getItem('user')
    if (stored) {
      const user = JSON.parse(stored)
      if (user.role !== 'admin') {
        next({ name: 'home' })  // 非管理员跳回首页
        return
      }
    }
  }
  next()
})
```text

**关键点**：

- `meta.requiresAdmin` — 标记需要 admin 权限的路由
- 用户信息在登录后通过 `auth.fetchUser()` 获取并存入 `localStorage.setItem('user', JSON.stringify(res))`
- 普通用户访问 `/admin` 会被重定向到首页

### 7.4 AdminLayout 侧边栏布局

`views/admin/AdminLayout.vue` 是 Admin 所有页面的容器：

```vue
<template>
  <div class="admin-layout">
    <!-- 侧边栏 -->
    <aside class="admin-sidebar">
      <div class="admin-sidebar-title">管理面板</div>
      <router-link to="/admin" class="admin-sidebar-link"
        :class="{ active: $route.path === '/admin' }">
        <span class="admin-sidebar-icon">📊</span>
        <span>仪表盘</span>
      </router-link>
      <router-link to="/admin/users" class="admin-sidebar-link"
        :class="{ active: $route.path.startsWith('/admin/users') }">
        <span class="admin-sidebar-icon">👥</span>
        <span>用户管理</span>
      </router-link>
      <!-- 更多菜单项... -->
    </aside>
    <!-- 内容区 -->
    <main class="admin-content">
      <router-view />  <!-- 子路由在此渲染 -->
    </main>
  </div>
</template>
```

**布局说明**：

- 侧边栏固定 220px 宽度，左侧深色背景
- 内容区 `margin-left: 220px`，灰色背景（`#f5f5f5`）
- `router-link` 的 `active` 类高亮当前菜单
- 响应式：小屏幕侧边栏缩窄到 60px（只显示图标）

**角色感知侧边栏**：超级管理员（`super_admin`）可以看到额外的"超级管理"区块，包含日志管理入口；普通管理员则看不到：

```vue
<!-- 所有管理员看到的菜单项 -->
<router-link to="/admin/users">用户管理</router-link>
<router-link to="/admin/orders">订单管理</router-link>

<!-- 仅超级管理员能看到 -->
<template v-if="currentUserRole === 'super_admin'">
  <div>超级管理</div>
  <router-link to="/admin/logs">
    <span>日志管理</span>
    <span>super</span>
  </router-link>
</template>
```text

角色状态从 `localStorage` 中的用户信息获取：

```javascript
const currentUserRole = computed(() => {
  try {
    const user = JSON.parse(localStorage.getItem('user') || '{}')
    return user.role || ''
  } catch {
    return ''
  }
})
```

### 7.5 仪表盘页面（DashboardView）

展示系统关键指标，包含统计卡片、收入趋势图、活动销售排行。

**超级管理员额外内容**：如果当前用户角色为 `super_admin`，仪表盘还会显示"今日概览"（今日注册数、今日登录数、总用户数、总收入）和"最近操作"（最近 10 条操作日志）：

```vue
<!-- 今日概览（仅超管可见） -->
<div v-if="dailyStats && isSuperAdmin" style="margin-bottom:20px">
  <div class="stats-grid">
    <div class="stat-card">
      <span class="stat-card-label">今日注册数</span>
      <span class="stat-card-value">{{ dailyStats.today_registrations }}</span>
    </div>
    <div class="stat-card">
      <span class="stat-card-label">今日登录数</span>
      <span class="stat-card-value">{{ dailyStats.today_logins }}</span>
    </div>
    <div class="stat-card">
      <span class="stat-card-label">总用户数</span>
      <span class="stat-card-value">{{ dailyStats.total_users }}</span>
    </div>
    <div class="stat-card">
      <span class="stat-card-label">总收入</span>
      <span class="stat-card-value">{{ formatMoney(dailyStats.total_revenue) }}</span>
    </div>
  </div>
</div>

<!-- 最近操作（仅超管可见） -->
<div v-if="recentOps.length && isSuperAdmin">
  <div>最近操作</div>
  <ul>
    <li v-for="op in recentOps" :key="op.id">
      <span>{{ logActionLabel(op.action) }}</span>
      {{ op.username }} - {{ op.details }}
      {{ op.ip_address }}
      {{ formatDate(op.created_at) }}
    </li>
  </ul>
</div>
```text

`isSuperAdmin` 计算属性从 localStorage 中的用户信息获取：

```javascript
const isSuperAdmin = (() => {
  try {
    const user = JSON.parse(localStorage.getItem('user') || '{}')
    return user.role === 'super_admin'
  } catch {
    return false
  }
})()
```

仪表盘核心代码：

```vue
<template>
  <div>
    <div class="page-title">仪表盘</div>

    <!-- 统计卡片 -->
    <div class="stats-grid" v-if="stats">
      <div class="stat-card">
        <span class="stat-card-label">用户总数</span>
        <span class="stat-card-value">{{ stats.total_users }}</span>
      </div>
      <div class="stat-card">
        <span class="stat-card-label">订单总数</span>
        <span class="stat-card-value">{{ stats.total_orders }}</span>
      </div>
      <div class="stat-card">
        <span class="stat-card-label">总营收</span>
        <span class="stat-card-value">¥{{ formatMoney(stats.total_revenue) }}</span>
      </div>
      <div class="stat-card">
        <span class="stat-card-label">待处理退款</span>
        <span class="stat-card-value"
          :style="{ color: stats.pending_refunds > 0 ? '#ff4d4f' : '#1a1a1a' }">
          {{ stats.pending_refunds }}
        </span>
      </div>
    </div>

    <!-- 收入趋势柱状图（纯 CSS，无第三方库） -->
    <div class="chart-container" v-if="trendData">
      <div class="chart-title">收入趋势</div>
      <div class="bar-chart">
        <div class="bar-item" v-for="(rev, i) in trendData.revenues" :key="i">
          <div class="bar" :style="{ height: barHeight(rev) + '%' }"></div>
          <span class="bar-label">{{ trendData.dates[i].slice(5) }}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { useAdminStore } from '../../stores/admin'

const adminStore = useAdminStore()
const stats = ref(null)

function formatMoney(v) {
  return Number(v || 0).toLocaleString('zh-CN', {
    minimumFractionDigits: 2, maximumFractionDigits: 2
  })
}

onMounted(async () => {
  stats.value = await adminStore.fetchStats()
  trendData.value = await adminStore.fetchRevenueTrends(14)
})
</script>
```text

**设计要点**：

- 使用 CSS `flex` 实现的柱状图，无需 Chart.js 等第三方库
- 柱高通过 `barHeight()` 计算（相对于最大值的百分比）
- 统计卡片用 `grid` 布局，自动响应式排列

### 7.6 用户管理页面（UserManageView）

完整的用户 CRUD + 封禁/解封功能：

```vue
<template>
  <div>
    <div class="page-title">用户管理</div>

    <!-- 搜索过滤栏 -->
    <div class="filter-bar">
      <input v-model="keyword" placeholder="搜索用户名/邮箱/手机号"
        @keyup.enter="search" />
      <select v-model="roleFilter" @change="search">
        <option value="">全部角色</option>
        <option value="user">普通用户</option>
        <option value="admin">管理员</option>
      </select>
      <select v-model="bannedFilter" @change="search">
        <option value="">全部状态</option>
        <option value="false">正常</option>
        <option value="true">已封禁</option>
      </select>
      <button class="btn btn-primary" @click="search">搜索</button>
    </div>

    <!-- 用户表格 -->
    <table class="data-table">
      <thead>
        <tr>
          <th>ID</th>
          <th>用户名</th>
          <th>邮箱</th>
          <th>角色</th>
          <th>状态</th>
          <th>操作</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="u in users" :key="u.id">
          <td>{{ u.id }}</td>
          <td><strong>{{ u.username }}</strong></td>
          <td>{{ u.email || '-' }}</td>
          <td><span class="status-badge">{{ u.role }}</span></td>
          <td>
            <span v-if="u.is_banned" style="color:#ff4d4f">已封禁</span>
            <span v-else style="color:#52c41a">正常</span>
          </td>
          <td>
            <button class="btn btn-sm btn-default" @click="viewDetail(u)">详情</button>
            <button v-if="!u.is_banned" class="btn btn-sm btn-warning"
              @click="confirmBan(u)">封禁</button>
            <button v-else class="btn btn-sm btn-success"
              @click="handleUnban(u)">解封</button>
          </td>
        </tr>
      </tbody>
    </table>

    <!-- 分页 -->
    <div class="pagination" v-if="totalPages > 1">
      <button :disabled="page <= 1" @click="goPage(page-1)">上一页</button>
      <button v-for="p in pages" :key="p"
        :class="{ active: p === page }" @click="goPage(p)">{{ p }}</button>
      <button :disabled="page >= totalPages" @click="goPage(page+1)">下一页</button>
      <span class="pagination-info">共 {{ total }} 条</span>
    </div>

    <!-- 封禁弹窗 -->
    <div class="modal-overlay" v-if="banTarget" @click.self="banTarget=null">
      <div class="modal-content">
        <div class="modal-title">封禁用户 - {{ banTarget.username }}</div>
        <textarea v-model="banReason" placeholder="请输入封禁原因..."></textarea>
        <div class="modal-actions">
          <button class="btn btn-default" @click="banTarget=null">取消</button>
          <button class="btn btn-danger" @click="handleBan">确认封禁</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import { useAdminStore } from '../../stores/admin'

const adminStore = useAdminStore()
const users = ref([])
const page = ref(1)
const keyword = ref('')

// 封禁操作
async function handleBan() {
  await adminStore.banUser(banTarget.value.id, banReason.value || '管理员封禁')
  banTarget.value = null
  fetchUsers()  // 刷新列表
}

// 解封操作
async function handleUnban(u) {
  if (!confirm(`确认解封「${u.username}」？`)) return
  await adminStore.unbanUser(u.id)
  fetchUsers()
}

onMounted(fetchUsers)
</script>
```

**关键模式**：

- **筛选栏**：`v-model` 双向绑定 + `@change="search"` 自动触发搜索
- **分页器**：计算 `totalPages`，渲染页码按钮，点击时 `goPage(p)`
- **弹窗**：使用 `modal-overlay` + `modal-content` 实现模态框
- **响应式表格**：`data-table` 样式自带 hover 高亮和交替行色

**超级管理员增强功能**：

1. **行内角色编辑**：超级管理员可以在表格中直接通过下拉框修改用户角色，无需进入弹窗

```vue
<select v-if="isSuperAdmin" v-model="u.role"
  class="role-select" @change="changeUserRole(u)">
  <option value="user">普通用户</option>
  <option value="organizer">组织者</option>
  <option value="admin">管理员</option>
  <option value="super_admin">超级管理员</option>
</select>
<span v-else class="admin-role-badge">  <!-- 普通管理员只读显示 -->
  {{ roleLabel(u.role) }}
</span>
```text

1. **IP 地址列 + IP 历史弹窗**：表格中显示用户的 `last_login_ip`，点击可弹出 IP 历史模态框

```vue
<!-- IP 地址列（可点击） -->
<td>
  <span v-if="u.last_login_ip" class="ip-address-link"
    @click="showIPHistory(u)">{{ u.last_login_ip }}</span>
</td>

<!-- IP 历史模态框 -->
<div class="modal-overlay" v-if="ipHistoryTarget">
  <table class="ip-history-table">
    <thead>
      <tr><th>IP地址</th><th>用户代理</th><th>登录时间</th><th>状态</th></tr>
    </thead>
    <tbody>
      <tr v-for="h in ipHistoryList" :key="h.id">
        <td style="font-family:monospace">{{ h.ip_address }}</td>
        <td>{{ h.user_agent }}</td>
        <td>{{ formatDate(h.login_at) }}</td>
        <td>{{ h.success ? '成功' : '失败' }}</td>
      </tr>
    </tbody>
  </table>
</div>
```

### 7.7 订单管理页面（OrderManageView）

支持多种筛选、退款审核、批量操作和空降订单创建：

```vue
<template>
  <div>
    <!-- 筛选栏 -->
    <div class="filter-bar">
      <select v-model="statusFilter" @change="search">
        <option value="">全部状态</option>
        <option value="pending_payment">待支付</option>
        <option value="paid">已支付</option>
        <option value="cancelled">已取消</option>
        <option value="refunded">已退款</option>
      </select>
      <select v-model="refundFilter" @change="search">
        <option value="">退款状态</option>
        <option value="pending">待审核</option>
        <option value="approved">已通过</option>
        <option value="rejected">已拒绝</option>
      </select>
      <input v-model.number="userIdFilter" type="number" placeholder="用户ID" />
      <button class="btn btn-primary" @click="search">搜索</button>
      <!-- 批量操作按钮 -->
      <button v-if="selectedOrders.length" class="btn btn-sm btn-success"
        @click="batchApprove">
        批量通过({{ selectedOrders.length }})
      </button>
    </div>

    <table class="data-table">
      <thead>
        <tr>
          <th><input type="checkbox" @change="toggleAll" /></th>
          <th>订单号</th>
          <th>金额</th>
          <th>状态</th>
          <th>退款状态</th>
          <th>操作</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="o in orders" :key="o.id"
          :style="o.refund_status === 'pending' ? { background: '#fffbe6' } : {}">
          <td><input type="checkbox" v-model="selectedOrders" :value="o.id" /></td>
          <td style="font-family:monospace">{{ o.order_no }}</td>
          <td><strong>¥{{ formatMoney(o.total_amount) }}</strong></td>
          <td><span class="status-badge" :class="o.status">{{ statusLabel(o.status) }}</span></td>
          <td>
            <span v-if="o.refund_status === 'pending'" style="color:#faad14">待审核</span>
            <span v-else-if="o.refund_status === 'approved'" style="color:#52c41a">已通过</span>
            <span v-else-if="o.refund_status === 'rejected'" style="color:#ff4d4f">已拒绝</span>
          </td>
          <td>
            <button v-if="o.refund_status === 'pending'" class="btn btn-sm btn-success"
              @click="handleReview(o.id, true)">通过</button>
            <button v-if="o.refund_status === 'pending'" class="btn btn-sm btn-danger"
              @click="handleReview(o.id, false)">拒绝</button>
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>
```text

**实现要点**：

- **批量选择**：使用 `selectedOrders` 数组 + checkbox 的 `v-model`
- **待审核高亮**：`refund_status === 'pending'` 的行背景色为 `#fffbe6`（浅黄色）
- **退款审核**：调用 `adminStore.reviewRefund(orderId, approve)` 后刷新列表
- **批量审核**：`batchReviewRefunds` 遍历 `selectedOrders` 数组

### 7.8 导航栏集成（NavBar）

在顶部导航栏中展示管理员入口和用户头像下拉菜单：

```vue
<template>
  <nav class="navbar">
    <router-link to="/" class="nav-brand">🎫 票务系统</router-link>
    <div class="nav-links">
      <router-link to="/events">活动</router-link>
      <router-link v-if="auth.isAdmin" to="/admin" class="admin-link">
        管理后台
      </router-link>
      <div v-if="auth.isLoggedIn" class="user-menu" @click="showMenu = !showMenu">
        <span class="user-avatar">{{ auth.user?.username[0] }}</span>
        <span class="user-name">{{ auth.user?.username }}</span>
        <div class="dropdown-menu" v-if="showMenu">
          <router-link to="/admin" class="dropdown-item">管理后台</router-link>
          <router-link to="/orders" class="dropdown-item">我的订单</router-link>
          <a class="dropdown-item logout" @click="handleLogout">退出登录</a>
        </div>
      </div>
    </div>
  </nav>
</template>
```

**关键点**：

- `auth.isAdmin` — 根据 `user.role === 'admin'` 计算得出，控制管理入口的显示
- 用户头像取用户名首字母，显示在圆形背景中
- 点击外部区域关闭下拉菜单（`v-click-outside` 自定义指令）

### 7.9 样式系统（admin.css）

Admin 样式独立文件，按组件分类：

```css
/* 侧边栏 */
.admin-sidebar {
  width: 220px; background: #001529;
  position: fixed; top: 60px; left: 0; bottom: 0;
}
.admin-sidebar-link {
  display: flex; align-items: center; gap: 10px;
  padding: 10px 24px; color: rgba(255,255,255,0.65);
  text-decoration: none; transition: all 0.2s;
}
.admin-sidebar-link:hover { color: #fff; background: rgba(255,255,255,0.08); }
.admin-sidebar-link.active { color: #fff; background: #1677ff; }

/* 统计卡片网格 */
.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
  gap: 16px; margin-bottom: 24px;
}
.stat-card { background: #fff; border-radius: 8px; padding: 20px 24px; }
.stat-card-value { font-size: 28px; font-weight: 700; }

/* 数据表格 */
.data-table { width: 100%; border-collapse: collapse; background: #fff; }
.data-table th { background: #fafafa; padding: 12px 16px; font-weight: 600; }
.data-table td { padding: 12px 16px; border-bottom: 1px solid #f0f0f0; }
.data-table tr:hover td { background: #fafafa; }

/* 模态框 */
.modal-overlay {
  position: fixed; inset: 0; background: rgba(0,0,0,0.45);
  display: flex; justify-content: center; align-items: center; z-index: 1000;
}
.modal-content { background: #fff; border-radius: 8px; padding: 24px;
  min-width: 480px; max-height: 80vh; overflow-y: auto; }

/* 分页器 */
.pagination { display: flex; justify-content: center; gap: 8px; margin-top: 20px; }
.pagination button.active { background: #1677ff; border-color: #1677ff; color: #fff; }

/* 响应式 */
@media (max-width: 768px) {
  .admin-sidebar { width: 60px; }
  .admin-sidebar-link span { display: none; }
  .admin-content { margin-left: 60px; }
}
```text

### 7.10 LogManageView — 日志管理页面（仅超管）

`frontend/src/views/admin/LogManageView.vue` 是超级管理员专属的日志查看器，提供多 Tab 切换、多维度筛选和分页功能：

```vue
<template>
  <div>
    <div class="page-title">日志管理</div>

    <!-- Tabs -->
    <div class="tab-bar">
      <button class="tab-item" :class="{ active: activeTab === 'all' }"
        @click="switchTab('all')">全部日志</button>
      <button class="tab-item" :class="{ active: activeTab === 'login' }"
        @click="switchTab('login')">登录日志</button>
      <button class="tab-item" :class="{ active: activeTab === 'register' }"
        @click="switchTab('register')">注册日志</button>
      <button class="tab-item" :class="{ active: activeTab === 'operation' }"
        @click="switchTab('operation')">操作日志</button>
    </div>

    <!-- 筛选栏 -->
    <div class="filter-bar">
      <select v-model="actionFilter" @change="search">
        <option value="">全部操作类型</option>
        <option value="login">登录</option>
        <option value="register">注册</option>
        <option value="ban">封禁</option>
        <option value="unban">解封</option>
        <option value="role_change">角色变更</option>
        <option value="other">其他</option>
      </select>
      <input type="date" v-model="startDate" @change="search" />
      <input type="date" v-model="endDate" @change="search" />
      <input v-model="keyword" placeholder="搜索用户/详情..." @keyup.enter="search" />
      <button class="btn btn-primary" @click="search">搜索</button>
      <button class="btn btn-default" @click="resetFilters">重置</button>
    </div>

    <!-- 日志表格 -->
    <table class="log-table">
      <thead>
        <tr>
          <th>ID</th>
          <th>用户</th>
          <th>操作类型</th>
          <th>IP地址</th>
          <th>详情</th>
          <th>时间</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="log in logs" :key="log.id">
          <td>{{ log.id }}</td>
          <td><strong>{{ log.username || '-' }}</strong></td>
          <td>
            <span class="log-badge" :class="logBadgeClass(log.action)">
              {{ logActionLabel(log.action) }}
            </span>
          </td>
          <td style="font-family:monospace">{{ log.ip_address || '-' }}</td>
          <td :title="log.details">{{ log.details || '-' }}</td>
          <td>{{ formatDate(log.created_at) }}</td>
        </tr>
      </tbody>
    </table>

    <!-- 分页 -->
    <div class="pagination" v-if="totalPages > 1">
      <button :disabled="page <= 1" @click="goPage(page-1)">上一页</button>
      <button v-for="p in pages" :key="p" :class="{ active: p === page }" @click="goPage(p)">{{ p }}</button>
      <button :disabled="page >= totalPages" @click="goPage(page+1)">下一页</button>
      <span class="pagination-info">共 {{ total }} 条</span>
    </div>
  </div>
</template>
```

**Tab 切换逻辑**：每个 Tab 调用不同的后端 API

```javascript
async function fetchLogs() {
  const params = { page: page.value, size: size.value }
  let res
  switch (activeTab.value) {
    case 'login':
      res = await adminStore.fetchLoginLogs(params)     // GET /admin/logs/login
      break
    case 'register':
      res = await adminStore.fetchRegisterLogs(params)  // GET /admin/logs/registration
      break
    case 'operation':
      res = await adminStore.fetchLogs(params)          // GET /admin/logs（带 action 过滤）
      break
    default:
      res = await adminStore.fetchLogs(params)           // GET /admin/logs
  }
  logs.value = res.items
  total.value = res.total
}
```text

**日志类型标签映射与颜色方案**：

```javascript
function logBadgeClass(action) {
  const map = {
    login: 'log-login',       // 蓝色
    register: 'log-register', // 绿色
    ban: 'log-ban',           // 红色
    unban: 'log-ban',
    role_change: 'log-role',  // 紫色
    logout: 'log-login',
  }
  return map[action] || 'log-other'
}
```

---

## 8. 最佳实践

### 8.1 安全性

- 所有管理接口必须校验 `admin` 角色，使用 `require_admin` 依赖
- 敏感操作（封禁、退票审核）记录操作日志（谁在什么时间做了什么）
- 退票审核涉及金额和库存，必须使用事务保护数据一致性
- 订单详情管理接口不做归属校验，但需要确保只有管理员可以访问

### 8.2 性能

- 列表接口强制分页（`size` 最大 100），防止一次加载过多数据
- 统计查询使用数据库聚合函数，避免在应用层计算
- 使用 `func.count()` 而非 `len(result.all())` 避免加载全部数据行
- 对于票证列表等大量关联查询，考虑使用 SQLAlchemy 的 `selectinload` 预加载

### 8.3 代码组织

- 服务层方法遵循单一职责原则，每个方法完成一个独立功能
- 路由层只负责参数解析和响应序列化，不包含业务逻辑
- 使用 Pydantic schema 确保响应格式的一致性
- 使用 `model_validate()` (Pydantic v2) 将 ORM 模型转换为响应模型

### 8.4 异常处理

```python
# 服务层抛出特定异常
raise NotFoundError("订单不存在")

# 路由层捕获并转换为 HTTP 响应
except AppError as e:
    raise HTTPException(status_code=e.status_code, detail=e.message)
```text

统一异常处理链：

1. 服务层抛出 `AppError` 子类（`NotFoundError`、`ConflictError` 等）
2. 路由层捕获后转为 `HTTPException`
3. 全局异常处理器（`app.exception_handler(AppError)`）兜底处理

### 8.5 测试

```python
@pytest.mark.asyncio
async def test_dashboard_stats_requires_admin(client: AsyncClient):
    """仪表盘接口需要 admin 角色"""
    resp = await client.get("/api/v1/admin/dashboard/stats")
    assert resp.status_code == 401
```

---

## 9. 超级管理员角色管理与日志审计

### 9.1 三级角色体系

本系统实现了三级角色权限体系：

```text
super_admin（超级管理员）—— 最高权限
  - 所有 admin 权限
  - 修改用户角色
  - 查看 IP 历史
  - 查看操作日志
  - 仪表盘额外统计

admin（普通管理员）
  - 用户管理（封禁/解封/编辑）
  - 订单管理
  - 活动/票证管理
  - 退票审核

user（普通用户）
  - 查看活动
  - 下单/支付
  - 查看自己的订单
```

### 9.2 require_super_admin 后端依赖

后端通过 RoleChecker 实现精细化权限控制：

```python
# app/middleware/auth.py
class RoleChecker:
    def __init__(self, allowed_roles: list[str]) -> None:
        self.allowed_roles = allowed_roles

    async def __call__(self, current_user: User = Depends(get_current_user)) -> User:
        if current_user.role not in self.allowed_roles:
            raise HTTPException(status_code=403, detail="权限不足")
        return current_user

# 预定义检查器
require_admin = RoleChecker(["admin", "super_admin"])        # admin 接口
require_super_admin = RoleChecker(["super_admin"])           # 超管专属接口
require_organizer = RoleChecker(["admin", "super_admin", "organizer"])
```text

**为什么 require_admin 允许 super_admin？** 这是角色继承的体现——super_admin 拥有 admin 的全部权限。只有日志管理和角色修改等超敏感操作才使用 `require_super_admin` 限制。

### 9.3 超管专属 API 端点

| 方法 | 路径 | 权限 | 说明 |
| ------ | ------ | ------ | ------ |
| PUT | `/admin/users/{id}/role` | super_admin | 修改用户角色 |
| GET | `/admin/users/{id}/ip-history` | super_admin | 查看用户 IP 历史 |
| GET | `/admin/logs` | admin+ | 操作日志列表 |
| GET | `/admin/logs/login` | admin+ | 登录日志 |
| GET | `/admin/logs/registration` | admin+ | 注册日志 |
| GET | `/admin/logs/statistics` | admin+ | 日志统计数据 |

### 9.4 路由守卫双层防护

前端通过路由 meta 字段和守卫实现双层权限控制：

```javascript
// router/index.js
{
  path: 'logs',
  component: () => import('../views/admin/LogManageView.vue'),
  meta: {
    requiresAuth: true,
    requiresAdmin: true,       // 第一层：必须是管理员
    requiresSuperAdmin: true,  // 第二层：必须是超级管理员
    title: '日志管理'
  },
}

// 路由守卫
router.beforeEach((to, from, next) => {
  const token = localStorage.getItem('token')

  // 检查登录
  if (to.meta.requiresAuth && !token) { /* 重定向登录 */ }

  // 检查 admin 角色
  if (to.meta.requiresAdmin && token) {
    const user = JSON.parse(localStorage.getItem('user') || '{}')
    if (!['admin', 'super_admin'].includes(user.role)) {
      next({ name: 'home' })
      return
    }
  }

  // 检查 super_admin 角色
  if (to.meta.requiresSuperAdmin && token) {
    const user = JSON.parse(localStorage.getItem('user') || '{}')
    if (user.role !== 'super_admin') {
      next({ path: '/admin' })  // 普通管理员跳回管理后台
      return
    }
  }
  next()
})
```

### 9.5 审计与日志

所有敏感操作都会记录到 `operation_logs` 表，包括：

| 操作 | action 值 | 触发时机 |
| ------ | ----------- | ---------- |
| 密码登录 | `login` | 用户通过密码登录 |
| 短信登录 | `sms_login` | 用户通过验证码登录 |
| 用户注册 | `register` | 用户注册新账号 |
| 用户登出 | `logout` | 用户登出 |
| 角色修改 | `role_change` | 超管修改用户角色 |
| 封禁用户 | `ban` | 管理员封禁用户 |

这些日志通过前端 LogManageView 查看，支持按操作类型、日期范围、关键词搜索和分页浏览。
