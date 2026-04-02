# benchmarks/safari.json 研究文档

## 场景与职责

`benchmarks/safari.json` 是 Pretext 项目在 **Apple Safari** 浏览器下的性能基准快照（checked-in benchmark snapshot）。它与 `benchmarks/chrome.json` 配对使用，共同构成跨浏览器性能监控的核心数据源：

1. **跨浏览器性能对标**：揭示同一文本布局引擎在 WebKit 与 Chromium 两种渲染后端下的行为差异，特别是 canvas 文本测量与 DOM 布局成本的差异。
2. **平台特定瓶颈探测**：Safari 的 `measureText` 实现、字体回退策略及 Intl.Segmenter 性能与 Chrome 存在显著差异，本快照是发现这些差异的第一现场。
3. **回归防护**：当引擎热路径（`src/analysis.ts`、`src/measurement.ts`、`src/line-break.ts`、`src/layout.ts`、`src/bidi.ts`）发生变更时，通过对比 Chrome 与 Safari 的同步刷新结果，捕获平台特定的退化。
4. **状态仪表板数据源**：被 `scripts/status-dashboard.ts` 消费，生成 `status/dashboard.json` 中的 `benchmarks.safari` 节点。

---

## 功能点目的

快照结构与 Chrome 完全一致，分为 5 个功能分组，但数值反映的是 WebKit 环境下的实际表现：

| 分组 | 字段名 | Safari 特定关切 |
|------|--------|-----------------|
| 顶层批处理 | `results` | 验证 `prepare()` / `layout()` 在 Safari 上是否仍保持对 DOM 布局的数量级优势。 |
| 富行 API（共享语料） | `richResults` | 监控 `walkLineRanges()` 等无字符串物化 API 在 WebKit 下的亚毫秒稳定性。 |
| 富行 API（pre-wrap 压力） | `richPreWrapResults` | 检测 pre-wrap 硬断块处理在 Safari 中是否存在额外的行盒构建开销。 |
| 富行 API（阿拉伯长文本压力） | `richLongResults` | 阿拉伯语复杂 shaping 在 WebKit 下的富 API 延迟；当前数据显示 `walkLineRanges()` 明显快于 Chrome（2.4ms vs 5.1ms），但 `layoutNextLine()` 略慢（7.7ms vs 7.4ms）。 |
| 长形式语料库应力 | `corpusResults` | 重点观察 `measureMs` 在 Safari 上的极端膨胀（如合成长可断行：213ms vs Chrome 的 10ms），这是 Safari canvas 测量的已知弱点。 |

---

## 具体技术实现

### 3.1 数据生成流程

与 Chrome 快照共用同一套流水线，仅浏览器后端不同：

1. **页面执行**：`pages/benchmark.ts` 在 Safari 渲染进程中运行（通过 `scripts/browser-automation.ts` 的 AppleScript/OSA 驱动）。
2. **自动化收集**：`scripts/benchmark-check.ts` 识别 `BENCHMARK_CHECK_BROWSER=safari` 环境变量或 `--browser=safari` 参数，启动 Safari 会话。
3. **报告传输**：同样通过 URL hash 协议（`shared/navigation-state.ts` + `pages/report-utils.ts`）回传 `BenchmarkReport`。
4. **快照写入**：命令 `bun run benchmark-check:safari` 默认不带 `--output`，但开发者可手动指定 `--output=benchmarks/safari.json` 完成刷新。

### 3.2 关键测试参数

参数与 Chrome 完全一致，定义于 `pages/benchmark.ts`：

```ts
const COUNT = 500
const WARMUP = 2
const RUNS = 10
const LAYOUT_SAMPLE_REPEATS = 200
const RICH_COUNT = 60
const RICH_LAYOUT_SAMPLE_REPEATS = 40
const CORPUS_WARMUP = 1
const CORPUS_RUNS = 7
const CORPUS_LAYOUT_SAMPLE_REPEATS = 200
```

所有耗时取 **中位数**。由于 Safari 的 `performance.now()` 分辨率与 Chrome 不同，部分结果呈现更粗的粒度（如 `prepare()` 恰好为 18ms，而 Chrome 为 18.6ms）。

### 3.3 核心数据结构

与 Chrome 快照共享同一 Schema：

```ts
type BenchmarkReport = {
  status: 'ready' | 'error'
  requestId?: string
  results?: BenchmarkResult[]
  richResults?: BenchmarkResult[]
  richPreWrapResults?: BenchmarkResult[]
  richLongResults?: BenchmarkResult[]
  corpusResults?: CorpusBenchmarkResult[]
}

type BenchmarkResult = { label: string; ms: number; desc: string }

type CorpusBenchmarkResult = {
  id: string; label: string; font: string; chars: number
  analysisSegments: number; segments: number; breakableSegments: number
  width: number; lineCount: number
  analysisMs: number; measureMs: number; prepareMs: number; layoutMs: number
}
```

