# Pretext 项目研究文档

## 1. 场景与职责

### 1.1 项目概述

Pretext 是一个浏览器环境下的高性能文本测量与布局库，旨在解决 DOM 文本测量导致的同步布局回流（layout reflow）性能问题。项目采用两阶段测量架构：

- **准备阶段（Prepare）**：通过 Canvas `measureText` 进行一次性文本分析和测量
- **布局阶段（Layout）**：基于缓存的宽度数据进行纯算术计算，实现快速重排

### 1.2 核心使用场景

| 场景 | 描述 |
|------|------|
| 评论流/社交应用 | 大量文本块需要频繁计算高度以支持虚拟滚动 |
| 响应式布局 | 窗口大小变化时快速重算文本换行 |
| 多语言内容 | 支持 CJK、阿拉伯语、希伯来语等复杂脚本 |
| 富文本编辑器 | 精确控制文本断行和光标位置 |

### 1.3 项目结构职责划分

```
pretext/
├── src/                    # 核心库源码（研究重点）
│   ├── layout.ts          # 主 API：prepare/layout
│   ├── analysis.ts        # 文本分析与分段
│   ├── line-break.ts      # 换行算法核心
│   ├── measurement.ts     # Canvas 测量与缓存
│   ├── bidi.ts            # 双向文本（BiDi）支持
│   ├── test-data.ts       # 测试数据
│   └── layout.test.ts     # 单元测试
├── scripts/               # 自动化脚本（研究重点）
│   ├── browser-automation.ts   # 浏览器自动化基础
│   ├── accuracy-check.ts       # 精度检查
│   ├── corpus-check.ts         # 语料库检查
│   ├── corpus-sweep.ts         # 语料库批量扫描
│   ├── benchmark-check.ts      # 性能基准测试
│   └── status-dashboard.ts     # 状态仪表板生成
├── pages/                 # 浏览器测试页面（研究重点）
│   ├── accuracy.ts        # 精度测试页面
│   ├── benchmark.ts       # 性能测试页面
│   ├── corpus.ts          # 语料库测试页面
│   └── demos/             # 演示页面
├── shared/                # 共享工具
│   └── navigation-state.ts    # 导航状态管理
├── corpora/               # 多语言语料库
├── accuracy/              # 精度检查结果快照
├── benchmarks/            # 性能基准快照
└── status/                # 状态仪表板
```

---

## 2. 功能点目的

### 2.1 核心 API 功能

#### 2.1.1 准备阶段 API

| API | 功能 | 返回值 |
|-----|------|--------|
| `prepare(text, font, options?)` | 准备文本（不透明句柄） | `PreparedText` |
| `prepareWithSegments(text, font, options?)` | 准备文本（暴露分段信息） | `PreparedTextWithSegments` |
| `profilePrepare(text, font)` | 性能分析版本 | `PrepareProfile` |
| `setLocale(locale?)` | 设置分词语言 | `void` |
| `clearCache()` | 清除所有缓存 | `void` |

#### 2.1.2 布局阶段 API

| API | 功能 | 适用场景 |
|-----|------|----------|
| `layout(prepared, maxWidth, lineHeight)` | 快速计算行数和高度 | 热路径（resize） |
| `layoutWithLines(prepared, maxWidth, lineHeight)` | 返回完整行内容 | 需要渲染文本 |
| `layoutNextLine(prepared, start, maxWidth)` | 流式单行布局 | 可变宽度布局 |
| `walkLineRanges(prepared, maxWidth, onLine)` | 非物化行范围遍历 | 收缩包装计算 |

### 2.2 支持的文本特性

| 特性 | 说明 |
|------|------|
| **空白处理** | `white-space: normal`（默认）和 `pre-wrap` 模式 |
| **换行策略** | `overflow-wrap: break-word`（在字素边界断行） |
| **软连字符** | `&shy;` / `\u00AD` 支持 |
| **零宽空格** | `\u200B` 作为显式断行机会 |
| **不间断空格** | `\u00A0` / `\u202F` / `\u2060` 作为粘合（glue） |
| **制表符** | `pre-wrap` 模式下支持标准制表位 |
| **硬换行** | `\n` 在 `pre-wrap` 模式下保留 |

### 2.3 国际化支持

