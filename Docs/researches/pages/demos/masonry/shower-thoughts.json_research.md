# Research: `pages/demos/masonry/shower-thoughts.json`

## 场景与职责

`pages/demos/masonry/shower-thoughts.json` 是 Pretext 库 Masonry 演示页面（Masonry Demo）的**静态语料数据源**。该文件是一个纯 JSON 数组，包含 1,904 条英文短文本（即 Reddit r/ShowerThoughts 风格的短句），总大小约 250 KB。这些文本被 Masonry 演示用作“卡片”内容，以展示 Pretext 在**无需 DOM 测量**的前提下，如何对大量文本块进行高度预测、流式布局与视口外裁剪（occlusion culling）。

在仓库中的职责定位：
- **演示数据锚点**：为 `pages/demos/masonry/index.ts` 提供可批量渲染的真实文本内容。
- **高度预测场景验证器**：通过长短不一的文本卡片，验证 `prepare()` + `layout()` 在 resize 热路径上的性能与准确性。
- **发布包内容物**：由于 `package.json` 的 `files` 字段包含 `pages/demos`，该 JSON 会随 npm 包一起发布，供消费者直接运行本地 demo。

## 功能点目的

### 1. 支撑 Masonry 布局的高度零延迟计算
Masonry 布局的核心难点在于：在将卡片插入 DOM 之前，必须知道每张卡片在特定列宽下的高度，否则无法决定下一卡片应放置在哪一列。传统实现依赖 `getBoundingClientRect` / `offsetHeight` 等 DOM 读操作，会导致同步布局抖动（layout thrashing）。本演示通过 Pretext 的纯算术布局 API，在**零 DOM 读取**的情况下预先算出每张卡片高度，从而：
- 避免布局抖动；
- 支持 resize 时的实时重排；
- 为虚拟化/裁剪提供精确几何信息。

### 2. 视口外节点裁剪（Occlusion Culling）
由于数据量接近 2,000 条，全部挂载 DOM 会造成不必要的内存与绘制开销。演示利用 Pretext 返回的 `height` 在 JS 层完成 Masonry 定位后，仅对滚动视口上下扩展 200px 范围内的卡片创建/保留 DOM 节点，其余节点直接 `remove()`。JSON 数据的大容量特性使这一优化变得必要且可见。

### 3. 作为 Pretext 公共 API 的端到端示例
该 demo 是 README 与 `pages/demos/index.html` 中明确列出的官方示例之一，直接向库用户展示如何将 `prepare()`（一次性测量）与 `layout()`（每次 resize 调用）结合到真实 UI 中。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 数据文件结构
`shower-thoughts.json` 的内容格式极为简单：
```json
[
  "Men's public restrooms are laid out all wrong...",
  "The 'richest people' list in actuality is...",
  ...
]
```
- **类型**：`string[]`
- **条数**：1,904
- **编码**：UTF-8
- **加载方式**：TypeScript 通过 `import rawThoughts from './shower-thoughts.json'` 以 JSON module 形式直接引入（`package.json` 已设 `"type": "module"`）。

### Masonry 演示的核心类型与常量
文件：`pages/demos/masonry/index.ts`（L1–L33）

```typescript
const font = '15px "Helvetica Neue", Helvetica, Arial, sans-serif'
const lineHeight = 22
const cardPadding = 16
const gap = 12
const maxColWidth = 400
const singleColumnMaxViewportWidth = 520

type Card = { text: string; prepared: PreparedText }
type PositionedCard = { cardIndex: number; x: number; y: number; h: number }
type LayoutState = { colWidth: number; contentHeight: number; positionedCards: PositionedCard[] }
type State = { cards: Card[] }
```

### 关键流程

#### 阶段 A：初始化（一次性）
`index.ts` L34–L40：
```typescript
const st: State = {
  cards: rawThoughts.map(text => ({
    text,
    prepared: prepare(text, font),
  })),
}
```
- 遍历全部 1,904 条文本，对每条调用 `prepare(text, font)`。
- `prepare()` 内部执行：
  1. `analyzeText()` —— 使用 `Intl.Segmenter` 分词、归一化空白、合并标点、CJK 拆字等；
  2. `measureAnalysis()` —— 通过 `<canvas>` 的 `measureText` 测量每个 segment 宽度，并按 `(segment, font)` 缓存；
  3. 若文本含 emoji，进行一次 DOM 校准以修正 canvas 过宽问题。
- 结果 `PreparedText` 是**与宽度无关**的句柄，可在后续任意 `maxWidth` 下复用。

