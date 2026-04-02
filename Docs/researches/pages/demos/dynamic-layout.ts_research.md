# Research: pages/demos/dynamic-layout.ts

## 场景与职责

`dynamic-layout.ts` 是 Pretext 库最具复杂度的浏览器演示之一。它实现了一个**固定视口高度、双栏杂志级编辑布局**，完全通过 Pretext API 在 JavaScript 中计算每行文本的位置与内容，再使用绝对定位的 `<span>` 注入 DOM。该演示的核心目标是验证并展示 Pretext 在真实产品场景中的能力：

- 自适应标题字号（二进制搜索拟合）
- 多栏连续文本流（cursor handoff）
- 动态障碍物避让（logo 多边形轮廓）
- 实时交互动画（点击 logo 旋转，文本即时重排）
- 零 DOM 测量热路径

## 功能点目的

1. **标题自适应字号拟合**：通过二分搜索找到不折断单词的前提下最大的 `font-size`，使标题在可用宽度内尽可能大。  
2. **两栏正文连续流**：左栏消费文本至底部后，将 `LayoutCursor` 传递给右栏，右栏无缝接续，模拟印刷杂志的分栏排版。  
3. **障碍物感知排版**：从 SVG logo 的 alpha 通道提取轮廓多边形，将其作为障碍物；对每一行文本计算可用的水平 slot，调用 `layoutNextLine` 填充。  
4. **交互式 logo**：点击 OpenAI / Claude logo 会触发旋转动画，旋转后的多边形轮廓即时参与下一帧的文本避让。  
5. **性能优化**：使用对象池（`syncPool`）复用 DOM 节点、文本投影缓存（`committedTextProjection`）避免无变化的 DOM 重写、RAF 调度合并渲染请求。

## 具体技术实现

### 数据模型

```ts
type PositionedLine = { x: number; y: number; width: number; text: string }
type ProjectedBodyLine = PositionedLine & { className: string }
type TextProjection = {
  pageWidth: number; pageHeight: number
  headlineFont: string; headlineLineHeight: number; headlineLines: PositionedLine[]
  creditLeft: number; creditTop: number
  bodyFont: string; bodyLineHeight: number; bodyLines: ProjectedBodyLine[]
}
type PageLayout = {
  isNarrow: boolean; gutter: number; pageWidth: number; pageHeight: number
  centerGap: number; columnWidth: number
  headlineRegion: Rect; headlineFont: string; headlineLineHeight: number
  creditGap: number; copyGap: number
  openaiRect: Rect; claudeRect: Rect
}
type BandObstacle =
  | { kind: 'polygon'; points: Point[]; horizontalPadding: number; verticalPadding: number }
  | { kind: 'rects'; rects: Rect[]; horizontalPadding: number; verticalPadding: number }
```

### 核心流程

#### 1. 初始化（模块顶层）

```ts
const [, openaiLayout, claudeLayout, openaiHit, claudeHit] = await Promise.all([
  document.fonts.ready,
  getWrapHull(OPENAI_LOGO_SRC, { smoothRadius: 6, mode: 'mean' }),
  getWrapHull(CLAUDE_LOGO_SRC, { smoothRadius: 6, mode: 'mean' }),
  getWrapHull(OPENAI_LOGO_SRC, { smoothRadius: 3, mode: 'mean' }),
  getWrapHull(CLAUDE_LOGO_SRC, { smoothRadius: 5, mode: 'mean' }),
])
```

- 等待字体就绪，同时并行解码两个 logo 的 SVG 并提取两套轮廓：
  - `smoothRadius: 6` 用于**文本避让**（layout wrapping），轮廓稍平滑。
  - `smoothRadius: 3/5` 用于**点击命中测试**（hit testing），轮廓更贴近原始形状。
- 预处理正文 `BODY_COPY` 与作者署名 `CREDIT_TEXT`。

#### 2. 布局构建（`buildLayout`）

