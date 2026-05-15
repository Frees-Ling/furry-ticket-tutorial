# 第18章：Vue 3 从零到高阶

## 本章学习目标

本章将学习 Vue 3 的全部核心概念，从组合式 API 到完整项目实战。

**学习路径：**

1. Vue 3 简介与对比
2. 开发环境搭建
3. 组合式 API（ref/reactive/computed/watch）
4. 模板语法与指令
5. 组件通信
6. Vue Router
7. Pinia 状态管理
8. 表单处理
9. API 集成
10. 组合式函数

---

## 1. Vue 3 简介

### 1.1 为什么选 Vue 3

| 特性 | Vue 3 | React | Angular |
| ------ | ------- | ------- | --------- |
| 学习曲线 | 平缓 | 中等 | 陡峭 |
| 模板语法 | HTML 原生 | JSX | HTML 扩展 |
| 响应式 | Proxy 自动追踪 | useState 手动 | Zone.js |
| 打包大小 | ~33KB | ~42KB | ~500KB |
| 中国生态 | 最流行 | 流行 | 较少 |
| TypeScript | 优秀 | 优秀 | 原生 |

### 1.2 组合式 API vs 选项式 API

```javascript
// 选项式 API (Vue 2 风格)
export default {
    data() {
        return { count: 0 };
    },
    methods: {
        increment() { this.count++; }
    },
    computed: {
        double() { return this.count * 2; }
    },
};

// 组合式 API (Vue 3 推荐)
import { ref, computed } from 'vue';

export default {
    setup() {
        const count = ref(0);
        const increment = () => count.value++;
        const double = computed(() => count.value * 2);
        return { count, increment, double };
    }
};

// 更简洁的语法:
// <script setup>
import { ref, computed } from 'vue';
const count = ref(0);
const increment = () => count.value++;
const double = computed(() => count.value * 2);
// </script>
```text

---

## 2. 开发环境搭建

```bash
# 使用 Vite 创建项目
npm create vite@latest ticket-frontend -- --template vue
cd ticket-frontend
npm install
npm install vue-router@4 pinia axios
npm run dev
```

**项目结构：**

```text
frontend/
├── index.html
├── package.json
├── vite.config.js
└── src/
    ├── main.js          # 入口
    ├── App.vue          # 根组件
    ├── router/          # 路由
    │   └── index.js
    ├── stores/          # 状态管理
    │   ├── auth.js
    │   └── event.js
    ├── views/           # 页面
    ├── components/      # 组件
    └── utils/           # 工具
        ├── api.js       # Axios 请求
        └── websocket.js # WebSocket
```

---

## 3. 组合式 API

### 3.1 ref — 响应式数据

```javascript
import { ref } from 'vue';

const count = ref(0);          // 基本类型
const user = ref({ name: 'Alice' });  // 对象

// 读取/修改必须通过 .value
console.log(count.value);      // 0
count.value++;
console.log(count.value);      // 1

// 模板中自动解包（不需要 .value）
// <template>
//   <p>{{ count }}</p>
//   <button @click="count++">+1</button>
// </template>
```text

### 3.2 reactive — 对象响应式

```javascript
import { reactive } from 'vue';

const state = reactive({
    user: null,
    orders: [],
    loading: false,
});

// reactive 不需要 .value，直接访问
state.loading = true;
state.user = { id: 1, name: 'Alice' };

// reactive vs ref
// reactive: 只能用于对象/数组，深层响应式
// ref: 用于基本类型或对象，通过 .value 访问
// 推荐：全部用 ref（统一，且 .value 明确标识响应式变量）
```

### 3.3 computed — 计算属性

```javascript
import { ref, computed } from 'vue';

const price = ref(680);
const quantity = ref(2);

// 只读计算属性
const total = computed(() => price.value * quantity.value);

// 可读写计算属性
const totalWithTax = computed({
    get: () => price.value * quantity.value * 1.13,
    set: (val) => {
        price.value = val / (quantity.value * 1.13);
    },
});
```text

### 3.4 watch — 侦听器

```javascript
import { ref, watch, watchEffect } from 'vue';

const search = ref('');
const results = ref([]);

// 监听单个源
watch(search, (newVal, oldVal) => {
    console.log(`搜索词从 "${oldVal}" 变为 "${newVal}"`);
    fetchResults(newVal);
});

// 监听多个源
watch([search, category], ([newSearch, newCat]) => {
    fetchResults(newSearch, newCat);
});

// watchEffect: 自动追踪依赖
watchEffect(async () => {
    // 任何 search.value 的变化都会触发
    results.value = await api.search(search.value);
});
```

