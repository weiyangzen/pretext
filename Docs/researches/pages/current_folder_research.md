# pages/ 目录深度研究报告

## 1. 场景与职责

`pages/` 目录是 Pretext 项目的浏览器端页面集合，承担以下核心职责：

### 1.1 核心定位
- **浏览器测试与验证平台**：提供浏览器环境下的文本布局算法精度测试、性能基准测试
- **可视化调试工具**：为文本断行问题提供详细的诊断和对比界面
- **API 使用示例**：展示 Pretext 库在实际应用场景中的各种用法
- **GitHub Pages 站点源**：作为项目演示站点直接部署

### 1.2 使用场景
| 场景 | 对应页面 | 目的 |
|------|----------|------|
| 算法精度回归测试 | `accuracy.ts` | 对比库预测高度与 DOM 实际高度 |
| 性能基准测试 | `benchmark.ts` | 测量 prepare/layout 各阶段性能 |
| 多语言语料测试 | `corpus.ts` | 长文本多语言断行验证 |
| 单文本精细诊断 | `probe.ts` | 特定文本片段的断行问题分析 |
| 经典英文文本测试 | `gatsby.ts` | 《了不起的盖茨比》开篇段落测试 |
| 交互式演示 | `demos/` | 气泡收缩、动态布局等实际应用 |

---

## 2. 功能点目的

### 2.1 核心测试页面

#### 2.1.1 accuracy.ts - 精度扫描页面
**目的**：系统性验证 Pretext 布局算法与浏览器 DOM 的一致性

**功能特性**：
- 多字体（4种）、多字号（8档）、多宽度（8档）的交叉测试矩阵
- 使用 `TEXTS` 测试数据集（拉丁、阿拉伯、希伯来、CJK、泰语、表情符号等）
- 通过 `getBrowserLines()` 使用 `Range.getClientRects()` 提取浏览器实际断行
- 生成详细的 mismatch 报告，包含逐行差异对比

**关键数据结构**：
```typescript
type Mismatch = {
  label: string
  font: string
  fontSize: number
  lineHeight: number
  width: number
  actual: number        // DOM 实际高度
  predicted: number     // 库预测高度
  diff: number
  text: string
  diagnosticLines?: string[]  // 逐行差异
}
```

#### 2.1.2 benchmark.ts - 性能基准页面
**目的**：测量 Pretext 各 API 的性能表现，与 DOM 操作对比

**测试维度**：
1. **prepare() 冷启动性能**：首次文本分析和测量耗时
2. **layout() 热路径性能**：基于缓存宽度的纯计算布局
3. **DOM 批量 vs 交错**：对比不同 DOM 读写模式的性能
4. **富 API 对比**：`layoutWithLines()` vs `walkLineRanges()` vs `layoutNextLine()`
5. **pre-wrap 模式**：硬断行和制表符处理的性能
6. **长文本压力测试**：阿拉伯语长文本的重复布局
7. **CJK vs Latin 扩展性**：不同字符集的增长曲线

**关键常量**：
```typescript
const COUNT = 500                    // 每批次文本数量
const WARMUP = 2                     // 预热轮数
const RUNS = 10                      // 测量轮数
const LAYOUT_SAMPLE_REPEATS = 200    // 布局重复次数
const CORPORA = [...]                // 13 种语言的长文本语料
```

#### 2.1.3 corpus.ts - 语料库压力测试
**目的**：多语言长文本的断行精度验证

**核心能力**：
- 支持 18 种语言的文学文本（日语、韩语、中文、泰语、缅甸语、阿拉伯语、乌尔都语、高棉语、印地语等）
- 动态宽度滑块控制（300-900px）
- 双模式浏览器行提取：`span-probe`（包裹 span）和 `range`（Range API）
- 完整的诊断报告：高度差异、行数差异、首个断行不匹配点分析

**诊断深度**：
- `diagnosticMode='light'`：仅高度对比
- `diagnosticMode='full'`：逐行内容对比、段宽度漂移分析、提取器敏感性检测

