---
title: 01-现代 CSS：布局、选择器与工程方案
publish: true
---

<ArticleTopAd></ArticleTopAd>

> 本章为 2026 年增补章节。原教程「02-CSS基础」「03-CSS进阶」在盒模型、浮动、定位、层叠这些**恒定知识**上依然扎实，不需重学。但 CSS 在 2019 之后补齐了大量以前必须靠 hack 或 JS 才能实现的能力，这部分原教程完全没有。

## 一、布局：Flex 之后是 Grid

### 1. Flex 主内，Grid 主外

这是最实用的分工原则：

| | Flexbox | Grid |
|---|---|---|
| 维度 | 一维（行 **或** 列） | **二维**（行 **和** 列） |
| 适合 | 导航条、按钮组、卡片内部对齐 | 页面骨架、卡片网格、仪表盘 |
| 尺寸决定 | 内容撑开容器 | 容器决定格子 |

```css
/* Flex：一行内元素分布 */
.toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;          /* 用 gap，不要再写 margin hack */
}

/* Grid：二维页面骨架 */
.layout {
  display: grid;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
  grid-template-columns: 240px 1fr;
  grid-template-rows: 64px 1fr 80px;
  min-height: 100vh;
  gap: 16px;
}

.layout > header { grid-area: header; }
.layout > aside  { grid-area: sidebar; }
.layout > main   { grid-area: main; }
.layout > footer { grid-area: footer; }
```

### 2. Grid 的弹性列

```css
/* 自动响应式卡片网格：不用写任何媒体查询 */
.cards {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 20px;
}
```

这一条几乎可以替代过去 90% 的"响应式卡片列表"媒体查询。

### 3. `gap` 取代 margin

```css
.list { display: flex; flex-direction: column; gap: 8px; }
```

好处：最后一项不再多出一个外边距，也不需要用 `:last-child` 去清除。

### 4. 居中

```css
/* 最简：单元素居中 */
.center { display: grid; place-items: center; }
```

## 二、现代选择器

### 1. `:has()` —— 父级选择器

CSS 有史以来最重要的补齐能力之一。以前"父元素根据子元素状态变化"只能靠 JS，现在纯 CSS 可写：

```css
/* 有勾选时高亮整行 */
.card:has(input:checked) {
  border-color: #2563eb;
  background: #eff6ff;
}

/* 表单里存在错误提示时，把整个字段标红 */
.field:has(.error) {
  color: #dc2626;
}

/* 图片加载失败时的容器样式 */
img:has(+ figcaption) { /* ... */ }
```

> 兼容性：2023 年后主流浏览器均已支持。

### 2. `:is()` 与 `:where()`

替代冗长的选择器列表：

```css
/* 以前 */
h1:hover, h2:hover, h3:hover { color: red; }

/* 现在 */
:is(h1, h2, h3):hover { color: red; }
```

区别：`:where()` 的**优先级为 0**，非常适合做基础样式重置，便于后续覆盖。

```css
:where(h1, h2, h3) {
  margin: 0;
  font-weight: 600;
}
```

### 3. `:not()` 升级版

```css
/* 除最后一项外的所有项加分隔线 */
li:not(:last-child) { border-bottom: 1px solid #eee; }

/* 排除多个 */
input:not([type="checkbox"], [type="radio"]) { ... }
```

### 4. `:nth-child()` 的公式仍然好用

```css
li:nth-child(odd) { background: #fafafa; }
li:nth-child(3n + 1) { clear: both; }
```

## 三、CSS 变量（自定义属性）

CSS 变量是**运行时**的：能被 JS 随时读写、能随 DOM 继承、能在 `@media` 条件里作为取值参与计算——这和 Sass / Less 的编译期变量是两回事。Sass 变量编译完就消失了，CSS 变量活到运行结束。

```css
:root {
  --color-primary: #2563eb;
  --color-text: #1f2937;
  --space-2: 8px;
  --radius: 8px;
  --header-h: 64px;
}

.button {
  background: var(--color-primary);
  padding: var(--space-2) calc(var(--space-2) * 2);
  border-radius: var(--radius);
}

/* 提供了回退值 */
.text { color: var(--color-muted, #6b7280); }

/* 组件内局部覆盖，实现主题化 */
:root[data-theme="dark"] {
  --color-text: #e5e7eb;
  --color-bg: #111827;
}
```