### 3.5 生命周期钩子

```javascript
import { onMounted, onUnmounted, onBeforeUnmount } from 'vue';

onMounted(() => {
    // 组件挂载后获取数据
    fetchData();
});

onUnmounted(() => {
    // 清理（定时器、WebSocket 连接）
    ws.close();
});

onBeforeUnmount(() => {
    // 卸载前清理
});
```text

---

## 4. 模板语法

### 4.1 插值与指令

```html
<!-- 文本插值 -->
<p>{{ message }}</p>

<!-- 原始 HTML -->
<div v-html="rawHtml"></div>

<!-- 属性绑定 -->
<img :src="imageUrl">
<button :disabled="isLoading">
<div :class="{ active: isActive, 'text-danger': hasError }">
<div :style="{ color: activeColor, fontSize: size + 'px' }">

<!-- 事件绑定 -->
<button @click="handleClick">
<form @submit.prevent="onSubmit">  <!-- .prevent = preventDefault -->
<input @keyup.enter="submit">      <!-- .enter = Enter 键 -->
</form>

<!-- 双向绑定 -->
<input v-model="username">
<!-- 等价于 -->
<input :value="username" @input="username = $event.target.value">
```

### 4.2 条件渲染

```html
<!-- v-if: 条件渲染（销毁/创建） -->
<div v-if="status === 'loading'">加载中...</div>
<div v-else-if="status === 'error'">加载失败</div>
<div v-else>内容已加载</div>

<!-- v-show: 切换 display（不销毁） -->
<div v-show="isVisible">始终存在，仅切换显示</div>
```text

### 4.3 列表渲染

```html
<!-- 基本列表 -->
<li v-for="(event, index) in events" :key="event.id">
    {{ index + 1 }}. {{ event.title }}
</li>

<!-- 对象遍历 -->
<div v-for="(value, key) in object" :key="key">
    {{ key }}: {{ value }}
</div>

<!-- 不要使用 index 作为 key（会导致渲染问题） -->
<!-- 正确: :key="event.id" -->
<!-- 错误: :key="index" -->
```

---

## 5. 组件通信

### 5.1 props — 父传子

```javascript
// ChildComponent.vue
const props = defineProps({
    title: { type: String, required: true },
    price: { type: Number, default: 0 },
    stock: { type: Number, validator: (v) => v >= 0 },
});

// ParentComponent.vue
// <ChildComponent title="音乐会" :price="680" :stock="100" />
```text

### 5.2 emit — 子传父

```javascript
// ChildComponent.vue
const emit = defineEmits(['buy', 'cancel']);

function handleBuy() {
    emit('buy', { quantity: 2 });
}

// ParentComponent.vue
// <ChildComponent @buy="onBuy" @cancel="onCancel" />
```

### 5.3 slots — 插槽

```html
<!-- ChildComponent.vue -->
<div class="card">
    <header><slot name="header">默认标题</slot></header>
    <main><slot>默认内容</slot></main>
    <footer><slot name="footer" :count="items.length">{{ count }} items</slot></footer>
</div>

<!-- ParentComponent.vue -->
<ChildComponent>
    <template #header>活动列表</template>
    <p>这里是主要内容</p>
    <template #footer="{ count }">共 {{ count }} 个活动</template>
</ChildComponent>
```text

### 5.4 provide/inject — 跨层级

```javascript
// 祖先组件
import { provide, ref } from 'vue';
const user = ref(null);
provide('user', user);

// 后代组件（任何层级）
import { inject } from 'vue';
const user = inject('user', null);  // 第二个参数是默认值
```

---

## 6. Vue Router

### 6.1 基本配置

