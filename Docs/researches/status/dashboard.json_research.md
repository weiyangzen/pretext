# status/dashboard.json 研究文档

## 场景与职责

`status/dashboard.json` 是 Pretext 项目的主状态仪表板（Main Status Dashboard），作为机器可读的核心状态快照文件，承担以下关键职责：

1. **统一状态聚合**：将分散在多个子系统的浏览器准确性（accuracy）和性能基准（benchmark）数据聚合为单一可信源
2. **回归检测门控**：为 CI/CD 流程和开发者提供快速判断库健康状况的指标
3. **跨浏览器对比**：汇总 Chrome、Safari、Firefox 三大主流浏览器的测试结果对比
4. **性能追踪**：记录核心 API（prepare/layout）及富文本 API 的性能基准数据

该文件是项目状态管理体系的中枢节点，上游依赖 `accuracy/*.json` 和 `benchmarks/*.json`，下游被文档引用（如 STATUS.md）和开发者工具使用。

---

## 功能点目的

### 1. 浏览器准确性汇总 (browserAccuracy)

**目的**：量化库在三大浏览器中的文本布局准确性

**数据含义**：
- `total`: 总测试用例数（当前为 7680 = 4字体 × 8字号 × 8宽度 × 30文本）
- `matchCount`: 与浏览器 DOM 实际渲染结果一致的用例数
- `mismatchCount`: 不一致的用例数

**当前状态**：
```json
{
  "chrome":  { "total": 7680, "matchCount": 7680, "mismatchCount": 0 },
  "safari":  { "total": 7680, "matchCount": 7680, "mismatchCount": 0 },
  "firefox": { "total": 7680, "matchCount": 7680, "mismatchCount": 0 }
}
```

所有浏览器均达到 100% 匹配，表明核心布局算法已成熟稳定。

### 2. 性能基准汇总 (benchmarks)

**目的**：追踪核心 API 和富文本 API 的性能表现，确保 resize 热路径保持高效

**测试分类**：

| 分类 | 说明 | 关键指标 |
|------|------|----------|
| `topLevel` | 基础 API 性能 | prepare(), layout() vs DOM 操作 |
| `richShared` | 富文本共享语料 | layoutWithLines(), walkLineRanges(), layoutNextLine() |
| `richPreWrap` | pre-wrap 模式 | 320 硬断点块的性能 |
| `richLong` | 阿拉伯长文本 | 8 篇阿拉伯语长文跨多宽度测试 |
| `longFormCorpusStress` | 长语料压力测试 | 13 种语言的实际文本性能剖析 |

### 3. 长语料压力测试详情

**目的**：分析真实长文本（日文、中文、韩文、泰文、缅甸文、阿拉伯文等）的各阶段耗时

**关键指标**：
- `analysisMs`: 文本分析阶段（分段、glue 规则应用）
- `measureMs`: 测量阶段（canvas measureText）
- `prepareMs`: 总准备时间（analysis + measure）
- `layoutMs`: 布局阶段（纯算术计算）
- `segments`: 最终段数
- `breakableSegments`: 可断行段数（用于 overflow-wrap）

**典型数据示例**（日文《蜘蛛の糸》）：
```json
{
  "id": "ja-kumo-no-ito",
  "chars": 2874,
  "analysisSegments": 1773,
  "segments": 2667,
  "breakableSegments": 0,
  "analysisMs": 2.1,
  "measureMs": 4,
  "prepareMs": 6.1,
  "layoutMs": 0.0265
}
```

---

## 具体技术实现

### 数据生成流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    status-dashboard.ts                           │
│                     (数据聚合脚本)                                │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ accuracy/     │    │ benchmarks/   │    │ corpora/      │
│ chrome.json   │    │ chrome.json   │    │ dashboard.json│
│ safari.json   │    │ safari.json   │    │               │
│ firefox.json  │    │               │    │               │
└───────────────┘    └───────────────┘    └───────────────┘
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              ▼
                    ┌─────────────────┐
                    │ status/dashboard│
                    │     .json       │
                    └─────────────────┘
```

### 核心数据结构

```typescript
// 聚合后的仪表板结构
type Dashboard = {
  generatedAt: string;           // ISO 8601 时间戳
  sources: {
    accuracy: {                  // 源文件引用
      chrome: string;
      safari: string;
      firefox: string;
    };
    benchmarks: {
      chrome: string;
      safari: string;
    };
    corpora: string;
  };
  browserAccuracy: {             // 准确性汇总
    chrome: AccuracySummary;
    safari: AccuracySummary;
    firefox: AccuracySummary;
  };
  benchmarks: {                  // 性能基准
    chrome: BrowserBenchmarks;
    safari: BrowserBenchmarks;
  };
};

