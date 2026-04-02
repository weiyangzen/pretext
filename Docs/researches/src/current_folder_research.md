# SRC 目录深度研究文档

## 1. 场景与职责

### 1.1 项目定位

`src/` 目录是 **Pretext** 库的核心源码目录，这是一个用于浏览器环境的**高性能文本测量与布局库**。该库通过 Canvas API 替代 DOM 测量来避免强制同步布局（Forced Synchronous Layout），从而显著提升大量文本块的测量性能。

### 1.2 核心问题域

- **DOM 测量性能瓶颈**：传统 `getBoundingClientRect`/`offsetHeight` 每次测量都会触发文档重排，500 个文本块每帧可能消耗 30ms+
- **国际化文本处理**：CJK（中日韩）、阿拉伯语、希伯来语、泰语等复杂脚本的换行规则
- **跨浏览器一致性**：不同浏览器在字体渲染、emoji 宽度、行高计算等方面的差异

### 1.3 职责边界

| 职责 | 说明 |
|------|------|
| 文本分析 | 使用 `Intl.Segmenter` 进行分词，处理空白字符归一化 |
| 宽度测量 | 基于 Canvas `measureText` 的字符/词元宽度缓存 |
| 行断决策 | 实现 CSS `white-space: normal` + `overflow-wrap: break-word` 语义 |
| 布局计算 | 纯算术计算行数/高度，无 DOM 操作 |
| 双向文本 | 提供简化版 Bidi 元数据用于自定义渲染 |

---

## 2. 功能点目的

### 2.1 核心 API 分层

```
┌─────────────────────────────────────────────────────────────┐
│  公开 API (layout.ts)                                        │
│  ├── prepare()          - 快速路径，不透明句柄               │
│  ├── prepareWithSegments() - 富路径，暴露段数据               │
│  ├── layout()           - 热路径，仅返回行数/高度             │
│  ├── layoutWithLines()  - 富路径，返回每行文本/宽度           │
│  ├── layoutNextLine()   - 流式布局，逐行计算                  │
│  └── walkLineRanges()   - 几何遍历，无字符串物化              │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│  内部模块                                                    │
│  ├── analysis.ts        - 文本分析与分段                     │
│  ├── measurement.ts     - Canvas 测量与缓存                  │
│  ├── line-break.ts      - 行断算法核心                       │
│  └── bidi.ts            - 双向文本元数据                     │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 各功能点详细说明

#### 2.2.1 两阶段架构设计

| 阶段 | 函数 | 调用时机 | 性能特征 |
|------|------|----------|----------|
| **Prepare** | `prepare()` / `prepareWithSegments()` | 文本首次出现时（如评论加载） | 一次性的，包含分词和测量 |
| **Layout** | `layout()` / `layoutWithLines()` | 每次容器宽度变化 | ~0.0002ms/文本块，纯算术 |

#### 2.2.2 文本分析阶段 (analysis.ts)

**目的**：将原始文本转换为可测量的段（segments）序列

**关键处理**：
1. **空白字符归一化**：将 `\t\n\r\f` 和多空格折叠为单空格（`white-space: normal` 模式）
2. **Intl.Segmenter 分词**：利用浏览器原生 API 处理 CJK、泰语等
3. **标点合并**：将 `"better."` 合并为单个测量单元（匹配 CSS 行为）
4. **特殊段类型识别**：
   - `text` - 普通文本
   - `space` - 可折叠空格
   - `preserved-space` - 保留空格（pre-wrap 模式）
   - `tab` - 制表符
   - `glue` - 非断行胶连（NBSP/NNBSP/WJ）
   - `zero-width-break` - 零宽换行机会（ZWSP）
   - `soft-hyphen` - 软连字符（SHY）
   - `hard-break` - 强制换行（\n）

**预处理规则**（脚本特定）：
- CJK 禁则处理（Kinsoku）：行首/行尾禁止标点附着
- 阿拉伯语无空格标点簇合并
- URL 识别与查询字符串特殊处理
- 数字/时间范围合并（如 `7:00-9:00`）

#### 2.2.3 测量阶段 (measurement.ts)

**目的**：缓存字体相关的宽度数据

**核心机制**：
```typescript
// 双层缓存结构
Map<font, Map<segment, SegmentMetrics>>
```

**Emoji 校正**：
- Chrome/Firefox 在 macOS 上 <24px 时 Canvas 测量的 emoji 比 DOM 实际渲染宽
- 通过一次 DOM 校准测量（每个字体）计算校正值
- Safari 无需校正（Canvas 和 DOM 一致）

**引擎特征检测**：
```typescript
type EngineProfile = {
  lineFitEpsilon: number        // Safari: 1/64, 其他: 0.005
  carryCJKAfterClosingQuote: boolean   // Chromium 特有
  preferPrefixWidthsForBreakableRuns: boolean  // Safari 特有
  preferEarlySoftHyphenBreak: boolean   // Safari 特有
}
```

#### 2.2.4 行断算法 (line-break.ts)

**目的**：基于已缓存宽度计算换行位置

**算法特性**：
- 贪心算法（与浏览器一致）
- 支持 grapheme 级别的紧急断行（overflow-wrap）
- 尾部空格悬挂（不计入行宽）
- 软连字符处理（可见尾部 `-`）

**双路径实现**：
| 路径 | 适用场景 | 复杂度 |
|------|----------|--------|
| `simpleLineWalkFastPath` | 纯文本，无特殊段类型 | O(n) |
| 完整路径 | 含 tab、软连字符、硬换行等 | O(n) + 块处理 |

#### 2.2.5 双向文本 (bidi.ts)

**目的**：为富路径提供段级别的 Bidi 层级信息

**实现特点**：
- 基于 pdf.js 的简化版 Unicode Bidi 算法
- 仅计算嵌入层级（embedding levels）
- 行断引擎不消费此数据（用于自定义渲染）

---

## 3. 具体技术实现

### 3.1 关键数据结构

#### 3.1.1 PreparedText 内部结构

```typescript
type PreparedCore = {
  widths: number[]                    // 段宽度
  lineEndFitAdvances: number[]        // 行尾适配推进（不含尾部空格）
  lineEndPaintAdvances: number[]      // 行尾绘制推进（含可见宽度）
  kinds: SegmentBreakKind[]           // 段类型
  simpleLineWalkFastPath: boolean     // 是否可走快速路径
  segLevels: Int8Array | null         // Bidi 层级（富路径）
  breakableWidths: (number[] | null)[]    // Grapheme 宽度（长词）
  breakablePrefixWidths: (number[] | null)[]  // 前缀累积宽度
  discretionaryHyphenWidth: number    // 软连字符宽度
  tabStopAdvance: number              // 制表位间隔
  chunks: PreparedLineChunk[]         // 硬换行分块
}
```

#### 3.1.2 游标系统

```typescript
type LayoutCursor = {
  segmentIndex: number    // 段索引
  graphemeIndex: number   // 段内 grapheme 索引
}
```

### 3.2 关键流程

#### 3.2.1 Prepare 流程

```
输入: text, font, whiteSpaceMode
  │
  ▼