```javascript
// router/index.js
import { createRouter, createWebHistory } from 'vue-router';

const routes = [
    { path: '/', name: 'home', component: () => import('../views/HomeView.vue') },
    { path: '/events', name: 'events', component: () => import('../views/EventList.vue') },
    { path: '/events/:id', name: 'event-detail', component: () => import('../views/EventDetail.vue') },
    { path: '/login', name: 'login', component: () => import('../views/LoginView.vue') },
    { path: '/checkout/:orderId', name: 'checkout', component: () => import('../views/Checkout.vue'), meta: { requiresAuth: true } },
    { path: '/orders', name: 'orders', component: () => import('../views/OrderHistory.vue'), meta: { requiresAuth: true } },
];

const router = createRouter({
    history: createWebHistory(),
    routes,
});

// 导航守卫
router.beforeEach((to, from, next) => {
    const token = localStorage.getItem('token');
    if (to.meta.requiresAuth && !token) {
        next({ name: 'login', query: { redirect: to.fullPath } });
    } else {
        next();
    }
});
```text

### 6.2 导航

```javascript
// 声明式
// <router-link :to="{ name: 'event-detail', params: { id: event.id } }">

// 编程式
import { useRouter, useRoute } from 'vue-router';

const router = useRouter();
const route = useRoute();

// 导航
router.push({ name: 'event-detail', params: { id: 1 } });
router.replace({ name: 'home' });
router.back();

// 获取参数
console.log(route.params.id);   // 动态路由参数
console.log(route.query.page);  // 查询参数
```

---

## 7. Pinia 状态管理

### 7.1 Store 定义

```javascript
// stores/auth.js
import { defineStore } from 'pinia';
import { ref, computed } from 'vue';
import { api } from '../utils/api';

export const useAuthStore = defineStore('auth', () => {
    // state
    const token = ref(localStorage.getItem('token') || '');
    const user = ref(null);

    // getters
    const isLoggedIn = computed(() => !!token.value);
    const isAdmin = computed(() => user.value?.role === 'admin');

    // actions
    async function login(username, password) {
        const res = await api.post('/auth/login', { username, password });
        token.value = res.access_token;
        localStorage.setItem('token', res.access_token);
    }

    async function fetchUser() {
        const res = await api.get('/auth/me');
        user.value = res;
    }

    function logout() {
        token.value = '';
        user.value = null;
        localStorage.removeItem('token');
    }

    return { token, user, isLoggedIn, isAdmin, login, fetchUser, logout };
});
```text

### 7.2 Store 使用

```javascript
// 组件中使用
import { useAuthStore } from '../stores/auth';
import { useEventStore } from '../stores/event';

const auth = useAuthStore();
const eventStore = useEventStore();

// 直接访问 state
console.log(auth.isLoggedIn);
console.log(eventStore.events);

// 调用 action
await auth.login('admin', 'pass123');
await eventStore.fetchEvents({ page: 1, size: 20 });
```

### 7.3 Event Store 示例

```javascript
// stores/event.js
import { defineStore } from 'pinia';
import { ref } from 'vue';
import { api } from '../utils/api';

export const useEventStore = defineStore('event', () => {
    const events = ref([]);
    const currentEvent = ref(null);
    const total = ref(0);
    const loading = ref(false);

    async function fetchEvents(params = {}) {
        loading.value = true;
        try {
            const res = await api.get('/events/', { params });
            events.value = res.items;
            total.value = res.total;
        } finally {
            loading.value = false;
        }
    }

    async function fetchEventDetail(id) {
        currentEvent.value = await api.get(`/events/${id}`);
    }

    return { events, currentEvent, total, loading, fetchEvents, fetchEventDetail };
});
```text

### 7.4 Admin Store — 一个 Store 管理多个资源

管理后台（`frontend/src/views/admin/`）使用了不同于前台 Store 的模式：**一个 Store 管理 5 组功能**。

```javascript
// stores/admin.js（简化）
import { defineStore } from 'pinia'
import { ref } from 'vue'
import { api } from '../utils/api'

export const useAdminStore = defineStore('admin', () => {
  const stats = ref(null)
  const loading = ref(false)

  // 仪表盘 API
  async function fetchStats() {
    stats.value = await api.get('/admin/dashboard/stats')
    return stats.value
  }

  // 用户管理 API
  async function fetchUsers(params) {
    return api.get('/admin/users', { params })
  }

  async function banUser(userId, reason) {
    return api.post(`/admin/users/${userId}/ban`, { reason })
  }

  // 活动管理 API
  async function createEvent(data) {
    return api.post('/events/', data)
  }

  // 订单管理 API（含退款审核）
  async function fetchAdminOrders(params) {
    return api.get('/admin/orders', { params })
  }

  // 批量退款审核
  async function batchReviewRefunds(orderIds, approve) {
    return api.post('/admin/refunds/batch-review', { order_ids: orderIds, approve })
  }

  // 票证管理 API
  async function fetchAdminTickets(params) {
    return api.get('/admin/tickets', { params })
  }

  return {
    stats, loading,
    fetchStats, fetchUsers, banUser,
    createEvent, fetchAdminOrders,
    batchReviewRefunds, fetchAdminTickets,
  }
})
```