type AccuracySummary = {
  total: number;
  matchCount: number;
  mismatchCount: number;
};

type BrowserBenchmarks = {
  topLevel: Record<string, BenchmarkEntry>;
  richShared: Record<string, BenchmarkEntry>;
  richPreWrap: Record<string, BenchmarkEntry>;
  richLong: Record<string, BenchmarkEntry>;
  longFormCorpusStress: CorpusBenchmarkResult[];
};

type BenchmarkEntry = {
  ms: number;
  desc: string;
};

type CorpusBenchmarkResult = {
  id: string;
  label: string;
  font: string;
  chars: number;
  analysisSegments: number;
  segments: number;
  breakableSegments: number;
  width: number;
  lineCount: number;
  analysisMs: number;
  measureMs: number;
  prepareMs: number;
  layoutMs: number;
};
```

### 生成脚本关键逻辑

**文件**：`scripts/status-dashboard.ts`

```typescript
// 1. 加载上游 JSON 快照
const chromeAccuracy = await loadJson<AccuracyReport>('accuracy/chrome.json');
const safariAccuracy = await loadJson<AccuracyReport>('accuracy/safari.json');
const firefoxAccuracy = await loadJson<AccuracyReport>('accuracy/firefox.json');
const chromeBenchmarks = await loadJson<BenchmarkReport>('benchmarks/chrome.json');
const safariBenchmarks = await loadJson<BenchmarkReport>('benchmarks/safari.json');

// 2. 聚合准确性数据
const browserAccuracy = {
  chrome: summarizeAccuracy(chromeAccuracy),
  safari: summarizeAccuracy(safariAccuracy),
  firefox: summarizeAccuracy(firefoxAccuracy),
};

// 3. 索引化基准测试结果（便于查询）
const topLevel = indexResults(chromeBenchmarks.results);
const richShared = indexResults(chromeBenchmarks.richResults);
// ...