| 脚本 | 特殊处理 |
|------|----------|
| CJK（中日韩） | 逐字素断行、禁则处理（kinsoku） |
| 阿拉伯语 | 无空格标点聚类、组合标记处理 |
| 希伯来语 | RTL 双向文本支持 |
| 泰语/老挝语/高棉语/缅甸语 | `Intl.Segmenter` 分词支持 |
| Emoji | 自动校正 Canvas 测量偏差 |

---

## 3. 具体技术实现

### 3.1 核心数据流

```
┌─────────────────────────────────────────────────────────────┐
│                         准备阶段 (Prepare)                    │
├─────────────────────────────────────────────────────────────┤
│  1. 空白规范化 (normal/normalizeWhitespacePreWrap)           │
│  2. Intl.Segmenter 分词 (word granularity)                   │
│  3. 分段合并规则（标点、CJK、URL、数字等）                    │
│  4. Canvas measureText 测量每个分段                           │
│  5. 长词字素预测量（用于 overflow-wrap）                      │
│  6. Emoji 校正计算                                           │
│  7. 可选：BiDi 分段级别计算                                   │
└─────────────────────────────────────────────────────────────┘
                              ↓
                    PreparedText 句柄
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                         布局阶段 (Layout)                     │
├─────────────────────────────────────────────────────────────┤
│  纯算术计算：遍历分段宽度，累加至 maxWidth，决定断行位置        │
│  - 支持 trailing whitespace 悬挂                             │
│  - 支持长词在字素边界强制断行                                 │
│  - 支持 soft-hyphen 断行并添加连字符                          │
│  - 支持 pre-wrap 硬换行和制表位                               │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 关键数据结构

#### 3.2.1 PreparedText 内部结构（`layout.ts`）

```typescript
type PreparedCore = {
  widths: number[]                    // 分段宽度
  lineEndFitAdvances: number[]        // 行尾适配宽度贡献
  lineEndPaintAdvances: number[]      // 行尾绘制宽度贡献
  kinds: SegmentBreakKind[]           // 分段断行类型
  simpleLineWalkFastPath: boolean     // 是否可使用简单行遍历
  segLevels: Int8Array | null         // BiDi 分段级别（富路径）
  breakableWidths: (number[] | null)[]    // 字素宽度（长词）
  breakablePrefixWidths: (number[] | null)[]  // 字素前缀宽度
  discretionaryHyphenWidth: number    // 软连字符宽度
  tabStopAdvance: number              // 制表位步进
  chunks: PreparedLineChunk[]         // 硬换行分块
}
```

#### 3.2.2 分段断行类型（`analysis.ts`）

```typescript
type SegmentBreakKind =
  | 'text'              // 普通文本
  | 'space'             // 可折叠空格
  | 'preserved-space'   // 保留空格（pre-wrap）
  | 'tab'               // 制表符
  | 'glue'              // 粘合（NBSP/NNBSP/WJ）
  | 'zero-width-break'  // 零宽空格断行机会
  | 'soft-hyphen'       // 软连字符
  | 'hard-break'        // 硬换行
```

### 3.3 文本分析流程（`analysis.ts`）

```
buildMergedSegmentation()
├── normalizeWhitespaceNormal/PreWrap()     # 空白规范化
├── wordSegmenter.segment()                  # Intl.Segmenter 分词
├── splitSegmentByBreakKind()                # 按断行类型拆分
├── 合并规则处理：
│   ├── CJK 引号后字符合并
│   ├── CJK 行首禁则字符合并
│   ├── 缅甸语中介标记合并
│   ├── 阿拉伯语无空格标点合并
│   ├── 转义引号聚类合并
│   ├── 前向粘性标点合并
│   └── glue 连接文本合并
├── mergeUrlLikeRuns()                       # URL 合并
├── mergeUrlQueryRuns()                      # URL 查询参数合并
├── mergeNumericRuns()                       # 数字/时间范围合并
├── mergeAsciiPunctuationChains()            # ASCII 标点链合并
├── splitHyphenatedNumericRuns()             # 连字符数字拆分
└── carryTrailingForwardStickyAcrossCJKBoundary()  # CJK 边界前向粘性处理
```

### 3.4 换行算法（`line-break.ts`）

#### 3.4.1 行遍历状态机

```typescript
type InternalLayoutLine = {
  startSegmentIndex: number
  startGraphemeIndex: number
  endSegmentIndex: number
  endGraphemeIndex: number
  width: number
}
```

#### 3.4.2 断行决策逻辑

```
对于每个分段：
  1. 如果当前行为空且分段宽度 > maxWidth：
     - 如果是可断行分段：在字素边界断行
     - 否则：强制放入（溢出）
  
  2. 如果当前行非空且添加后 > maxWidth：
     - 检查当前分段是否允许断后（space/tab/soft-hyphen/ZWSP）
     - 是：断行，当前分段放入新行
     - 否：检查 pendingBreak（前一个可断行点）
       - 有 pendingBreak：回退到该点断行
       - 无 pendingBreak：强制断行
  
  3. 更新 pendingBreak（如果当前分段允许断后）
