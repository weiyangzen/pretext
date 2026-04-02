# Research: pages/demos/bubbles.ts

## 场景与职责

`bubbles.ts` 是 Bubbles（Shrinkwrap）演示的 UI 驱动层。它在页面加载后接管右侧聊天面板的动态宽度计算，通过 Pretext 的 `prepareWithSegments` + `walkLineRanges` + `layout` 实现**零 DOM 测量**的气泡 shrinkwrap。同时负责绑定 slider 输入、窗口 resize、字体加载完成等事件，并在 `requestAnimationFrame` 中统一更新 DOM。

## 功能点目的

1. **无 DOM 测量的实时 shrinkwrap**：在 slider 拖动或窗口 resize 时，仅通过 Pretext 算术重新计算每个气泡的最优宽度，不读取任何元素的 `offsetWidth` / `getBoundingClientRect`。  
2. **状态与事件批处理**：使用 `scheduledRaf` 将高频输入（slider `input`、resize）合并到单一渲染帧，避免每输入一像素就重算一次。  
3. **与 HTML 内联脚本的接力**：读取 HTML 同步脚本写入的 `dataset['bubblesChatWidth']` 作为初始状态，随后完全由本模块管理。  
4. **量化对比展示**：调用 `computeBubbleRender()` 得到 `totalWastedPixels`，实时更新左侧 CSS 栏的浪费像素计数器。

## 具体技术实现

### 状态模型

```ts
type State = {
  requestedChatWidth: number   // 用户通过 slider 请求的宽度（可能被视口限制截断）
  events: {
    sliderValue: number | null // 本帧待处理的 slider 值
  }
}
```

### DOM 缓存（`domCache`）

在模块顶层直接查询并缓存关键节点：
- `root: document.documentElement`
- `chatShrink: #chat-shrink`
- `slider: #slider`
- `valLabel: #val`
- `cssWaste: #css-waste`
- `shrinkWaste: #shrink-waste`

### 初始化流程

1. 读取 `#chat-shrink` 下所有 `.msg` 节点，提取 `textContent`。
2. 调用 `prepareBubbleTexts(shrinkNodes.map(readNodeText))` 一次性预处理所有消息文本。
3. 从 `dataset` 或 `slider.value` 获取初始 `requestedChatWidth`。
4. 绑定事件：
   - `slider.addEventListener('input')` → 记录 `sliderValue` → `scheduleRender()`
   - `window.addEventListener('resize')` → `scheduleRender()`
   - `document.fonts.ready.then()` → `scheduleRender()`
5. 立即 `scheduleRender()`。

### 渲染帧（`render`）

```ts
function render(): void {
  // 1. 计算实际聊天容器宽度
  const minWidth = parseInt(slider.min, 10)
  let requestedChatWidth = st.requestedChatWidth
  if (st.events.sliderValue !== null) requestedChatWidth = st.events.sliderValue
  const maxWidth = getMaxChatWidth(minWidth, document.documentElement.clientWidth)
  const chatWidth = Math.min(requestedChatWidth, maxWidth)

  // 2. 更新 slider UI
  domCache.slider.max = String(maxWidth)
  domCache.slider.value = String(chatWidth)
  domCache.valLabel.textContent = `${chatWidth}px`

  // 3. 更新气泡宽度与浪费像素
  updateBubbles(chatWidth)
}
```

### 气泡更新（`updateBubbles`）

```ts
function updateBubbles(chatWidth: number): void {
  const renderState = computeBubbleRender(preparedBubbles, chatWidth)
  // 将计算结果写回 CSS 变量
  domCache.root.style.setProperty('--chat-width', `${renderState.chatWidth}px`)
  domCache.root.style.setProperty('--bubble-max-width', `${renderState.bubbleMaxWidth}px`)

  // 仅对 shrinkwrap 侧的气泡逐个设置 style.width = tightWidth
  for (let index = 0; index < shrinkNodes.length; index++) {
    const shrinkNode = shrinkNodes[index]!
    const widths = renderState.widths[index]!
    shrinkNode.style.maxWidth = `${renderState.bubbleMaxWidth}px`
    shrinkNode.style.width = `${widths.tightWidth}px`
  }

  // 更新浪费像素标签
  domCache.cssWaste.textContent = formatPixelCount(renderState.totalWastedPixels)
  domCache.shrinkWaste.textContent = '0'
}
```

注意：左侧 CSS 栏的宽度由浏览器自身根据 `width: fit-content` 渲染，本模块不直接操作左侧气泡；它仅将 Pretext 算出的“如果左侧也这么宽会浪费多少像素”以数字形式展示在左侧 `.metric` 中。

## 关键代码路径与文件引用

- **同目录共享模块**：
  - `bubbles-shared.ts` — 导入 `computeBubbleRender`、`formatPixelCount`、`getMaxChatWidth`、`prepareBubbleTexts`。
- **上游库**：
  - `../../src/layout.ts` — 被 `bubbles-shared.ts` 引入，提供 `prepareWithSegments`、`walkLineRanges`、`layout`。
- **同目录页面**：
  - `bubbles.html` — 提供双栏 DOM 骨架、CSS 变量、初始 dataset。

## 依赖与外部交互

- **浏览器 API**：`document.fonts.ready`、`requestAnimationFrame`、`document.documentElement.clientWidth`、CSS Custom Properties。
- **与 HTML 的契约**：
  - 期望 `#chat-shrink` 中存在与 `preparedBubbles` 数量一致的 `.msg` 节点。
  - 期望 `bubbles.html` 的同步脚本已将 `bubblesChatWidth` 写入 `dataset`。
- **与共享模块的契约**：`getMaxChatWidth` 和 `computeBubbleRender` 中的布局常量必须与 HTML/CSS 保持一致。

## 风险、边界与改进建议

1. **DOM 写入频率**：`updateBubbles` 在每次 RAF 中对每个气泡设置 `style.width`。对于 7 条消息无感，但对于数百条消息列表，频繁设置内联样式可能触发样式重计算。建议：
   - 使用 CSS 变量为每个气泡分配宽度（如 `--bubble-0-width`），减少内联样式写入；或
   - 仅对可见区域内的气泡执行 shrinkwrap。  
2. **缺少 `min-width` 保护**：`computeBubbleRender` 返回的 `tightWidth` 理论上可能极小（如单字符单词）。虽然 `max-width: bubbleMaxWidth` 会限制上限，但无下限保护，极端情况下气泡可能窄到只剩几个字母宽。建议在 `bubbles-shared.ts` 或此处增加 `min(tightWidth, bubbleMaxWidth)` 的同时再设一个 `absoluteMinWidth`（如 60px）。  
3. **事件监听范围**：`resize` 监听在 `window`，`slider input` 在局部。若页面被嵌入 iframe 且 iframe 尺寸变化由父级控制，`resize` 不会触发，导致 `chatWidth` 未按新的 `maxWidth` 截断。此场景在独立演示中无需考虑。  
4. **字体就绪后的重复计算**：`document.fonts.ready` 只触发一次。若页面后续通过 JS 动态加载新字体（如用户切换主题字体），`preparedBubbles` 不会自动刷新。演示场景下可接受。  
5. **改进建议**：
   - 增加 `ResizeObserver` 监听 `.page` 或 `.grid` 容器，以替代 `window.resize`，更精确地响应布局变化。
   - 在 `updateBubbles` 中对比新旧宽度，若某气泡 `tightWidth` 未变，则跳过 `style.width` 赋值，减少无效 DOM 操作。
   - 将 `shrinkWaste` 硬编码的 `'0'` 改为基于实际测量的验证值（可用于自动化测试断言）。