#### 阶段 B：Resize / Scroll 热路径
`computeLayout(windowWidth)`（L55–L99）：
1. **列数计算**：根据视口宽度决定 1 列或多列（`colCount`），并计算 `colWidth`。
2. **文本可用宽度**：`textWidth = colWidth - cardPadding * 2`。
3. **Masonry 贪心放置**：维护 `colHeights: Float64Array`，每次将下一张卡片放到当前最短的列上。
   - 卡片高度来源：`layout(st.cards[i].prepared, textWidth, lineHeight).height`。
   - 总卡片高度：`totalH = height + cardPadding * 2`。
4. 返回完整的 `LayoutState`，包含所有卡片的 `(x, y, h)`。

#### 阶段 C：渲染与裁剪
`render()`（L126–L159）：
1. **DOM 读取**：一次性读取 `clientWidth`、`clientHeight`、`scrollY`。
2. **布局计算**：调用 `computeLayout()`。
3. **可见性判断**：
   - 视口范围 = `[scrollTop - 200, scrollTop + windowHeight + 200]`。
   - 遍历 `positionedCards`，若卡片与视口相交，则标记 `visibleFlags[cardIndex] = 1`，并更新 DOM 节点的 `left/top/width/height`。
4. **节点回收**：遍历 `domCache.cards`，对不可见的已创建节点执行 `node.remove()` 并清空缓存引用。

### 数据结构细节
- **`Float64Array(colCount)`**：用于列高累加，保证数值稳定性。
- **`Uint8Array(st.cards.length)`**：可见性标记位图，避免在 DOM 写入循环中做额外分配。
- **`DomCache`**：
  - `container` 生命周期与页面相同；
  - `cards` 生命周期受可见性驱动，元素在进出视口时被创建/销毁。

### 构建与部署协议
- **开发服务器**：`bun run start` 会扫描 `pages/demos/*/index.html`，因此 `pages/demos/masonry/index.html` 可直接通过 `http://localhost:3000/demos/masonry` 访问。
- **静态站点构建**：`scripts/build-demo-site.ts` 将 `pages/demos/masonry/index.html` 作为 `entrypoints` 之一传入 `bun build`，输出到 `site/masonry/index.html`。构建脚本还会对相对路径进行重基（rebase）与链接重写。
- **npm 包发布**：`package.json` 的 `exports` 字段暴露 `"./demos/*": "./pages/demos/*"`，因此安装包的用户可通过 `@chenglou/pretext/demos/masonry/index.html` 访问该演示。

## 关键代码路径与文件引用

| 文件 | 角色 | 与本 JSON 的关系 |
|------|------|------------------|
| `pages/demos/masonry/shower-thoughts.json` | **目标文件** | 1,904 条短文本的静态数据源 |
| `pages/demos/masonry/index.ts` | 演示逻辑入口 | `import rawThoughts from './shower-thoughts.json'`（L2）；映射为 `Card[]` 并调用 `prepare()`（L36–L39） |
| `pages/demos/masonry/index.html` | 演示页面壳 | 加载 `index.ts`，提供 `.card` CSS 样式 |
| `src/layout.ts` | 核心库 API | 导出 `prepare()`（L472）与 `layout()`（L495），被 `index.ts` 调用 |
| `src/analysis.ts` | 文本分析阶段 | `prepare()` 内部调用 `analyzeText()`，处理分词、空白折叠、CJK、标点合并等 |
| `src/measurement.ts` | Canvas 测量运行时 | `prepare()` 内部调用 `getFontMeasurementState()` 与 `getSegmentMetrics()`，维护 segment 宽度缓存 |
| `src/line-break.ts` | 行遍历核心 | `layout()` 内部调用 `countPreparedLines()`，基于缓存宽度做纯算术断行 |
| `scripts/build-demo-site.ts` | Demo 站点构建脚本 | 将 `pages/demos/masonry/index.html` 列为 `entrypoints`（L13）与 `targets`（L38） |
| `pages/demos/index.html` | Demo 导航首页 | 包含指向 `/demos/masonry` 的链接卡片（L135–L138） |
| `README.md` | 项目文档 | 在“应用场景”段落中以 masonry 为例，说明 `height` 预测对 masonry 布局的价值（L46–L48） |
| `pages/demos/editorial-engine.ts` | 另一演示 | 在文案中提及 masonry 作为需要文本高度测量的典型场景（L190） |

## 依赖与外部交互

### 内部依赖
1. **Pretext 核心库 (`src/layout.ts`)**
   - `prepare(text, font)` → `PreparedText`
   - `layout(prepared, maxWidth, lineHeight)` → `{ lineCount, height }`
   - 这是本演示存在的根本理由：用库 API 替代 DOM 测量。

2. **Bun 运行时 / 构建系统**
   - JSON module import 依赖 Bun/Node ESM 对 `.json` 的加载支持（在 Bun 中默认可用）。
   - `bun build` 负责将 `index.ts` 及其依赖（包括本 JSON）打包为浏览器可执行的 bundle。

3. **开发服务器**
   - `bun run start` 使用 Bun 内置 dev server，自动将 `pages/demos/masonry/index.html` 及其相对资源（含 `.json` 与 `.ts`）暴露为 HTTP 端点。