运行时切换换肤：

```js
document.documentElement.dataset.theme = 'dark';
```

**这是暗黑模式的主流实现方式**，不需要维护两套 CSS。

## 四、容器查询

媒体查询看的是**视口**宽度；容器查询看的是**父容器**宽度。这让组件真正实现"在哪都自适应"。

```css
.card-wrap {
  container-type: inline-size;
  container-name: card;
}

@container card (min-width: 480px) {
  .card {
    display: flex;
    gap: 16px;
  }
}

@container card (max-width: 479px) {
  .card__cover {
    width: 100%;
  }
}
```

**实际意义**：同一个卡片组件放进侧边栏（窄）和主内容区（宽），不用改一行 JS，自动切换布局。这是原教程的浮动/媒体查询知识体系无法覆盖的。

## 五、响应式单位与函数

### 1. `clamp()` —— 流式排版

```css
.title {
  /* 最小 24px，随视口增长，最大 56px */
  font-size: clamp(24px, 4vw + 1rem, 56px);
}

.section {
  padding-block: clamp(24px, 6vw, 80px);
}
```

一行替代过去"多个断点各写一次 font-size"的写法。

### 2. 其它有用的单位

| 单位 | 含义 |
|---|---|
| `rem` | 相对根字号，无障碍友好，**字号首选** |
| `em` | 相对当前字体，适合内边距 |
| `dvh` / `svh` | 动态视口高度，**解决移动端 `100vh` 被地址栏截断的老问题** |
| `ch` | 字符宽度，适合限制正文行宽 |
| `cqi` | 容器内联尺寸的百分比（配合容器查询） |

```css
.hero {
  min-height: 100dvh;     /* 移动端不再跳动 */
}

.prose {
  max-width: 65ch;        /* 正文最佳阅读行宽 */
}
```

### 3. `aspect-ratio`

```css
.thumb {
  aspect-ratio: 16 / 9;
  object-fit: cover;
  width: 100%;
}
```

彻底告别 padding-top 百分比占位法。

### 4. `object-fit`

```css
img { width: 100%; height: 200px; object-fit: cover; }
```

## 六、现代视觉效果

```css
/* 圆角与阴影分层 */
.card {
  border-radius: 12px;
  box-shadow:
    0 1px 2px rgb(0 0 0 / 0.06),
    0 8px 24px rgb(0 0 0 / 0.08);
}

/* 毛玻璃 */
.glass {
  background: rgb(255 255 255 / 0.6);
  backdrop-filter: blur(12px);
  border: 1px solid rgb(255 255 255 / 0.3);
}

/* 粘性定位 */
.toc { position: sticky; top: 16px; }

/* 平滑滚动 + 尊重用户偏好 */
html { scroll-behavior: smooth; }
@media (prefers-reduced-motion: reduce) {
  html { scroll-behavior: auto; }
  *, *::before, *::after { animation-duration: 0.01ms !important; }
}

/* 暗色模式自动适配 */
@media (prefers-color-scheme: dark) {
  :root { --color-bg: #111827; }
}

/* 颜色空间：更鲜艳的渐变 */
.gradient {
  background: linear-gradient(135deg in oklch, #2563eb, #7c3aed);
}
```

## 七、动画：优先用 transform 与 opacity

```css
.box {
  transition: transform 0.2s ease, opacity 0.2s ease;
  will-change: transform;
}

.box:hover {
  transform: translateY(-2px) scale(1.02);
}

/* 进入动画 */
@keyframes fade-up {
  from { opacity: 0; transform: translateY(8px); }
  to   { opacity: 1; transform: none; }
}
.fade-up { animation: fade-up 0.3s ease both; }
```

**性能要点**：`transform` 与 `opacity` 只触发**合成**，不触发布局与重绘，是最可靠的高性能动画属性。动画里改 `width` / `top` / `margin` 会触发布局，务必避免。

> 补充：`filter`、`clip-path`、`backdrop-filter` 等属性在多数浏览器中也能走合成层，但支持度与表现不如 `transform`/`opacity` 稳定，性能敏感场景仍优先用后两者。

### `@starting-style` 与 `transition-behavior`（新特性）

