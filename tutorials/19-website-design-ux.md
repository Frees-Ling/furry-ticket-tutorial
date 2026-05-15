# 第19章：网站设计与 UX

## 本章学习目标

本章将学习网站设计核心原理，从 HTML5 语义化到 CSS3 布局、响应式设计和宣发功能。

**学习路径：**

1. HTML5 语义化标签
2. CSS Flexbox 完全教程
3. CSS Grid 完全教程
4. CSS 动画与过渡
5. 响应式设计
6. Web 可访问性
7. UI/UX 设计原则
8. 活动列表页设计
9. 宣发功能设计

---

## 1. HTML5 语义化标签

### 1.1 语义标签

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>票务系统</title>
</head>
<body>
    <header>
        <nav>
            <ul>
                <li><a href="/">首页</a></li>
                <li><a href="/events">活动</a></li>
            </ul>
        </nav>
    </header>

    <main>
        <section class="hero">
            <h1>发现精彩演出</h1>
            <p>在线选座，轻松购票</p>
        </section>

        <section class="events">
            <h2>热门活动</h2>
            <article>
                <header>
                    <h3>周杰伦演唱会</h3>
                    <time datetime="2026-06-15">6月15日</time>
                </header>
                <p>国家体育场</p>
                <footer>
                    <span>¥680 起</span>
                </footer>
            </article>
        </section>
    </main>

    <aside>
        <h2>推荐</h2>
        <!-- 侧边推荐内容 -->
    </aside>

    <footer>
        <p>&copy; 2026 票务系统</p>
    </footer>
</body>
</html>
```text

### 1.2 语义标签的意义

| 标签 | 作用 | SEO 影响 |
| ------ | ------ | ---------- |
| `<header>` | 页眉/区块头部 | 提高 |
| `<nav>` | 导航 | 提高 |
| `<main>` | 主要内容（页面唯一） | 提高 |
| `<article>` | 独立内容块 | 提高 |
| `<section>` | 章节分组 | 中 |
| `<aside>` | 侧边栏/补充 | 中 |
| `<footer>` | 页脚/区块尾部 | 中 |
| `<time>` | 时间 | 低 |

---

## 2. CSS Flexbox 完全教程

### 2.1 基础概念

```css
/* Flex 容器属性 */
.container {
    display: flex;           /* 或 inline-flex */
    flex-direction: row;     /* row | row-reverse | column | column-reverse */
    flex-wrap: wrap;        /* nowrap | wrap | wrap-reverse */
    justify-content: center; /* 主轴上对齐方式 */
    align-items: center;    /* 交叉轴上对齐方式 */
    align-content: center;  /* 多行时交叉轴对齐 */
    gap: 16px;              /* 间距（推荐代替 margin） */
}

/* Flex 项目属性 */
.item {
    flex: 1;                /* flex-grow flex-shrink flex-basis 缩写 */
    flex-grow: 1;           /* 放大比例（默认为 0） */
    flex-shrink: 1;         /* 缩小比例（默认为 1） */
    flex-basis: auto;       /* 初始大小 */
    align-self: center;     /* 单个项目交叉轴对齐 */
    order: 1;               /* 排序（默认 0） */
}
```

### 2.2 justify-content 取值

```css
/* 主轴对齐（默认 direction: row 时水平方向） */
justify-content: flex-start;   /* 起始位置 */
justify-content: flex-end;     /* 结束位置 */
justify-content: center;       /* 居中 */
justify-content: space-between; /* 两端对齐 */
justify-content: space-around;  /* 项目两侧间距相等 */
justify-content: space-evenly;  /* 所有间距相等 */
```text

### 2.3 align-items 取值

```css
/* 交叉轴对齐 */
align-items: stretch;    /* 拉伸（默认） */
align-items: flex-start; /* 起始位置 */
align-items: flex-end;   /* 结束位置 */
align-items: center;     /* 居中 */
align-items: baseline;   /* 基线对齐 */
```

### 2.4 Flex 弹性盒模型算法

```text
Flex 容器宽度 = W
项目初始宽度 = flex-basis 之和