[空白字符归一化] ──→ normalized text
  │
  ▼
[Intl.Segmenter 分词] ──→ 初始段序列
  │
  ▼
[buildMergedSegmentation] 
  ├── 标点合并
  ├── CJK 禁则处理
  ├── 阿拉伯语特殊处理
  ├── URL/数字合并
  └── Glue 连接处理
  │
  ▼
[compileAnalysisChunks] ──→ 硬换行块
  │
  ▼
[measureAnalysis]
  ├── Canvas 宽度测量（带缓存）
  ├── CJK Grapheme 分割
  ├── 长词 Grapheme 宽度预计算
  └── Emoji 校正应用
  │
  ▼
输出: PreparedText / PreparedTextWithSegments
```

#### 3.2.2 Layout 流程

```
输入: prepared, maxWidth, lineHeight
  │
  ▼
[walkPreparedLines / countPreparedLines]
  ├── 遍历 chunks（硬换行块）
  ├── 每块内贪心行填充
  │   ├── 尝试添加整段
  │   ├── 检查溢出：寻找可断点
  │   ├── 紧急断行：Grapheme 级分割
  │   └── 软连字符特殊处理
  └── 返回行数
  │
  ▼
输出: { lineCount, height }
```

### 3.3 缓存策略

| 缓存 | 作用域 | 清理方式 |
|------|--------|----------|
| `segmentMetricCaches` | 全局，按字体 | `clearCache()` |
| `emojiCorrectionCache` | 全局，按字体 | `clearCache()` |
| `sharedWordSegmenter` | 全局 | `setLocale()` / `clearCache()` |
| `sharedGraphemeSegmenter` | 全局 | `clearCache()` |
| `sharedLineTextCaches` | WeakMap，按 PreparedText | 自动 GC |

### 3.4 浏览器自动化测试协议

**脚本层** (`scripts/`):
- `accuracy-check.ts` - 精度检查，对比 Canvas 预测与 DOM 实际高度
- `corpus-check.ts` - 长文语料库检查
- `corpus-sweep.ts` - 多宽度批量扫描
- `browser-automation.ts` - 浏览器控制抽象（Chrome/Safari/Firefox）

**页面层** (`pages/`):
- `accuracy.ts` - 精度测试页面，输出 JSON 报告
- `corpus.ts` - 语料库测试页面
- `benchmark.ts` - 性能基准测试

**报告传输**：
- 小报告：URL hash (`#report=...`)
- 大报告：本地 POST 服务器 (`report-server.ts`)

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件清单