### 3.4 Safari 与 Chrome 的数值差异摘要

#### 3.4.1 顶层批处理（`results`）

| 指标 | Chrome | Safari | 差异 |
|------|--------|--------|------|
| `prepare()` | 18.60ms | 18ms | 基本持平 |
| `layout()` | 0.16ms | 0.16ms | 高度一致 |
| DOM batch | 3.70ms | 84.50ms | **Safari 慢 23×** |
| DOM interleaved | 41.70ms | 153.50ms | **Safari 慢 3.7×** |

**解读**：Safari 的 DOM 写读布局成本远高于 Chrome，但 Pretext 库的纯 JS 布局路径（`prepare`/`layout`）几乎不受影响，这使得 Safari 上的“库 vs DOM”优势比 Chrome 更为悬殊。

#### 3.4.2 长形式语料库 `measureMs` 异常点

| 语料 | Chrome measureMs | Safari measureMs | 膨胀倍数 |
|------|------------------|------------------|----------|
| synthetic-long-breakable-runs | 10.1ms | **213.0ms** | **21×** |
| ar-risalat-al-ghufran-part-1 | 33.8ms | **190.0ms** | **5.6×** |
| hi-eidgah | 6.2ms | **34.0ms** | **5.5×** |
| ur-chughd | 3.2ms | **43.0ms** | **13×** |
| km-prachum-reuang-preng-khmer-volume-7-stories-1-10 | 4.4ms | **15.0ms** | **3.4×** |
| th-nithan-vetal-story-1 | 5.4ms | **26.0ms** | **4.8×** |

**解读**：Safari 的 canvas 2D `measureText` 在处理长可断行单元、复杂脚本（阿拉伯语、乌尔都语 Nastaliq、印地语 Devanagari、高棉语、泰语）时存在显著的性能劣化。这通常与 WebKit 的文本 shaping 后端（CoreText）在频繁测量短子串时的开销有关。

#### 3.4.3 富行 API（阿拉伯长文本压力）

| API | Chrome | Safari | 差异 |
|-----|--------|--------|------|
| `layoutWithLines()` | 8.20ms | 6.38ms | Safari 更快 |
| `walkLineRanges()` | 5.06ms | 2.43ms | **Safari 快 2×** |
| `layoutNextLine()` | 7.39ms | 7.74ms | 基本持平 |

**解读**：在已经 `prepare` 完成的前提下，Safari 的行遍历反而更快，说明 WebKit 的 JS 执行或数组访问路径在此场景下有优势；`layoutNextLine()` 因涉及逐行 grapheme 缓存簿记，两者拉平。

---

## 关键代码路径与文件引用

| 文件 | 角色 |
|------|------|
| `pages/benchmark.ts` | 基准测试页面；所有浏览器共用同一套测试逻辑与常量。 |
| `scripts/benchmark-check.ts` | 通过 `--browser=safari` 或环境变量切换为 Safari 会话。 |
| `scripts/browser-automation.ts` | Safari 侧使用 AppleScript/OSA 打开/关闭标签页、导航 URL、读取页面标题/地址。 |
| `pages/report-utils.ts` | 页面端通过 URL hash 发布报告。 |
| `shared/navigation-state.ts` | Hash 编解码协议。 |
| `scripts/status-dashboard.ts` | 读取本文件，聚合到 `status/dashboard.json`。 |
| `src/layout.ts` | 被测 API 实现。 |
| `src/measurement.ts` | Safari `measureMs` 异常的直接责任方；包含 canvas 测量运行时、段指标缓存、emoji 修正与引擎 profile shim。 |
| `package.json` | 提供 `benchmark-check:safari` 命令别名。 |

---

## 依赖与外部交互

### 5.1 内部依赖

- **测量层**：`src/measurement.ts` 的段指标缓存策略（`Map<font, Map<segment, metrics>>`）对 Safari 的慢速 `measureText` 尤为关键。缓存命中可显著缓解 Safari 的测量劣势。
- **分析层**：`src/analysis.ts` 中的脚本特定预处理（如阿拉伯语无空格标点聚类、CJK 处理）影响 `analysisSegments` 与 `analysisMs`。
- **仪表板生成**：`scripts/status-dashboard.ts` 将本文件与 `benchmarks/chrome.json` 一并索引，生成 `benchmarks.chrome` 与 `benchmarks.safari` 两个并列节点。