根据视口宽度分为**窄屏**（`< 760px`）与**宽屏**两套几何参数：
- 计算 `gutter`、`centerGap`、`columnWidth`。
- 计算标题区域 `headlineRegion`、OpenAI logo 矩形 `openaiRect`、Claude logo 矩形 `claudeRect`。
- 窄屏时标题字号上限 48px，宽屏时通过 `fitHeadlineFontSize()` 动态拟合。

#### 3. 标题拟合（`fitHeadlineFontSize`）

```ts
while (low <= high) {
  const size = Math.floor((low + high) / 2)
  const font = `700 ${size}px ${HEADLINE_FONT_FAMILY}`
  const headlinePrepared = getPrepared(HEADLINE_TEXT, font)
  if (!headlineBreaksInsideWord(headlinePrepared, headlineWidth)) {
    best = size
    low = size + 1
  } else {
    hi = size - 1
  }
}
```

- 二分搜索范围 `[ceil(max(22, pageWidth*0.026)), floor(min(94.4, max(55.2, pageWidth*0.055)))]`。
- 判定条件：`walkLineRanges` 遍历标题，检查是否有任何一行的结束点落在某个 segment 的中间（`line.end.graphemeIndex !== 0`），即折行是否打断了一个单词。

#### 4. 栏排版（`layoutColumn`）

这是整个演示最核心的算法：

```ts
function layoutColumn(
  prepared, startCursor, region, lineHeight, obstacles, side
): { lines: PositionedLine[], cursor: LayoutCursor }
```

对每一行（`lineTop` 到 `lineTop + lineHeight`）：
1. 遍历所有 `obstacles`，调用 `getObstacleIntervals` 计算该行垂直 band 被障碍物遮挡的水平区间。
2. 调用 `carveTextLineSlots(baseInterval, blockedIntervals)` 得到剩余可用 slot 列表。
3. 选择最优 slot：宽屏下左栏优先最左侧最宽 slot，右栏优先最右侧最宽 slot（通过 `side` 参数控制）。
4. 调用 `layoutNextLine(prepared, cursor, slotWidth)` 得到一行文本。
5. 将文本放入 slot，更新 `cursor`，继续下一行。

#### 5. 评估布局（`evaluateLayout`）

- 标题栏：使用 `layoutColumn` 在 `headlineRegion` 内排版，障碍物仅 OpenAI logo。
- 将标题的每一行转换为 `RectObstacle`（`titleObstacle`），作为右栏正文的障碍物之一。
- 作者署名：在标题底部下方，计算避开 OpenAI（宽屏）或 OpenAI+Claude（窄屏）的可用 slot，左对齐放置。
- 正文：
  - 窄屏：单栏，障碍物为 Claude + OpenAI。
  - 宽屏：左栏（障碍物 OpenAI）→ 右栏（障碍物 titleObstacle + Claude + OpenAI），使用同一 `preparedBody` 的 cursor 接力。

#### 6. 动画与交互（`render` / `commitFrame`）

- `render(now)` 处理鼠标/点击事件，检测是否点中 logo（使用 `isPointInPolygon` 对旋转后的 hit hull 做射线法测试）。
- 若点中，调用 `startLogoSpin(kind, direction, now)` 设置旋转动画状态。
- `commitFrame(now)` 重新计算整个布局（包括旋转后的 logo 轮廓），生成新的 `TextProjection`。
- `textProjectionEqual()` 对比新旧投影，仅在真正变化时调用 `projectTextProjection()` 更新 DOM。

#### 7. DOM 投影（`projectTextProjection`）

- 标题行：注入到 `.headline` 容器内，使用 `.headline-line` 类。
- 正文行：注入到 `#stage`，使用 `.line` 类。
- 使用 `syncPool()` 复用已有 `<span>`，避免频繁创建/销毁节点。

### 关键工具函数

- `getPrepared(text, font)`：基于 `Map` 的缓存，避免相同文本+字体的重复 `prepareWithSegments`。
- `getPreparedSingleLineWidth()`：用 `walkLineRanges(..., 100_000)` 快速测量单行宽度（用于作者署名）。
- `getObstacleIntervals()`：将多边形或矩形障碍物转换为当前 band 的 blocked 区间。
- `easeSpin(t)`：三次缓动函数，用于 logo 旋转动画。