如果 初始宽度 < W:
    剩余空间 = W - 初始宽度
    按 flex-grow 比例分配剩余空间
否则:
    超出空间 = 初始宽度 - W
    按 flex-shrink * flex-basis 比例缩减
```

**示例：**

```css
.container { display: flex; width: 600px; }
.item-a { flex: 1; }   /* 200px + (600-300)*1/3 ≈ 300px */
.item-b { flex: 1; }   /* 200px + (600-300)*1/3 ≈ 300px */
.item-c { flex: 1; }   /* 0px + (600-300)*1/3 ≈ 300px (flex-basis: 0 平分) */

/* 更常用的写法：等分 */
.item { flex: 1; }  /* 等价于 flex: 1 1 0 */
/* 所有项目平分容器宽度 */
```text

---

## 3. CSS Grid 完全教程

### 3.1 基本用法

```css
.grid-container {
    display: grid;
    grid-template-columns: repeat(3, 1fr);  /* 3 列等宽 */
    grid-template-rows: auto 1fr auto;        /* 3 行 */
    gap: 16px;                                /* 间距 */
    grid-template-areas:                       /* 区域命名 */
        "header header header"
        "main   main   aside"
        "footer footer footer";
}

.header { grid-area: header; }
.main   { grid-area: main; }
.aside  { grid-area: aside; }
.footer { grid-area: footer; }
```

### 3.2 Grid 布局算法

```text
Grid 容器宽度 = W
grid-template-columns = [col1, col2, ..., colN]

每列宽度计算:
  fr 单位: 按比例分配剩余空间
  px/%/em: 固定宽度
  auto: 内容自适应
  
如果有 1fr 1fr 1fr:
  每列 = (W - gap * (N-1)) / 3

如果有 200px 1fr 1fr:
  第一列 = 200px
  剩余 = (W - 200 - gap * (N-1))
  后两列各 = 剩余 / 2
```

### 3.3 网格线定位

```css
.item {
    grid-column: 1 / 3;     /* 从第 1 条网格线到第 3 条 */
    grid-row: 1 / 3;
    /* 等价于 */
    grid-column: 1 / span 2; /* 跨越 2 列 */
    grid-row: 1 / span 2;
}
```text

---

## 3.5 现代 CSS 核心特性

### 3.5.1 CSS 自定义属性（变量）

自定义属性是 CSS 中最强大的特性之一，实现主题化和设计系统的基础。

```css
:root {
  /* 颜色系统 */
  --color-primary: #4a90d9;
  --color-danger: #e74c3c;
  --color-success: #2ecc71;
  --color-text: #333;
  --color-bg: #fff;

  /* 间距系统 */
  --space-xs: 4px;
  --space-sm: 8px;
  --space-md: 16px;
  --space-lg: 24px;
  --space-xl: 48px;

  /* 排版系统 */
  --font-size-sm: 0.875rem;
  --font-size-base: 1rem;
  --font-size-lg: 1.25rem;
  --font-size-xl: 2rem;
}

/* 使用变量 */
.card {
  background: var(--color-bg);
  color: var(--color-text);
  padding: var(--space-md);
  font-size: var(--font-size-base);
}