### 5.2 外部依赖

- **Safari / WebKit**：快照数值高度依赖当前 macOS 上安装的 Safari 版本及其底层 CoreText 实现。不同 macOS 版本（如 Sonoma vs Sequoia）可能带来显著波动。
- **AppleScript 自动化**：`browser-automation.ts` 依赖 macOS 的 `osascript` 与 Safari 的 JavaScript 注入能力，无法在 Linux/Windows 上运行。
- **字体栈**：与 Chrome 相同，部分语料指定了 macOS 专有字体（如 `Geeza Pro`、`Noto Nastaliq Urdu`），字体缺失会导致回退到通用字体，改变 `measureMs` 与 `lineCount`。

### 5.3 交互协议

- **单所有者锁**：`browser-automation.ts` 实现了基于文件锁的浏览器自动化锁，防止多个脚本实例同时操作 Safari。该锁具备“自修复”机制，可清理过期的死所有者文件。
- **前台要求**：与 accuracy/corpus 检查器不同，benchmark 运行明确要求 `foreground: true`（`benchmark-check.ts` 第 117 行），因为后台/节流标签页会严重扭曲性能数据。AGENTS.md 也强调“不要优化掉 benchmark 焦点”。

---

## 风险、边界与改进建议

### 6.1 已知风险

1. **Safari 测量瓶颈被误读为引擎退化**：由于 Safari 的 `measureMs` 天然数倍于 Chrome，若开发者未对照 Safari 基线而仅看 Chrome 数据，可能在 Safari 上出现“假阳性”回归警报。
2. **AppleScript 自动化脆弱性**：Safari 的 UI 自动化比 Chrome CDP 慢且更易受系统弹窗、权限提示、焦点丢失影响，导致 `benchmark-check:safari` 偶尔超时或锁竞争。
3. **粒度截断**：Safari 的计时器分辨率较低，多个结果恰好为整数（如 18ms、3ms、5ms），降低了微小性能变化的敏感度。
4. **无 Firefox 对照**：缺少第三浏览器基准，无法判断某些差异是“Safari 慢”还是“Chrome 快”。

### 6.2 边界条件

- **DOM 基准的不可比性**：Safari 的 DOM batch（84.5ms）与 Chrome（3.7ms）差距过大，已超出“同一测试代码”的纯引擎差异，涉及 WebKit 的合成器、布局引擎及 GPU 路径差异。该数字更适合作为“库在 Safari 上优势更大”的定性证据，而非精确的跨浏览器 DOM 成本对标。
- **长可断行合成的极端值**：`synthetic-long-breakable-runs` 在 Safari 上 `measureMs` 高达 213ms，是 Chrome 的 21 倍。该语料由大量 URL、缓存键、时间范围等长单元组成，触发了 WebKit 测量路径的最坏情况，但真实应用中的文本分布通常不会如此极端。

### 6.3 改进建议

1. **为 Safari 引入运行时校准**：AGENTS.md 的 Open Questions 已提到“line-fit tolerance 是否应移至运行时校准”。同理，可考虑在 Safari 上运行时检测 `measureText` 的基准延迟，动态调整缓存策略或测量批大小。
2. **拆分 `measureMs` 黑盒**：在 `profilePrepare` 中增加 `canvasMeasureMs` 与 `segmenterMs` 两个子字段，明确 Safari 的 213ms 是否全部花在 `measureText`，还是也包含 `Intl.Segmenter` 的 shaping 开销。
3. **增加合成语料的缓存命中率测试**：在 `pages/benchmark.ts` 中增加一轮“warm cache”测量，验证 `src/measurement.ts` 的 `Map` 缓存是否能在 Safari 上有效摊平 213ms 的首次测量成本。
4. **考虑 Safari 专用测量降级策略**：若未来发现 Safari 的 `measureText` 在特定脚本（Nastaliq、Devanagari）下存在已知的 O(n) 或更差行为，可在 `src/measurement.ts` 中针对 WebKit 引入更激进的子串合并或批测量策略。
5. **记录 Safari 版本与 macOS 版本**：在 `BenchmarkReport` 中增加 `userAgent` 或 `platform` 字段，使历史快照具备版本可追溯性，避免“数值变化源于系统升级”的困惑。
6. **优先使用 Chrome 进行 font-matrix 初筛**：AGENTS.md 已建议“Prefer Chrome for the first font-matrix pass. Safari font-matrix automation is slower and noisier”。该原则同样适用于 benchmark 初筛：任何新的基准假设应先在 Chrome 验证，再作为 Safari 跟进 smoke test。
