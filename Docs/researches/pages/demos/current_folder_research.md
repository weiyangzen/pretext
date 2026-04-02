# pages/demos 目录深度研究文档

## 1. 场景与职责

`pages/demos/` 是 Pretext 库的**交互式演示目录**，承担以下核心职责：

### 1.1 功能定位
- **API 展示平台**：通过可视化 demo 展示 Pretext 核心 API (`prepare`, `layout`, `prepareWithSegments`, `layoutWithLines`, `walkLineRanges`, `layoutNextLine`) 的实际应用场景
- **性能验证工具**：证明无 DOM 测量文本布局的性能优势（~0.05ms vs 30ms DOM 测量）
- **GitHub Pages 站点根**：`index.html` 作为项目演示站点入口（chenglou.me/pretext）
- **开发调试辅助**：为开发者提供实时交互式测试环境

### 1.2 目标受众
- **终端用户**：了解 Pretext 能解决什么实际问题
- **潜在采用者**：评估库的功能完整性和性能表现
- **贡献者**：理解 API 设计意图和最佳实践

---

## 2. 功能点目的

### 2.1 Demo 清单与核心展示点

| Demo | 文件名 | 核心展示功能 | 技术亮点 |
|------|--------|-------------|----------|
| **Accordion** | `accordion.ts/html` | 手风琴组件高度计算 | `prepare` + `layout` 基础 API，展示无 DOM 测量的展开/收起动画 |
| **Bubbles** | `bubbles.ts/html` + `bubbles-shared.ts` | 消息气泡 shrinkwrap 布局 | `walkLineRanges` 二分搜索最紧凑宽度，对比 CSS `fit-content` 的浪费空间 |
| **Dynamic Layout** | `dynamic-layout.ts/html` + `dynamic-layout-text.ts` + `wrap-geometry.ts` | 复杂编辑排版（双栏+障碍物绕行） | `layoutNextLine` 流式布局，SVG logo 轮廓提取与旋转，响应式多栏 |
| **Variable Typographic ASCII** | `variable-typographic-ascii.ts/html` | 粒子驱动的 ASCII 艺术 | `prepareWithSegments` 精确字符宽度测量，亮度场驱动的字符选择 |
| **Editorial Engine** | `editorial-engine.ts/html` | 高级编辑排版（多栏+浮动引用+首字下沉） | `layoutNextLine` 多栏连续流，动态标题字号适配，可拖拽障碍物 |
| **Justification Comparison** | `justification-comparison.ts/html` + `.model.ts` + `.ui.ts` + `.data.ts` | 文本对齐算法对比 | Knuth-Plass 最优断行 vs 贪心算法，河流检测可视化 |
| **Rich Note** | `rich-note.ts/html` | 富文本内联布局（文本+代码+链接+标签） | 混合内容类型的行内布局，`layoutNextLine` 跨片段断行 |
| **Masonry** | `masonry/index.ts/html` + `shower-thoughts.json` | 瀑布流虚拟滚动 | `prepare` + `layout` 批量高度计算，DOM 节点池化与视口裁剪 |

### 2.2 各 Demo 详细分析

#### 2.2.1 Accordion (`accordion.ts`)
```typescript
// 核心 API 使用
const prepared = prepare(item.text, font)  // 一次性准备
const metrics = layout(preparedCache.items[index]!, contentWidth, lineHeight)  // 热路径纯算术计算
```
- **目的**：展示最基础的 `prepare` + `layout` 使用模式
- **交互**：点击展开/收起，实时计算面板高度
- **性能点**：resize 时只调用 `layout`，不重新 `prepare`

#### 2.2.2 Bubbles (`bubbles.ts`, `bubbles-shared.ts`)
```typescript
// Shrinkwrap 核心算法
export function findTightWrapMetrics(prepared: PreparedTextWithSegments, maxWidth: number): WrapMetrics {
  const initial = collectWrapMetrics(prepared, maxWidth)
  let lo = 1, hi = Math.max(1, Math.ceil(maxWidth))
  while (lo < hi) {
    const mid = Math.floor((lo + hi) / 2)
    const midLineCount = layout(prepared, mid, LINE_HEIGHT).lineCount
    if (midLineCount <= initial.lineCount) hi = mid
    else lo = mid + 1
  }
  return collectWrapMetrics(prepared, lo)
}
```
- **目的**：展示 CSS 无法实现的功能——多行文本的最紧凑宽度
- **对比**：左侧 CSS `fit-content`（按最长行）vs 右侧 Pretext shrinkwrap（保持行数的最小宽度）
- **关键 API**：`walkLineRanges` 收集行几何信息