// 4. 写入输出文件
writeFileSync(output, JSON.stringify(dashboard, null, 2));
```

### 上游数据生成链

#### 1. 准确性数据 (`accuracy/*.json`)

**生成命令**：
```bash
bun run accuracy-snapshot        # Chrome
bun run accuracy-snapshot:safari # Safari
bun run accuracy-snapshot:firefox # Firefox
```

**执行流程**：
```
accuracy-check.ts
    └── browser-automation.ts  (浏览器自动化)
            ├── 启动临时页面服务器 (bun)
            ├── 打开 /accuracy 页面
            ├── 页面执行测试逻辑 (pages/accuracy.ts)
            │       ├── 使用 src/layout.ts 计算预测值
            │       ├── 使用 DOM 测量实际值
            │       └── 对比生成报告
            └── 捕获报告写入 JSON
```

**测试矩阵**：4字体 × 8字号 × 8宽度 × 30文本 = 7680 用例

#### 2. 基准数据 (`benchmarks/*.json`)

**生成命令**：
```bash
bun run benchmark-check        # Chrome
bun run benchmark-check:safari # Safari
```

**关键测试项**：

| 测试项 | 描述 | 当前值 (Chrome) |
|--------|------|-----------------|
| prepare() | 500 文本冷批处理 | ~18.6ms |
| layout() | 热路径吞吐量 | ~0.16ms |
| DOM batch | 批量 resize (写后读) | ~3.7ms |
| DOM interleaved | 交错 resize (每 div 读写) | ~41.7ms |

**性能对比**：`layout()` 比 DOM interleaved 快约 260 倍

---

## 关键代码路径与文件引用

### 核心文件依赖图

```
status/dashboard.json
    ▲
    │ (generated by)
    │
scripts/status-dashboard.ts
    │
    ├── accuracy/chrome.json ───────┐
    ├── accuracy/safari.json ───────┼── scripts/accuracy-check.ts
    ├── accuracy/firefox.json ──────┘   └── pages/accuracy.ts
    │                                       └── src/layout.ts
    │
    ├── benchmarks/chrome.json ─────┐
    ├── benchmarks/safari.json ─────┼── scripts/benchmark-check.ts
    │                               │   └── pages/benchmark.ts
    │                               │       └── src/layout.ts
    │                               │
    └── corpora/dashboard.json ─────┘── scripts/corpus-status.ts
                                        └── corpora/*.json
```

### 关键源文件详解

#### 1. 核心库 (`src/layout.ts`)

**公开 API**：
- `prepare(text, font, options?)`: 一次性文本分析 + 测量
- `layout(prepared, maxWidth, lineHeight)`: 热路径纯算术布局
- `prepareWithSegments()`: 富文本变体，返回段信息
- `layoutWithLines()`: 返回行文本和宽度
- `walkLineRanges()`: 仅几何信息，无字符串物化
- `layoutNextLine()`: 流式逐行布局

**内部数据结构**：
```typescript
type PreparedCore = {
  widths: number[];                    // 段宽度
  lineEndFitAdvances: number[];        // 行尾适配进宽
  lineEndPaintAdvances: number[];      // 行尾绘制宽
  kinds: SegmentBreakKind[];           // 断行类型
  simpleLineWalkFastPath: boolean;     // 是否可用快速路径
  segLevels: Int8Array | null;         // bidi 元数据
  breakableWidths: (number[] | null)[]; // 可断行 grapheme 宽
  breakablePrefixWidths: (number[] | null)[];
  discretionaryHyphenWidth: number;    // 软连字符宽
  tabStopAdvance: number;              // tab 停止位
  chunks: PreparedLineChunk[];         // 硬断块
};
```

#### 2. 文本分析 (`src/analysis.ts`)

**职责**：
- Unicode 文本规范化（空白字符处理）
- 使用 `Intl.Segmenter` 进行词/字分割
- Glue 规则应用（标点合并到前词）
- CJK 特殊处理（kinsoku 禁则）
- 支持两种 white-space 模式：`normal` | `pre-wrap`

**段类型定义**：
```typescript
type SegmentBreakKind =
  | 'text'              // 普通文本
  | 'space'             // 可折叠空格
  | 'preserved-space'   // 保留空格 (pre-wrap)
  | 'tab'               // 制表符
  | 'glue'              // NBSP/NNBSP/WJ 粘合
  | 'zero-width-break'  // ZWSP 零宽断点
  | 'soft-hyphen'       // 软连字符
  | 'hard-break';       // 硬换行
```

#### 3. 测量模块 (`src/measurement.ts`)

**核心功能**：
- Canvas `measureText()` 封装
- 段度量缓存（`Map<font, Map<segment, metrics>>`）
- Emoji 校正（canvas vs DOM 差异自动检测）
- 引擎特征检测（Chromium vs Safari vs Firefox）

**引擎特征配置**：
```typescript
type EngineProfile = {
  lineFitEpsilon: number;              // 行适配容差
  carryCJKAfterClosingQuote: boolean;  // Chromium 特有
  preferPrefixWidthsForBreakableRuns: boolean; // Safari 优化
  preferEarlySoftHyphenBreak: boolean; // Safari 优化
};

// 实际值
const profile = {
  lineFitEpsilon: isSafari ? 1/64 : 0.005,
  carryCJKAfterClosingQuote: isChromium,
  preferPrefixWidthsForBreakableRuns: isSafari,
  preferEarlySoftHyphenBreak: isSafari,
};
```

#### 4. 断行引擎 (`src/line-break.ts`)

**算法**：贪心 + 可断行段 grapheme 回退

**关键函数**：
- `countPreparedLines()`: 热路径行计数
- `layoutNextLineRange()`: 富 API 行范围计算
- `walkPreparedLines()`: 行遍历

**断行规则**（匹配 CSS `white-space: normal` + `overflow-wrap: break-word`）：
1. 在非空格段前断行（若会溢出）
2. 尾部空格悬挂（不触发断行）
3. 超宽段在 grapheme 边界断行

---

## 依赖与外部交互

### 上游依赖

| 依赖项 | 类型 | 说明 |
|--------|------|------|
| `accuracy/*.json` | 数据文件 | 浏览器准确性原始快照 |
| `benchmarks/*.json` | 数据文件 | 性能基准原始快照 |
| `corpora/dashboard.json` | 数据文件 | 语料库仪表板引用 |
| `scripts/status-dashboard.ts` | 生成脚本 | 聚合逻辑实现 |

### 下游消费者

| 消费者 | 用途 |
|--------|------|
| `STATUS.md` | 文档引用，提供人类可读的状态概览 |
| 开发者 | 运行 `bun run status-dashboard` 刷新 |
| CI/CD | 可作为健康检查的数据源 |

### 外部系统交互

1. **浏览器自动化**（数据生成阶段）
   - Chrome: 通过 AppleScript + Chrome 远程调试协议
   - Safari: 通过 AppleScript + Safari 驱动
   - Firefox: 通过类似机制

2. **Canvas API**（运行时测量）
   - `CanvasRenderingContext2D.measureText()`
   - `OffscreenCanvas` 优先，降级到 DOM canvas

3. **Intl.Segmenter**（文本分割）
   - `granularity: 'word'` - 词级分割
   - `granularity: 'grapheme'` - 字素分割

---

## 风险、边界与改进建议

### 当前风险

#### 1. 数据时效性风险

**问题**：`dashboard.json` 是生成的派生文件，可能与生成的源文件不同步

**缓解**：
- 文件包含 `generatedAt` 时间戳
- 包含 `sources` 字段明确记录源文件路径
- 文档明确说明需手动运行 `bun run status-dashboard` 刷新

#### 2. Safari 测量噪声

**问题**：Safari 基准测试比 Chrome 更嘈杂，热身行为不一致

**现状**：
- 代码中已设置更长的超时（240s vs 120s）
- 注释明确说明 Safari 数字仅供参考

#### 3. 语料库覆盖盲区

**当前已知盲区**（来自 RESEARCH.md）：
- 中文 Chrome 正偏差场（one-line positive field）
- 缅甸文 quote/follower 类残留问题
- 乌尔都语 Nastaliq/Naskh 窄宽负偏差

### 边界条件

#### 1. 字体限制

- `system-ui` 字体在 macOS 上不安全（canvas 与 DOM 解析不同）
- 必须使用具名字体（如 `"Helvetica Neue"`, `"Songti SC"`）

#### 2. 引擎容差

```typescript
// Chromium/Gecko
lineFitEpsilon = 0.005;

// Safari/WebKit  
lineFitEpsilon = 1/64;  // ~0.0156
```

Safari 需要更大容差 due to 阿拉伯文细宽场问题。

#### 3. 文本长度边界

- 短文本：`layout()` ~0.0002ms/块
- 长文本（10万+字符）：`prepare()` 可能达 60ms+，但 `layout()` 仍保持亚毫秒级

### 改进建议

#### 1. 自动化集成

**现状**：需手动运行 `bun run status-dashboard`

**建议**：
```bash
# 在 package.json 中添加前置钩子
"accuracy-snapshot": "... && bun run status-dashboard"
"benchmark-check": "... && bun run status-dashboard"
```

#### 2. 增量更新支持

**现状**：每次全量重写

**建议**：考虑仅更新变更字段，保留历史趋势数据：
```json
{
  "generatedAt": "...",
  "history": [
    { "date": "...", "chromeMatch": 7680, ... }
  ]
}
```

#### 3. 扩展 Firefox 基准

**现状**：仅 Chrome 和 Safari 有基准数据

**建议**：添加 Firefox 基准支持，完善跨浏览器性能对比

#### 4. 行适配容差运行时校准

**现状**：容差是硬编码的浏览器特定值

**建议**：如 AGENTS.md 所述，考虑与 emoji 校正一起进行运行时校准

#### 5. 添加健康检查端点

**建议**：添加验证脚本检查 dashboard.json 完整性：
```typescript
// scripts/validate-dashboard.ts
function validateDashboard(dashboard: Dashboard): boolean {
  // 检查所有必需字段
  // 检查数值合理性（如 matchCount <= total）
  // 检查时间戳新鲜度
}
```

---

## 附录：关键命令速查

```bash
# 刷新仪表板
bun run status-dashboard

# 刷新准确性快照
bun run accuracy-snapshot
bun run accuracy-snapshot:safari
bun run accuracy-snapshot:firefox

# 刷新基准快照
bun run benchmark-check
bun run benchmark-check:safari

# 完整语料库刷新
bun run corpus-status:refresh

# 开发服务器
bun start
```

---

## 附录：文件引用索引

| 文件路径 | 角色 | 关键导出/功能 |
|----------|------|---------------|
| `status/dashboard.json` | 本研究目标 | 聚合状态仪表板 |
| `scripts/status-dashboard.ts` | 生成脚本 | 数据聚合逻辑 |
| `scripts/accuracy-check.ts` | 准确性检查 | 浏览器自动化 |
| `scripts/benchmark-check.ts` | 基准检查 | 性能测试 |
| `pages/accuracy.ts` | 准确性页面 | 浏览器端测试逻辑 |
| `pages/benchmark.ts` | 基准页面 | 浏览器端性能测试 |
| `src/layout.ts` | 核心库 | prepare, layout API |
| `src/analysis.ts` | 文本分析 | 分段、glue 规则 |
| `src/measurement.ts` | 测量模块 | canvas 测量、缓存 |
| `src/line-break.ts` | 断行引擎 | 行计算算法 |
| `STATUS.md` | 状态文档 | 人类可读状态概览 |
| `DEVELOPMENT.md` | 开发文档 | 命令参考 |
| `RESEARCH.md` | 研究日志 | 历史决策记录 |