```css
/* 元素首次出现时也播放过渡（以前只能靠 JS 加类） */
.dialog {
  transition: opacity 0.3s, display 0.3s allow-discrete;
  opacity: 1;

  @starting-style {
    opacity: 0;
  }
}
```

## 八、工程方案：怎么组织 CSS

| 方案 | 代表 | 适用 |
|---|---|---|
| **原子化 CSS** | Tailwind CSS | 中大型项目主流，产物小、无命名成本 |
| CSS Modules | Vite 内置支持 | 传统组件样式隔离 |
| 组件内 `<style scoped>` | Vue SFC | Vue 项目默认 |
| CSS-in-JS | styled-components / Emotion | 逐渐降温，运行时开销 |
| 预处理器 | Sass / Less | 仍需处理嵌套与 mixin 时可留 |

### Tailwind 速览

```html
<div class="flex items-center gap-3 rounded-xl bg-white p-4 shadow-sm">
  <img class="h-10 w-10 rounded-full object-cover" src="a.png" />
  <div class="min-w-0">
    <p class="truncate text-sm font-medium text-gray-900">标题</p>
    <p class="truncate text-xs text-gray-500">描述</p>
  </div>
</div>
```

优点：样式与结构在一起、不用起名、删代码即删样式、产物自动只含用到的类。
缺点：类名串长，需要团队约定（可用 `@apply` 或组件封装抽离）。

### 现代重置（推荐基底）

```css
*, *::before, *::after { box-sizing: border-box; }

* { margin: 0; }

body {
  line-height: 1.5;
  -webkit-font-smoothing: antialiased;
}

img, picture, video, canvas, svg {
  display: block;
  max-width: 100%;
}

input, button, textarea, select {
  font: inherit;
}

p, h1, h2, h3, h4, h5, h6 {
  overflow-wrap: break-word;
}

#root, #app {
  isolation: isolate;
}
```

## 九、兼容性判断方法

不必记兼容表，用 [caniuse.com](https://caniuse.com/) 或 [Baseline](https://web-platform-dx.github.io/web-features/) 查询。

**Baseline 的概念**：某特性在所有主流浏览器稳定可用后进入 Baseline。新项目可放心用 Baseline 特性；要求兼容老设备时用 `@supports` 做降级：

```css
.flex { display: flex; }

@supports (container-type: inline-size) {
  .wrap { container-type: inline-size; }
}
```

## 十、原教程 CSS 内容的有效性判断

| 原教程内容 | 现在 |
|---|---|
| 盒模型、`box-sizing` | ✅ 完全有效，必学 |
| 浮动 `float` 布局 | ⚠️ 仅用于文字环绕，**布局基本不用了** |
| 定位 `position` | ✅ 有效，`sticky` 是新增重点 |
| 层叠、优先级、`z-index` | ✅ 有效 |
| `display: inline-block` 布局 | ⚠️ 已被 Flex/Grid 取代 |
| 媒体查询 | ✅ 有效，但很多场景可被容器查询/clamp 替代 |
| 清除浮动 | ⚠️ 布局不再需要，仅遗留代码维护 |
| Flex | ✅ 有效，需补 `gap` |
| **Grid** | ❌ 原教程几乎未覆盖，**必须补** |
| **CSS 变量** | ❌ 几乎未覆盖，**必须补** |
| **`:has()` / 容器查询 / `clamp()` / `dvh`** | ❌ 完全未覆盖 |

## 十一、小结

- 布局：Flex 管一维，Grid 管二维；`grid-template-areas` 画骨架，`auto-fill + minmax` 做自适应网格。
- `gap` 替代 margin 间距 hack。
- `:has()` 让纯 CSS 也能感知子元素状态。
- CSS 变量是主题化与运行时换肤的基础。
- 容器查询让组件真正"在哪都自适应"。
- `clamp()` + `dvh` + `aspect-ratio` 是移动端三件套。
- 动画只动 `transform` / `opacity`。
- 工程上 Tailwind 是当前最主流选择，CSS Modules 仍然稳妥。

## 参考

- MDN CSS 布局：<https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_grid_layout>
- 现代 CSS 重置：<https://www.joshwcomeau.com/css/custom-css-reset/>
- Tailwind CSS：<https://tailwindcss.com/docs>
- Baseline：<https://web-platform-dx.github.io/web-features/>