#### 2.2.3 Dynamic Layout (`dynamic-layout.ts`, `wrap-geometry.ts`)
```typescript
// 核心布局循环
function layoutColumn(
  prepared: PreparedTextWithSegments,
  startCursor: LayoutCursor,
  region: Rect,
  lineHeight: number,
  obstacles: BandObstacle[],
  side: 'left' | 'right',
): { lines: PositionedLine[], cursor: LayoutCursor } {
  // 对每一行：1) 计算障碍物阻挡区间 2)  carve 可用文本槽 3) layoutNextLine
  const slots = carveTextLineSlots({ left: region.x, right: region.x + region.width }, blocked)
  const line = layoutNextLine(prepared, cursor, width)
}
```
- **目的**：展示复杂编辑排版——双栏连续流、障碍物绕行、响应式布局
- **技术点**：
  - SVG logo 光栅化提取轮廓 (`getWrapHull` in `wrap-geometry.ts`)
  - 旋转后的多边形碰撞检测 (`isPointInPolygon`)
  - 标题字号二分搜索适配 (`fitHeadlineFontSize`)
  - 窄屏/宽屏双模式布局

#### 2.2.4 Variable Typographic ASCII (`variable-typographic-ascii.ts`)
```typescript
// 字符宽度测量用于亮度匹配
const width = measureWidth(ch, font)  // 使用 prepareWithSegments
const brightness = estimateBrightness(ch, font)  // Canvas 像素分析
const match = findBest(targetBrightness)  // 亮度+宽度双目标优化
```
- **目的**：展示精确字符宽度测量的创意应用
- **技术点**：
  - 粒子系统驱动的亮度场
  - Georgia 3 字重 × 正/斜体 变体 vs 等宽字体对比
  - 亮度-宽度双目标字符选择

#### 2.2.5 Editorial Engine (`editorial-engine.ts`)
```typescript
// 多栏连续流
const leftResult = layoutColumn(preparedBody, startCursor, leftRegion, ...)
const rightResult = layoutColumn(preparedBody, leftResult.cursor, rightRegion, ...)

// 可拖拽障碍物
const circleObstacles: CircleObstacle[] = orbs.map(orb => ({
  cx: orb.x, cy: orb.y, r: orb.r * orbRadiusScale,
  hPad: isNarrow ? 10 : 14, vPad: isNarrow ? 2 : 4,
}))
```
- **目的**：展示生产级编辑排版功能
- **功能**：
  - 3 栏/2 栏/1 栏响应式切换
  - 浮动引用块 (pull quotes)
  - 首字下沉 (drop cap)
  - 发光球体障碍物（可拖拽、可暂停动画）
  - 60fps 实时重排

#### 2.2.6 Justification Comparison (`justification-comparison.*`)
```typescript
// Knuth-Plass 最优断行算法
const dp: number[] = new Array(candidateCount).fill(Infinity)
for (let toCandidate = 1; toCandidate < candidateCount; toCandidate++) {
  for (let fromCandidate = toCandidate - 1; fromCandidate >= 0; fromCandidate--) {
    const lineStats = getLineStatsFromBreakCandidates(...)
    const totalBadness = dp[fromCandidate]! + lineBadness(lineStats, maxWidth, ...)
    if (totalBadness < dp[toCandidate]!) {
      dp[toCandidate] = totalBadness
      previous[toCandidate] = fromCandidate
    }
  }
}
```
- **目的**：展示 Pretext 支持复杂排版算法
- **对比维度**：
  - CSS/贪心对齐：浏览器默认行为
  - Pretext+连字符：贪心+软连字符
  - Knuth-Plass：全局最优断行（TeX 算法）
- **可视化**：红色叠加层显示"河流"（过宽词间距）

