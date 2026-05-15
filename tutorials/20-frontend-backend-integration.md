# 第20章：前后端集成

## 本章学习目标

本章将学习前端与后端完整集成，包括 CORS、JWT 管理、Axios 配置、WebSocket 客户端、支付流程和完整集成测试。

**学习路径：**

1. CORS 深入理解
2. JWT 前端管理
3. Axios 完整配置
4. WebSocket 客户端
5. 实时座位更新
6. 支付前端流程
7. 完整集成测试

---

## 1. CORS 深入理解

### 1.1 浏览器同源策略

**同源定义：** 协议 + 域名 + 端口 都相同

```text
http://localhost:8000      和 http://localhost:5173
  协议相同 ✓  域名相同 ✓   端口不同 ✗  → 跨域
```

### 1.2 简单请求 vs 预检请求

**简单请求条件（全部满足）：**

1. GET/HEAD/POST 请求
2. 仅允许的 Header：Accept/Accept-Language/Content-Language/Content-Type
3. Content-Type 只能是：application/x-www-form-urlencoded / multipart/form-data / text/plain

**预检请求（OPTIONS）：**
当条件不满足时，浏览器先发 OPTIONS 请求询问服务器是否允许真实请求。

```http
// 预检请求
OPTIONS /api/v1/orders/ HTTP/1.1
Origin: http://localhost:5173
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Authorization, Content-Type

// 服务器响应
HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://localhost:5173
Access-Control-Allow-Methods: POST, GET, OPTIONS
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
```text

### 1.3 凭据（Credentials）

```javascript
// 前端：携带 cookie 时需要
fetch('/api/user', {
    credentials: 'include',  // 或 'same-origin'
});

// 后端：允许凭据时不能设置 allow_origins=*
// 必须指定具体域名
```

### 1.4 FastAPI CORS 配置

```python
# app/main.py 中已配置
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # 开发阶段允许所有
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 生产环境应限制来源
allow_origins=[
    "https://ticket.your-domain.com",
    "http://localhost:5173",  # 开发
],
```text

---

## 2. JWT 前端管理

### 2.1 存储位置对比

| 方式 | 优点 | 缺点 | 推荐场景 |
| ------ | ------ | ------ | ---------- |
| localStorage | 简单，持久 | XSS 可读取 | Access Token |
| sessionStorage | 标签页隔离 | 关闭即消失 | 临时数据 |
| Cookie (httpOnly) | XSS 不可读 | 需要 CSRF 防护 | Refresh Token |
| 内存变量 | 最安全 | 刷新即丢失 | 当前用户信息 |

**推荐策略：**

- Access Token → localStorage（配合 HttpOnly Cookie 存 Refresh Token 更安全，但实现复杂）
- Refresh Token → localStorage 或 httpOnly Cookie
- 本项目：Access Token + Refresh Token 都存 localStorage

### 2.2 自动刷新

```javascript
// utils/api.js
let isRefreshing = false;
let refreshSubscribers = [];

function onRefreshed(newToken) {
    refreshSubscribers.forEach(cb => cb(newToken));
    refreshSubscribers = [];
}

function addRefreshSubscriber(cb) {
    refreshSubscribers.push(cb);
}

api.interceptors.response.use(
    (response) => response,
    async (error) => {
        const originalRequest = error.config;

        if (error.response?.status === 401 && !originalRequest._retry) {
            if (isRefreshing) {
                // 等待其他请求刷新完成
                return new Promise(resolve => {
                    addRefreshSubscriber(newToken => {
                        originalRequest.headers.Authorization = `Bearer ${newToken}`;
                        resolve(api(originalRequest));
                    });
                });
            }

            originalRequest._retry = true;
            isRefreshing = true;

            try {
                const refreshToken = localStorage.getItem('refresh_token');
                const res = await axios.post('/api/v1/auth/refresh', {
                    refresh_token: refreshToken,
                });

                const newToken = res.data.access_token;
                localStorage.setItem('token', newToken);
                onRefreshed(newToken);

                originalRequest.headers.Authorization = `Bearer ${newToken}`;
                return api(originalRequest);
            } catch (refreshError) {
                // 刷新失败，跳转登录
                localStorage.clear();
                window.location.href = '/login';
                return Promise.reject(refreshError);
            } finally {
                isRefreshing = false;
            }
        }

        return Promise.reject(error);
    }
);
```

