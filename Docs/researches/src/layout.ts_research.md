# src/layout.ts 研究文档

## 场景与职责

`layout.ts` 是 Pretext 库的公共 API 入口和核心布局引擎，实现了基于 Canvas 测量的高性能文本布局。它采用两阶段架构（准备阶段 + 布局阶段），旨在解决 DOM 测量导致的布局抖动（layout thrashing）问题。

### 核心职责
1. **公共 API 暴露**：提供 `prepare`/`prepareWithSegments`、`layout`/`layoutWithLines`、`layoutNextLine`、`walkLineRanges` 等接口
2. **两阶段布局协调**：协调文本分析（analysis）和测量（measurement）阶段
3. **富文本路径支持**：支持需要自定义渲染的富文本元数据（段内容、双向级别）
4. **性能优化**：保持热路径（`layout()`）无 DOM 读取、无 Canvas 调用、无字符串操作

### 架构定位
```
┌─────────────────────────────────────────────────────────────┐
│                        layout.ts                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │   prepare    │  │ prepareWith  │  │ profilePrepare   │  │
│  │   (opaque)   │  │   Segments   │  │   (diagnostic)   │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────────────┘  │
│         │                 │                                 │
│         └────────┬────────┘                                 │
│                  ▼                                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              prepareInternal                        │   │
│  │  ┌─────────────┐  ┌─────────────────────────────┐   │   │
│  │  │ analyzeText │  │      measureAnalysis        │   │   │
│  │  │(analysis.ts)│  │  ┌─────────────────────┐    │   │   │
│  │  └─────────────┘  │  │  CJK grapheme split │    │   │   │
│  │                   │  │  width measurement  │    │   │   │
│  │                   │  │  breakable widths   │    │   │   │
│  │                   │  └─────────────────────┘    │   │   │
│  │                   └─────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                                │
│         ┌──────────────────┼──────────────────┐            │
│         ▼                  ▼                  ▼            │
│  ┌─────────────┐  ┌─────────────────┐  ┌──────────────┐   │
│  │   layout    │  │ layoutWithLines │  │ layoutNextLine│   │
│  │  (hot path) │  │  (rich path)    │  │ (streaming)  │   │
│  └─────────────┘  └─────────────────┘  └──────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 功能点目的

### 1. 准备阶段 API

#### `prepare(text, font, options?)`
- **用途**：快速路径，返回不透明句柄
- **特点**：不包含段文本，仅用于 `layout()` 计算行数
- **适用场景**：只需要行数/高度的简单布局

#### `prepareWithSegments(text, font, options?)`
- **用途**：富路径，返回包含段数组的句柄
- **特点**：包含段文本、双向级别等元数据
- **适用场景**：自定义渲染、需要访问行内容

#### `profilePrepare(text, font, options?)`
- **用途**：诊断工具，分离分析和测量阶段耗时
- **返回**：`PrepareProfile` 包含各阶段毫秒数和段统计

### 2. 布局阶段 API

#### `layout(prepared, maxWidth, lineHeight)`
- **用途**：热路径，仅计算行数和高度
- **性能目标**：~0.0002ms 每文本块
- **特点**：无字符串操作、无 Canvas 调用、纯算术

#### `layoutWithLines(prepared, maxWidth, lineHeight)`
- **用途**：富路径，返回每行文本和宽度
- **特点**：使用 `walkPreparedLines` 遍历，物化每行内容
- **开销**：额外的字符串拼接和数组分配

#### `layoutNextLine(prepared, start, maxWidth)`
- **用途**：流式布局，从指定光标计算下一行
- **特点**：支持可变宽度布局（每行不同 maxWidth）
- **适用场景**：编辑器、动态布局

#### `walkLineRanges(prepared, maxWidth, onLine)`
- **用途**：非物化行几何遍历
- **特点**：不返回字符串，仅回调行范围（start/end 光标）
- **适用场景**：shrinkwrap 计算、聚合几何操作

### 3. 状态管理 API

#### `setLocale(locale?)`
- 设置 `Intl.Segmenter` 的语言
- 自动清除相关缓存

#### `clearCache()`
- 清除所有模块级缓存
- 包括分析缓存、测量缓存、字素缓存

---

## 具体技术实现

### 核心数据结构

#### 准备文本核心（内部表示）
```typescript
type PreparedCore = {
  widths: number[]                    // 段宽度
  lineEndFitAdvances: number[]        // 行尾适配宽度贡献
  lineEndPaintAdvances: number[]      // 行尾绘制宽度贡献
  kinds: SegmentBreakKind[]           // 段断点类型
  simpleLineWalkFastPath: boolean     // 是否可使用简化行遍历
  segLevels: Int8Array | null         // 双向级别（富路径）
  breakableWidths: (number[] | null)[] // 字素宽度（长词断行）
  breakablePrefixWidths: (number[] | null)[] // 前缀宽度（Safari 优化）
  discretionaryHyphenWidth: number    // 软连字符宽度
  tabStopAdvance: number              // 制表位间隔
  chunks: PreparedLineChunk[]         // 硬换行分块
}