#### 2.2.7 Rich Note (`rich-note.ts`)
```typescript
// 混合内容类型布局
type InlineItem = TextInlineItem | ChipInlineItem

function layoutInlineItems(items: InlineItem[], maxWidth: number): RichLine[] {
  // 文本片段：使用 layoutNextLine 断行
  // 标签片段：原子单元，整颗换行
}
```
- **目的**：展示富文本内联布局（类似 Slack/Notion 消息）
- **内容类型**：普通文本、代码片段、链接、彩色标签（mention/status/priority/time/count）
- **约束**：标签保持完整，文本自然换行

#### 2.2.8 Masonry (`masonry/index.ts`)
```typescript
// 瀑布流 + 虚拟滚动
const { height } = layout(st.cards[i]!.prepared, textWidth, lineHeight)  // 预计算高度
const positionedCards = computeMasonryLayout(st.cards, colCount, colWidth)  // 最短列优先

// 视口裁剪
const visibleFlags = new Uint8Array(st.cards.length)
for (const card of positionedCards) {
  if (card.y > viewBottom || card.y + card.h < viewTop) continue
  visibleFlags[card.cardIndex] = 1
  // 渲染可见卡片
}
```
- **目的**：展示批量高度计算 + 虚拟滚动
- **数据**：100 条 Reddit Shower Thoughts
- **性能**：DOM 节点池化，仅渲染视口内卡片

---

## 3. 具体技术实现

### 3.1 核心 API 使用模式

#### 模式 1：基础高度计算（Accordion, Masonry）
```typescript
// 初始化
const prepared = prepare(text, font)

// 热路径（resize 时）
const { height, lineCount } = layout(prepared, maxWidth, lineHeight)
```

#### 模式 2：行几何信息收集（Bubbles）
```typescript
const prepared = prepareWithSegments(text, font)
let maxLineWidth = 0
const lineCount = walkLineRanges(prepared, maxWidth, line => {
  if (line.width > maxLineWidth) maxLineWidth = line.width
})
```

#### 模式 3：流式逐行布局（Dynamic Layout, Editorial Engine）
```typescript
const prepared = prepareWithSegments(text, font)
let cursor: LayoutCursor = { segmentIndex: 0, graphemeIndex: 0 }
while (true) {
  const line = layoutNextLine(prepared, cursor, availableWidth)
  if (line === null) break
  // 使用 line.text, line.width, line.start, line.end
  cursor = line.end
}
```

#### 模式 4：完整行信息获取（Justification Comparison）
```typescript
const prepared = prepareWithSegments(text, font)
const { lines, height, lineCount } = layoutWithLines(prepared, maxWidth, lineHeight)
// lines: Array<{ text: string, width: number, start: LayoutCursor, end: LayoutCursor }>
```

### 3.2 共享工具模块

#### `bubbles-shared.ts`
```typescript
export type PreparedBubble = { prepared: PreparedTextWithSegments }
export type BubbleRenderState = {
  chatWidth: number
  bubbleMaxWidth: number
  totalWastedPixels: number
  widths: BubbleRenderWidths[]
}
export function computeBubbleRender(preparedBubbles: PreparedBubble[], chatWidth: number): BubbleRenderState
```
- 被 `bubbles.ts` 使用
- 封装 shrinkwrap 算法逻辑

#### `wrap-geometry.ts`
```typescript
export type Point = { x: number; y: number }
export type Rect = { x: number; y: number; width: number; height: number }
export type Interval = { left: number; right: number }

// SVG 图像轮廓提取
export function getWrapHull(src: string, options: WrapHullOptions): Promise<Point[]>

// 多边形变换与碰撞检测
export function transformWrapPoints(points: Point[], rect: Rect, angle: number): Point[]
export function isPointInPolygon(points: Point[], x: number, y: number): boolean

// 文本行槽计算（障碍物绕行核心）
export function carveTextLineSlots(base: Interval, blocked: Interval[]): Interval[]
export function getPolygonIntervalForBand(...): Interval | null
```
- 被 `dynamic-layout.ts` 使用
- 提供复杂排版所需的几何工具