---

## 3. Axios 完整配置

### 3.1 实例创建

```javascript
// utils/api.js
import axios from 'axios';

const api = axios.create({
    baseURL: '/api/v1',
    timeout: 15000,
    headers: { 'Content-Type': 'application/json' },
});

// 请求拦截器
api.interceptors.request.use(
    (config) => {
        const token = localStorage.getItem('token');
        if (token) {
            config.headers.Authorization = `Bearer ${token}`;
        }

        // 请求开始时间（用于日志）
        config.metadata = { startTime: Date.now() };
        return config;
    },
    (error) => Promise.reject(error)
);

// 响应拦截器
api.interceptors.response.use(
    (response) => {
        // 请求耗时日志
        const duration = Date.now() - response.config.metadata.startTime;
        if (duration > 1000) {
            console.warn(`慢请求 [${duration}ms]: ${response.config.url}`);
        }
        return response.data;  // 直接返回 data
    },
    async (error) => {
        // 网络错误
        if (!error.response) {
            console.error('网络连接失败');
            return Promise.reject(new Error('网络连接失败，请检查网络'));
        }

        const { status, data } = error.response;
        const message = data?.detail || getDefaultMessage(status);

        // 401 处理见自动刷新逻辑
        // 403
        if (status === 403) {
            console.warn('权限不足');
        }

        // 429 (Rate Limited)
        if (status === 429) {
            console.warn('请求过于频繁，请稍后重试');
        }

        // 500
        if (status >= 500) {
            console.error('服务器错误');
        }

        return Promise.reject(new Error(message));
    }
);

function getDefaultMessage(status) {
    const messages = {
        400: '请求参数错误',
        401: '请先登录',
        403: '权限不足',
        404: '资源不存在',
        429: '请求过于频繁',
        500: '服务器内部错误',
        502: '网关错误',
        503: '服务不可用',
    };
    return messages[status] || `请求失败 (${status})`;
}

export { api };
```text

### 3.2 API 模块封装

```javascript
// utils/api/events.js
import { api } from '../api';

export const eventApi = {
    list(params) {
        return api.get('/events/', { params });
    },
    get(id) {
        return api.get(`/events/${id}`);
    },
    create(data) {
        return api.post('/events/', data);
    },
    update(id, data) {
        return api.put(`/events/${id}`, data);
    },
    delete(id) {
        return api.delete(`/events/${id}`);
    },
};

// utils/api/orders.js
export const orderApi = {
    create(data) {
        return api.post('/orders/', data);
    },
    list(params) {
        return api.get('/orders/', { params });
    },
    get(id) {
        return api.get(`/orders/${id}`);
    },
    cancel(id) {
        return api.post(`/orders/${id}/cancel`);
    },
};

// utils/api/payments.js
export const paymentApi = {
    alipay(data) {
        return api.post('/payments/alipay', data);
    },
    refund(data) {
        return api.post('/payments/refund', data);
    },
};
```

---

## 4. WebSocket 客户端

### 4.1 连接管理

