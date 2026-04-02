# Research: pages/demos/accordion.html

## 场景与职责

`accordion.html` 是 Pretext 库“Accordion”演示的静态页面壳层。它提供一个语义化、可访问的折叠面板（accordion）DOM 骨架，包含 4 个可折叠条目。页面本身不包含业务逻辑，所有动态行为（高度计算、展开/收起动画、事件绑定）均由同目录下的 `accordion.ts` 驱动。该页面用于展示 Pretext 如何在**不读取 DOM 高度**的情况下，通过纯 JavaScript 算术预先计算每个面板的展开高度。

## 功能点目的

1. **提供无脚本也可渲染的静态骨架**：4 组 `.accordion-item` 结构在 HTML 中硬编码，保证首屏即有可交互元素。  
2. **CSS 驱动的过渡动画**：`.accordion-body` 使用 `height` + `transition` 实现平滑展开/收起，避免 JavaScript 动画帧开销。  
3. **响应式与无障碍**：包含 `aria-expanded`、`aria-label`、`:hover` 状态，以及针对 `prefers-reduced-motion` 的媒体查询。  
4. **为 TS 模块预留挂载点**：每个条目内的 `.accordion-title`、`.accordion-meta`、`.accordion-copy` 均为空节点，由 `accordion.ts` 在 `boot()` 阶段填充文本并计算高度。

## 具体技术实现

### DOM 结构

- 外层：`<main class="page">` 限制最大宽度 `min(780px, 100vw - 32px)`。
- 列表：`<section id="list" class="stack">` 包含 4 个 `<article class="accordion-item">`。
- 每个条目内部：
  - `<button class="accordion-toggle" aria-expanded="false">`：可点击标题区。
    - `.accordion-heading > .accordion-title`：标题文本容器。
    - `.accordion-meta`：用于展示“Measurement: N lines · H px”。
    - `.accordion-glyph`：旋转箭头（CSS `::before` 绘制三角形）。
  - `.accordion-body > .accordion-inner > .accordion-copy`：正文段落容器。

### CSS 关键细节

- `.accordion-body` 默认 `height: 0; overflow: clip;`，展开时由 JS 写入具体像素高度。
- `.accordion-glyph` 的旋转通过 `transform: rotate(90deg)` 实现，与 body 高度过渡同用 `180ms ease`。
- 移动端断点 `@media (max-width: 640px)` 收紧内边距与间距。

### 脚本引用

```html
<script type="module" src="./accordion.ts"></script>
```

## 关键代码路径与文件引用

- 同目录：
  - `accordion.ts` — 负责填充文本、调用 Pretext API 计算高度、绑定点击事件。
- 上游库：
  - `../../src/layout.ts` — 被 `accordion.ts` 引入，提供 `prepare()` / `layout()`。

## 依赖与外部交互

- **无直接外部依赖**：纯 HTML/CSS 文件，不引用任何第三方库或 CDN。
- **与 TS 模块的契约**：
  - HTML 中 `.accordion-item` 的数量（4 个）必须与 `accordion.ts` 中 `items` 数组长度一致，否则 `getAccordionItemNodes()` 会抛出 `accordion item count mismatch`。
  - `.accordion-copy` 的 `font` 与 `line-height` 通过 `getComputedStyle()` 被 `accordion.ts` 读取，用于构造 Pretext 的 `font` 字符串。

## 风险、边界与改进建议

1. **数量硬编码风险**：HTML 中条目数量与 TS 中数据数组长度耦合。若只改一方会在运行时抛错。建议未来将条目模板化（如用 `<template>` + JS 克隆），但当前作为演示代码，显式硬编码更易阅读。  
2. **首屏无内容风险**：`.accordion-title` 和 `.accordion-copy` 初始为空，若 `accordion.ts` 加载失败或执行异常，用户将看到空白折叠面板。考虑到这是本地演示页面，风险可接受。  
3. **CSS 变量未使用**：颜色通过 `--page`、`--panel` 等 CSS 变量管理，但无暗色模式切换逻辑。若后续增加主题切换，只需在 `:root` 外再包一层 `data-theme` 选择器即可。  
4. **改进建议**：可在 `<head>` 中增加 `<link rel="modulepreload" href="./accordion.ts">` 以加速模块解析，减少首屏内容填充的延迟。