#### 2.1.4 probe.ts - 单文本探针
**目的**：快速诊断特定文本片段的断行行为

**特点**：
- 通过 URL 参数接收文本、宽度、字体等配置
- 支持 `whiteSpace: 'normal' | 'pre-wrap'` 模式
- 可切换 `method=span|range` 对比不同提取方式
- 输出结构化的 `ProbeReport`，包含首个断行不匹配详情

#### 2.1.5 gatsby.ts - 经典文本测试
**目的**：使用《了不起的盖茨比》开篇段落作为稳定的英文文本基准

**特殊功能**：
- 支持批量宽度扫描（`?widths=300,400,500`）
- 详细的断行边界描述（`describeBoundary()`）
- 段窗口分析（`getSegmentWindow()`）：展示断行点周围的上下文段

### 2.2 演示页面（demos/）

#### 2.2.1 bubbles.ts - 气泡收缩包装
**目的**：展示如何使用 `walkLineRanges()` 实现紧凑的多行消息气泡

**核心算法**：
```typescript
function findTightWrapMetrics(prepared, maxWidth) {
  // 二分查找保持相同行数的最小宽度
  while (lo < hi) {
    const mid = Math.floor((lo + hi) / 2)
    const midLineCount = layout(prepared, mid, LINE_HEIGHT).lineCount
    if (midLineCount <= initial.lineCount) hi = mid
    else lo = mid + 1
  }
}
```

#### 2.2.2 dynamic-layout.ts - 动态编辑布局
**目的**：展示高级布局能力——障碍物感知文本环绕

**技术亮点**：
- 使用 `layoutNextLine()` 实现逐行流式布局
- SVG Logo 的 alpha 通道提取生成环绕轮廓（`getWrapHull()`）
- 旋转 Logo 的实时碰撞检测和文本重排
- 双栏连续流：左栏消费后光标传递给右栏
- 标题字体大小的自适应拟合（二分查找）

#### 2.2.3 justification-comparison.ts - 对齐算法对比
**目的**：对比 CSS 两端对齐、贪婪断行和 Knuth-Plass 算法的视觉效果

#### 2.2.4 其他演示
- **accordion.ts**：手风琴展开/折叠的高度预测
- **variable-typographic-ascii.ts**：可变字体 ASCII 艺术
- **editorial-engine.ts**：动画球体和实时文本重排
- **rich-note.ts**：富文本内联布局
- **masonry.ts**：瀑布流布局

---

## 3. 具体技术实现

### 3.1 浏览器行提取技术

页面使用两种技术获取浏览器的实际断行结果：

#### 3.1.1 Span Probe 方法（`getBrowserLinesFromSpans`）
```typescript
// 为每个诊断单元创建 span，通过 getBoundingClientRect().top 检测换行
for (const unit of units) {
  const span = document.createElement('span')
  span.textContent = unit.text
  div.appendChild(span)
  spans.push(span)
}
// 检测 top 值变化识别新行
if (rectTop > lastTop + 0.5) { /* 新行 */ }
```

**适用场景**：普通文本，性能较好

#### 3.1.2 Range 方法（`getBrowserLinesFromRange`）
```typescript
// 使用 DOM Range API 获取精确位置
const range = document.createRange()
range.setStart(textNode, unit.start)
range.setEnd(textNode, unit.end)
const rects = range.getClientRects()
```

**适用场景**：
- 东南亚文字（泰语、老挝语、高棉语、缅甸语）
- RTL 文本（阿拉伯语、希伯来语、乌尔都语）
- 当 span probe 与 DOM 高度不一致时作为 fallback

### 3.2 诊断工具函数（diagnostic-utils.ts）