```javascript
// utils/websocket.js
export class WebSocketClient {
    constructor(url, options = {}) {
        this.url = url;
        this.options = {
            reconnectInterval: 3000,
            maxReconnects: 10,
            heartbeatInterval: 30000,
            ...options,
        };
        this.reconnectCount = 0;
        this.listeners = new Map();
        this.connect();
    }

    connect() {
        this.ws = new WebSocket(this.url);

        this.ws.onopen = () => {
            console.log('WebSocket 已连接');
            this.reconnectCount = 0;
            this.startHeartbeat();
            this.emit('connected');
        };

        this.ws.onmessage = (event) => {
            try {
                const data = JSON.parse(event.data);
                this.emit(data.type, data);
                this.emit('message', data);
            } catch (e) {
                console.error('WebSocket 消息解析失败:', e);
            }
        };

        this.ws.onclose = (event) => {
            console.log('WebSocket 断开:', event.code);
            this.stopHeartbeat();
            this.emit('disconnected');
            this.reconnect();
        };

        this.ws.onerror = (error) => {
            console.error('WebSocket 错误:', error);
            this.emit('error', error);
        };
    }

    reconnect() {
        if (this.reconnectCount >= this.options.maxReconnects) {
            console.error('WebSocket 重连次数已达上限');
            this.emit('reconnect_failed');
            return;
        }

        this.reconnectCount++;
        console.log(`WebSocket 重连 (${this.reconnectCount}/${this.options.maxReconnects})...`);
        setTimeout(() => this.connect(), this.options.reconnectInterval);
    }

    send(data) {
        if (this.ws?.readyState === WebSocket.OPEN) {
            this.ws.send(JSON.stringify(data));
        }
    }

    startHeartbeat() {
        this.heartbeatTimer = setInterval(() => {
            this.send({ type: 'ping' });
        }, this.options.heartbeatInterval);
    }

    stopHeartbeat() {
        if (this.heartbeatTimer) {
            clearInterval(this.heartbeatTimer);
            this.heartbeatTimer = null;
        }
    }

    on(event, callback) {
        if (!this.listeners.has(event)) {
            this.listeners.set(event, []);
        }
        this.listeners.get(event).push(callback);
        return () => this.off(event, callback);
    }

    off(event, callback) {
        const cbs = this.listeners.get(event);
        if (cbs) {
            this.listeners.set(event, cbs.filter(cb => cb !== callback));
        }
    }

    emit(event, data) {
        const cbs = this.listeners.get(event) || [];
        cbs.forEach(cb => cb(data));
    }

    close() {
        this.stopHeartbeat();
        this.ws?.close();
    }
}
```text

### 4.2 使用示例

```javascript
// composables/useWebSocket.js
import { ref, onUnmounted } from 'vue';
import { WebSocketClient } from '../utils/websocket';

export function useSeatWebSocket(eventId) {
    const connected = ref(false);
    const seats = ref({});
    let ws = null;

    function connect() {
        const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
        const host = window.location.host;
        ws = new WebSocketClient(`${protocol}//${host}/ws/shows/${eventId}/seats`);

        ws.on('connected', () => { connected.value = true; });
        ws.on('disconnected', () => { connected.value = false; });
        ws.on('seat_selected', (data) => {
            seats.value[data.seat] = { selected: true, user: data.user };
        });
        ws.on('seat_released', (data) => {
            delete seats.value[data.seat];
        });
    }

    function selectSeat(seat, user) {
        ws?.send({ type: 'select', seat, user });
    }

    function releaseSeat(seat) {
        ws?.send({ type: 'release', seat });
    }

    onUnmounted(() => {
        ws?.close();
    });

    return { connected, seats, connect, selectSeat, releaseSeat };
}
```

---

## 5. 支付前端流程

### 5.1 支付宝支付流程

```text
用户点击"去支付"
    ↓
POST /api/v1/payments/alipay (携带 order_id)
    ↓
服务器返回 pay_url
    ↓
打开新窗口（window.open）到支付宝页面
    ↓
用户完成支付
    ↓
支付宝调 return_url（GET 请求）
    ↓
支付宝发异步通知到 notify_url（POST 请求）
    ↓
前端轮询订单状态
    ↓
订单状态变为 "paid"
    ↓
跳转到订单成功页
```

### 5.2 前端实现

```javascript
// Checkout.vue
import { ref, onUnmounted } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import { paymentApi } from '../utils/api/payments';
import { orderApi } from '../utils/api/orders';