```

#### 3.4.3 软连字符特殊处理

```typescript
function fitSoftHyphenBreak(
  graphemeWidths: number[],
  initialWidth: number,
  maxWidth: number,
  lineFitEpsilon: number,
  discretionaryHyphenWidth: number,
  cumulativeWidths: boolean,
): { fitCount: number, fittedWidth: number }
```

### 3.5 测量与缓存（`measurement.ts`）

#### 3.5.1 缓存架构

```
segmentMetricCaches: Map<font, Map<segment, SegmentMetrics>>
├── 按字体分桶
│   └── 按分段文本缓存
│       ├── width: number           // 测量宽度
│       ├── containsCJK: boolean    // 是否含 CJK
│       ├── emojiCount?: number     // Emoji 数量（延迟计算）
│       ├── graphemeWidths?: number[]      // 字素宽度（延迟）
│       └── graphemePrefixWidths?: number[] // 字素前缀宽度（延迟）
```

#### 3.5.2 Emoji 校正机制

```typescript
function getEmojiCorrection(font: string, fontSize: number): number {
  // 1. 测量 Canvas 中 emoji 宽度
  // 2. 创建 DOM span 测量实际渲染宽度
  // 3. 计算差值作为校正量
  // 4. 缓存结果
}
```

#### 3.5.3 引擎配置文件

```typescript
type EngineProfile = {
  lineFitEpsilon: number              // 行适配容差（Safari: 1/64, 其他: 0.005）
  carryCJKAfterClosingQuote: boolean  // Chromium 特性
  preferPrefixWidthsForBreakableRuns: boolean  // Safari 特性
  preferEarlySoftHyphenBreak: boolean // Safari 特性
}
```

### 3.6 双向文本支持（`bidi.ts`）

基于 Unicode BiDi 算法简化实现：
- 字符分类为 L/R/AL/AN/EN/ES/ET/CS/ON/BN/B/S/WS/NSM
- 计算嵌入级别（embedding levels）
- 仅用于富路径的自定义渲染，不影响换行决策

---

## 4. 关键代码路径与文件引用

### 4.1 核心库文件

| 文件 | 行数 | 关键功能 | 主要导出 |
|------|------|----------|----------|
| `src/layout.ts` | 717 | 主 API 实现 | `prepare`, `layout`, `layoutWithLines` |
| `src/analysis.ts` | 1000+ | 文本分析与分段 | `analyzeText`, `SegmentBreakKind` |
| `src/line-break.ts` | 1000+ | 换行算法 | `countPreparedLines`, `walkPreparedLines` |
| `src/measurement.ts` | 231 | Canvas 测量 | `getSegmentMetrics`, `getEngineProfile` |
| `src/bidi.ts` | 173 | 双向文本 | `computeSegmentLevels` |
| `src/test-data.ts` | 59 | 测试数据 | `TEXTS`, `SIZES`, `WIDTHS` |
| `src/layout.test.ts` | 906 | 单元测试 | - |

### 4.2 关键代码路径

#### 4.2.1 准备路径

```
prepare() / prepareWithSegments()
  └── prepareInternal()
      ├── analyzeText() [analysis.ts:993]
      │   ├── normalizeWhitespaceNormal/PreWrap()
      │   ├── buildMergedSegmentation()
      │   │   ├── getSharedWordSegmenter() [Intl.Segmenter]
      │   │   └── 各种合并规则...
      │   └── compileAnalysisChunks()
      └── measureAnalysis() [layout.ts:191]
          ├── getFontMeasurementState()
          │   ├── getSegmentMetricCache()
          │   └── getEmojiCorrection()
          ├── 遍历分析分段测量
          │   ├── getSegmentMetrics() [measurement.ts:52]
          │   └── getCorrectedSegmentWidth()
          └── 处理 CJK 字素拆分
