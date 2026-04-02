# Masonry Demo 研究文档

## 场景与职责

### 定位

Masonry Demo 是 Pretext 库的一个浏览器端演示应用，位于 `pages/demos/masonry/` 目录下。它是 Pretext 官方 Demo 集合中的 8 个演示之一（其他包括 Accordion、Bubbles、Dynamic Layout、Variable Typographic ASCII、Editorial Engine、Justification Comparison、Rich Text）。

### 核心职责

该演示展示 Pretext 在**瀑布流（Masonry）布局**场景下的应用能力：

1. **文本卡片高度预测**：利用 Pretext 的 `prepare()` + `layout()` API，在完全不触碰 DOM 的情况下计算每张文本卡片的高度
2. **虚拟化/遮挡剔除（Occlusion Culling）**：仅渲染视口可见的卡片，移除视口外的卡片以优化性能
3. **响应式列布局**：根据视口宽度动态计算列数（1-多列），实现自适应瀑布流

### 解决的问题

传统瀑布流布局需要依赖 DOM 测量（如 `getBoundingClientRect`、`offsetHeight`）来获取卡片高度，这会触发强制同步布局（Forced Synchronous Layout），导致性能问题。Masonry Demo 展示了如何使用 Pretext 的纯数学计算替代 DOM 测量，实现：

- **零布局回流（Zero Layout Reflow）**：所有高度计算通过 canvas 预测量 + 纯算术完成
- **流畅的 resize 响应**：窗口大小变化时只需重新执行 `layout()` 计算，无需重新测量文本
- **大规模数据支持**：演示使用 1904 条 Reddit "Shower Thoughts" 数据，验证大数据量下的性能表现

---

## 功能点目的

### 1. 瀑布流布局算法

**目的**：将不定高度的文本卡片按列填充，保持整体布局紧凑。

**实现策略**：
- 使用**最短列优先**算法：每次将新卡片放入当前高度最小的列
- 列数根据视口宽度动态计算：
  - 窄屏（≤520px）：单列
  - 宽屏：根据 `minColWidth = 100 + windowWidth * 0.1` 计算最大可容纳列数
- 列宽上限 400px，保证可读性

### 2. 虚拟化渲染

**目的**：处理大量卡片（1904 张）时保持流畅滚动性能。

**实现策略**：
- 视口检测：计算每张卡片是否在视口上下 200px 的缓冲区内
- DOM 节点池：仅创建可见卡片的 DOM 节点，不可见时移除并释放
- 批量 DOM 操作：DOM reads 和 writes 分离，避免布局抖动

### 3. 响应式重排

**目的**：窗口大小变化时快速重新计算布局。

**实现策略**：
- `requestAnimationFrame` 节流渲染
- 复用已 `prepare()` 的文本数据，只重新执行 `layout()`
- 列数、列宽、卡片位置全部重新计算

---

## 具体技术实现

### 关键流程

```
初始化流程:
1. 加载 shower-thoughts.json (1904 条文本)
2. 对每条文本执行 prepare(text, font) → PreparedText
3. 存储在 State.cards 中供后续复用

渲染流程 (scheduleRender → render):
1. DOM reads: 获取 windowWidth, windowHeight, scrollTop
2. 计算布局 (computeLayout):
   a. 根据 windowWidth 确定 colCount 和 colWidth
   b. 遍历所有卡片，使用 layout(prepared, textWidth, lineHeight) 计算高度
   c. 将卡片放入最短列，记录 x, y, h
3. DOM writes:
   a. 设置容器高度
   b. 对视口内卡片：创建/更新 DOM 节点，设置 left/top/width/height
   c. 对视口外卡片：移除 DOM 节点
```

### 数据结构

```typescript
// 核心类型定义 (pages/demos/masonry/index.ts)

// 卡片原始数据 + 预处理结果
type Card = {
  text: string           // 原始文本
  prepared: PreparedText // Pretext 预处理结果（缓存的段宽数据）
}

// 布局后的卡片位置信息
type PositionedCard = {
  cardIndex: number  // 在 cards 数组中的索引
  x: number          // 水平位置（px）
  y: number          // 垂直位置（px）
  h: number          // 总高度（含 padding）
}

// 完整布局状态
type LayoutState = {
  colWidth: number         // 列宽（px）
  contentHeight: number    // 内容总高度（px）
  positionedCards: PositionedCard[] // 所有卡片的布局信息
}

// 应用状态
type State = {
  cards: Card[] // 所有卡片的预处理后数据
}

// DOM 缓存结构
type DomCache = {
  container: HTMLDivElement              // 布局容器
  cards: Array<HTMLDivElement | undefined> // 卡片节点池（稀疏数组）
}
```

