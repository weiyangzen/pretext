# Research: pages/demos/bubbles.html

## 场景与职责

`bubbles.html` 是 Pretext “Shrinkwrap Showdown” 演示的完整页面。它通过**左右双栏实时对比**，直观展示 CSS `width: fit-content` 与 Pretext shrinkwrap 算法在多行聊天气泡上的差异。左侧为浏览器原生行为（存在右侧留白），右侧为 Pretext 计算的最优宽度（零浪费像素）。页面包含内联启动脚本（消除首屏闪烁）和外部模块脚本（`bubbles.ts`）两部分。

## 功能点目的

1. **首屏无闪烁初始化**：`<head>` 中的同步脚本在 DOM 构建前计算初始聊天宽度与气泡最大宽度，直接写入 CSS 变量与 `dataset`，避免模块加载完成前的布局跳动。  
2. **双栏对比**：左侧 `#chat-css` 使用 CSS `width: fit-content` + `max-width: 80%`；右侧 `#chat-shrink` 的宽度由 Pretext 在运行时逐条计算并赋值。  
3. **交互式宽度调节**：提供 range slider，允许用户在 220px–760px 之间动态改变聊天容器宽度，实时观察两种布局的响应差异。  
4. **自包含的降级体验**：即使 `bubbles.ts` 模块加载失败，页面仍保留完整的 HTML 结构与初始内联脚本设置的气泡宽度，用户至少能看到左侧 CSS 版本。

## 具体技术实现

### 同步初始化脚本（`<head>` 内）

```js
(function syncInitialBubbleGeometry() {
  const viewportWidth = root.clientWidth
  const pageWidth = Math.min(pageMaxWidth, viewportWidth - (viewportWidth <= 760 ? mobilePageMargin : desktopPageMargin))
  const columnWidth = viewportWidth <= 760 ? pageWidth : (pageWidth - gridGap) / 2
  const panelContentWidth = Math.max(1, Math.floor(columnWidth - panelPaddingX))
  const maxWidth = Math.max(minWidth, panelContentWidth)
  const chatWidth = Math.min(requestedWidth, maxWidth)
  const bubbleMaxWidth = Math.floor(chatWidth * 0.8)

  root.dataset['bubblesChatWidth'] = String(chatWidth)
  root.dataset['bubblesMaxWidth'] = String(maxWidth)
  root.style.setProperty('--chat-width', `${chatWidth}px`)
  root.style.setProperty('--bubble-max-width', `${bubbleMaxWidth}px`)
})()
```

该脚本与 `bubbles-shared.ts` 中的 `getMaxChatWidth()` 逻辑**硬编码一致**，确保在模块解析前 CSS 变量已就绪。

### 内联启动脚本（`<body>` 底部）

#### 1. `applyInitialBubbleControls()`
读取 `dataset` 中的 `chatWidth` 与 `maxWidth`，初始化 slider 的 `max` 和 `value`，并更新标签 `#val`。

#### 2. `applyInitialShrinkwrapRender()`
这是页面最复杂的内联逻辑，负责在 `bubbles.ts` 接管前，用**原生 DOM 测量**为右侧 shrinkwrap 气泡设定一个合理的初始宽度：
- 遍历 `#chat-css` 与 `#chat-shrink` 中对应的 `.msg` 节点对。
- 将左侧 CSS 气泡设为 `width: fit-content`，读取其 `getBoundingClientRect()` 得到实际宽度和通过高度换算出的行数。
- 对右侧气泡执行**原生二分搜索**：不断缩小宽度并读取高度，直到行数即将增加，找到 tight width。
- 该内联脚本仅执行一次，作为模块加载前的“视觉占位”；真正的无 DOM 测量重算由 `bubbles.ts` 在 `requestAnimationFrame` 中完成。

### CSS 架构

- `:root` 定义了页面配色与两个动态变量：
  - `--chat-width: 340px`
  - `--bubble-max-width: 272px`
- `.chat` 宽度绑定 `--chat-width`。
- `.msg` 的 `max-width` 绑定 `--bubble-max-width`；左侧额外设置 `width: fit-content`。
- 响应式断点 `@media (max-width: 760px)` 将双列 `.grid` 改为单列。

### DOM 结构要点

- 7 条聊天消息硬编码在 HTML 中，内容涵盖英文、韩文、阿拉伯文、emoji，用于测试 Pretext 的国际化折行。
- 左右两侧消息文本**完全一致**，保证对比的公平性。

## 关键代码路径与文件引用

- **同目录脚本**：
  - `bubbles.ts` — 模块级逻辑，监听 slider / resize / fonts.ready，调用 `bubbles-shared.ts` 的函数重新计算并更新右侧气泡宽度。
  - `bubbles-shared.ts` — 提供 `computeBubbleRender`、`getMaxChatWidth`、`prepareBubbleTexts` 等纯函数。
- **上游库**：
  - `../../src/layout.ts` — 被 `bubbles-shared.ts` 与 `bubbles.ts` 引入。
- **预加载提示**：
  - `<link rel="modulepreload" href="./bubbles-shared.ts">`
  - `<link rel="modulepreload" href="../../src/layout.ts">`

## 依赖与外部交互

- **浏览器 API**：`document.documentElement.clientWidth`、`getBoundingClientRect()`、CSS 自定义属性（Custom Properties）。
- **与 TS 的隐式契约**：
  - HTML 中 `.msg` 的数量、顺序、文本内容必须与 `bubbles.ts` 中通过 `readNodeText()` 读取的节点数组一致。
  - 布局常量（`pageMaxWidth`、`gridGap` 等）必须与 `bubbles-shared.ts` 保持同步。

## 风险、边界与改进建议

1. **重复逻辑维护成本**：同步初始化脚本、内联 shrinkwrap 脚本、`bubbles-shared.ts` 三处均包含宽度计算或二分搜索逻辑。任何布局参数变更需要改三处，极易遗漏。建议：
   - 将同步块精简为仅设置一个保守默认值（如 `--chat-width: 340px`），把精确计算完全交给 `bubbles.ts`；或
   - 使用一个极小的内联 `type="module"` 脚本替代 IIFE，直接 import 共享常量（但会阻塞首屏）。  
2. **内联脚本的 DOM 测量**：`applyInitialShrinkwrapRender()` 在模块加载前进行了大量 `getBoundingClientRect()` 和二分搜索，虽然只跑一次，但在低端设备上仍可能触发多次重排。考虑到这是演示页面，可接受；若作为生产代码则应移除。  
3. **Slider 最大值硬编码**：`<input max="760">` 与 `bubbles-shared.ts` 中未显式关联。若设计规范变更，需同步修改。  
4. **无障碍改进空间**：slider 仅有 `<span>Container width:</span>` 标签，缺少 `<label for="slider">`。右侧 `.metric` 中的“Wasted pixels”数字变化对屏幕阅读器不够友好，建议增加 `aria-live="polite"` 区域。  
5. **暗色/浅色模式**：页面当前为固定浅色主题，无切换逻辑。若后续增加，需确保 `.msg.sent` / `.msg.recv` 的颜色对比度仍符合 WCAG 标准。