| 文件 | 行数 | 职责 | 关键导出 |
|------|------|------|----------|
| `src/layout.ts` | 717 | 主 API 与协调 | `prepare`, `layout`, `layoutWithLines`, `layoutNextLine`, `walkLineRanges` |
| `src/analysis.ts` | 1019 | 文本分析与分段 | `analyzeText`, `normalizeWhitespaceNormal`, `isCJK`, `kinsokuStart/End` |
| `src/measurement.ts` | 231 | Canvas 测量 | `getSegmentMetrics`, `getEngineProfile`, `getCorrectedSegmentWidth` |
| `src/line-break.ts` | 1059 | 行断算法 | `countPreparedLines`, `walkPreparedLines`, `layoutNextLineRange` |
| `src/bidi.ts` | 173 | 双向文本 | `computeSegmentLevels` |
| `src/test-data.ts` | 59 | 测试数据 | `TEXTS`, `SIZES`, `WIDTHS` |
| `src/layout.test.ts` | 906 | 单元测试 | Bun 测试套件 |

### 4.2 关键代码路径

#### 4.2.1 热路径（Resize 时）

```
layout() 
  → getInternalPrepared()
  → countPreparedLines() [line-break.ts:166]
    → walkPreparedLinesSimple() [line-break.ts:177] (fast path)
    或 walkPreparedLines() [line-break.ts:353] (full path)
```

#### 4.2.2 冷路径（文本首次出现）

```
prepare()
  → prepareInternal()
  → analyzeText() [analysis.ts:993]
    → normalizeWhitespaceNormal() [analysis.ts:56]
    → buildMergedSegmentation() [analysis.ts:795]
  → measureAnalysis() [layout.ts:191]
    → getFontMeasurementState() [measurement.ts:214]
    → getCorrectedSegmentWidth() [measurement.ts:169]
```

#### 4.2.3 CJK 特殊处理

```
measureAnalysis() 中 CJK 段处理 [layout.ts:279-318]
  → isCJK() [analysis.ts:105]
  → graphemeSegmenter.segment()
  → kinsokuEnd/kinsokuStart 检查 [analysis.ts:129-172]
  → getSegmentMetrics() [measurement.ts:52]
```

#### 4.2.4 软连字符处理

```
walkPreparedLines() [line-break.ts:353]
  → fitSoftHyphenBreak() [line-break.ts:81]
  → continueSoftHyphenBreakableSegment() [line-break.ts:489]
```

---

## 5. 依赖与外部交互

### 5.1 外部 API 依赖

| API | 用途 | 文件 |
|-----|------|------|
| `Intl.Segmenter` | 分词（word/grapheme） | `analysis.ts`, `layout.ts`, `measurement.ts` |
| `CanvasRenderingContext2D.measureText()` | 宽度测量 | `measurement.ts` |
| `OffscreenCanvas` | 后台测量上下文 | `measurement.ts` |
| `document.createElement('canvas')` | 主线程测量回退 | `measurement.ts` |
| `navigator.userAgent` | 引擎特征检测 | `measurement.ts` |

### 5.2 内部模块依赖图

```
layout.ts
  ├── analysis.ts (文本分析)
  ├── measurement.ts (测量缓存)
  └── line-break.ts (行断算法)

line-break.ts
  └── measurement.ts (EngineProfile)

bidi.ts (独立，仅被 layout.ts 引用)
```

### 5.3 配置与数据文件

| 文件 | 类型 | 用途 |
|------|------|------|
| `tsconfig.json` | 配置 | TypeScript 编译配置（开发） |
| `tsconfig.build.json` | 配置 | 发布构建配置 |
| `package.json` | 配置 | 包定义与脚本 |
| `corpora/sources.json` | 数据 | 语料库元数据 |
| `accuracy/*.json` | 数据 | 浏览器精度快照 |
| `benchmarks/*.json` | 数据 | 性能基准快照 |
| `status/dashboard.json` | 数据 | 聚合仪表板 |

### 5.4 脚本与页面交互