### 核心算法

#### 1. 列数计算算法

```typescript
// pages/demos/masonry/index.ts:55-65
function computeLayout(windowWidth: number): LayoutState {
  let colCount: number
  let colWidth: number
  if (windowWidth <= singleColumnMaxViewportWidth) {
    // 窄屏：单列，宽度为视口宽度减去两侧间隙
    colCount = 1
    colWidth = Math.min(maxColWidth, windowWidth - gap * 2)
  } else {
    // 宽屏：根据最小列宽计算可容纳列数
    const minColWidth = 100 + windowWidth * 0.1
    colCount = Math.max(2, Math.floor((windowWidth + gap) / (minColWidth + gap)))
    colWidth = Math.min(maxColWidth, (windowWidth - (colCount + 1) * gap) / colCount)
  }
  // ...
}
```

#### 2. 最短列填充算法

```typescript
// pages/demos/masonry/index.ts:70-91
const colHeights = new Float64Array(colCount) // 每列当前高度
for (let c = 0; c < colCount; c++) colHeights[c] = gap

const positionedCards: PositionedCard[] = []
for (let i = 0; i < st.cards.length; i++) {
  // 1. 找到当前高度最小的列
  let shortest = 0
  for (let c = 1; c < colCount; c++) {
    if (colHeights[c]! < colHeights[shortest]!) shortest = c
  }

  // 2. 使用 Pretext layout() 计算卡片高度
  const { height } = layout(st.cards[i]!.prepared, textWidth, lineHeight)
  const totalH = height + cardPadding * 2

  // 3. 记录卡片位置
  positionedCards.push({
    cardIndex: i,
    x: offsetLeft + shortest * (colWidth + gap),
    y: colHeights[shortest]!,
    h: totalH,
  })

  // 4. 更新该列高度
  colHeights[shortest]! += totalH + gap
}
```

#### 3. 虚拟化渲染算法

```typescript
// pages/demos/masonry/index.ts:126-158
function render() {
  // DOM reads
  const windowWidth = document.documentElement.clientWidth
  const windowHeight = document.documentElement.clientHeight
  const scrollTop = window.scrollY

  const layoutState = computeLayout(windowWidth)
  domCache.container.style.height = `${layoutState.contentHeight}px`

  // 视口范围 + 200px 缓冲
  const viewTop = scrollTop - 200
  const viewBottom = scrollTop + windowHeight + 200
  const visibleFlags = new Uint8Array(st.cards.length)

  // 第一遍：处理可见卡片
  for (let i = 0; i < layoutState.positionedCards.length; i++) {
    const positionedCard = layoutState.positionedCards[i]!
    if (positionedCard.y > viewBottom || positionedCard.y + positionedCard.h < viewTop) continue

    visibleFlags[positionedCard.cardIndex] = 1
    const node = getOrCreateCardNode(positionedCard.cardIndex)
    node.style.left = `${positionedCard.x}px`
    node.style.top = `${positionedCard.y}px`
    node.style.width = `${layoutState.colWidth}px`
    node.style.height = `${positionedCard.h}px`
  }

  // 第二遍：移除非可见卡片
  for (let cardIndex = 0; cardIndex < domCache.cards.length; cardIndex++) {
    const node = domCache.cards[cardIndex]
    if (node && visibleFlags[cardIndex] === 0) {
      node.remove()
      domCache.cards[cardIndex] = undefined
    }
  }
}
```

### 配置常量

```typescript
// pages/demos/masonry/index.ts:4-10
const font = '15px "Helvetica Neue", Helvetica, Arial, sans-serif'
const lineHeight = 22           // 行高（px）
const cardPadding = 16          // 卡片内边距（px）
const gap = 12                  // 卡片间隙（px）
const maxColWidth = 400         // 最大列宽（px）
const singleColumnMaxViewportWidth = 520  // 单列模式阈值（px）
```

---

## 关键代码路径与文件引用

### 本目录文件