// 不透明句柄（公共 API）
export type PreparedText = {
  readonly [preparedTextBrand]: true
}

// 内部句柄（类型断言使用）
type InternalPreparedText = PreparedText & PreparedCore

// 富文本句柄（公共 API）
export type PreparedTextWithSegments = InternalPreparedText & {
  segments: string[]
}
```

#### 布局结果类型
```typescript
export type LayoutResult = {
  lineCount: number
  height: number
}

export type LayoutLine = {
  text: string
  width: number
  start: LayoutCursor
  end: LayoutCursor
}

export type LayoutCursor = {
  segmentIndex: number
  graphemeIndex: number
}
```

### 关键处理流程

#### 1. 准备流程 `prepareInternal()`

```
输入: text, font, includeSegments, options
输出: InternalPreparedText | PreparedTextWithSegments

1. 调用 analyzeText() 进行文本分析
   - 规范化空白
   - 分段和合并
   - 生成分析块

2. 调用 measureAnalysis() 进行测量
   a. 获取引擎配置 (getEngineProfile)
   b. 获取字体测量状态 (getFontMeasurementState)
      - 获取/创建 Canvas 上下文
      - 获取段度量缓存
      - 计算 emoji 校正
   c. 遍历分析段：
      - 软连字符：零宽度，记录连字符宽度
      - 硬换行/制表符：零宽度
      - CJK 文本：字素拆分，应用禁则合并
      - 普通文本：测量宽度，计算行尾贡献
      - 长词：预计算字素宽度（breakableWidths）
   d. 映射分析块到准备块
   e. 计算段级别（富路径）

3. 返回准备结果
```

#### 2. CJK 字素拆分 `measureAnalysis()` 中的 CJK 处理

```typescript
if (segKind === 'text' && segMetrics.containsCJK) {
  let unitText = ''
  let unitStart = 0
  
  for (const gs of graphemeSegmenter.segment(segText)) {
    const grapheme = gs.segment
    
    // 禁则检查
    if (kinsokuEnd.has(unitText) ||           // 行尾禁止
        kinsokuStart.has(grapheme) ||         // 行首禁止
        leftStickyPunctuation.has(grapheme) || // 左粘附标点
        (engineProfile.carryCJKAfterClosingQuote && 
         isCJK(grapheme) && endsWithClosingQuote(unitText))) {
      unitText += grapheme  // 合并到当前单元
      continue
    }
    
    // 输出当前单元，开始新单元
    pushMeasuredSegment(unitText, w, w, w, 'text', ...)
    unitText = grapheme
    unitStart = gs.index
  }
  
  // 输出最后一个单元
  if (unitText.length > 0) {
    pushMeasuredSegment(unitText, w, w, w, 'text', ...)
  }
}
```

#### 3. 行文本物化 `buildLineTextFromRange()`

```
输入: segments, kinds, cache, startSegment, startGrapheme, endSegment, endGrapheme
输出: lineText