**为什么 Admin Store 不同于前台 Store？**

| 对比项 | 前台 Store（auth/event/order） | Admin Store |
| -------- | ------------------------------- | ------------- |
| 数量 | 多个独立 Store | 一个 Store 管理全部 |
| 职责 | 每个 Store 一个资源 | 一个 Store 多个资源 |
| 状态 | 缓存列表和详情 | 仅缓存 dashboard stats |
| 使用范围 | 多个页面共享 | 仅在管理后台使用 |

前台系统中，用户、活动、订单是三个独立的业务域，分别创建 Store 有利于模块化。但是在管理后台中，**5 个管理页面共享相同的"管理员"上下文**，合并到一个 Store 更方便维护：

1. 所有管理 API 调用集中在同一处，修改 URL 或参数时只需要改一个文件
2. 避免创建多个 Store 导致的重复 import
3. 管理员上下文天然聚合，不会与前台 Store 冲突

---

## 8. 表单处理

### 8.1 v-model 修饰符

```html
<!-- 常用修饰符 -->
<input v-model.lazy="username">     <!-- 懒同步（change 事件触发） -->
<input v-model.number="age">        <!-- 自动转数字 -->
<input v-model.trim="search">       <!-- 自动去空格 -->

<!-- 不同输入类型 -->
<input type="text" v-model="name">
<textarea v-model="description"></textarea>
<input type="checkbox" v-model="agree">    <!-- 单个: boolean -->
<input type="checkbox" v-model="hobbies">   <!-- 多个: array -->
<input type="radio" v-model="gender" value="male">
<select v-model="selected">
    <option value="">请选择</option>
    <option value="concert">音乐会</option>
    <option value="sports">体育</option>
</select>
```text

### 8.2 自定义组件 v-model

```javascript
// SearchInput.vue
const model = defineModel();  // Vue 3.4+

// <input :value="model" @input="model = $event.target.value">

// 使用:
// <SearchInput v-model="searchText" />
```

---

## 9. API 集成

### 9.1 Axios 实例配置

```javascript
// utils/api.js
import axios from 'axios';
import { useAuthStore } from '../stores/auth';

const api = axios.create({
    baseURL: '/api/v1',
    timeout: 10000,
    headers: { 'Content-Type': 'application/json' },
});

// 请求拦截器：自动添加 token
api.interceptors.request.use((config) => {
    const token = localStorage.getItem('token');
    if (token) {
        config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
});

// 响应拦截器：统一错误处理
api.interceptors.response.use(
    (response) => response.data,
    (error) => {
        if (error.response?.status === 401) {
            const auth = useAuthStore();
            auth.logout();
            window.location.href = '/login';
        }
        return Promise.reject(error.response?.data || error);
    }
);

export { api };
```text

---

## 10. 组合式函数

### 10.1 useAuth

```javascript
// composables/useAuth.js
import { computed } from 'vue';
import { useAuthStore } from '../stores/auth';

export function useAuth() {
    const auth = useAuthStore();

    const requireAuth = () => {
        if (!auth.isLoggedIn) {
            throw new Error('需要登录');
        }
    };

    return {
        user: computed(() => auth.user),
        isLoggedIn: auth.isLoggedIn,
        isAdmin: auth.isAdmin,
        login: auth.login,
        logout: auth.logout,
        requireAuth,
    };
}
```

### 10.2 useWebSocket

```javascript
// composables/useWebSocket.js
import { ref, onUnmounted } from 'vue';

export function useWebSocket(url) {
    const ws = ref(null);
    const connected = ref(false);
    const messages = ref([]);

    function connect() {
        ws.value = new WebSocket(url);

        ws.value.onopen = () => {
            connected.value = true;
            console.log('WebSocket 已连接');
        };

        ws.value.onmessage = (event) => {
            const data = JSON.parse(event.data);
            messages.value.push(data);
        };

        ws.value.onclose = () => {
            connected.value = false;
            // 自动重连
            setTimeout(connect, 3000);
        };
    }

    function send(data) {
        if (ws.value?.readyState === WebSocket.OPEN) {
            ws.value.send(JSON.stringify(data));
        }
    }

    onUnmounted(() => {
        ws.value?.close();
    });

    return { connected, messages, connect, send };
}
```text