| 文件 | 行数 | 职责 |
|------|------|------|
| `pages/demos/masonry/index.html` | 26 | 页面骨架，引入样式和 TypeScript 模块 |
| `pages/demos/masonry/index.ts` | 161 | 核心逻辑：布局算法、虚拟化渲染、事件处理 |
| `pages/demos/masonry/shower-thoughts.json` | 1 | 1904 条 Reddit Shower Thoughts 文本数据 |

### 依赖的 Pretext 核心 API

```typescript
// pages/demos/masonry/index.ts:1
import { prepare, layout, type PreparedText } from '../../../src/layout.ts'
```

| API | 来源文件 | 用途 |
|-----|----------|------|
| `prepare(text, font)` | `src/layout.ts:472` | 一次性预处理文本，测量段宽并缓存 |
| `layout(prepared, maxWidth, lineHeight)` | `src/layout.ts:495` | 热路径计算行数和高度，纯算术操作 |
| `PreparedText` | `src/layout.ts:99` | 预处理结果的类型定义（opaque handle） |

### 核心依赖链

```
pages/demos/masonry/index.ts
  ├── src/layout.ts (prepare, layout)
  │     ├── src/analysis.ts (文本分析、分段、归一化)
  │     ├── src/measurement.ts (canvas 测量、emoji 校正)
  │     └── src/line-break.ts (换行算法、行计数)
  └── pages/demos/masonry/shower-thoughts.json (数据)
```

### 构建与部署相关

```
scripts/build-demo-site.ts
  ├── 将 pages/demos/masonry/index.html 作为 entrypoint 构建
  └── 输出到 site/masonry/index.html

package.json
  └── "site:build": "rm -rf site && bun run scripts/build-demo-site.ts"
```

---

## 依赖与外部交互

### 运行时依赖

| 依赖 | 类型 | 说明 |
|------|------|------|
| Pretext Core (`src/layout.ts`) | 内部库 | 文本预处理和布局计算的核心能力 |
| Browser Canvas API | 浏览器原生 | `measureText()` 用于字体测量（在 prepare 阶段调用） |
| Intl.Segmenter | 浏览器原生 | 文本分段（词级、字素级） |
| DOM API | 浏览器原生 | 节点创建、样式设置、事件监听 |

### 数据依赖

- `shower-thoughts.json`：包含 1904 条字符串的 JSON 数组，每条是一个 "Shower Thought" 文本片段
- 数据来源：Reddit r/ShowerThoughts 社区
- 数据特征：短文本（通常 1-3 句话），英文为主，长度不一

### 事件交互

```typescript
// pages/demos/masonry/index.ts:113-124
window.addEventListener('resize', () => scheduleRender())
window.addEventListener('scroll', () => scheduleRender(), true)
```

- **resize**：窗口大小变化时重新计算列数和布局
- **scroll**：滚动时更新虚拟化渲染的可见卡片集合

### 样式系统

```css
/* pages/demos/masonry/index.html:6-23 */
* { margin: 0; padding: 0; box-sizing: border-box; }
body {
  font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif;
  background: #f0f0f0;
  overflow-y: scroll;
}
.card {
  position: absolute;  /* 绝对定位，通过 left/top 放置 */
  background: white;
  border-radius: 8px;
  padding: 16px;
  font-size: 15px;
  line-height: 22px;
  color: #333;
  box-shadow: 0 1px 3px rgba(0,0,0,0.08);
  overflow: hidden;
}
```

---

## 风险、边界与改进建议

### 当前风险

#### 1. 字体回退差异风险

**问题**：演示使用 `Helvetica Neue, Helvetica, Arial, sans-serif` 字体栈，在不同操作系统上可能回退到不同字体（如 Windows 上的 Arial vs macOS 上的 Helvetica Neue）。

**影响**：`prepare()` 阶段使用 canvas 测量字体宽度，如果实际渲染字体与测量字体不同，会导致高度计算偏差。

**缓解**：AGENTS.md 中已指出 `system-ui` 字体不安全，演示使用了具体命名的字体栈。

#### 2. 大数据量内存占用

**问题**：1904 条文本全部预处理后存储在内存中，每条文本的 `PreparedText` 包含多个并行数组（widths, kinds, breakableWidths 等）。

**估算**：假设平均每条文本 100 字节原始数据，预处理后可能膨胀到 1-2KB，总计约 2-4MB 内存占用。

**缓解**：对于更大规模数据，应考虑分页加载或虚拟化预处理。

#### 3. 列数计算抖动