```typescript
// 获取诊断单元：对可断行段进行字素级拆分
export function getDiagnosticUnits(prepared: PreparedTextWithSegments): DiagnosticUnit[] {
  if (prepared.breakableWidths[i] !== null) {
    // 字素级拆分
    for (const grapheme of diagnosticGraphemeSegmenter.segment(segment)) {
      units.push({ text: grapheme.segment, start, end })
    }
  }
}

// 测量 Canvas 文本宽度
export function measureCanvasTextWidth(ctx, text, font): number

// 测量 DOM 文本宽度（创建临时 span）
export function measureDomTextWidth(doc, text, font, direction): number

// 格式化断行上下文
export function formatBreakContext(text, breakOffset, radius = 32): string
```

### 3.3 导航状态报告（report-utils.ts）

用于浏览器自动化测试时的状态通信：

```typescript
// 阶段报告：loading | measuring | posting
export function publishNavigationPhase(phase: NavigationPhase, requestId?: string)

// 完整报告（通过 URL hash 传递）
export function publishNavigationReport(report: unknown)
```

### 3.4 环绕几何计算（wrap-geometry.ts）

dynamic-layout.ts 的核心几何工具：

```typescript
// 从 SVG 图像提取环绕轮廓
export function getWrapHull(src: string, options: WrapHullOptions): Promise<Point[]>

// 将归一化轮廓点变换到实际布局矩形
export function transformWrapPoints(points: Point[], rect: Rect, angle: number): Point[]

// 计算多边形与水平带的交集区间
export function getPolygonIntervalForBand(
  points: Point[],
  bandTop: number,
  bandBottom: number,
  horizontalPadding: number,
  verticalPadding: number,
): Interval | null

// 从被阻挡区间雕刻出可用文本槽
export function carveTextLineSlots(base: Interval, blocked: Interval[]): Interval[]
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心依赖链

```
pages/accuracy.ts
├── ../src/layout.ts
│   ├── prepareWithSegments()  → 分析 + 测量
│   ├── layout()               → 纯计算行数
│   └── layoutWithLines()      → 带行内容布局
├── ./diagnostic-utils.ts
│   └── getDiagnosticUnits()   → 字素级诊断单元
├── ./report-utils.ts
│   └── publishNavigationPhase() → 自动化状态报告
└── ../src/test-data.ts
    └── TEXTS, SIZES, WIDTHS   → 测试参数