const route = useRoute();
const router = useRouter();
const order = ref(null);
const paying = ref(false);
const paymentWindow = ref(null);

async function handlePayment() {
    paying.value = true;
    try {
        const result = await paymentApi.alipay({
            order_id: parseInt(route.params.orderId),
        });

        // 打开支付宝支付页面
        paymentWindow.value = window.open(result.pay_url, '_blank');

        // 开始轮询订单状态
        startPolling();
    } catch (error) {
        alert('支付请求失败: ' + error.message);
        paying.value = false;
    }
}

// 轮询订单状态
let pollTimer = null;

function startPolling() {
    pollTimer = setInterval(async () => {
        try {
            const orderDetail = await orderApi.get(route.params.orderId);
            if (orderDetail.status === 'paid') {
                clearInterval(pollTimer);
                paying.value = false;

                // 关闭支付窗口
                paymentWindow.value?.close();

                // 跳转到成功页
                alert('支付成功！');
                router.push({ name: 'orders' });
            } else if (orderDetail.status === 'cancelled') {
                clearInterval(pollTimer);
                paying.value = false;
                alert('订单已取消');
            }
        } catch (e) {
            console.error('轮询失败:', e);
        }
    }, 2000);  // 2 秒轮询一次
}

onUnmounted(() => {
    clearInterval(pollTimer);
});
```text

### 5.3 支付按钮组件

```html
<!-- PaymentButton.vue -->
<template>
    <button
        class="payment-button"
        :class="{ loading: processing }"
        :disabled="processing || disabled"
        @click="handlePay"
    >
        <span v-if="processing" class="spinner"></span>
        <span v-else>支付宝支付 ¥{{ amount }}</span>
    </button>
</template>

<script setup>
const props = defineProps({
    orderId: { type: Number, required: true },
    amount: { type: Number, required: true },
    disabled: { type: Boolean, default: false },
});

const emit = defineEmits(['success', 'error']);
const processing = ref(false);

async function handlePay() {
    processing.value = true;
    try {
        const result = await paymentApi.alipay({ order_id: props.orderId });
        window.open(result.pay_url, '_blank');
        emit('success', result);
    } catch (error) {
        emit('error', error);
    } finally {
        processing.value = false;
    }
}
</script>