```
scripts/accuracy-check.ts ──► pages/accuracy.ts (浏览器页面)
scripts/corpus-check.ts   ──► pages/corpus.ts
scripts/corpus-sweep.ts   ──► pages/corpus.ts
scripts/benchmark-check.ts──► pages/benchmark.ts
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 字体相关风险

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| `system-ui` 字体 | macOS 上 Canvas 和 DOM 解析到不同光学变体 | 文档明确建议使用命名字体 |
| 字体回退 | 复杂回退链可能导致测量偏差 | 测试覆盖多字体场景 |
| 字体加载时机 | 自定义字体未加载完成时测量不准 | 调用方需确保字体就绪 |

#### 6.1.2 浏览器差异

| 浏览器 | 特殊处理 | 代码位置 |
|--------|----------|----------|
| Safari | lineFitEpsilon = 1/64（更宽松） | `measurement.ts:95` |
| Safari | preferPrefixWidthsForBreakableRuns = true | `measurement.ts:97` |
| Chromium | carryCJKAfterClosingQuote = true | `measurement.ts:96` |
| Firefox | 通过 BiDi 协议自动化 | `browser-automation.ts` |

#### 6.1.3 精度边界

- **行适配容差**：Chrome/Gecko 使用 `0.005`，Safari 使用 `1/64`
- **阿拉伯语细宽度**：剩余未解决的差异字段（AGENTS.md 记录）
- **软连字符**：某些宽度下 Chrome 的 `710px` 软连字符缺失（提取器敏感）

### 6.2 当前限制

1. **CSS 支持范围**：
   - ✅ `white-space: normal` / `pre-wrap`
   - ✅ `overflow-wrap: break-word`
   - ✅ `word-break: normal`
   - ❌ `break-all`, `keep-all`, `strict`, `loose`, `anywhere`

2. **Bidi 支持**：
   - 提供元数据但不参与行断决策
   - 自定义渲染器需自行处理重排

3. **垂直文本**：
   - 当前仅支持水平布局

### 6.3 改进建议

#### 6.3.1 架构层面

1. **行适配容差运行时校准**
   - 当前：硬编码浏览器特定值
   - 建议：与 emoji 校正类似，运行时采样校准

2. **ASCII 快速路径**
   - 当前：所有文本走统一路径
   - 建议：纯 ASCII 文本跳过 CJK、Bidi、Emoji 开销

3. **更丰富的断行策略**
   - 当前：贪心算法
   - 建议：可选的 Knuth-Plass 算法（排版质量优先场景）

#### 6.3.2 代码层面

1. **缓存持久化**
   - 当前：内存缓存，页面刷新丢失
   - 建议：考虑 IndexedDB 持久化（权衡复杂度）

2. **Web Worker 支持**
   - 当前：主线程测量
   - 建议：探索 OffscreenCanvas 在 Worker 中的测量

3. **增量 Prepare**
   - 当前：完整文本重新分析
   - 建议：编辑器场景下的增量更新

#### 6.3.3 测试层面

1. **自动化覆盖率**
   - 当前：手动触发浏览器检查
   - 建议：CI 集成（GitHub Actions 有难度，需 headless 浏览器）

2. **视觉回归测试**
   - 当前：高度数值对比
   - 建议：像素级对比（复杂但更全面）

### 6.4 维护注意事项

1. **JSON 快照更新时机**（AGENTS.md 规定）：
   - `accuracy/*.json`：行断方法或文本引擎变更时
   - `benchmarks/*.json`：热路径代码变更时
   - `corpora/representative.json`：锚点行为变更时

2. **浏览器自动化锁**：
   - 单浏览器单所有者（`browser-automation.ts`）
   - 死锁自恢复（检查 PID 存活）

3. **发布检查**：
   - `package-smoke-test.ts` 验证 tarball 消费
   - `dist/` 为发布时生成，不提交到仓库

---

## 附录：关键常量与集合

### CJK 禁则字符集

```typescript
// 行首禁止 (kinsokuStart)
'\uFF0C', '\uFF0E', '\uFF01', '\uFF1A', '\uFF1B', '\uFF1F',  // 全角标点
'\u3001', '\u3002', '\u30FB', '\uFF09', '\u3015', ...        // 日文标点

// 行尾禁止 (kinsokuEnd)
'"', '(', '[', '{', '\u201C', '\u2018', '\u00AB', '\u2039',   // 开引号/括号
'\uFF08', '\u3014', '\u3008', '\u300A', ...                    // 全角开括号
```

### 阿拉伯语无空格尾随标点

```typescript
arabicNoSpaceTrailingPunctuation = new Set([':', '.', '\u060C', '\u061B'])
// 对应：冒号、句点、阿拉伯逗号、阿拉伯分号
```

### 数字连接符

```typescript
numericJoinerChars = new Set([
  ':', '-', '/', '\u00D7', ',', '.', '+',  // ASCII
  '\u2013', '\u2014',                       // En dash, Em dash
])
```

---

*文档生成时间: 2026-04-02*
*研究范围: src/**/*.ts, scripts/**/*.ts, pages/**/*.ts, shared/**/*.ts, .github/workflows/*, accuracy/*.json, benchmarks/*.json, status/*.json*
*模型版本: k2p5*