pages/corpus.ts
├── ../corpora/sources.json    → 语料元数据
├── ../corpora/*.txt           → 多语言文本内容（通过 import with { type: 'text' }）
└── （共享 diagnostic-utils.ts, report-utils.ts）

pages/benchmark.ts
├── ../corpora/*.txt           → 长文本语料
└── profilePrepare()           → 分阶段性能分析

pages/demos/dynamic-layout.ts
├── ./wrap-geometry.ts         → 环绕几何
├── ./dynamic-layout-text.ts   → 正文内容
└── ../assets/*.svg            → Logo 资源
```

### 4.2 URL 参数协议

各页面支持的查询参数：

| 页面 | 参数 | 说明 |
|------|------|------|
| accuracy | `requestId`, `full`, `reportEndpoint` | 自动化报告 |
| corpus | `id`, `width`, `widths`, `font`, `lineHeight`, `sliceStart/End`, `diagnostic`, `method` | 语料选择 + 诊断控制 |
| probe | `text`, `width`, `font`, `lineHeight`, `dir`, `lang`, `method`, `verbose`, `whiteSpace` | 单文本测试 |
| gatsby | `width`, `widths`, `report`, `diagnostic`, `requestId`, `reportEndpoint` | 宽度扫描 |
| benchmark | `report`, `requestId` | 报告模式 |

### 4.3 全局报告接口

页面通过 `window` 对象暴露报告供自动化脚本读取：

```typescript
// accuracy.ts
declare global {
  interface Window {
    __ACCURACY_REPORT__?: AccuracyReport
  }
}

// corpus.ts
interface Window {
  __CORPUS_REPORT__?: CorpusReport
  __CORPUS_DEBUG__?: { ... }
}

// probe.ts
interface Window {
  __PROBE_REPORT__?: ProbeReport
}

// gatsby.ts
interface Window {
  __GATSBY_REPORT__?: GatsbyReport
}

// benchmark.ts
interface Window {
  __BENCHMARK_REPORT__?: BenchmarkReport
}
```

---

## 5. 依赖与外部交互

### 5.1 源码依赖（src/）

| 模块 | 用途 |
|------|------|
| `layout.ts` | 核心 API：`prepare`, `prepareWithSegments`, `layout`, `layoutWithLines`, `walkLineRanges`, `layoutNextLine`, `clearCache` |
| `test-data.ts` | 共享测试数据：`TEXTS`（多语言短文本）、`SIZES`、`WIDTHS` |

### 5.2 共享模块（shared/）

| 模块 | 用途 |
|------|------|
| `navigation-state.ts` | URL hash 状态管理：`buildNavigationPhaseHash`, `buildNavigationReportHash`, `readNavigationPhaseState` |

### 5.3 语料资源（corpora/）

页面通过 `import with { type: 'text' }` 直接导入：

```typescript
import jaKumoNoIto from '../corpora/ja-kumo-no-ito.txt' with { type: 'text' }
import arRisalatAlGhufranPart1 from '../corpora/ar-risalat-al-ghufran-part-1.txt' with { type: 'text' }
// ... 共 18+ 种语言文本
```

### 5.4 浏览器 API 使用

| API | 用途 |
|-----|------|
| `Intl.Segmenter` | 字素分割（grapheme segmentation） |
| `CanvasRenderingContext2D.measureText()` | 文本宽度测量 |
| `Range.getClientRects()` | 精确获取文本位置（诊断） |
| `document.fonts.ready` | 等待字体加载完成 |
| `OffscreenCanvas` | Logo 轮廓提取（wrap-geometry.ts） |
| `getBoundingClientRect()` | DOM 高度验证 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 提取器敏感性（Extractor Sensitivity）
**问题**：不同浏览器行提取方法（span vs range）可能产生不同结果，导致诊断困惑。

**当前缓解**：
```typescript
const extractorSensitivity =
  firstBreakMismatch !== null && alternateFirstBreakMismatch === null
    ? `${browserLineMethod} mismatch disappears with ${alternateBrowserLineMethod}`
    : null
```

**建议**：在报告中明确标记 extractor-sensitive 的 mismatch，避免基于单一提取器的引擎修改。

#### 6.1.2 字体加载时序
**问题**：`document.fonts.ready` 在某些浏览器可能不可靠，导致初始测量使用回退字体。

**代码位置**：`corpus.ts:1055-1057`, `probe.ts:493-496`

#### 6.1.3 软连字符（Soft Hyphen）处理
**问题**：`layoutWithLines()` 在行尾添加可见连字符，但行类型没有单独标记软连字符元数据。

**AGENTS.md 注释**：
> If a soft hyphen wins the break, the rich line APIs should still expose the visible trailing `-` in `line.text`

### 6.2 边界条件

| 边界 | 处理 | 位置 |
|------|------|------|
| 空文本 | `createEmptyPrepared()` 返回空数组 | `layout.ts:159-189` |
| 零宽度 | `layout()` 返回 `lineCount: 0, height: 0` | 隐式处理 |
| 超长单词 | `breakableWidths` 存储字素级宽度用于溢出断行 | `layout.ts:331-357` |
| RTL 文本 | `dir` 属性设置 + `range` 提取方法 | `corpus.ts:327-328` |
| 东南亚文字 | 强制使用 `range` 提取方法 | `corpus.ts:165-166, 725-739` |

### 6.3 改进建议

#### 6.3.1 性能优化
- **问题**：`corpus.ts` 的 `measureWidth()` 每次调用都重新计算 `ourLines`，即使只改变宽度
- **建议**：缓存 prepared 结果和行分析，仅重新执行布局计算

#### 6.3.2 诊断增强
- **建议**：添加可视化叠加层，直接在页面上高亮显示 mismatch 行
- **建议**：导出 CSV/JSON 格式报告便于自动化分析

#### 6.3.3 代码组织
- **问题**：`corpus.ts` 和 `gatsby.ts` 有大量重复的浏览器行提取逻辑
- **建议**：将通用诊断逻辑进一步提取到 `diagnostic-utils.ts`

#### 6.3.4 类型安全
- **问题**：部分页面使用 `declare global` 扩展 Window 接口，分散在不同文件
- **建议**：集中定义在 `shared/window-types.ts`

#### 6.3.5 测试覆盖
- **建议**：为 `wrap-geometry.ts` 的几何计算添加单元测试
- **建议**：为各演示页面添加视觉回归测试

### 6.4 维护注意事项

1. **JSON 快照更新**：修改 `src/analysis.ts`, `src/measurement.ts`, `src/line-break.ts`, `src/layout.ts`, `src/bidi.ts` 或 `pages/accuracy.ts` 后，需刷新 `accuracy/chrome.json`, `accuracy/safari.json`, `accuracy/firefox.json`

2. **基准刷新**：修改性能相关代码后，需刷新 `benchmarks/chrome.json`, `benchmarks/safari.json`

3. **语料更新**：添加新语料时，需同步更新：
   - `corpora/sources.json`
   - `pages/corpus.ts` 的 `loadText()` switch 语句
   - `pages/benchmark.ts` 的 `CORPORA` 数组

---

## 附录：文件清单

### 核心页面（44 个文件）

```
pages/
├── accuracy.html          # 精度扫描页面 HTML
├── accuracy.ts            # 精度扫描逻辑（372 行）
├── benchmark.html         # 性能基准页面 HTML
├── benchmark.ts           # 性能基准逻辑（762 行）
├── corpus.html            # 语料测试页面 HTML
├── corpus.ts              # 语料测试逻辑（1137 行）
├── probe.html             # 单文本探针 HTML
├── probe.ts               # 单文本探针逻辑（497 行）
├── gatsby.html            # 盖茨比测试页面 HTML
├── gatsby.ts              # 盖茨比测试逻辑（791 行）
├── emoji-test.html        # 表情符号测试
├── justification-comparison.html
├── diagnostic-utils.ts    # 共享诊断工具（74 行）
├── report-utils.ts        # 报告状态工具（21 行）
├── assets/                # SVG 资源
│   ├── claude-symbol.svg
│   └── openai-symbol.svg
└── demos/                 # 演示页面（14 个 TS 文件）
    ├── index.html         # 演示首页
    ├── accordion.ts
    ├── bubbles.ts
    ├── bubbles-shared.ts
    ├── dynamic-layout.ts
    ├── dynamic-layout-text.ts
    ├── editorial-engine.ts
    ├── justification-comparison.ts
    ├── justification-comparison.model.ts
    ├── justification-comparison.ui.ts
    ├── justification-comparison.data.ts
    ├── rich-note.ts
    ├── variable-typographic-ascii.ts
    ├── wrap-geometry.ts   # 环绕几何计算（328 行）
    └── masonry/
```

### 关键代码统计

| 文件 | 行数 | 复杂度 | 维护优先级 |
|------|------|--------|-----------|
| `corpus.ts` | 1137 | 高 | 最高 |
| `dynamic-layout.ts` | 922 | 高 | 高 |
| `gatsby.ts` | 791 | 中 | 高 |
| `benchmark.ts` | 762 | 中 | 中 |
| `probe.ts` | 497 | 中 | 中 |
| `accuracy.ts` | 372 | 中 | 高 |
| `wrap-geometry.ts` | 328 | 高 | 中 |
| `bubbles-shared.ts` | 107 | 低 | 低 |
| `diagnostic-utils.ts` | 74 | 低 | 高 |
| `report-utils.ts` | 21 | 低 | 中 |
