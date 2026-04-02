# 研究文档：pages/demos/masonry/index.ts

## 场景与职责

`pages/demos/masonry/index.ts` 是 Pretext 库 **Masonry（瀑布流）布局演示** 的核心逻辑文件。它展示了如何在用户态（userland）中利用 Pretext 提供的 `prepare()` / `layout()` API，实现一个具有以下特性的高性能瀑布流页面：

- **响应式列数**：根据视口宽度动态计算列数和列宽；窄屏（≤520px）退化为单列。
- **精确高度预测**：在将卡片插入 DOM 之前，先通过 Pretext 的纯 JS 文本布局引擎计算出每张卡片在指定列宽下会占用多少行，从而得到精确高度。
- **最短列优先填充**：经典的 masonry 算法——每次将下一张卡片放到当前高度最短的列上。
- **虚拟化渲染**：仅渲染视口上下 200px 缓冲区内的卡片，超出范围的卡片从 DOM 中移除，以控制大量（约 250KB 原始文本）卡片场景下的内存与渲染开销。
- **RAF 节流**：resize 与 scroll 事件通过 `requestAnimationFrame` 进行调度，避免高频事件导致布局抖动（layout thrashing）。

## 功能点目的

| 功能模块 | 目的 |
|----------|------|
| ** upfront `prepare()` ** | 在页面加载时一次性对所有文本进行预处理（分词、测量、缓存），使后续的 resize/layout 只走纯数学计算，无需再访问 Canvas 或 DOM 进行测量。 |
| **`computeLayout()`** | 将视口宽度映射为列配置（列数、列宽、内容总宽、左偏移），然后为每张卡片计算其在 masonry 网格中的 `(x, y, h)`。 |
| **`getOrCreateCardNode()`** | 维护一个懒创建的 DOM 节点缓存池，避免重复创建/销毁 DOM 节点，减少 GC 压力。 |
| **`scheduleRender()` / `render()`** | 结合 RAF 与视口裁剪，实现“读-算-写”分离的渲染管线，最小化 DOM 操作次数。 |

## 具体技术实现

### 1. 配置常量

```ts
const font = '15px "Helvetica Neue", Helvetica, Arial, sans-serif'
const lineHeight = 22
const cardPadding = 16
const gap = 12
const maxColWidth = 400
const singleColumnMaxViewportWidth = 520
```

- `font` 必须与 `index.html` 中 `.card` 的 `font-family` 及 `font-size` 严格匹配，否则测量失真。
- `cardPadding` 与 CSS 中的 `padding: 16px` 对应，用于从列宽中扣除以得到文本可用宽度 `textWidth = colWidth - cardPadding * 2`。

### 2. 数据类型定义

```ts
type Card = {
  text: string
  prepared: PreparedText
}

type PositionedCard = {
  cardIndex: number
  x: number
  y: number
  h: number
}

type LayoutState = {
  colWidth: number
  contentHeight: number
  positionedCards: PositionedCard[]
}
```

- `Card` 将原始文本与 Pretext 的预处理句柄绑定，保证 `layout()` 调用时只需传入句柄和几何参数。
- `PositionedCard` 是 masonry 算法的输出原子，包含绝对定位所需的全部信息。
- `LayoutState` 是一次完整布局计算的快照，可被 `render()` 消费。

### 3. 初始化：一次性 `prepare()`

```ts
const st: State = {
  cards: rawThoughts.map(text => ({
    text,
    prepared: prepare(text, font),
  })),
}
```

- `rawThoughts` 来自同目录的 `shower-thoughts.json`，是一个字符串数组（约 250KB）。
- 在模块顶层立即执行，意味着页面加载后、任何渲染发生前，所有文本已完成分析。这是 Pretext 推荐的使用模式：将测量成本前置，让 resize/layout 走热路径。

### 4. DOM 缓存池

```ts
type DomCache = {
  container: HTMLDivElement
  cards: Array<HTMLDivElement | undefined>
}
```

- `container` 生命周期与页面相同，所有卡片节点都挂载在其下。
- `cards` 数组通过 `cardIndex` 索引，实现 O(1) 的节点复用查找。节点仅在首次进入视口时创建，离开视口时调用 `.remove()` 并将数组对应位置置为 `undefined`。

### 5. 布局计算：`computeLayout()`

#### 列数与列宽决策

```ts
if (windowWidth <= singleColumnMaxViewportWidth) {
  colCount = 1
  colWidth = Math.min(maxColWidth, windowWidth - gap * 2)
} else {
  const minColWidth = 100 + windowWidth * 0.1
  colCount = Math.max(2, Math.floor((windowWidth + gap) / (minColWidth + gap)))
  colWidth = Math.min(maxColWidth, (windowWidth - (colCount + 1) * gap) / colCount)
}
```