```

#### 4.2.2 布局路径

```
layout()
  └── countPreparedLines() [line-break.ts:166]
      ├── walkPreparedLinesSimple() [fast path]
      └── walkPreparedLines() [full path]
          └── 遍历 chunks，处理各种断行类型

layoutWithLines()
  └── walkPreparedLines()
      └── materializeLayoutLine()
          └── buildLineTextFromRange()
```

### 4.3 自动化脚本文件

| 文件 | 功能 | 关键依赖 |
|------|------|----------|
| `scripts/browser-automation.ts` | 浏览器会话管理、锁机制 | AppleScript (Safari/Chrome), Firefox BiDi |
| `scripts/accuracy-check.ts` | 精度检查自动化 | browser-automation.ts |
| `scripts/corpus-check.ts` | 语料库详细检查 | browser-automation.ts |
| `scripts/corpus-sweep.ts` | 语料库批量扫描 | browser-automation.ts, report-server.ts |
| `scripts/benchmark-check.ts` | 性能基准测试 | browser-automation.ts |
| `scripts/status-dashboard.ts` | 生成状态仪表板 | JSON 快照聚合 |

### 4.4 浏览器测试页面

| 文件 | 功能 | 测试范围 |
|------|------|----------|
| `pages/accuracy.ts` | 精度测试 | 4 字体 × 8 字号 × 8 宽度 × 55 文本 |
| `pages/benchmark.ts` | 性能测试 | 准备/布局/DOM 对比、富 API、长文本 |
| `pages/corpus.ts` | 语料库测试 | 17 个多语言语料库 |
| `pages/diagnostic-utils.ts` | 诊断工具 | 行提取、宽度测量 |

---

## 5. 依赖与外部交互

### 5.1 运行时依赖

| 依赖 | 用途 | 说明 |
|------|------|------|
| `Intl.Segmenter` | 文本分段 | 分词（word）和字素（grapheme）粒度 |
| `CanvasRenderingContext2D.measureText()` | 宽度测量 | 核心测量原语 |
| `OffscreenCanvas` | 离屏测量 | Worker 环境支持 |
| `navigator.userAgent` | 引擎检测 | Safari/Chromium 特性区分 |

### 5.2 开发依赖

| 依赖 | 用途 |
|------|------|
| `bun` | 运行时和包管理 |
| `typescript` | 类型检查 |
| `oxlint` | 代码检查 |

### 5.3 浏览器自动化依赖

| 平台 | 依赖 | 说明 |
|------|------|------|
| macOS | AppleScript | Safari/Chrome 控制 |
| 所有 | Firefox BiDi | Firefox 远程调试协议 |
| 所有 | Bun 服务器 | 本地页面服务 |

### 5.4 外部交互流程

```
自动化脚本 (Node/Bun)
    │
    ├── 获取浏览器锁 (PID 文件锁)
    │
    ├── 启动/复用页面服务器 (bun --port=xxx pages/*.html)
    │
    ├── 创建浏览器会话
    │   ├── Safari: osascript (AppleScript)
    │   ├── Chrome: osascript (AppleScript)
    │   └── Firefox: WebSocket BiDi 协议
    │
    ├── 导航到测试页面
    │   └── URL 参数: ?report=1&requestId=xxx
    │
    ├── 等待页面报告
    │   ├── 方式1: 轮询 URL hash (#report=...)
    │   └── 方式2: POST 到本地报告服务器
    │
    └── 解析报告结果
```

---

## 6. 风险、边界与改进建议

### 6.1 已知限制与风险

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **system-ui 字体** | Canvas 和 DOM 在 macOS 上解析不同光学变体 | 文档明确建议使用命名字体 |
| **字体回退差异** | 复杂回退链可能导致测量偏差 | 使用特定字体而非通用族 |
| **阿拉伯语精度** | 复杂 shaping 上下文可能仍有偏差 | 持续语料库调优、预处理规则 |
| **Safari 容差** | 需要更大的 lineFitEpsilon (1/64) | 引擎配置文件自动检测 |
| **Emoji 校正** | 仅针对 Apple Color Emoji 优化 | 按字体大小自动检测 |

### 6.2 边界条件

| 边界 | 处理 |
|------|------|
| 空文本 | 返回零行零高度 |
| 纯空白 | `normal` 模式返回空，`pre-wrap` 保留可见 |
| 超长无空格词 | 在字素边界强制断行 |
| 零宽度容器 | 每字符一行（CJK）或溢出 |
| 混合方向文本 | BiDi 元数据用于渲染，不影响换行 |

### 6.3 改进建议

#### 6.3.1 架构层面

1. **行适配容差动态校准**
   - 当前：硬编码浏览器特定值
   - 建议：与 emoji 校正类似，运行时通过简单探测自动校准

2. **ASCII 快速路径**
   - 当前：所有文本走统一路径
   - 建议：纯 ASCII 文本跳过 CJK、BiDi、Emoji 处理开销

3. **更丰富的断行策略**
   - 当前：`overflow-wrap: break-word` 等价
   - 建议：支持 `break-all`、`keep-all`、`strict`、`loose` 等 CSS 配置

#### 6.3.2 实现层面

4. **缓存粒度优化**
   - 当前：按 (font, segment) 缓存
   - 建议：考虑 LRU 淘汰策略防止内存无限增长

5. **Web Worker 支持**
   - 当前：依赖 OffscreenCanvas
   - 建议：明确测试和文档化 Worker 环境使用方式

6. **更精确的阿拉伯语支持**
   - 当前：基于预处理的启发式规则
   - 建议：考虑集成轻量级 shaping 引擎（如 harfbuzzjs）作为可选路径

#### 6.3.3 测试与监控

7. **持续集成**
   - 当前：本地浏览器自动化
   - 建议：添加 CI 流程（GitHub Actions）运行基础精度检查

8. **性能回归测试**
   - 当前：手动运行 benchmark
   - 建议：自动化性能基准对比，检测 PR 回归

9. **更多语料库覆盖**
   - 当前：17 个语料库
   - 建议：添加梵文（Devanagari）、泰米尔语等南亚脚本

---

## 7. 附录

### 7.1 文件引用索引

| 类别 | 文件路径 | 说明 |
|------|----------|------|
| 入口 | `src/layout.ts` | 主 API 导出 |
| 类型定义 | `src/layout.ts` (80-156行) | 公共类型定义 |
| 配置 | `tsconfig.build.json` | 发布构建配置 |
| 包定义 | `package.json` | npm 包配置 |
| 语料库定义 | `corpora/sources.json` | 17 个语料库元数据 |
| 状态仪表板 | `status/dashboard.json` | 聚合状态快照 |

### 7.2 关键常量

```typescript
// 行适配容差
const LINE_FIT_EPSILON = isSafari ? 1 / 64 : 0.005

// CJK Unicode 范围（analysis.ts:105-127）
const CJK_RANGES = [
  [0x4E00, 0x9FFF],    // CJK Unified Ideographs
  [0x3400, 0x4DBF],    // CJK Extension A
  [0x20000, 0x2A6DF],  // CJK Extension B
  // ... 更多扩展
]

// 禁则字符集（analysis.ts:129-172）
const kinsokuStart = new Set(['\uFF0C', '\uFF0E', ...])  // 行首禁止
const kinsokuEnd = new Set(['"', '(', '[', ...])         // 行尾禁止
```

### 7.3 性能指标参考

基于 `benchmarks/chrome.json` 和 `benchmarks/safari.json`：

| 操作 | 典型耗时 | 备注 |
|------|----------|------|
| `prepare()` 500 文本批次 | ~1-2ms | 冷启动，含测量 |
| `layout()` 500 文本批次 | ~0.01-0.02ms | 热路径 |
| DOM 批量读写 | ~10-20ms | 对比基准 |
| DOM 交错读写 | ~30-50ms | 对比基准 |

---

*文档生成时间: 2026-04-02*
*研究范围: src/**, scripts/**, shared/**, pages/**, package.json, corpora/sources.json, status/dashboard.json*