<style scoped>
.payment-button {
    width: 100%;
    padding: 14px 24px;
    background: linear-gradient(135deg, #1677ff, #4096ff);
    color: white;
    border: none;
    border-radius: 8px;
    font-size: 16px;
    font-weight: bold;
    cursor: pointer;
    transition: opacity 0.2s;
}
.payment-button:hover { opacity: 0.9; }
.payment-button:disabled { opacity: 0.5; cursor: not-allowed; }
.payment-button.loading { pointer-events: none; }
.spinner {
    display: inline-block;
    width: 20px;
    height: 20px;
    border: 2px solid rgba(255,255,255,0.3);
    border-top-color: white;
    border-radius: 50%;
    animation: spin 0.6s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }
</style>
```

---

## 6. 完整集成测试

### 6.1 用户完整流程

```javascript
// 测试场景: 新用户完整购票流程
async function testUserJourney() {
    // 1. 注册
    const registerRes = await api.post('/auth/register', {
        username: `test_${Date.now()}`,
        password: 'Test123456',
        email: `test_${Date.now()}@test.com`,
    });
    console.log('注册成功:', registerRes);

    // 2. 登录
    const loginRes = await api.post('/auth/login', {
        username: registerRes.username,
        password: 'Test123456',
    });
    const token = loginRes.access_token;
    api.defaults.headers.common.Authorization = `Bearer ${token}`;
    console.log('登录成功');

    // 3. 浏览活动列表
    const eventsRes = await api.get('/events/', {
        params: { page: 1, size: 10, category: 'concert' },
    });
    console.log('活动列表:', eventsRes.items);

    // 4. 查看活动详情
    if (eventsRes.items.length > 0) {
        const eventDetail = await api.get(`/events/${eventsRes.items[0].id}`);
        console.log('活动详情:', eventDetail.title);
    }

    // 5. 创建订单
    const orderRes = await api.post('/orders/', {
        event_id: 1,
        quantity: 2,
    });
    console.log('订单创建成功:', orderRes.order_no);

    // 6. 获取支付链接
    const paymentRes = await api.post('/payments/alipay', {
        order_id: orderRes.id,
    });
    console.log('支付链接:', paymentRes.pay_url);

    // 7. 查看订单列表
    const ordersRes = await api.get('/orders/');
    console.log('我的订单:', ordersRes.total, '条');

    console.log('✅ 完整购票流程测试通过');
}
```text

---

## 7. 管理后台前端-后端集成

### 7.1 管理后台 API 概览

管理后台的前端 API 调用全部封装在 `frontend/src/stores/admin.js` 中，通过统一的 `api` 实例（Axios）向后端 `admin` 路由器发送请求。

后端对应路由在 `app/routers/admin.py`，挂载在 `/api/v1/admin` 路径下。

| 功能 | 前端 Store 方法 | API 端点 | HTTP 方法 |
| ------ | ---------------- | ---------- | ----------- |
| 仪表盘统计 | `fetchStats()` | `/admin/dashboard/stats` | GET |
| 收入趋势 | `fetchRevenueTrends(days)` | `/admin/dashboard/revenue-trends` | GET |
| 活动排行 | `fetchEventRankings(limit)` | `/admin/dashboard/event-rankings` | GET |
| 用户列表 | `fetchUsers(params)` | `/admin/users` | GET |
| 用户详情 | `fetchUserDetail(userId)` | `/admin/users/{userId}` | GET |
| 封禁用户 | `banUser(userId, reason)` | `/admin/users/{userId}/ban` | POST |
| 解封用户 | `unbanUser(userId)` | `/admin/users/{userId}/unban` | POST |
| 修改用户角色 | `setUserRole(userId, role)` | `/admin/users/{userId}/role` | PUT |
| 用户登录历史 | `fetchLoginHistory(userId)` | `/admin/users/{userId}/login-history` | GET |
| 用户 IP 历史 | `fetchUserIPHistory(userId)` | `/admin/users/{userId}/ip-history` | GET |
| 操作日志列表 | `fetchLogs(params)` | `/admin/logs` | GET |
| 登录日志 | `fetchLoginLogs(params)` | `/admin/logs/login` | GET |
| 注册日志 | `fetchRegisterLogs(params)` | `/admin/logs/registration` | GET |
| 日志统计数据 | `fetchDashboardStats(params)` | `/admin/logs/statistics` | GET |
| 创建活动 | `createEvent(data)` | `/events/` | POST |
| 编辑活动 | `updateEvent(eventId, data)` | `/events/{eventId}` | PUT |
| 订单列表 | `fetchAdminOrders(params)` | `/admin/orders` | GET |
| 订单详情 | `fetchOrderDetail(orderId)` | `/admin/orders/{orderId}` | GET |
| 空降订单 | `createAdminOrder(data)` | `/admin/orders` | POST |
| 退款审核 | `reviewRefund(orderId, approve)` | `/admin/refunds/{orderId}/review` | POST |
| 批量退款 | `batchReviewRefunds(orderIds, approve)` | `/admin/refunds/batch-review` | POST |
| 票证列表 | `fetchAdminTickets(params)` | `/admin/tickets` | GET |

### 7.2 管理员认证流程

管理后台的 API 请求通过统一的 JWT 认证机制保护：

```

用户登录 → 后端返回 JWT → 前端存入 localStorage
  → Axios 请求拦截器自动注入 Authorization: Bearer <token>
  → 后端 AuthMiddleware 解码 JWT → 提取 user_id 和 role
  → 后端 RoleChecker 检查 role === "admin"
  → 允许访问或返回 403

```text

### 7.3 管理后台 Store 调用示例

```javascript
// 组件中调用
import { useAdminStore } from '../../stores/admin'

const adminStore = useAdminStore()

// 获取仪表盘数据
const stats = await adminStore.fetchStats()
const trend = await adminStore.fetchRevenueTrends(14)
const ranking = await adminStore.fetchEventRankings(10)

// 管理用户
const users = await adminStore.fetchUsers({ page: 1, size: 20, keyword: 'test' })
await adminStore.banUser(42, '恶意刷票')

// 退款审核
await adminStore.reviewRefund(1001, true)      // 单个通过
await adminStore.batchReviewRefunds([1002, 1003], false)  // 批量拒绝
```

### 7.4 错误处理模式

管理后台的错误处理遵循统一的 try/catch 模式：

```javascript
async function fetchUsers() {
  loading.value = true
  try {
    const res = await adminStore.fetchUsers(params)
    users.value = res.items
    total.value = res.total
  } catch (e) {
    console.error('获取用户列表失败', e)
    // 可选：显示错误提示
  } finally {
    loading.value = false  // 无论成功失败都结束加载
  }
}
```text

**为什么 using try/catch 而不是 Axios 拦截器统一处理？**

- 管理后台每个页面的错误处理策略不同：列表页可能静默失败，而提交操作需要明确提示用户
- 提交失败时需要保持表单状态（不刷新页面），让用户可以修正后重试
- 批量操作需要显示具体的成功/失败数量

### 7.5 前后端交互流程示例

以"管理员审核退款"为例：

```

管理员点击"通过"按钮
    │
    ├─→ adminStore.reviewRefund(orderId, approve)
    │     POST /api/v1/admin/refunds/:orderId/review
    │     Body: { "approve": true }
    │
    ▼
后端 AdminService.review_refund()
    │  1. 查询订单 → 验证 status === "paid"
    │  2. 更新 refund_status = "approved"
    │  3. 调用支付宝退款 API → 如果已支付
    │  4. 更新订单 status = "refunded"
    │
    ▼
返回 { success: true }
    │
    ├─→ 前端刷新列表: fetchOrders()
    └─→ 显示成功提示

```text

### 7.6 日志管理 API 集成

操作日志管理是超级管理员特有的功能，后端提供三个日志查询端点，前端通过 Tab 切换视图。

**日志管理前端集成** (`frontend/src/views/admin/LogManageView.vue`)：

```vue
<!-- Tabs: 全部日志 / 登录日志 / 注册日志 / 操作日志 -->
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
    <!-- ... -->
  </select>
  <input type="date" v-model="startDate" @change="search" />
  <input type="date" v-model="endDate" @change="search" />
  <input v-model="keyword" placeholder="搜索用户/详情..." @keyup.enter="search" />
  <button class="btn btn-primary" @click="search">搜索</button>
</div>
```

**Tab 切换与 API 调用映射**：

```javascript
async function fetchLogs() {
  const params = { page: page.value, size: size.value }
  // 根据 Tab 调用不同的 API
  let res
  switch (activeTab.value) {
    case 'login':
      res = await adminStore.fetchLoginLogs(params)    // GET /admin/logs/login
      break
    case 'register':
      res = await adminStore.fetchRegisterLogs(params) // GET /admin/logs/registration
      break
    default:
      res = await adminStore.fetchLogs(params)          // GET /admin/logs
  }
  logs.value = res.items
  total.value = res.total
}
```text

**日志操作类型与标签映射**：

```javascript
const actionLabelMap = {
  login: '登录', register: '注册', ban: '封禁',
  unban: '解封', role_change: '角色变更', logout: '登出',
}

const badgeClassMap = {
  login: 'log-login',       // 蓝色
  register: 'log-register', // 绿色
  ban: 'log-ban',           // 红色
  role_change: 'log-role',  // 紫色
}
```

**Store 方法** (`frontend/src/stores/admin.js`)：

```javascript
// ===== Super Admin: Logs =====
async function fetchLogs(params = {}) {          // GET /admin/logs
  return api.get('/admin/logs', { params })
}
async function fetchLoginLogs(params = {}) {     // GET /admin/logs/login
  return api.get('/admin/logs/login', { params })
}
async function fetchRegisterLogs(params = {}) {  // GET /admin/logs/registration
  return api.get('/admin/logs/registration', { params })
}
async function fetchDashboardStats(params = {}) {// GET /admin/logs/statistics
  return api.get('/admin/logs/statistics', { params })
}

// ===== Super Admin: User Role =====
async function setUserRole(userId, role) {       // PUT /admin/users/{id}/role
  return api.put(`/admin/users/${userId}/role`, { role })
}
async function fetchUserIPHistory(userId) {      // GET /admin/users/{id}/ip-history
  return api.get(`/admin/users/${userId}/ip-history`)
}
```text

### 7.7 超级管理员路由守卫

日志管理页面仅限 `super_admin` 角色访问，需要额外的路由守卫检查：

```javascript
// router/index.js
{
  path: 'logs',
  component: () => import('../views/admin/LogManageView.vue'),
  meta: {
    requiresAuth: true,        // 需要登录
    requiresAdmin: true,       // 需要 admin 或 super_admin
    requiresSuperAdmin: true,  // 额外需要 super_admin
    title: '日志管理'
  },
}

// 路由守卫中的超管检查
if (to.meta.requiresSuperAdmin && token) {
  const user = JSON.parse(localStorage.getItem('user') || '{}')
  if (user.role !== 'super_admin') {
    next({ path: '/admin' })  // 非超管跳转管理后台首页
    return
  }
}
```

这样即使普通管理员手动访问 `/admin/logs` URL，也会被路由守卫拦截并重定向。

---

## 8. 项目部署检查清单

### 7.1 前端构建

```bash
# 构建生产版本
npm run build

# 输出在 dist/ 目录
# 部署到 Nginx 静态文件目录
cp -r dist/* /var/www/ticket/static/
```text

### 7.2 Nginx 配置检查

```nginx
# 1. 静态文件服务
location / {
    root /var/www/ticket/static;
    try_files $uri $uri/ /index.html;  # SPA 路由
}

# 2. API 反向代理
location /api/ {
    proxy_pass http://localhost:8000;
}

# 3. WebSocket 代理
location /ws/ {
    proxy_pass http://localhost:8000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

### 7.3 安全加固

```javascript
// 1. XSS 防护 - 不要使用 v-html 渲染用户输入
// 2. CSRF 防护 - 使用 SameSite Cookie
// 3. 敏感信息 - 不要在前端代码中硬编码密钥
// 4. HTTPS - 生产环境必须使用
// 5. 输入验证 - 前后端都要做
```text

---

## 本章总结

| 知识点 | 掌握程度 |
| -------- | --------- |
| CORS | 理解预检请求和配置 |
| JWT 前端管理 | 能实现自动刷新 |
| Axios 配置 | 能封装完整的请求模块 |
| WebSocket 客户端 | 能实现自动重连 |
| 支付流程 | 理解支付宝前端流程 |
| 集成测试 | 能测试完整用户旅程 |

**项目里程碑：**

- [ ] 前端正确调用后端 API
- [ ] Token 自动刷新
- [ ] WebSocket 实时座位同步
- [ ] 支付宝支付流程完整

---

## 安全防护

### CORS 配置

```python
# app/main.py — 仅允许白名单中的前端域名
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],  # ★ 明确指定，非 "*"
    allow_credentials=True,                   # 允许携带 Authorization 头
    allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH"],
    allow_headers=["Authorization", "Content-Type"],
)
```

### JWT 存储安全对比

| 存储位置 | XSS 风险 | CSRF 风险 | 适用场景 |
| --------- | --------- | ----------- | --------- |
| localStorage | 高（JS 可读） | 无（不自动发送） | 本系统当前选择 |
| sessionStorage | 高（JS 可读） | 无（不自动发送） | 单标签页应用 |
| httpOnly Cookie | 低（JS 不可读） | 高（自动发送） | 需额外 CSRF token |
| BFF 模式 | 无（令牌在服务端） | 无 | 企业级首选 |

**本系统的选择：localStorage + CSP 缓解**

由于我们配置了 CSP（`script-src 'self'`），内联脚本和外站脚本无法执行，大幅降低了 XSS 风险。这是本地开发部署的合理选择。

### 令牌刷新安全流程

```text
1. Axios 响应拦截器检测 401
2. 自动调用 POST /api/v1/auth/refresh
3. 后端验证旧 refresh_token
4. 后端将旧 refresh_token 加入黑名单（令牌轮换）
5. 后端返回新 access_token + 新 refresh_token
6. 前端更新 localStorage
7. 重试原始请求
```

### WebSocket 认证

WebSocket 连接通过查询参数 `?token=xxx` 传递 JWT 令牌。连接建立时服务端验证令牌有效性，之后所有消息中的用户身份都从令牌提取（客户端无法伪造）。

### 支付前端安全

1. 支付 URL 通过 HTTPS 获取，不在前端拼接
2. 支付完成后轮询订单状态（不信任浏览器回调）
3. 新窗口打开支付宝页面（防止钓鱼页面篡改主窗口）
4. 订单金额在服务端计算（前端传的金额仅用于展示）

- [ ] 首屏加载 < 2 秒

---

## 常见问题与排错

| 问题 | 原因 | 解决 |
| ------ | ------ | ------ |
| CORS 预检请求失败（OPTIONS 返回 404） | Nginx 未配置 OPTIONS 处理 | 在 Nginx 中添加 `if ($request_method = OPTIONS)` 处理 |
| 前端报 401 后不断刷新 Token | 多个并发请求都触发刷新 | 参照实际代码用 `_retry` 标志位防重入 |
| WebSocket 连接后立即断开 | 路径或认证错误 | 检查连接路径是否匹配后端路由，Token 是否有效 |
| 支付宝回调未被前端接收 | 新窗口打开无法直接通信 | 改用轮询订单状态替代窗口通信 |
| Axios 响应拦截器未捕获到 401 | 请求被 catch 提前处理 | 确保所有请求统一通过同一 Axios 实例发送 |
| 静态资源 404（JS/CSS 加载失败） | 构建后资源路径不对 | Vite 配置 `base: '/'`，Nginx 配置 `try_files` |
| 生产环境 WebSocket 被断开 | 负载均衡未配置 WebSocket 支持 | Nginx 必须设置 `proxy_set_header Upgrade $http_upgrade` |

---

## 本章练习

1. **实现 Axios 实例**：对照 `frontend/src/utils/api.js`，实现包含请求拦截器（注入 token）和响应拦截器（自动刷新 token）的完整 Axios 封装
2. **实现 WebSocket 客户端**：对照 `frontend/src/utils/websocket.js`，实现带自动重连和心跳检测的 WebSocket 客户端
3. **分析实际 CORS 配置**：阅读 `app/main.py` 中的 CORS 中间件配置，说明为什么生产环境不能使用 `allow_origins=["*"]`
4. **集成测试电商流程**：编写自动测试脚本实现完整用户旅程：注册→登录→浏览活动→下单→模拟支付→查看订单状态
5. **部署检查**：完成生产环境 Nginx 配置，包含静态文件服务、API 反向代理、WebSocket 代理和 HTTPS 配置

---

## 下一章预告

第 21 章将系统讲解票务系统的完整安全体系，覆盖 OWASP Top 10 每个风险点，包括密码存储、JWT 令牌安全、CSP 策略、速率限制、支付安全和数据库安全等核心内容。