## 关键代码路径与文件引用

- **上游 Pretext API**：
  - `../../src/layout.ts`：`layoutNextLine`、`prepareWithSegments`、`walkLineRanges`、`LayoutCursor`、`PreparedTextWithSegments`。
  - `../../src/line-break.ts`：被 `layout.ts` 内部调用，执行实际的折行遍历。
- **同目录依赖**：
  - `dynamic-layout-text.ts`：`BODY_COPY` 正文数据源。
  - `wrap-geometry.ts`：`getWrapHull`、`transformWrapPoints`、`isPointInPolygon`、`carveTextLineSlots`、`getPolygonIntervalForBand`、`getRectIntervalsForBand`。
- **静态资源**：
  - `../assets/openai-symbol.svg`
  - `../assets/claude-symbol.svg`
- **宿主页面**：
  - `dynamic-layout.html` — 提供 `#stage`、`.page`、CSS 基础样式。

## 依赖与外部交互

- **浏览器 API**：
  - `document.fonts.ready` — 等待字体加载。
  - `OffscreenCanvas` / `Image.decode()` — 在 `wrap-geometry.ts` 中用于提取 SVG alpha 轮廓。
  - `requestAnimationFrame` — 动画与渲染调度。
  - `getComputedStyle` — 未使用（零 DOM 测量热路径）。
- **Pretext 库**：
  - 依赖 `prepareWithSegments` 的 canvas 测宽与 CJK/emoji/阿拉伯文分词。
  - 依赖 `layoutNextLine` 的逐行 cursor 推进机制，实现跨栏无缝接续。

## 风险、边界与改进建议

1. **首屏等待时间**：模块顶层 `await Promise.all([document.fonts.ready, ...getWrapHull(...)])` 会阻塞脚本执行。在慢网络或大量字体未缓存时，用户可能看到数秒空白。建议：
   - 在等待期间显示一个轻量级骨架屏或加载指示器。
   - 将 `getWrapHull` 的结果序列化为 JSON 内联到 HTML 中，避免每次刷新都重新 rasterize SVG。  
2. **布局计算量随文本长度线性增长**：每帧 `commitFrame` 都会从头遍历所有正文行。虽然 Pretext 单行计算极快，但当文本量极大（如数万词）时，总时间可能超过 16ms。建议：
   - 引入可见区域裁剪（viewport culling），仅计算视口内 ±N 行的布局。
   - 对静态文本预计算并缓存每行的布局结果，仅在障碍物变化时重算受影响区域。  
3. **障碍物避让的精度边界**：`getPolygonIntervalForBand` 对多边形障碍物在整数 Y 坐标上采样，若 logo 旋转角度非 0 且边缘极斜，可能产生 1px 级别的 jitter。当前通过 `Math.round` 和 padding 已能掩盖，但在更高 DPI 或更大缩放比例下可能暴露。  
4. **内存泄漏风险**：`preparedByKey` 是模块级 `Map`，持续缓存所有 `(text, font)` 组合。在标题字号二分搜索过程中会生成大量不同 `font` 的 `PreparedTextWithSegments`，虽然现代浏览器 GC 能处理，但长期运行的 SPA 中应考虑设置 `Map` 大小上限或 LRU 策略。  
5. **窄屏单栏的文本截断**：窄屏时正文仅一栏，若文本长度超过视口高度，超出部分被 `overflow: hidden` 直接截断，无滚动或“继续阅读”提示。作为演示可接受，但非生产就绪。  
6. **改进建议**：
   - 将 `buildLayout` 与 `evaluateLayout` 抽取为独立的“布局引擎”类，使 `dynamic-layout.ts` 专注于事件与动画，提升可测试性。
   - 增加 `IntersectionObserver` 监测 `#stage` 是否进入视口，在不可见时暂停 RAF 动画，节省电池。
   - 对 `wrapHulls` 增加 Web Worker 离线计算分支，避免在主线程 rasterize 大尺寸 SVG。