1. 检查是否以软连字符结尾
2. 遍历段范围：
   - 跳过软连字符和硬换行段
   - 起始段部分：从 startGrapheme 切片
   - 中间段：完整内容
   - 结束段部分：到 endGrapheme 切片
3. 如果以软连字符断行，追加连字符 '-'
```

### 缓存管理

```typescript
// 共享字素分段器
let sharedGraphemeSegmenter: Intl.Segmenter | null = null

// 行文本缓存（WeakMap，按 prepared 句柄隔离）
let sharedLineTextCaches = new WeakMap<PreparedTextWithSegments, Map<number, string[]>>()

// 获取/创建缓存
function getLineTextCache(prepared: PreparedTextWithSegments): Map<number, string[]>

// 获取段字素（带缓存）
function getSegmentGraphemes(
  segmentIndex: number,
  segments: string[],
  cache: Map<number, string[]>
): string[]
```

---

## 关键代码路径与文件引用

### 主要导出函数

| 函数 | 行号 | 用途 |
|------|------|------|
| `prepare()` | 472-474 | 快速准备路径 |
| `prepareWithSegments()` | 478-480 | 富准备路径 |
| `profilePrepare()` | 436-456 | 性能分析 |
| `layout()` | 490-501 | 热路径布局 |
| `layoutWithLines()` | 695-705 | 富路径布局 |
| `layoutNextLine()` | 681-689 | 流式布局 |
| `walkLineRanges()` | 669-679 | 非物化遍历 |
| `setLocale()` | 714-717 | 设置语言 |
| `clearCache()` | 707-712 | 清除缓存 |

### 内部调用关系

```
prepare / prepareWithSegments
└── prepareInternal
    ├── analyzeText (from analysis.ts)
    └── measureAnalysis
        ├── getEngineProfile (from measurement.ts)
        ├── getFontMeasurementState (from measurement.ts)
        ├── getSegmentMetrics (from measurement.ts)
        ├── getCorrectedSegmentWidth (from measurement.ts)
        ├── getSegmentGraphemeWidths (from measurement.ts)
        ├── getSegmentGraphemePrefixWidths (from measurement.ts)
        ├── getSharedGraphemeSegmenter
        ├── pushMeasuredSegment (内部)
        ├── mapAnalysisChunksToPreparedChunks (内部)
        └── computeSegmentLevels (from bidi.ts)

layout
└── countPreparedLines (from line-break.ts)

layoutWithLines
└── walkPreparedLines (from line-break.ts)
    └── materializeLayoutLine (内部)
        └── buildLineTextFromRange (内部)

layoutNextLine
└── stepLineRange (内部)
    └── stepPreparedLineRange (from line-break.ts)
        └── materializeLine (内部)

walkLineRanges
└── walkPreparedLines (from line-break.ts)
```

### 依赖导入

```typescript
// 双向文本
import { computeSegmentLevels } from './bidi.js'

// 文本分析
import {
  analyzeText, clearAnalysisCaches, endsWithClosingQuote, isCJK,
  kinsokuEnd, kinsokuStart, leftStickyPunctuation, setAnalysisLocale,
  type AnalysisChunk, type SegmentBreakKind, type TextAnalysis, type WhiteSpaceMode,
} from './analysis.js'

// 测量
import {
  clearMeasurementCaches, getCorrectedSegmentWidth, getEngineProfile,
  getFontMeasurementState, getSegmentGraphemePrefixWidths, getSegmentGraphemeWidths,
  getSegmentMetrics, textMayContainEmoji,
} from './measurement.js'