### 外部交互
- **浏览器 DOM**：
  - 读取：`document.documentElement.clientWidth`、`window.scrollY`。
  - 写入：创建/更新/移除 `.card` div 的 `style.left/top/width/height`。
- **Canvas API（间接）**：`prepare()` 内部创建离屏 `<canvas>` 进行 `measureText`，但 demo 代码本身不直接操作 canvas。
- **无网络请求**：`shower-thoughts.json` 是本地静态文件，演示不依赖任何外部 API 或服务。

## 风险、边界与改进建议

### 风险

1. **JSON 文件体积对 bundle 的影响**
   - 约 250 KB 的纯文本数组会在构建时被打包进 Masonry demo 的 bundle。对于仅想查看核心库 API 的用户而言，这增加了无意义的下载量。虽然 demo 页面本身不是生产代码，但在慢速网络下首次加载会有可感知延迟。

2. **初始化时全量 `prepare()` 的阻塞风险**
   - `index.ts` L36–L39 在模块顶层同步对 1,904 条文本执行 `prepare()`。`prepare()` 涉及 `Intl.Segmenter` 分词和 canvas 测量，虽然单条很快，但 1,904 条累积仍可能在低端设备上造成数十毫秒的主线程阻塞，影响首次渲染的 TTI/FID。

3. **DOM 节点创建/销毁的抖动**
   - 快速滚动时，`render()` 会频繁创建和移除卡片节点。虽然使用了 `requestAnimationFrame` 节流，但大量节点的 `createElement` + `appendChild` / `remove()` 仍可能触发 GC 压力与样式重算，尤其是在列宽变化导致大量卡片同时进出视口时。

4. **字体回退导致的测量偏差**
   - 演示硬编码字体为 `"Helvetica Neue", Helvetica, Arial, sans-serif`（L5）。如果用户系统缺少这些字体，浏览器会回退到默认 sans-serif，而 Pretext 的 canvas 测量与实际 DOM 渲染可能产生细微宽度差异，导致卡片高度预测不准，出现文字截断或底部留白。

5. **单语言、单脚本语料的局限性**
   - `shower-thoughts.json` 全是英文 ASCII + 少量标点/数字。它无法展示 Pretext 对 CJK、阿拉伯语、泰语等复杂脚本的处理能力，也无法验证 emoji 修正逻辑在该 demo 中的效果（因为语料中几乎没有 emoji）。

### 边界

- **无动态数据更新**：JSON 是静态只读数据，演示不包含新增/删除/编辑卡片的功能。
- **无 SSR 路径**：`index.ts` 直接操作 `document`/`window`，无法在 Node/Bun 服务端运行。
- **无测试覆盖**：仓库中不存在针对 Masonry demo 或其 JSON 数据的自动化测试（如视觉回归或高度断言测试）。
- **无错误处理**：JSON import 失败或 `prepare()` 抛出异常时，演示没有降级逻辑，页面会直接白屏。

### 改进建议

1. **延迟/分批 `prepare()`**
   - 将 1,904 条卡片的 `prepare()` 工作从模块顶层移入应用初始化逻辑，并使用 `requestIdleCallback` 或基于 `requestAnimationFrame` 的时间切片（time-slicing）分批处理。优先准备视口附近的前 N 条，其余在空闲时后台准备。

2. **节点池复用（Object Pooling）**
   - 当前不可见卡片直接被 `remove()`。可以改为将节点移出视口（`display: none` 或保留在 DOM 中但脱离文档流）并放入对象池，待再次可见时复用，减少 `createElement` 和 GC 开销。

3. **语料多样化或动态切换**
   - 建议增加一个小的语料切换器，允许用户在英文 shower thoughts、混合 app 文本（`src/test-data.ts` 中已有）、CJK 文本之间切换。这能更好地展示 Pretext 的跨脚本能力，也能在手动测试时发现特定脚本的回归。

4. **添加轻量自动化断言**
   - 在 `src/layout.test.ts` 或单独的 demo 测试脚本中，增加对 Masonry demo 布局逻辑的单元测试：给定一组固定文本和固定视口宽度，断言 `computeLayout()` 返回的 `contentHeight` 和卡片 `y` 坐标是否符合预期。这可以在不启动浏览器的情况下捕获 `layout()` 回归。

5. **字体加载检测**
   - 若演示继续使用自定义字体，应监听 `document.fonts.ready`，在字体实际加载完成后再执行首次 `render()`，避免系统回退字体与 canvas 测量字体不一致导致的布局漂移。

6. **JSON 体积优化**
   - 若演示不需要全部 1,904 条数据，可只保留前 200–400 条作为默认语料，或将完整数据集改为懒加载的 `.json` fetch，以减小初始 bundle 体积。
