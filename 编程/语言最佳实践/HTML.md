# HTML 最佳实践

## 文档结构

### 标准文档模板

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta name="description" content="页面描述，用于 SEO">
  <title>页面标题</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <header>
    <nav aria-label="主导航">
      <!-- 导航内容 -->
    </nav>
  </header>

  <main>
    <article>
      <h1>页面主标题</h1>
      <!-- 主要内容 -->
    </article>
  </main>

  <footer>
    <!-- 页脚内容 -->
  </footer>
</body>
</html>
```

## 语义化标签

### 正确使用语义元素

```html
<!-- ❌ Bad - div 滥用 -->
<div class="header">
  <div class="nav">...</div>
</div>
<div class="main">...</div>
<div class="footer">...</div>

<!-- ✅ Good - 语义化 -->
<header>
  <nav>...</nav>
</header>
<main>...</main>
<footer>...</footer>
```

### 语义元素速查

| 标签 | 用途 |
|------|------|
| `<header>` | 页头或章节头部 |
| `<nav>` | 导航链接区域 |
| `<main>` | 页面主要内容（每页仅一个） |
| `<article>` | 独立的完整内容块 |
| `<section>` | 主题性分组 |
| `<aside>` | 侧边栏、辅助内容 |
| `<footer>` | 页脚或章节尾部 |
| `<figure>` | 自包含的图片/图表/代码 |
| `<figcaption>` | figure 的说明文字 |
| `<time>` | 日期/时间 |
| `<address>` | 联系信息 |
| `<details>` | 可展开的详情 |
| `<mark>` | 标记/高亮文本 |

### 列表与表格

```html
<!-- ✅ 导航用列表 -->
<nav aria-label="主导航">
  <ul>
    <li><a href="/" aria-current="page">首页</a></li>
    <li><a href="/about">关于</a></li>
    <li><a href="/contact">联系</a></li>
  </ul>
</nav>

<!-- ✅ 表格语义化 -->
<table>
  <caption>2024年销售数据</caption>
  <thead>
    <tr>
      <th scope="col">月份</th>
      <th scope="col">销售额</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1月</td>
      <td>¥100,000</td>
    </tr>
  </tbody>
  <tfoot>
    <tr>
      <th scope="row">合计</th>
      <td>¥1,200,000</td>
    </tr>
  </tfoot>
</table>
```

## 表单

### 可访问的表单

```html
<form action="/api/users" method="POST">
  <!-- ✅ label 关联 -->
  <div>
    <label for="name">姓名</label>
    <input type="text" id="name" name="name" required
           minlength="2" maxlength="50"
           autocomplete="name">
  </div>

  <!-- ✅ 分组 -->
  <fieldset>
    <legend>联系方式</legend>
    <div>
      <label for="email">邮箱</label>
      <input type="email" id="email" name="email" required
             autocomplete="email">
    </div>
    <div>
      <label for="phone">电话</label>
      <input type="tel" id="phone" name="phone"
             pattern="[0-9]{11}"
             autocomplete="tel">
    </div>
  </fieldset>

  <!-- ✅ 正确的 input type -->
  <div>
    <label for="birthday">出生日期</label>
    <input type="date" id="birthday" name="birthday">
  </div>

  <button type="submit">提交</button>
</form>
```

## 可访问性（a11y）

### ARIA 属性

```html
<!-- ✅ 图片必须有 alt -->
<img src="logo.png" alt="公司 Logo">

<!-- ✅ 装饰性图片用空 alt -->
<img src="decoration.png" alt="" role="presentation">

<!-- ✅ 动态内容 -->
<div role="alert" aria-live="polite">
  表单提交成功
</div>

<!-- ✅ 按钮有可访问名称 -->
<button aria-label="关闭对话框">
  <svg aria-hidden="true">...</svg>
</button>

<!-- ✅ 当前页面高亮 -->
<a href="/" aria-current="page">首页</a>

<!-- ✅ 展开/折叠状态 -->
<button aria-expanded="false" aria-controls="details">
  展开详情
</button>
<div id="details" hidden>
  详情内容
</div>
```

### 键盘导航

```html
<!-- ✅ 使用原生可聚焦元素 -->
<button>点击</button>          <!-- 原生支持键盘 -->
<a href="/page">链接</a>       <!-- 原生支持键盘 -->

<!-- 自定义可聚焦元素需添加 tabindex -->
<div tabindex="0" role="button"
     onkeydown="if(event.key==='Enter') handleClick()">
  自定义按钮
</div>
```

## 性能

### 资源加载优化

```html
<!-- ✅ CSS 放 head，阻塞渲染 -->
<head>
  <link rel="stylesheet" href="critical.css">
</head>

<!-- ✅ JS 放 body 底部，或使用 defer/async -->
<script src="app.js" defer></script>        <!-- DOM 就绪后执行 -->
<script src="analytics.js" async></script>  <!-- 异步加载执行 -->

<!-- ✅ 预加载关键资源 -->
<link rel="preload" href="font.woff2" as="font" type="font/woff2" crossorigin>
<link rel="preload" href="hero.jpg" as="image">

<!-- ✅ 预连接外部域 -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="dns-prefetch" href="https://analytics.example.com">

<!-- ✅ 图片优化 -->
<img src="photo.jpg"
     alt="描述文字"
     loading="lazy"
     decoding="async"
     width="800"
     height="600">

<!-- ✅ 响应式图片 -->
<picture>
  <source srcset="hero.webp" type="image/webp">
  <source srcset="hero.jpg" type="image/jpeg">
  <img src="hero.jpg" alt="描述文字" width="800" height="600">
</picture>

<!-- ✅ 不同尺寸 -->
<img srcset="small.jpg 480w, medium.jpg 800w, large.jpg 1200w"
     sizes="(max-width: 600px) 480px, (max-width: 1000px) 800px, 1200px"
     src="medium.jpg"
     alt="描述文字">
```

### 减少 DOM 嵌套

```html
<!-- ❌ Bad - 过度嵌套 -->
<div class="wrapper">
  <div class="container">
    <div class="row">
      <div class="col">
        <div class="card">
          <p>内容</p>
        </div>
      </div>
    </div>
  </div>
</div>

<!-- ✅ Good - 精简结构 -->
<div class="card">
  <p>内容</p>
</div>
```

## HTML5 新特性

```html
<!-- ✅ details/summary 无 JS 折叠 -->
<details>
  <summary>常见问题</summary>
  <p>这里是答案内容</p>
</details>

<!-- ✅ dialog 原生对话框 -->
<dialog id="modal">
  <form method="dialog">
    <p>确认操作？</p>
    <button value="confirm">确认</button>
    <button value="cancel">取消</button>
  </form>
</dialog>

<!-- ✅ output 元素 -->
<form oninput="result.value = a.valueAsNumber + b.valueAsNumber">
  <input type="number" id="a" name="a"> +
  <input type="number" id="b" name="b"> =
  <output name="result" for="a b">0</output>
</form>

<!-- ✅ template 元素 -->
<template id="card-template">
  <div class="card">
    <slot name="title"></slot>
    <slot></slot>
  </div>
</template>
```

## 参考资源

- [MDN HTML 教程](https://developer.mozilla.org/zh-CN/docs/Web/HTML)
- [HTML Living Standard](https://html.spec.whatwg.org/)
- [Web.dev - Learn HTML](https://web.dev/learn/html/)
- [W3C WAI-ARIA](https://www.w3.org/WAI/ARIA/apg/)