#### `dynamic-layout-text.ts`
- 仅包含 `BODY_COPY` 常量（Leopold Aschenbrenner 文章摘录）
- 为 Dynamic Layout demo 提供长文本内容

#### `justification-comparison.data.ts`
- 包含 4 段关于排版研究的英文段落
- 连字符词典（HYPHEN_EXCEPTIONS）：130+ 单词的手动分音节数据
- 前缀/后缀列表用于规则-based 连字符

#### `masonry/shower-thoughts.json`
- 100 条 Reddit r/ShowerThoughts 帖子
- 用于瀑布流虚拟滚动演示

### 3.3 渲染架构模式

所有 demos 遵循统一的渲染架构：

```typescript
// 1. 状态管理
type State = {
  // 当前控制状态
  events: { /* 待处理的输入事件 */ }
}

// 2. DOM 缓存
const domCache = {
  root: document.documentElement,
  // 其他持久 DOM 引用
}

// 3. RAF 调度
let scheduledRaf: number | null = null
function scheduleRender(): void {
  if (scheduledRaf !== null) return
  scheduledRaf = requestAnimationFrame(function renderFrame(now) {
    scheduledRaf = null
    render(now)
  })
}

// 4. 事件监听触发重绘
window.addEventListener('resize', scheduleRender)
domCache.slider.addEventListener('input', () => {
  st.events.sliderValue = Number.parseInt(domCache.slider.value, 10)
  scheduleRender()
})

// 5. 渲染函数结构
function render(now: number): boolean {
  // DOM reads
  const viewportWidth = document.documentElement.clientWidth
  
  // Layout computation（纯 JS，无 DOM 操作）
  const layoutState = computeLayout(viewportWidth)
  
  // DOM writes
  applyLayoutToDom(layoutState)
  
  return isAnimating  // 返回是否需要下一帧
}
```

### 3.4 响应式设计模式

```typescript
const NARROW_BREAKPOINT = 760
const isNarrow = pageWidth < NARROW_BREAKPOINT

if (isNarrow) {
  // 移动端：单栏、缩小间距、减少障碍物
} else {
  // 桌面端：多栏、完整功能
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 API 导入路径
```typescript
// 所有 demos 统一从 src/layout.ts 导入
import {
  prepare,
  prepareWithSegments,
  layout,
  layoutWithLines,
  walkLineRanges,
  layoutNextLine,
  type PreparedText,
  type PreparedTextWithSegments,
  type LayoutCursor,
  type LayoutLine,
} from '../../src/layout.ts'
```

### 4.2 核心调用链

```
accordion.ts
  └── prepare() → layout()
      └── src/layout.ts:prepareInternal()
          ├── src/analysis.ts:analyzeText()      [文本分析]
          └── src/measurement.ts:getSegmentMetrics()  [Canvas 测量]

bubbles.ts
  └── bubbles-shared.ts:computeBubbleRender()
      ├── walkLineRanges()                      [收集行宽]
      └── findTightWrapMetrics()                [二分搜索]
          └── layout()                          [行数计算]

dynamic-layout.ts
  └── layoutColumn()
      ├── wrap-geometry.ts:getPolygonIntervalForBand()  [障碍物计算]
      ├── wrap-geometry.ts:carveTextLineSlots()         [可用槽计算]
      └── layoutNextLine()                              [单行布局]

editorial-engine.ts
  └── layoutColumn()
      ├── fitHeadline()                         [字号二分搜索]
      ├── layoutNextLine()                      [正文流式布局]
      └── walkLineRanges()                      [引用块布局]