---

## 本章总结

| 知识点 | 掌握程度 |
| -------- | --------- |
| 组合式 API | 熟练使用 ref/reactive/computed/watch |
| 模板语法 | 掌握指令/条件/列表渲染 |
| 组件通信 | 掌握 props/emit/slots/provide-inject |
| Vue Router | 能配置路由/守卫/参数传递 |
| Pinia | 能定义和使用 store |
| 表单处理 | 掌握 v-model 和自定义模型 |
| Axios 集成 | 能配置拦截器和错误处理 |

---

## 安全防护

### 模板 XSS 防护

Vue 3 默认对模板插值进行 HTML 编码，这防止了大部分 XSS 攻击：

```vue
<!-- ✅ 安全：Vue 自动编码特殊字符 -->
<p>{{ userInput }}</p>
<!-- 输出：&lt;script&gt;alert('xss')&lt;/script&gt; -->

<!-- ❌ 危险：v-html 跳过编码 -->
<p v-html="userInput"></p>
<!-- 输出：<script>alert('xss')</script> -->
```

**安全规则：** 除非绝对必要且内容已消毒，否则绝不使用 `v-html`。

### 导航守卫鉴权

在路由级别控制页面访问权限：

```javascript
// router/index.js
router.beforeEach((to, from, next) => {
    const token = localStorage.getItem('token')
    if (to.meta.requiresAuth && !token) {
        next('/login')  // 未登录 → 跳转登录页
    } else {
        next()
    }
})
```text

### 敏感信息展示

用户信息只在前端展示必要字段，不暴露 `password_hash` 等敏感字段。API 返回的 UserResponse schema 已经排除了密码字段。

### 安全最佳实践

1. **使用 CSP 限制脚本来源**（后端配置）
2. **接口请求统一通过 Axios 实例发送**（自动处理认证头）
3. **不在 URL 参数中传递敏感数据**
4. **路由懒加载不暴露未授权页面代码**
5. **生产环境禁用 Vue Devtools**

---

## 常见问题与排错

| 问题 | 原因 | 解决 |
| ------ | ------ | ------ |
| `ref` 在模板中自动解包，在 script 中不自动解包 | Vue 的响应式设计 | script 中使用 `.value` 访问，模板中不用 |
| `reactive` 重新赋值后失去响应性 | reactive 只对初始对象代理 | 用 `ref` 存整个对象，或修改属性不替换对象 |
| `watch` 监听 `reactive` 对象不触发 | 默认只监听引用变化 | 开启 `{ deep: true }` 监听深层变化 |
| 组件未更新但控制台数据已变 | Vue 的异步更新队列 | 用 `nextTick()` 等待 DOM 更新完成 |
| `v-for` + `v-if` 使用警告 | Vue 3 中 `v-if` 优先级高于 `v-for` | 用计算属性过滤列表替代 |
| `provide/inject` 传递的 ref 不是响应式 | 传入了 ref.value 而非 ref 本身 | 传入 ref 对象，让子组件 `.value` 访问 |
| `<Suspense>` 未捕获异步依赖 | 需要父组件用 Suspense 包裹 | 检查组件树是否都在 Suspense 边界内 |
| 路由切换后页面不刷新 | 组件复用了相同路由 | 添加 `:key="$route.fullPath"` |
| Pinia store 修改不触发 UI 更新 | 直接解构了 state 导致失去响应性 | 用 `storeToRefs()` 解构 |

---

### 9.2 核销管理 API 集成示例

对于需要组织者或管理员权限的核销管理功能，可以添加以下 API 请求：

```javascript
// 验票核销
async function verifyTicket(code) {
    return api.post('/verification/verify', { code });
}

// 批量核销
async function batchVerify(codes) {
    return api.post('/verification/batch-verify', { codes });
}

// 查询活动核销统计
async function getVerificationStats(eventId) {
    return api.get(`/events/${eventId}/verification/stats`);
}

// 获取活动票证列表（核销管理）
async function getEventTickets(eventId, params = {}) {
    return api.get(`/events/${eventId}/verification/tickets`, { params });
}
```