// 行断点
import {
  countPreparedLines, layoutNextLineRange as stepPreparedLineRange,
  walkPreparedLines, type InternalLayoutLine,
} from './line-break.js'
```

---

## 依赖与外部交互

### 外部依赖

| 依赖 | 来源 | 用途 |
|------|------|------|
| `Intl.Segmenter` | 浏览器原生 | 字素分段 |
| `performance.now()` | 浏览器原生 | 性能分析计时 |
| `OffscreenCanvas` / `Canvas` | 浏览器原生 | 文本测量（通过 measurement.ts） |

### 被依赖方

| 文件 | 导入内容 |
|------|----------|
| `layout.test.ts` | 所有公共 API |
| `pages/benchmark.ts` | `prepare`, `layout`, `profilePrepare` |
| `pages/accuracy.ts` | `prepare`, `layout`, `prepareWithSegments` |
| `pages/demos/*.ts` | 各种布局 API |
| `scripts/*.ts` | 诊断和检查脚本 |

---

## 风险、边界与改进建议

### 已知风险

1. **类型安全**
   - 使用 `as unknown as` 类型断言转换内部/公共类型
   - `PreparedText` 的不透明品牌仅编译时检查，运行时可伪造

2. **内存管理**
   - `sharedLineTextCaches` 使用 WeakMap，但内部 Map 可能累积
   - 长生命周期的 prepared 句柄可能导致字素缓存增长

3. **性能陷阱**
   - `layoutWithLines` 物化所有行，大文本可能分配大量字符串
   - CJK 字素拆分的内层循环在超长 CJK 文本中可能成为热点

4. **浏览器差异**
   - 依赖 `Intl.Segmenter` 的浏览器支持
   - Canvas 测量在不同浏览器/字体组合下可能有差异

### 边界条件

1. **空文本**
   - `createEmptyPrepared()` 返回空数组的 prepared 句柄
   - 所有布局 API 正确处理空 prepared（返回 0 行）

2. **零宽度**
   - `layout(prepared, 0, lineHeight)` 行为取决于内容
   - 每个字符/字素都触发断行

3. **极大宽度**
   - 单行布局，无断行

4. **光标边界**
   - `layoutNextLine` 接受任意起始光标
   - 超出范围的光标返回 null

5. **pre-wrap 空行**
   - 连续硬换行产生空行
   - 尾随硬换行不产生额外空行

### 改进建议

1. **类型安全**
   - 考虑使用更严格的内部/公共类型分离
   - 添加运行时品牌验证（开发模式）

2. **性能优化**
   - 为 `layoutWithLines` 添加迭代器版本，避免数组分配
   - 考虑使用字符串池减少小字符串分配
   - 为超长文本添加流式处理模式

3. **功能扩展**
   - 支持 `text-align`（左/右/居中/两端对齐）
   - 支持 `text-indent`
   - 支持 `hanging-punctuation`
   - 支持垂直文本布局

4. **可观测性**
   - 添加运行时指标收集（缓存命中率、平均段数等）
   - 添加开发模式警告（如检测到 `system-ui` 字体）

5. **测试覆盖**
   - 添加压力测试（百万字符文本）
   - 添加内存使用测试
   - 添加并发测试（验证缓存线程安全）

### 相关文档

- `AGENTS.md`: 详细的实现笔记和架构决策
- `README.md`: 公共 API 文档和使用示例
- `RESEARCH.md`: 浏览器精度研究和性能基准
- `DEVELOPMENT.md`: 开发命令和工作流

### 关键设计决策

根据 `AGENTS.md` 和代码注释：

1. **两阶段架构**
   > `prepare()` / `prepareWithSegments()` do horizontal-only work. `layout()` / `layoutWithLines()` take explicit `lineHeight`.

2. **热路径优化**
   > `layout()` is the resize hot path: no DOM reads, no canvas calls, no string work, and avoid gratuitous allocations.

3. **不透明句柄**
   > Keep the main prepared handle opaque so the public API does not accidentally calcify around the current parallel-array representation.

4. **富路径分离**
   > `prepare()` should stay the opaque fast-path handle. If a page/script needs segment arrays, that should usually flow through `prepareWithSegments()` instead of re-exposing internals on the main prepared type.
