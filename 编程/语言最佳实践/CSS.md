# CSS 最佳实践

## 现代布局

### Flexbox 一维布局

```css
/* ✅ 水平居中 */
.container {
  display: flex;
  justify-content: center;
  align-items: center;
}

/* ✅ 导航栏：Logo 左，链接右 */
.navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 1rem;
}

/* ✅ 卡片等高 */
.card-list {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
}

.card {
  flex: 1 1 300px; /* 最小 300px，等分空间 */
}
```

### Grid 二维布局

```css
/* ✅ 经典圣杯布局 */
.layout {
  display: grid;
  grid-template-columns: 250px 1fr 200px;
  grid-template-rows: auto 1fr auto;
  grid-template-areas:
    "header header header"
    "sidebar main aside"
    "footer footer footer";
  min-height: 100vh;
}

.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
.main    { grid-area: main; }
.aside   { grid-area: aside; }
.footer  { grid-area: footer; }

/* ✅ 响应式网格 */
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 1.5rem;
}

/* ✅ 子网格（subgrid） */
.card {
  display: grid;
  grid-template-rows: subgrid;
  grid-row: span 3; /* 继承父级行定义 */
}
```

## 现代 CSS 特性

### CSS 自定义属性（变量）

```css
/* ✅ 定义设计令牌 */
:root {
  /* 颜色 */
  --color-primary: #3b82f6;
  --color-primary-hover: #2563eb;
  --color-text: #1f2937;
  --color-bg: #ffffff;
  --color-border: #e5e7eb;

  /* 间距 */
  --space-xs: 0.25rem;
  --space-sm: 0.5rem;
  --space-md: 1rem;
  --space-lg: 1.5rem;
  --space-xl: 2rem;

  /* 圆角 */
  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;
  --radius-lg: 1rem;

  /* 阴影 */
  --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.05);
  --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.1);
}

/* ✅ 暗色模式 */
@media (prefers-color-scheme: dark) {
  :root {
    --color-text: #f9fafb;
    --color-bg: #111827;
    --color-border: #374151;
  }
}

/* ✅ 或通过 class 切换 */
[data-theme="dark"] {
  --color-text: #f9fafb;
  --color-bg: #111827;
}
```

### 逻辑属性

```css
/* ✅ 使用逻辑属性支持 RTL */
.element {
  margin-inline-start: 1rem;  /* 替代 margin-left */
  padding-block: 0.5rem;      /* 替代 padding-top/bottom */
  border-inline-end: 1px solid var(--color-border); /* 替代 border-right */
  inset-inline-start: 0;      /* 替代 left */
}
```

### 容器查询

```css
/* ✅ 基于容器尺寸响应 */
.card-container {
  container-type: inline-size;
  container-name: card;
}

@container card (min-width: 400px) {
  .card {
    flex-direction: row;
  }
}

@container card (max-width: 399px) {
  .card {
    flex-direction: column;
  }
}
```

### `:has()` 选择器

```css
/* ✅ 父级选择 */
.form-group:has(input:invalid) {
  border-color: red;
}

/* ✅ 条件样式 */
.card:has(img) {
  grid-template-rows: auto 1fr;
}

/* ✅ 空状态 */
.list:has(> :nth-child(1)) .empty-state {
  display: none;
}
```

## 命名约定

### BEM 命名

```css
/* ✅ BEM: Block__Element--Modifier */
.card { }
.card__title { }
.card__body { }
.card__footer { }
.card--featured { }
.card__title--large { }

/* ✅ 避免过度嵌套 */
/* ❌ Bad */
.card .card-body .card-body-title .card-body-title-text { }

/* ✅ Good */
.card__body-title { }
```

## 响应式设计

### 移动优先

```css
/* ✅ 基础样式 = 移动端 */
.container {
  padding: var(--space-sm);
  font-size: 14px;
}

/* 平板 */
@media (min-width: 768px) {
  .container {
    padding: var(--space-md);
    font-size: 16px;
  }
}

/* 桌面 */
@media (min-width: 1024px) {
  .container {
    padding: var(--space-lg);
    max-width: 1200px;
    margin: 0 auto;
  }
}
```

### 流体排版

```css
/* ✅ clamp() 实现流体尺寸 */
h1 {
  font-size: clamp(1.5rem, 4vw, 3rem);
  line-height: 1.2;
}

.container {
  padding: clamp(1rem, 3vw, 3rem);
}

/* ✅ 流体间距 */
.stack {
  display: flex;
  flex-direction: column;
  gap: clamp(0.5rem, 2vw, 2rem);
}
```

## 组件模式

### 按钮系统

```css
/* ✅ 可复用的按钮 */
.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: var(--space-sm);
  padding: var(--space-sm) var(--space-md);
  border: 2px solid transparent;
  border-radius: var(--radius-md);
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
}

.btn--primary {
  background: var(--color-primary);
  color: white;
}

.btn--primary:hover {
  background: var(--color-primary-hover);
}

.btn--outline {
  background: transparent;
  border-color: var(--color-primary);
  color: var(--color-primary);
}

.btn--sm { padding: var(--space-xs) var(--space-sm); font-size: 0.875rem; }
.btn--lg { padding: var(--space-md) var(--space-lg); font-size: 1.125rem; }

/* 聚焦样式 */
.btn:focus-visible {
  outline: 3px solid var(--color-primary);
  outline-offset: 2px;
}
```

### 卡片组件

```css
.card {
  background: var(--color-bg);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-lg);
  box-shadow: var(--shadow-sm);
  overflow: hidden;
  transition: box-shadow 0.2s ease;
}

.card:hover {
  box-shadow: var(--shadow-md);
}

.card__image {
  width: 100%;
  aspect-ratio: 16 / 9;
  object-fit: cover;
}

.card__content {
  padding: var(--space-md);
}
```

## 动画与过渡

```css
/* ✅ 尊重用户偏好 */
@media (prefers-reduced-motion: no-preference) {
  .fade-in {
    animation: fadeIn 0.3s ease-out;
  }
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to   { opacity: 1; transform: translateY(0); }
}

/* ✅ 过渡只作用于预期属性 */
.element {
  transition: opacity 0.2s ease, transform 0.2s ease;
}

/* ✅ 视差效果 */
.parallax {
  transform: translateZ(-1px) scale(2);
}
```

## 性能建议

- 使用 `will-change` 仅对已知会动画的元素
- 优先使用 `transform` 和 `opacity` 做动画（GPU 加速）
- 避免 `@import`，使用 `<link>` 加载样式
- 使用 `content-visibility: auto` 延迟渲染屏幕外内容
- 图片使用 `aspect-ratio` 避免 CLS（累积布局偏移）
- 使用 `contain` 属性隔离布局计算

## 参考资源

- [MDN CSS 教程](https://developer.mozilla.org/zh-CN/docs/Web/CSS)
- [CSS Tricks](https://css-tricks.com/)
- [Every Layout](https://every-layout.dev/)
- [CSS for JS Developers](https://css-for-js.dev/)
- [Web.dev - Learn CSS](https://web.dev/learn/css/)
