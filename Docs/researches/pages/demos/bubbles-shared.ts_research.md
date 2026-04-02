# Research: pages/demos/bubbles-shared.ts

## 场景与职责

`bubbles-shared.ts` 是 Bubbles（Shrinkwrap）演示的共享逻辑层，被 `bubbles.ts` 和潜在的测试/对比页面复用。它封装了“聊天消息气泡紧包裹（shrinkwrap）”算法的核心几何计算：给定一组预处理好的文本，在指定聊天容器宽度下，计算每个气泡的**最窄宽度**（tight width），使得折行数与 CSS `width: fit-content` 完全一致，但消除右侧留白。

## 功能点目的

1. **提取可复用的 shrinkwrap 算法**：将气泡宽度计算、聊天容器宽度解析、浪费像素统计抽象为纯函数，便于 UI 层调用。  
2. **对比 CSS `fit-content` 与 Pretext 最优宽度**：通过 `collectWrapMetrics` + `findTightWrapMetrics` 的二分搜索，找到保持相同行数的最小宽度。  
3. **量化收益**：计算并汇总所有气泡因使用 tight width 而节省的像素总面积（`totalWastedPixels`）。

## 具体技术实现

### 类型与常量

```ts
export type WrapMetrics = { lineCount: number; height: number; maxLineWidth: number }
export type PreparedBubble = { prepared: PreparedTextWithSegments }
export type BubbleRenderWidths = { cssWidth: number; tightWidth: number }
export type BubbleRenderState = {
  chatWidth: number
  bubbleMaxWidth: number
  totalWastedPixels: number
  widths: BubbleRenderWidths[]
}
```

常量定义了视觉系统的网格参数：
- `FONT = '15px "Helvetica Neue", Helvetica, Arial, sans-serif'`
- `LINE_HEIGHT = 20`
- `PADDING_H = 12`、`PADDING_V = 8`（气泡内部 padding）
- `BUBBLE_MAX_RATIO = 0.8`（气泡最大占聊天容器宽度的比例）
- `PAGE_MAX_WIDTH = 1080` 等响应式断点参数

### 核心函数

#### `prepareBubbleTexts(texts: string[]): PreparedBubble[]`

对每条消息文本调用 `prepareWithSegments(text, FONT)`，返回带有 segment 信息的预处理对象数组。这是**一次性开销**，在页面启动时完成。

#### `getMaxChatWidth(minWidth, viewportWidth): number`

根据视口宽度计算聊天内容区的实际可用宽度：
- 视口 ≤ 760px 时单列，否则双列。
- 扣除页面边距（`MOBILE_PAGE_MARGIN` / `DESKTOP_PAGE_MARGIN`）、网格间隙（`GRID_GAP`）、面板内边距（`PANEL_PADDING_X`）。

#### `collectWrapMetrics(prepared, maxWidth): WrapMetrics`

使用 `walkLineRanges()`（非物化行文本的轻量 API）遍历所有折行，统计：
- `lineCount`：总行数
- `height = lineCount * LINE_HEIGHT`
- `maxLineWidth`：所有行中最宽一行的宽度

#### `findTightWrapMetrics(prepared, maxWidth): WrapMetrics`

**二分搜索**最窄宽度：
- 先以 `maxWidth` 得到初始 `lineCount`（即 CSS `fit-content` 下的行数）。
- 在区间 `[1, maxWidth]` 内二分，找到最小的 `mid` 使得 `layout(prepared, mid, LINE_HEIGHT).lineCount` **不大于**初始行数。
- 最终用该最小宽度重新 `collectWrapMetrics`，得到 tight 状态下的 `maxLineWidth`。

时间复杂度：O(log(maxWidth) × lineWalkCost)。由于 `layout()` 是纯算术遍历，单次调用极快（~0.0002ms），整体可忽略。

#### `computeBubbleRender(preparedBubbles, chatWidth): BubbleRenderState`

对每个气泡执行：
1. `cssMetrics = collectWrapMetrics(bubble.prepared, contentMaxWidth)`
2. `tightMetrics = findTightWrapMetrics(bubble.prepared, contentMaxWidth)`
3. 计算气泡 CSS 宽度与 tight 宽度（均加上左右 padding）：
   ```ts
   cssWidth = Math.ceil(cssMetrics.maxLineWidth) + PADDING_H * 2
   tightWidth = Math.ceil(tightMetrics.maxLineWidth) + PADDING_H * 2
   ```
4. 累加浪费像素：`totalWastedPixels += max(0, cssWidth - tightWidth) * cssHeight`

### 工具函数

- `formatPixelCount(value)`：将数字四舍五入并本地化输出，用于 UI 展示。

## 关键代码路径与文件引用

- **上游 API**：
  - `../../src/layout.ts`：导入 `layout`、`prepareWithSegments`、`walkLineRanges`、`PreparedTextWithSegments`。
    - `walkLineRanges` 是 `layoutWithLines` 的轻量替代，不生成字符串，仅回调每行的几何范围，适合 shrinkwrap 这种只需要宽度的场景。
    - `layout` 用于二分搜索中的快速行数判定。
- **下游消费者**：
  - `bubbles.ts` — 绑定滑块与 resize 事件，调用 `computeBubbleRender` 并更新 DOM。
  - `bubbles.html` — 提供双栏对比 UI（CSS fit-content vs Pretext shrinkwrap）。

## 依赖与外部交互

- **纯函数设计**：除 `prepareWithSegments` 内部使用 canvas / `Intl.Segmenter` 外，本文件所有逻辑不依赖 DOM、window 或浏览器事件。
- **与 HTML 的隐式契约**：`PAGE_MAX_WIDTH`、`DESKTOP_PAGE_MARGIN`、`GRID_GAP`、`PANEL_PADDING_X` 等常量必须与 `bubbles.html` 中 CSS 布局保持同步，否则计算出的 `chatWidth` 会与实际 CSS 渲染宽度不一致。

## 风险、边界与改进建议

1. **常量同步风险**：`bubbles.html` 的 `<script>` 同步块中也硬编码了相同的布局参数（`pageMaxWidth`、`desktopPageMargin` 等）。若修改一方未同步另一方，会导致首屏闪烁或宽度错位。建议将这些参数导出为共享 JSON 配置，或由 `bubbles-shared.ts` 在运行时计算后通过 CSS 变量注入。  
2. **二分搜索边界**：`findTightWrapMetrics` 的搜索上界是 `Math.ceil(maxWidth)`。若某段文本在 `maxWidth` 下仅 1 行，而宽度为 1px 时仍只有 1 行（例如单个短单词），二分将收敛到 1px，tightWidth 可能极窄。这在聊天 UI 中通常由 `min-width` 兜底，但本模块未显式限制。  
3. **性能假设**：`computeBubbleRender` 对每个气泡都做一次二分搜索。若气泡数量从 7 个增加到数千个，总耗时线性增长。对于批量 shrinkwrap，可考虑缓存相同文本的结果，或并行计算（Web Workers）。  
4. **改进建议**：
   - 将 `findTightWrapMetrics` 的搜索结果按 `(text, maxWidth)` 缓存，避免 resize 时重复计算未改变的文本。
   - 增加 `minWidth` 参数传入，防止极端窄气泡。
   - 当前 `totalWastedPixels` 只统计水平浪费 × 高度，未考虑多行气泡底部最后一行的垂直空间浪费（若 CSS 行高大于文本实际高度）。若需更精确对比，可引入 Pretext 的逐行实际高度数据。