```

### 4.3 文件依赖图

```
pages/demos/
├── index.html                          [独立，导航入口]
├── accordion.html/ts                   [依赖: src/layout.ts]
├── bubbles.html/ts                     [依赖: bubbles-shared.ts → src/layout.ts]
├── bubbles-shared.ts                   [依赖: src/layout.ts]
├── dynamic-layout.html/ts              [依赖: wrap-geometry.ts, dynamic-layout-text.ts → src/layout.ts]
├── dynamic-layout-text.ts              [纯文本常量]
├── wrap-geometry.ts                    [几何工具库，依赖: pages/assets/*.svg]
├── variable-typographic-ascii.html/ts  [依赖: src/layout.ts]
├── editorial-engine.html/ts            [依赖: src/layout.ts]
├── justification-comparison.html/ts    [依赖: .model.ts, .ui.ts → src/layout.ts]
├── justification-comparison.model.ts   [算法实现，依赖: .data.ts → src/layout.ts]
├── justification-comparison.ui.ts      [渲染层，依赖: .model.ts]
├── justification-comparison.data.ts    [纯数据]
├── rich-note.html/ts                   [依赖: src/layout.ts]
├── masonry/
│   ├── index.html                      [依赖: index.ts]
│   └── index.ts                        [依赖: shower-thoughts.json → src/layout.ts]
│   └── shower-thoughts.json            [纯数据]
└── svg.d.ts                            [SVG 模块类型声明]
```

---

## 5. 依赖与外部交互

### 5.1 内部依赖

| 依赖 | 用途 | 说明 |
|------|------|------|
| `src/layout.ts` | 核心 API | 所有 demos 的基础依赖 |
| `pages/assets/*.svg` | Logo 资源 | dynamic-layout 使用的 OpenAI/Claude SVG |
| `pages/demos/masonry/shower-thoughts.json` | 测试数据 | Masonry demo 的卡片内容 |

### 5.2 浏览器 API 使用

```typescript
// Canvas 2D（用于测量和渲染）
const ctx = canvas.getContext('2d')
ctx.measureText(text)           // 备用宽度测量
ctx.fillText(text, x, y)        // justification-comparison 渲染
ctx.getImageData()              // variable-typographic-ascii 亮度分析

// OffscreenCanvas（wrap-geometry.ts）
const canvas = new OffscreenCanvas(width, height)  // SVG 轮廓提取

// Intl.Segmenter（src/layout.ts 内部）
new Intl.Segmenter(undefined, { granularity: 'grapheme' })  // 字符分割

// Font Loading API
document.fonts.ready            // 所有 demos 等待字体就绪

// ResizeObserver（未使用，演示替代方案）
// 所有 demos 使用 window.resize + RAF 调度

// Pointer/Touch 事件
window.addEventListener('pointerdown/move/up', ...)  // editorial-engine 拖拽
```

### 5.3 运行时环境要求

- **ES Modules**：所有 `.ts` 文件使用 ES module 语法
- **TypeScript**：通过 Bun 的 TypeScript 支持直接运行
- **现代浏览器**：依赖 `Intl.Segmenter`, `OffscreenCanvas`, CSS 变量

---

## 6. 风险、边界与改进建议

### 6.1 当前风险

#### 风险 1：字体加载时序
```typescript
// 当前做法：等待所有字体就绪
document.fonts.ready.then(() => scheduleRender())
```
- **问题**：如果指定字体不可用，会静默回退到系统字体，导致测量偏差
- **缓解**：`AGENTS.md` 明确警告不要使用 `system-ui` 字体

#### 风险 2：硬编码常量
```typescript
// dynamic-layout.ts
const NARROW_BREAKPOINT = 760
const BODY_FONT = '20px "Iowan Old Style", "Palatino Linotype", ...'
```
- **问题**：断点和字体配置分散在各 demo 中，难以统一维护
- **影响**：设计系统变更需要修改多个文件

#### 风险 3：内存泄漏风险
```typescript
// wrap-geometry.ts
const wrapHullByKey = new Map<string, Promise<Point[]>>()
```
- **问题**：缓存永不清除，长时间运行可能累积
- **缓解**：demos 通常是短期会话，实际影响有限

#### 风险 4：无障碍支持不足
- **问题**：动态布局的绝对定位元素可能破坏屏幕阅读器的阅读顺序
- **现状**：部分 demos（如 editorial-engine）使用 `user-select: text` 保留文本选择

### 6.2 边界限制

| 边界 | 说明 |
|------|------|
| **语言支持** | 主要测试英语、CJK、阿拉伯语；其他脚本可能需额外验证 |
| **字体限制** | 不支持 `system-ui`（macOS 上 canvas 与 DOM 解析不同） |
| **CSS 模式** | 仅支持 `white-space: normal` 或 `pre-wrap`，不支持 `break-all`, `keep-all` 等 |
| **浏览器** | 依赖现代浏览器 API（Intl.Segmenter, OffscreenCanvas） |
| **性能基线** | 假设单次 `prepare` + 多次 `layout` 模式；频繁变更文本会抵消性能优势 |

### 6.3 改进建议

#### 建议 1：提取共享配置
```typescript
// 新建 demos/shared/config.ts
export const BREAKPOINTS = { narrow: 760, wide: 1200 }
export const FONTS = {
  body: '17px "Helvetica Neue", Helvetica, Arial, sans-serif',
  headline: '700 32px Georgia, "Times New Roman", serif',
}
```

#### 建议 2：统一动画调度
```typescript
// 新建 demos/shared/animation.ts
export function createAnimationLoop(render: (now: number) => boolean): void
```
- 当前各 demo 有自己的 RAF 调度逻辑，可统一

#### 建议 3：增加无障碍支持
- 为动态布局添加 `aria-live` 区域通知布局变化
- 提供"简化视图"模式（无障碍版无动画、无绝对定位）

#### 建议 4：测试覆盖
```typescript
// 建议添加 demos/e2e-tests/
// 使用 Playwright 或 Puppeteer 自动化验证：
// - 各 demo 无控制台错误
// - resize 后布局正确
// - 交互（拖拽、滑动）正常工作
```

#### 建议 5：性能监控
```typescript
// 在 demos 中集成性能标记
performance.mark('pretext-layout-start')
const result = layout(prepared, width, lineHeight)
performance.mark('pretext-layout-end')
performance.measure('pretext-layout', 'pretext-layout-start', 'pretext-layout-end')
```

#### 建议 6：文档增强
- 为每个 demo 添加"查看源码"链接到 GitHub
- 添加性能对比仪表板（实时显示 layout 耗时）

---

## 7. 附录：关键数据结构

### 7.1 Pretext 核心类型
```typescript
// src/layout.ts 导出
type PreparedText = { readonly [preparedTextBrand]: true }
type PreparedTextWithSegments = PreparedText & {
  segments: string[]
  widths: number[]
  // ... 内部字段
}
type LayoutCursor = { segmentIndex: number; graphemeIndex: number }
type LayoutLine = { text: string; width: number; start: LayoutCursor; end: LayoutCursor }
type LayoutLineRange = { width: number; start: LayoutCursor; end: LayoutCursor }
```

### 7.2 Demo 特定类型
```typescript
// bubbles-shared.ts
type WrapMetrics = { lineCount: number; height: number; maxLineWidth: number }
type BubbleRenderWidths = { cssWidth: number; tightWidth: number }

// wrap-geometry.ts
type Point = { x: number; y: number }
type Rect = { x: number; y: number; width: number; height: number }
type Interval = { left: number; right: number }
type BandObstacle = 
  | { kind: 'polygon'; points: Point[]; horizontalPadding: number; verticalPadding: number }
  | { kind: 'rects'; rects: Rect[]; horizontalPadding: number; verticalPadding: number }

// justification-comparison.model.ts
type MeasuredLine = {
  segments: LineSegment[]
  wordWidth: number
  spaceCount: number
  naturalWidth: number
  maxWidth: number
  ending: 'paragraph-end' | 'wrap'
  trailingMarker: 'none' | 'soft-hyphen'
}
type QualityMetrics = { avgDeviation: number; maxDeviation: number; riverCount: number; lineCount: number }
```

---

## 8. 总结

`pages/demos/` 是 Pretext 库的**能力展示中心**和**使用范例库**。通过 8 个精心设计的交互式演示，全面展示了：

1. **基础 API**：`prepare` + `layout` 的高度计算
2. **高级 API**：`layoutNextLine` 的流式布局、`walkLineRanges` 的几何收集
3. **复杂场景**：障碍物绕行、多栏连续流、富文本内联布局
4. **算法集成**：Knuth-Plass 最优断行、shrinkwrap 二分搜索
5. **性能优化**：虚拟滚动、DOM 节点池化、RAF 调度

这些 demos 不仅是文档的补充，更是**生产代码的参考实现**，为采用者提供了可直接借鉴的设计模式和最佳实践。