- 窄屏（≤520px）直接退化为单列，两侧保留 `gap` 边距。
- 宽屏使用动态最小列宽公式 `minColWidth = 100 + windowWidth * 0.1`，列数随视口变宽而增加，但单列宽度上限为 `maxColWidth = 400px`。
- 内容区总宽 `contentWidth = colCount * colWidth + (colCount - 1) * gap`，然后计算水平居中偏移 `offsetLeft = (windowWidth - contentWidth) / 2`。

#### Masonry 填充算法

```ts
const colHeights = new Float64Array(colCount)
for (let c = 0; c < colCount; c++) colHeights[c] = gap

for (let i = 0; i < st.cards.length; i++) {
  let shortest = 0
  for (let c = 1; c < colCount; c++) {
    if (colHeights[c]! < colHeights[shortest]!) shortest = c
  }

  const { height } = layout(st.cards[i]!.prepared, textWidth, lineHeight)
  const totalH = height + cardPadding * 2

  positionedCards.push({ cardIndex: i, x: ..., y: ..., h: totalH })
  colHeights[shortest]! += totalH + gap
}
```

- 使用 `Float64Array` 跟踪每列当前高度，初始值为 `gap`（顶部边距）。
- 每次通过线性扫描 `O(colCount)` 找到最短列。由于列数通常很小（2~4 列），线性扫描比优先队列更轻量。
- 对每张卡片调用 `layout(prepared, textWidth, lineHeight)`，返回 `{ lineCount, height }`。`height` 是文本块高度（`lineCount * lineHeight`），加上上下 `cardPadding * 2` 得到卡片总高 `totalH`。
- 更新最短列高度：`colHeights[shortest] += totalH + gap`。

### 6. 渲染管线：`render()`

```ts
function render() {
  // --- DOM reads ---
  const windowWidth = document.documentElement.clientWidth
  const windowHeight = document.documentElement.clientHeight
  const scrollTop = window.scrollY

  const layoutState = computeLayout(windowWidth)
  domCache.container.style.height = `${layoutState.contentHeight}px`

  // --- visibility + DOM writes (single pass) ---
  const viewTop = scrollTop - 200
  const viewBottom = scrollTop + windowHeight + 200
  const visibleFlags = new Uint8Array(st.cards.length)

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

  for (let cardIndex = 0; cardIndex < domCache.cards.length; cardIndex++) {
    const node = domCache.cards[cardIndex]
    if (node && visibleFlags[cardIndex] === 0) {
      node.remove()
      domCache.cards[cardIndex] = undefined
    }
  }
}
```

- **DOM 读取集中化**：`clientWidth`、`clientHeight`、`scrollY` 在函数开头一次性读取，避免在写入循环中触发强制同步布局（forced synchronous layout）。
- **容器高度同步**：将 `layoutState.contentHeight` 写入容器，确保页面滚动条长度正确，即使大量卡片被虚拟化移除。
- **视口裁剪**：上下各扩展 200px 作为缓冲，减少快速滚动时的白屏。
- **单遍写入**：对可见卡片，调用 `getOrCreateCardNode()` 获取节点并直接设置内联样式（`left`/`top`/`width`/`height`），不修改 `textContent`，因此复用节点时无文本重排成本。
- **回收阶段**：遍历整个 `domCache.cards`，对不可见节点执行 `.remove()` 并清空缓存引用。

### 7. RAF 调度

```ts
let scheduledRaf: number | null = null
function scheduleRender() {
  if (scheduledRaf != null) return
  scheduledRaf = requestAnimationFrame(function renderAndMaybeScheduleAnotherRender() {
    scheduledRaf = null
    render()
  })
}
```

- 标准的 RAF 节流模式：多个 resize/scroll 事件在 16ms 内只会触发一次 `render()`。
- 事件监听：`window.addEventListener('resize', ...)` 和 `window.addEventListener('scroll', ..., true)`（`true` 表示捕获阶段监听，确保在子元素滚动也能响应，虽然本页面结构简单，但增加了鲁棒性）。

## 关键代码路径与文件引用

- **自身路径**：`pages/demos/masonry/index.ts`
- **Pretext 核心 API**：`../../../src/layout.ts`
  - `prepare(text: string, font: string): PreparedText` —— 文本预处理
  - `layout(prepared: PreparedText, maxWidth: number, lineHeight: number): LayoutResult` —— 热路径布局计算
  - `PreparedText` —— 不透明预处理句柄类型
- **本地数据源**：`./shower-thoughts.json` —— 约 250KB 的淋浴想法文本数组
- **宿主 HTML**：`./index.html` —— 加载本模块并提供 `.card` 的基础 CSS
- **构建脚本**：`scripts/build-demo-site.ts` —— 将本 demo 打包进静态站点
- **入口页链接**：`pages/demos/index.html` —— 演示列表中的 masonry 入口

## 依赖与外部交互