这些接口与前面定义的 `api` Axios 实例兼容，自动携带 JWT token。

---

## 11. 管理后台前端 Admin Dashboard

### 11.1 页面概览

管理后台包含 5 个核心管理页面，代码在 `frontend/src/views/admin/` 目录下：

| 页面 | 文件 | 核心功能 |
| ------ | ------ | --------- |
| 仪表盘 | `DashboardView.vue` | 6 张统计卡片、CSS 柱状图收入趋势、活动销售排行 |
| 用户管理 | `UserManageView.vue` | 搜索/筛选/编辑/封禁/解封、登录历史弹窗 |
| 活动管理 | `EventManageView.vue` | 活动 CRUD、售罄率进度条、创建/编辑共用表单 |
| 订单管理 | `OrderManageView.vue` | 多维筛选、订单详情、退款审核（含批量）、空降订单创建 |
| 票证管理 | `TicketManageView.vue` | 票证搜索/筛选/使用状态，只读页面 |

### 11.2 布局模式

使用 **嵌套路由** + **侧边栏布局**（`AdminLayout.vue`）：

```vue
<template>
  <div class="admin-layout">
    <aside class="admin-sidebar">
      <router-link to="/admin" class="admin-sidebar-link">仪表盘</router-link>
      <router-link to="/admin/users" class="admin-sidebar-link">用户管理</router-link>
      <!-- ... -->
    </aside>
    <main class="admin-content">
      <router-view />
    </main>
  </div>
</template>
```text

### 11.3 管理员路由守卫

路由配置中使用 `meta.requiresAdmin` 标识需要管理员权限的页面：

```javascript
{
  path: '/admin',
  component: AdminLayout,
  meta: { requiresAuth: true, requiresAdmin: true },
  children: [
    { path: '', component: DashboardView },
    { path: 'users', component: UserManageView },
    // ...
  ],
}
```

导航守卫中从 `localStorage` 读取缓存的用户角色进行同步检查：

```javascript
router.beforeEach((to, from, next) => {
  if (to.meta.requiresAdmin) {
    const stored = localStorage.getItem('user')
    if (stored) {
      const user = JSON.parse(stored)
      if (user.role !== 'admin') {
        next({ name: 'home' })
        return
      }
    }
  }
  next()
})
```text

### 11.4 公共模式

所有管理页面共享 4 种模式：

1. **筛选栏** — 搜索框 + 下拉选择 + 搜索按钮，回车或切换即触发搜索
2. **数据表格** — `data-table` 类表格，统一的加载态/空数据/行渲染
3. **分页** — 计算属性生成页码范围，显示当前页周围 5 个按钮
4. **模态弹窗** — `modal-overlay` + `modal-content` 两层结构，`@click.self` 点击背景关闭

### 11.5 样式选择

管理后台没有使用 Element Plus 等 UI 库，而是采用纯 CSS 自定义样式：

- **原因**：页面数量少（5 个），引入 UI 库会增加 2MB+ 的包体积
- **实现**：统一的 `data-table`、`btn`、`modal-overlay`、`filter-bar` 等 CSS 类
- **图表**：使用 CSS 柱状图替代 ECharts/Chart.js，零依赖

---

## 本章练习

1. **实现 EventCard 组件**：对照 `frontend/src/components/EventCard.vue`，用 Vue 3 Composition API + props 实现活动卡片组件
2. **实现 Pinia 订单 store**：对照 `frontend/src/stores/order.js`，实现订单列表获取和取消订单功能
3. **实现 useCountdown composable**：对照 `frontend/src/components/PaymentCountdown.vue`，实现一个通用的倒计时组合式函数
4. **路由守卫增强**：在 `router/index.js` 中添加角色检查，让 admin 页面只能被 admin 角色访问
5. **WebSocket 集成**：对照 `frontend/src/composables/useWebSocket.js`，在 EventDetail 页面中集成实时座位更新
6. **实现 Admin Store**：对照 `frontend/src/stores/admin.js`，封装管理后台的仪表盘/用户/活动/订单/票证 API 调用
7. **CSS 柱状图**：对照 `DashboardView.vue` 中的 `barHeight` 函数，用 CSS 实现一个简单的柱状图组件

---

## 下一章预告

第 18.5 章将学习前端测试，包括 Vitest 环境搭建、Vue 组件测试、Pinia Store 测试、Vue Router 导航守卫测试以及组合式函数测试。