/* 变量默认值 */
.title {
  color: var(--color-primary, #4a90d9);  /* 第二个参数为默认值 */
}

/* 在 JavaScript 中修改（实现主题切换） */
// document.documentElement.style.setProperty('--color-primary', '#ff6b6b');
```

> **本项目应用**：`frontend/src/assets/styles/components.css` 中使用了 CSS 变量管理组件主题。

### 3.5.2 box-sizing: border-box

这是 CSS 布局中最关键的属性之一，决定元素的尺寸计算方式。

```css
/* 标准盒模型（默认） */
.box-standard {
  box-sizing: content-box;
  width: 100px;
  padding: 10px;
  border: 1px solid;
  /* 实际宽度 = 100 + 10*2 + 1*2 = 122px */
}

/* 怪异盒模型（推荐） */
.box-border {
  box-sizing: border-box;
  width: 100px;
  padding: 10px;
  border: 1px solid;
  /* 实际宽度 = 100px（padding和border向内挤压） */
}

/* 全局重置（推荐） */
*, *::before, *::after {
  box-sizing: border-box;
}
```text

### 3.5.3 calc() / min() / max() / clamp()

CSS 数学函数让响应式布局不再需要媒体查询。

```css
/* calc — 任意单位的混合计算 */
.sidebar {
  width: calc(100% - 250px);  /* 剩余宽度 */
}
.column {
  width: calc(100% / 3 - 20px);  /* 三等分，带间距 */
}

/* min — 取最小值（响应式首选） */
.container {
  width: min(90%, 1200px);  /* 在 1200px 以下为 90%，以上为 1200px */
}

/* max — 取最大值 */
.text {
  font-size: max(1rem, 2.5vw);  /* 不小于 1rem */
}

/* clamp — 限制在范围内（流体排版） */
.title {
  font-size: clamp(1.5rem, 3vw + 1rem, 3rem);
  /* 最小值 1.5rem，优选 3vw+1rem，最大值 3rem */
}

/* 流体间距 */
.card {
  padding: clamp(1rem, 5vw, 3rem);
}
```

### 3.5.4 position 定位

```css
/* static（默认）— 正常文档流 */
.static-box { position: static; }

/* relative — 相对自身原始位置偏移 */
.relative-box {
  position: relative;
  top: 10px;   /* 向下偏移 10px */
  left: 20px;  /* 向右偏移 20px */
  /* 原始位置保留，其他元素不受影响 */
}

/* absolute — 脱离文档流，相对于最近的非 static 祖先定位 */
.absolute-box {
  position: absolute;
  top: 0;
  right: 0;
  /* 相对于最近的 position:relative/absolute/fixed 祖先 */
}
.card-container { position: relative; }  /* 创建定位参考 */

/* fixed — 相对于视口固定 */
.fixed-header {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  z-index: 1000;
}

/* sticky — 粘性定位（滚动到阈值后固定） */
.sticky-nav {
  position: sticky;
  top: 0;  /* 滚动到顶部时固定 */
  z-index: 100;
}
```text

### 3.5.5 z-index 与层叠上下文

`z-index` 是 CSS 中最大的坑之一，理解层叠上下文是解决 z-index 问题的关键。

```css
/* 基础用法 */
.modal-overlay {
  position: fixed;
  z-index: 1000;  /* 数值越大越靠前 */
}

/* 层叠上下文：每个上下文内部独立比较 z-index */
/* 以下代码中，.box-b 的 z-index 为 999，但实际在 .box-a（z-index:1）之下 */
/* 因为 .box-b 的父容器 .container-b 的 z-index 小于 .container-a */

.container-a { position: relative; z-index: 1; }
.container-b { position: relative; z-index: 0; }
.box-a { position: absolute; z-index: 1; }
.box-b { position: absolute; z-index: 999; }  /* 仍被 .box-a 覆盖！ */

/* 创建新层叠上下文的方式 */
.isolate {
  isolation: isolate;  /* 最简单的方式，强制创建独立上下文 */
}
/* 其他方式：opacity < 1、transform 非 none、filter 非 none 等 */
```

---

## 4. CSS 动画

### 4.1 transition

```css
.button {
    background-color: #4CAF50;
    transition: background-color 0.3s ease, transform 0.2s;
    /* transition: property duration timing-function delay */
}

.button:hover {
    background-color: #45a049;
    transform: translateY(-2px);
}
```text

### 4.2 @keyframes

```css
@keyframes fadeIn {
    from { opacity: 0; transform: translateY(20px); }
    to   { opacity: 1; transform: translateY(0); }
}

@keyframes pulse {
    0%   { transform: scale(1); }
    50%  { transform: scale(1.05); }
    100% { transform: scale(1); }
}

.element {
    animation: fadeIn 0.5s ease-out, pulse 2s infinite;
}
```

---

## 5. 响应式设计

### 5.1 断点系统

```css
/* Mobile First 设计：先写移动端样式，再用媒体查询增强 */

/* 基础样式（移动端） */
.container { padding: 16px; }
.grid { grid-template-columns: 1fr; }

/* 平板（≥768px） */
@media (min-width: 768px) {
    .container { padding: 24px; }
    .grid { grid-template-columns: repeat(2, 1fr); }
}

/* 桌面（≥1024px） */
@media (min-width: 1024px) {
    .container { padding: 32px; max-width: 1200px; margin: 0 auto; }
    .grid { grid-template-columns: repeat(3, 1fr); }
}

/* 大屏（≥1440px） */
@media (min-width: 1440px) {
    .grid { grid-template-columns: repeat(4, 1fr); }
}
```text

### 5.2 相对单位

```css
/* 推荐使用 rem/em/% 代替 px */
html { font-size: 16px; }     /* 基础字号 */

h1 { font-size: 2rem; }       /* 32px */
p  { font-size: 1rem; }       /* 16px */
small { font-size: 0.875rem; } /* 14px */

.container { width: 100%; max-width: 1200px; }  /* 流式宽度 */
.padding   { padding: 1em; }   /* 相对当前字号 */

/* vw/vh 视口单位 */
.hero { height: 60vh; }        /* 60% 视口高度 */
.title { font-size: 5vw; }     /* 响应式字号 */
```

### 5.3 图片自适应

```css
img {
    max-width: 100%;
    height: auto;
    display: block;
}

/* 背景图片自适应 */
.hero {
    background-image: url('hero.jpg');
    background-size: cover;
    background-position: center;
    background-repeat: no-repeat;
}
```text

---

## 6. Web 可访问性

### 6.1 ARIA 属性

```html
<!-- 按钮有 aria-label 方便屏幕阅读器 -->
<button aria-label="关闭" onclick="close()">
    <span aria-hidden="true">×</span>
</button>

<!-- 角色标注 -->
<nav role="navigation" aria-label="主导航">
<ul role="menubar">
    <li role="menuitem"><a href="/">首页</a></li>
</ul>
</nav>

<!-- 状态提示 -->
<div role="alert" aria-live="polite">
    订单创建成功
</div>
```

### 6.2 键盘导航

```css
/* 焦点样式 */
:focus-visible {
    outline: 2px solid #4CAF50;
    outline-offset: 2px;
}

/* 移除鼠标焦点（保留键盘焦点） */
:focus:not(:focus-visible) {
    outline: none;
}
```text

```html
<!-- tabindex 控制 Tab 顺序 -->
<input tabindex="1">
<button tabindex="2">
```

---

## 7. UI/UX 设计原则

### 7.1 F 型浏览模式

用户浏览网页的视线呈 F 型：

1. 水平扫描顶部（读取导航和标题）
2. 向下移动，水平扫描中间（读取副标题）
3. 垂直扫描左侧（快速浏览列表）

**设计启示：**

- 重要信息和 CTA（Call to Action）放在左上角
- 使用清晰的标题层级（h1 → h2 → h3）
- 列表项左侧对齐，便于垂直扫描

### 7.2 希克定律（Hick's Law）

**定义：** 用户做出决定所需时间与可用选项数量成正比。

```text
T = a + b * log₂(n)
T = 反应时间, n = 选项数量, a/b = 常数
```

**设计启示：**

- 导航选项不超过 7 个
- 分步表单优于长表单
- 使用渐进式展示（先显示主要选项，次要选项折叠）

### 7.3 费茨定律（Fitts' Law）

**定义：** 移动到目标所需时间取决于目标距离和大小。

```text
T = a + b * log₂(D/W + 1)
T = 移动时间, D = 距离, W = 目标宽度
```

**设计启示：**

- 按钮要大（至少 44×44px）
- 重要按钮放在边缘/角落（屏幕边缘无限大）
- 相关操作按钮距离要近

### 7.4 雅各布定律（Jakob's Law）

**定义：** 用户在其他网站花的时间比在你的网站多，所以他们希望你的网站像其他网站一样工作。

**设计启示：**

- 遵循常见设计模式（购物车在右上角、Logo 在左上角）
- 不要重新发明轮子（使用标准 UI 组件）
- 保持一致的设计语言

---

## 8. 宣发功能设计

### 8.1 首页轮播

```html
<section class="hero-carousel">
    <div class="carousel-container">
        <div class="slide active">
            <img src="/images/banner1.jpg" alt="周杰伦演唱会">
            <div class="slide-content">
                <h2>周杰伦 2026 巡回演唱会</h2>
                <p>6月15日 · 国家体育场</p>
                <a href="/events/1" class="btn-primary">立即购票</a>
            </div>
        </div>
    </div>
</section>
```text

```css
.hero-carousel { height: 60vh; min-height: 400px; position: relative; overflow: hidden; }
.slide { position: absolute; inset: 0; opacity: 0; transition: opacity 0.5s; }
.slide.active { opacity: 1; }
.slide-content { position: absolute; bottom: 20%; left: 10%; color: white; }
.btn-primary { display: inline-block; padding: 12px 32px; background: #ff6b35; color: white; border-radius: 8px; text-decoration: none; font-weight: bold; }
```

### 8.2 热门活动推荐

```html
<section class="featured-events">
    <h2>热门推荐</h2>
    <div class="event-grid">
        <article v-for="event in hotEvents" :key="event.id" class="event-card">
            <div class="event-image">
                <img :src="event.cover_image || '/images/placeholder.jpg'" :alt="event.title">
                <span class="event-category">{{ event.category }}</span>
            </div>
            <div class="event-info">
                <h3>{{ event.title }}</h3>
                <p class="event-venue">📍 {{ event.venue }}</p>
                <p class="event-date">📅 {{ formatDate(event.start_date) }}</p>
                <div class="event-footer">
                    <span class="event-price">¥{{ event.price }} 起</span>
                    <span class="event-stock" :class="{ low: event.sold_count / event.total_stock > 0.8 }">
                        {{ event.total_stock - event.sold_count }} 张剩余
                    </span>
                </div>
            </div>
        </article>
    </div>
</section>
```text

### 8.3 分享功能

```javascript
// 分享链接
function shareEvent(event) {
    const url = `${window.location.origin}/events/${event.id}`;
    const text = `🎫 ${event.title} - ${event.venue} @ ${event.start_date}`;

    // Web Share API (移动端原生分享)
    if (navigator.share) {
        navigator.share({ title: event.title, text, url });
    } else {
        // 复制链接到剪贴板
        navigator.clipboard.writeText(url);
        alert('链接已复制');
    }
}
```

---

## 本章总结

| 知识点 | 掌握程度 |
| -------- | --------- |
| HTML5 语义化 | 理解标签含义和使用场景 |
| Flexbox 布局 | 能进行任何单维布局 |
| Grid 布局 | 能进行二维网格布局 |
| 响应式设计 | 能实现移动优先的适配 |
| 可访问性 | 了解 ARIA 和键盘导航 |
| UX 原则 | 理解 F 型/希克/费茨/雅各布定律 |

---

## 安全防护

### 表单安全

表单是 XSS 和数据注入的主要入口：

1. **所有用户输入必须在服务端校验**（前端校验仅用于用户体验）
2. **输出编码**：用户生成的内容必须 HTML 编码后展示
3. **CSRF 防护**：本系统使用 Bearer Token（非 cookie），天然抗 CSRF
4. **表单长度限制**：用户名 3-50 字，密码 8-128 字

### 安全标头与设计

本系统后端自动添加以下安全标头，影响前端行为：

| 标头 | 对前端的影响 |
| ------ | ------------- |
| `X-Frame-Options: DENY` | 页面不能在 iframe 中加载 |
| `Content-Security-Policy` | 限制资源加载来源 |
| `Referrer-Policy` | 控制 Referer 头信息 |
| `X-Content-Type-Options: nosniff` | 防止 MIME 嗅探 |

### HTTPS 与 HSTS

生产环境必须启用 HTTPS。HSTS 标头告诉浏览器：未来一年内只允许 HTTPS 访问，即使用户手动输入 `http://` 也会被浏览器自动转换为 `https://`。

### 可访问性与安全

ARIA 标签和语义化 HTML 不仅提升可访问性，也间接提升安全性：

- 清晰的表单标签帮助用户识别钓鱼攻击
- 语义化结构减少对 JavaScript 的依赖（降低 XSS 风险）
- 键盘导航不依赖事件监听（减少攻击面）

---

## 常见问题与排错

| 问题 | 原因 | 解决 |
| ------ | ------ | ------ |
| Flex 布局溢出容器 | 未设置 `flex-wrap: wrap` | 添加 `flex-wrap: wrap` 或减少 `flex-basis` |
| Grid 内容超出单元格 | 未设置 `min-width: 0` | 给 grid item 添加 `min-width: 0` 覆盖默认值 |
| 响应式断点不生效 | CSS 顺序问题 | 移动端基础样式在前，`@media` 查询在后（Mobile First） |
| `z-index` 无效 | 未设置 `position` | `z-index` 只对定位元素生效（relative/absolute/fixed） |
| 字体大小在不同设备不一致 | 未设置 `html { font-size }` | 设置基础字号后使用 `rem` 单位 |
| 图片在移动端溢出 | 未限制最大宽度 | `img { max-width: 100%; height: auto; }` |
| CSS 动画卡顿 | 触发了重排（reflow） | 只用 `transform` 和 `opacity` 做动画，避免改变宽高 |
| Safari 下 100vh 溢出 | Safari 视口定义不同 | 使用 `100dvh`（dynamic viewport height）或 JavaScript 计算 |
| :focus-visible 样式未出现 | 浏览器兼容性 | 添加 `:focus` 回退样式 |
| ARIA 标签未生效 | 屏幕阅读器兼容 | 同时使用语义化标签 + ARIA 辅助 |

---

## 本章练习

1. **实现 EventCard 组件**：使用 Flexbox 完成票务系统的活动卡片组件，包含图片、标题、场馆、价格和剩余票数，并添加 hover 效果
2. **宣发首页轮播**：用 CSS 实现自动轮播的首页 Banner（至少 3 张幻灯片），支持自动播放和指示器
3. **响应式活动列表**：实现 Mobile First 的活动网格——移动端 1 列、平板 2 列、桌面 3 列
4. **表单验证状态**：设计登录表单的 3 种视觉状态（默认、聚焦、错误），并考虑无障碍提示
5. **项目 CSS 重构**：对照 `frontend/src/assets/styles/components.css`，分析实际项目的 CSS 设计，找出可以改进的地方
6. **可访问性审计**：使用 Chrome Lighthouse 的 Accessibility 面板审计项目的 EventList 页面，修复发现的问题

---

## 下一章预告

第 20 章将学习前后端完整集成，包括 CORS 跨域配置、JWT 令牌前端管理、Axios 请求封装、WebSocket 客户端、支付宝支付流程和完整的集成测试。