| 依赖项 | 关系 | 说明 |
|--------|------|------|
| `src/layout.ts` | 核心库 | 提供 `prepare` / `layout` / `PreparedText`。这是本 demo 的技术基石；没有 DOM 测量，所有高度预测都依赖该模块。 |
| `shower-thoughts.json` | 本地数据 | 同目录下的 JSON 语料，模块通过默认导入获取字符串数组。数据量大（~250KB），是验证虚拟化必要性的关键。 |
| `index.html` | 宿主页面 | 提供全局样式和脚本加载上下文；CSS 中的字体、字号、padding 必须与本 TS 中的常量保持一致。 |
| `scripts/build-demo-site.ts` | 构建消费者 | 本文件通过 `index.html` 的模块引用间接参与 Bun 构建流程。 |
| 浏览器 API | 外部环境 | `document.documentElement.clientWidth/Height`、`window.scrollY`、`requestAnimationFrame`、`DOM` 操作。 |

## 风险、边界与改进建议

### 风险

1. **字体/样式失同步（最高优先级）**：
   - `index.ts` 中的 `font`、`lineHeight`、`cardPadding` 与 `index.html` 的 CSS 是硬编码耦合的。任何一方的修改若未同步，会导致 `layout()` 预测高度与实际渲染高度不一致，产生卡片重叠或间隙。
   - 同样，`maxColWidth`、`gap`、`singleColumnMaxViewportWidth` 也分散在 TS 中，缺乏统一配置源。

2. **列数扫描的线性复杂度**：
   - `computeLayout()` 中对最短列的查找是 `O(colCount)` 每张卡片。虽然列数通常很小，但在极端宽屏或未来支持更多列时，可能成为瓶颈。

3. **`Float64Array` 的过度精确**：
   - 使用 `Float64Array` 跟踪列高在数学上精确，但浏览器实际布局是亚像素（sub-pixel）且各引擎舍入策略不同。长期累积的浮点误差可能导致最后一列高度与真实 DOM 有 1px 级偏差。

4. **虚拟化无节点复用池**：
   - 当前策略是“离开视口即移除 DOM、进入视口即重新创建”。快速来回滚动会导致频繁的 DOM 创建/销毁，虽然比保留所有节点轻量，但不如对象池（object pool）模式高效。

5. **缺少错误边界**：
   - 若 `shower-thoughts.json` 损坏或 `prepare()` 抛出异常，页面将白屏，没有降级展示。

### 边界

- **无 SSR**：所有逻辑在浏览器端运行，首屏需要等待 JS 下载、执行、prepare、layout 完成后才能看到内容。
- **无动画/过渡**：卡片位置变化是瞬时跳变，没有 CSS transform 过渡或 FLIP 动画。
- **固定字体**：不支持用户通过浏览器设置更改字号后的重新布局（未监听 `matchMedia` 或系统字号变化）。
- **纯文本卡片**：每张卡片仅展示纯文本，不支持富文本、图片、内嵌链接等更复杂的内容类型。

### 改进建议

1. **统一配置源**：
   - 在 `index.html` 的 `:root` 中定义 CSS 自定义属性（`--masonry-font`、`--masonry-line-height`、`--masonry-gap` 等），`index.ts` 在初始化时通过 `getComputedStyle` 读取，消除硬编码重复。

2. **引入最小堆优化列查找**：
   - 若未来列数可能显著增加，可将 `colHeights` 维护为最小堆（priority queue），将每次查找最短列的复杂度从 `O(C)` 降到 `O(log C)`。

3. **DOM 节点对象池**：
   - 将 `domCache.cards` 从“存或 undefined”改为维护一个可见节点数组和一个隐藏节点池。卡片离开视口时不 `.remove()`，而是 `display: none` 或从容器中 detach 并放入池子，再次进入视口时优先从池中取出，避免反复的 `createElement` 和 `textContent` 赋值。

4. **增加首屏骨架或分批 prepare**：
   - 当前 `prepare()` 在模块顶层同步执行大量文本，若语料进一步增大可能阻塞主线程。可改为：
     - 先 `prepare` 前 N 张卡片并立即渲染首屏；
     - 使用 `requestIdleCallback` 或分帧 `setTimeout` 异步处理剩余卡片。

5. **监听系统字号变化**：
   - 通过 `matchMedia('(resolution: ...)')` 或 `ResizeObserver` 观察 `document.documentElement` 的尺寸变化（而不仅仅是 `window.resize`），以响应用户缩放或系统辅助功能设置导致的字号变化。

6. **增加 `will-change` / `contain` 提示**：
   - 在 `.card` 的创建逻辑中加入 `node.style.willChange = 'transform'`（若未来改用 transform 位移）或利用 `CSS.contain` 限制重排范围，可进一步提升滚动性能。

7. **增加 IntersectionObserver 替代手动裁剪**：
   - 当前手动计算 `viewTop`/`viewBottom` 是有效的，但在更复杂的容器滚动场景下，使用 `IntersectionObserver` 配合 `rootMargin: "200px"` 可以将可见性判断委托给浏览器，代码更简洁且能处理嵌套滚动容器。
