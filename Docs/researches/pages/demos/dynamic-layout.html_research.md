# Research: pages/demos/dynamic-layout.html

## 场景与职责

`dynamic-layout.html` 是 Pretext “Dynamic Layout” 演示的静态页面壳层。该页面展示了一个**固定视口高度、纯 JS 驱动排版**的杂志级编辑布局：标题自适应字号、正文在两栏间连续流动、OpenAI 与 Claude 品牌 logo 作为动态障碍物被文本环绕。HTML 文件本身极度精简，仅提供舞台容器 `#stage` 和基础样式，所有复杂布局逻辑均由 `dynamic-layout.ts` 通过绝对定位的 `<span>` 逐行注入完成。

## 功能点目的

1. **提供零 JS 亦可渲染的最小骨架**：包含背景氛围层（`.atmosphere`）、提示 pill（`.hint-pill`）和舞台 `#stage`，即使脚本异常也能保持页面不空白。  
2. **CSS 仅负责氛围与基础层级**：所有文本元素（`.headline-line`、`.line`、`.credit`、`.logo`）的样式定义在 HTML 的 `<style>` 中，但位置、尺寸、旋转角度完全由 JS 在运行时通过 `style` 属性赋值。  
3. **支持响应式断点**：通过 `@media (max-width: 760px)` 隐藏提示 pill，并在 `dynamic-layout.ts` 中读取视口宽度切换窄屏/宽屏布局算法。  
4. **为绝对定位文本渲染预留样式契约**：定义了 `.headline-line`（标题行）、`.line`（正文行）、`.credit`（作者署名）、`.logo`（可旋转 logo）的 `position: absolute`、`white-space: pre`、`user-select: text` 等基础行为。

## 具体技术实现

### DOM 结构

```html
<main class="page">
  <div class="atmosphere atmosphere--left"></div>
  <div class="atmosphere atmosphere--right"></div>
  <p class="hint-pill">Everything laid out in JS. Resize horizontally and vertically, then click the logos.</p>
  <div id="stage" class="stage"></div>
</main>
```

- `#stage` 是所有动态创建元素的挂载点（标题、正文行、logo、作者行）。
- `.atmosphere` 使用 `radial-gradient` 与 `linear-gradient` 营造杂志封面级背景，不依赖任何图片资源。

### CSS 关键规则

- `.page`：
  - `height: 100vh; overflow: hidden;` — 固定高度视口，溢出隐藏，模拟杂志单页。
  - `.page--mobile` 变体使用 `100svh` 适配移动端动态工具栏。
- `.headline-line` / `.line`：
  - `position: absolute; white-space: pre;` — JS 逐行设置 `left` / `top` / `textContent`。
  - `.line:hover { color: var(--accent); }` — 纯 CSS 的悬停效果，无需 JS 参与。
- `.logo`：
  - `transform-origin: center center;` — 为 `dynamic-layout.ts` 中的旋转动画提供正确锚点。
  - 针对 OpenAI / Claude 分别定义了不同的 `drop-shadow`。
- `.hint-pill`：
  - `position: fixed; top: 16px; left: 50%; transform: translateX(-50%);` — 居中固定提示。
  - 在窄屏下 `display: none`。

### 脚本引用

```html
<script type="module" src="./dynamic-layout.ts"></script>
```

## 关键代码路径与文件引用

- **同目录脚本**：
  - `dynamic-layout.ts` — 核心布局引擎，负责 headline 字号拟合、两栏正文流、logo 障碍物计算、动画帧调度。
  - `dynamic-layout-text.ts` — 提供正文长文本 `BODY_COPY`。
  - `wrap-geometry.ts` — 提供 logo 的 alpha 通道轮廓提取与多边形变换工具。
- **上游库**：
  - `../../src/layout.ts` — 被 `dynamic-layout.ts` 引入，提供 `layoutNextLine`、`prepareWithSegments`、`walkLineRanges`。
- **静态资源**：
  - `../assets/openai-symbol.svg`
  - `../assets/claude-symbol.svg`

## 依赖与外部交互

- **无第三方依赖**：纯原生 HTML/CSS，不引用任何外部库或 CDN。
- **与 TS 模块的契约**：
  - `#stage` 必须存在且为 `HTMLDivElement`，否则 `dynamic-layout.ts` 在顶层即抛错。
  - `.page` 类名被 `dynamic-layout.ts` 用于动态切换 `.page--mobile`，修改类名需同步更新 TS 中的选择器。

## 风险、边界与改进建议

1. **绝对定位的可访问性**：所有文本行都是绝对定位的 `<span>`，虽然保留了 `user-select: text`，但屏幕阅读器可能无法正确识别阅读顺序（因为 DOM 顺序不等于视觉顺序）。建议：
   - 为 `#stage` 增加 `role="region" aria-label="Editorial layout"`。
   - 在 DOM 中按视觉阅读顺序（左上 → 左下 → 右上 → 右下）插入行元素，当前 `dynamic-layout.ts` 已按左栏后右栏的顺序生成，基本符合。  
2. **固定高度的溢出截断**：`overflow: hidden` 意味着若文本过长，超出两栏底部的内容将被直接截断，无滚动提示。作为演示页面这是设计选择，但在真实编辑产品中需增加“继续阅读”指示或允许滚动。  
3. **字体加载前的空白**：页面等待 `document.fonts.ready` 后才进行首次布局（见 `dynamic-layout.ts`），因此在慢网络下首屏可能短暂空白。可考虑在 HTML 中增加一个极轻量的骨架占位（如模糊色块），提升感知性能。  
4. **暗色/浅色模式缺失**：当前仅有一套暖色调主题。若需支持主题切换，可将 CSS 变量化进一步扩展（如 `--paper`、`--ink` 已具备，只需增加 `data-theme` 选择器）。  
5. **改进建议**：
   - 增加 `<link rel="preload" href="../assets/openai-symbol.svg" as="image">` 与 Claude logo 的预加载，减少 `wrap-geometry.ts` 中 `getWrapHull` 的等待时间。
   - 在 `<head>` 中增加 `<meta name="theme-color" content="#f6f0e6">`，使移动端浏览器地址栏颜色与页面背景一致。