**问题**：`minColWidth = 100 + windowWidth * 0.1` 是线性函数，在特定宽度阈值附近可能导致列数频繁变化（如 2 列 ↔ 3 列）。

**影响**：用户调整窗口大小时，布局可能发生剧烈变化。

### 边界条件

| 边界 | 当前处理 | 说明 |
|------|----------|------|
| 空文本 | 未显式处理 | shower-thoughts.json 中无空字符串 |
| 超长文本 | 正常处理 | 高度计算正确，但卡片会非常长 |
| 超窄视口 | 单列模式 | ≤520px 时强制单列 |
| 字体加载失败 | 依赖浏览器回退 | 可能影响测量精度 |
| 快速滚动 | RAF 节流 | 可能产生短暂空白 |

### 改进建议

#### 1. 布局稳定性优化

**建议**：引入列数变化的滞后（hysteresis）机制，避免在阈值附近抖动：

```typescript
// 伪代码
function computeColCount(windowWidth: number, currentColCount: number): number {
  const minColWidth = 100 + windowWidth * 0.1
  const newColCount = Math.max(2, Math.floor((windowWidth + gap) / (minColWidth + gap)))
  
  // 滞后：只有当变化超过 1 列时才更新
  if (Math.abs(newColCount - currentColCount) > 1) {
    return newColCount
  }
  return currentColCount
}
```

#### 2. 渐进式预处理

**建议**：对于更大规模数据，采用虚拟化预处理策略：

```typescript
// 仅预处理视口附近的文本
function prepareVisibleCards(scrollTop: number, viewportHeight: number) {
  const visibleRange = estimateVisibleRange(scrollTop, viewportHeight)
  for (let i = visibleRange.start; i < visibleRange.end; i++) {
    if (!st.cards[i]!.prepared) {
      st.cards[i]!.prepared = prepare(st.cards[i]!.text, font)
    }
  }
}
```

#### 3. 增强虚拟化

**建议**：当前虚拟化只处理 DOM 节点，但布局计算仍遍历所有卡片。可引入分层布局：

- 第一层：快速估算所有卡片高度（使用平均行数 × lineHeight）
- 第二层：精确计算视口附近卡片高度
- 滚动停止后，逐步精确化可见区域外的布局

#### 4. 可访问性增强

**建议**：
- 添加 `aria-label` 和 `role` 属性，使屏幕阅读器能正确识别卡片
- 支持键盘导航（方向键在卡片间移动）
- 支持减少动画偏好（`prefers-reduced-motion`）

#### 5. 性能监控

**建议**：添加性能指标收集：

```typescript
// 测量 layout() 调用耗时
const t0 = performance.now()
const { height } = layout(prepared, textWidth, lineHeight)
layoutTime += performance.now() - t0

// 定期上报
if (cardIndex % 100 === 0) {
  console.log(`Average layout time: ${(layoutTime / 100).toFixed(3)}ms`)
}
```

#### 6. 错误处理

**建议**：添加对 `prepare()` 和 `layout()` 的异常处理：

```typescript
try {
  const prepared = prepare(text, font)
  const { height } = layout(prepared, textWidth, lineHeight)
} catch (error) {
  console.error('Pretext layout failed:', error)
  // 回退到 DOM 测量或显示错误占位符
}
```

### 与 Pretext 库演进的关联

1. **API 兼容性**：当前使用 `prepare()` + `layout()` 基础 API，未来可考虑使用 `prepareWithSegments()` + `walkLineRanges()` 实现更复杂的卡片内排版（如首行缩进）

2. **新特性适配**：当 Pretext 支持 `whiteSpace: 'pre-wrap'` 模式时，演示可扩展支持保留换行符的文本卡片

3. **性能基准**：该演示可作为 Pretext 大规模数据集性能的基准测试场景，建议在 CI 中加入性能回归检测

---

## 附录：文件引用速查

| 引用目标 | 路径 |
|----------|------|
| 入口 HTML | `pages/demos/masonry/index.html` |
| 核心逻辑 | `pages/demos/masonry/index.ts` |
| 数据文件 | `pages/demos/masonry/shower-thoughts.json` |
| Pretext API | `src/layout.ts` |
| 行中断实现 | `src/line-break.ts` |
| 文本分析 | `src/analysis.ts` |
| 测量引擎 | `src/measurement.ts` |
| Demo 导航页 | `pages/demos/index.html` |
| 构建脚本 | `scripts/build-demo-site.ts` |
