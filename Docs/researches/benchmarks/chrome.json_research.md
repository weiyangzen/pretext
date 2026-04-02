# benchmarks/chrome.json 研究文档

## 场景与职责

`benchmarks/chrome.json` 是 Pretext 项目在 **Google Chrome** 浏览器下的性能基准快照（checked-in benchmark snapshot）。它作为项目持续集成与状态监控的核心数据源之一，承担着以下职责：

1. **性能基线锚定**：记录当前文本布局引擎在 Chrome 中的关键路径耗时，为后续回归检测提供可比对的基准。
2. **库 vs DOM 对比验证**：量化 `prepare()` / `layout()` 相对于原生 DOM 批量/交错读写布局的性能优势。
3. **富 API 路径监控**：跟踪 `layoutWithLines()`、`walkLineRanges()`、`layoutNextLine()` 等高级 API 在共享语料、pre-wrap 块压力、阿拉伯长文本压力三种场景下的表现。
4. **长形式语料库应力测试**：覆盖 13 种真实多语言语料（CJK、阿拉伯语、乌尔都语、泰语、缅甸语、高棉语、印地语等），暴露脚本特定的 `prepare()` 回归或长可断行测量的性能问题。

该文件与 `benchmarks/safari.json` 共同构成跨浏览器性能视图，并被 `status/dashboard.json` 消费以生成机器可读的项目健康仪表板。

---

## 功能点目的

快照内包含 5 个功能分组，各自对应不同的工程关切：

| 分组 | 字段名 | 目的 |
|------|--------|------|
| 顶层批处理 | `results` | 测量 500 文本批次的 `prepare()` 冷启动成本与 `layout()` 热路径吞吐，并与 DOM batch/interleaved 做数量级对比。 |
| 富行 API（共享语料） | `richResults` | 在 60 条共享语料、180/220/260px 多宽度下，比较三种富 API 的延迟差异，验证“无字符串物化”的 `walkLineRanges()` 是否最快。 |
| 富行 API（pre-wrap 压力） | `richPreWrapResults` | 使用 12 条生成的 pre-wrap 文本（含 320 个硬断块、tab、空行、尾部空格），专门监控 `{ whiteSpace: 'pre-wrap' }` 模式下的富 API 开销。 |
| 富行 API（阿拉伯长文本压力） | `richLongResults` | 8 份阿拉伯长语料在 240/300/360px 下的最坏情况路径，用于捕获 RTL/复杂 shaping 场景的富 API 退化。 |
| 长形式语料库应力 | `corpusResults` | 逐语料拆分 `analysisMs`（文本分析）、`measureMs`（canvas 测量）、`prepareMs`（总冷启动）、`layoutMs`（热布局），定位脚本特定的瓶颈。 |

---

## 具体技术实现

### 3.1 数据生成流程

数据并非手工维护，而是通过浏览器自动化流水线生成：

1. **入口页面执行**：`pages/benchmark.ts` 作为浏览器页面加载，直接在真实 Chrome 渲染进程中运行性能测试。
2. **自动化收集**：`scripts/benchmark-check.ts` 调用 `scripts/browser-automation.ts` 中的 CDP（Chrome DevTools Protocol）会话，启动临时 Bun 页面服务器（`ensurePageServer`），导航至 `http://localhost:<port>/benchmark?report=1&requestId=...`。
3. **报告传输**：页面通过 `pages/report-utils.ts` 将完成的 `BenchmarkReport` 序列化后写入 URL hash（基于 `shared/navigation-state.ts` 的协议）。自动化脚本轮询 hash，解析出 JSON 报告。
4. **快照写入**：当带 `--output=benchmarks/chrome.json` 运行时，`benchmark-check.ts` 将报告格式化写入本文件。

### 3.2 关键测试参数（pages/benchmark.ts）

```ts
const COUNT = 500                    // 顶层批处理文本数
const WARMUP = 2                     // 预热轮数
const RUNS = 10                      // 实测轮数
const LAYOUT_SAMPLE_REPEATS = 200    // layout() 内部重复以稳定亚毫秒计时
const LAYOUT_SAMPLE_WIDTHS = [200, 250, 300, 350, 400]
const RICH_COUNT = 60                // 富 API 共享语料数
const RICH_LAYOUT_SAMPLE_REPEATS = 40
const RICH_LAYOUT_SAMPLE_WIDTHS = [180, 220, 260]
const CORPUS_WARMUP = 1
const CORPUS_RUNS = 7
const CORPUS_LAYOUT_SAMPLE_REPEATS = 200
```

所有计时均取 **中位数（median）**，以消除异常值影响。`bench()` 函数在每次运行前执行指定轮数的 warmup。

### 3.3 核心数据结构

```ts
type BenchmarkReport = {
  status: 'ready' | 'error'
  requestId?: string
  results?: BenchmarkResult[]           // 顶层批处理
  richResults?: BenchmarkResult[]       // 共享语料富 API
  richPreWrapResults?: BenchmarkResult[]// pre-wrap 压力
  richLongResults?: BenchmarkResult[]   // 阿拉伯长文本压力
  corpusResults?: CorpusBenchmarkResult[] // 长形式语料库
}

type BenchmarkResult = {
  label: string   // 如 "Our library: layout()"
  ms: number      // 中位耗时（毫秒）
  desc: string    // 人类可读的场景描述
}

type CorpusBenchmarkResult = {
  id: string
  label: string
  font: string
  chars: number
  analysisSegments: number   // 分析阶段产出的段数
  segments: number           // 最终 prepared 段数
  breakableSegments: number  // 可断行段数
  width: number
  lineCount: number
  analysisMs: number
  measureMs: number
  prepareMs: number
  layoutMs: number
}
```

### 3.4 语料库来源

`corpusResults` 中的文本直接通过 `with { type: 'text' }` 导入 `corpora/*.txt` 真实语料，以及一条合成数据：

- `ja-kumo-no-ito.txt`, `ja-rashomon.txt`（日语）
- `ko-unsu-joh-eun-nal.txt`（韩语）
- `zh-zhufu.txt`, `zh-guxiang.txt`（中文）
- `th-nithan-vetal-story-1.txt`（泰语）
- `my-cunning-heron-teacher.txt`, `my-bad-deeds-return-to-you-teacher.txt`（缅甸语）
- `ur-chughd.txt`（乌尔都语）
- `km-prachum-reuang-preng-khmer-volume-7-stories-1-10.txt`（高棉语）
- `hi-eidgah.txt`（印地语）
- `ar-risalat-al-ghufran-part-1.txt`（阿拉伯语）
- `synthetic-long-breakable-runs`：由 `buildLongBreakableStressText()` 生成，包含 URL、缓存键、时间范围等长可断行单元

---

## 关键代码路径与文件引用

| 文件 | 角色 |
|------|------|
| `pages/benchmark.ts` | 基准测试页面的核心实现；直接调用 `src/layout.ts` 的 API 进行计时与对比。 |
| `scripts/benchmark-check.ts` | Chrome/Safari 基准自动化入口；负责启停浏览器会话、临时服务器、收集报告、写入 JSON。 |
| `scripts/browser-automation.ts` | 底层浏览器控制（CDP for Chrome，AppleScript/OSA for Safari）与页面服务器生命周期管理。 |
| `pages/report-utils.ts` | 页面端报告发布工具，通过 `history.replaceState` 将报告编码进 URL hash。 |
| `shared/navigation-state.ts` | 定义 hash 编码/解码协议，供页面与自动化脚本共享。 |
| `scripts/status-dashboard.ts` | 读取 `benchmarks/chrome.json` 与 `benchmarks/safari.json`，聚合生成 `status/dashboard.json`。 |
| `src/layout.ts` | 被测库的核心 API：`prepare`, `prepareWithSegments`, `layout`, `layoutWithLines`, `walkLineRanges`, `layoutNextLine`, `clearCache`, `profilePrepare`。 |
| `src/test-data.ts` | 提供 `TEXTS` 共享语料，用于 500 文本顶层批处理。 |
| `package.json` | 定义 `benchmark-check` / `benchmark-check:safari` 等命令别名。 |

---

## 依赖与外部交互

### 5.1 内部依赖

- **引擎热路径**：任何对 `src/analysis.ts`、`src/measurement.ts`、`src/line-break.ts`、`src/layout.ts`、`src/bidi.ts` 的修改都会直接影响本快照数值。AGENTS.md 明确要求：当这些文件或 `pages/benchmark.ts` 发生变更时，必须刷新本快照并重新生成 `status/dashboard.json`。
- **状态仪表板**：`scripts/status-dashboard.ts` 在 `bun run status-dashboard` 时读取本文件，将其内容转录到 `status/dashboard.json` 的 `benchmarks.chrome` 节点下。

### 5.2 外部依赖

- **Chrome 浏览器**：快照反映的是运行时刻的 Chrome 版本及其底层 canvas 2D 文本测量实现。不同版本的 Chrome 可能因 `measureText` 优化或 Intl.Segmenter 行为微调而导致数值波动。
- **操作系统字体**：语料库测试使用了平台特定字体回退栈（如 `"Hiragino Mincho ProN"`、`"Songti SC"`、`"Geeza Pro"` 等），字体可用性会直接影响 `measureMs` 与 `layoutMs`。
- **Bun 运行时**：`benchmark-check.ts` 与 `status-dashboard.ts` 均使用 Bun 特有的 API（如 `Bun.file().json()`）。

### 5.3 交互协议

- **Hash-based 报告传输**：页面与自动化脚本之间通过 URL hash 传递 JSON 报告。该协议轻量、无 CORS 问题，但受 URL 长度限制（约 2KB~32KB 因浏览器而异）。对于 benchmark 数据（当前约 8.7KB）而言安全，但若未来增加更多语料或宽度采样，需警惕逼近上限。

---

## 风险、边界与改进建议

### 6.1 已知风险

1. **平台/版本敏感**：Chrome 的 canvas 文本测量在不同版本间可能存在优化或回归，导致同一 commit 在不同时刻生成的快照不可比。
2. **字体差异**：部分语料指定了 macOS 优先字体（如 `Hiragino Mincho ProN`、`Geeza Pro`），在 Linux/Windows CI 上运行会产生显著不同的 `measureMs`，因此当前流程主要在开发者本地 macOS 环境执行。
3. **无 Firefox 基准**：项目仅有 `chrome.json` 与 `safari.json`，缺少 Firefox 的 benchmark 快照，无法形成完整的三浏览器性能矩阵。
4. **Hash 传输上限**：虽然当前 8.7KB 安全，但 `corpusResults` 若持续膨胀，未来可能触及 URL hash 长度限制，届时需迁移到 `POST` 侧通道（accuracy/corpus 检查器已部分采用）。

### 6.2 边界条件

- **亚毫秒计时稳定性**：`layout()` 热路径在 Chrome 上仅约 0.16ms/500 文本批次，已接近 `performance.now()` 的分辨率边界。代码通过 `LAYOUT_SAMPLE_REPEATS = 200` 与多宽度循环来放大信号，但在更低延迟硬件上可能仍出现 `<0.01` 的截断显示。
- **DOM 基准的相对性**：DOM batch（3.7ms）与 DOM interleaved（41.7ms）作为参照系，受浏览器合成策略、GPU 加速、容器是否可见等因素影响。页面使用了 `position:relative` 与 `overflow:hidden;height:1px` 的可见容器，尽量贴近真实布局成本，但仍是简化模型。

### 6.3 改进建议

1. **引入 Firefox 基准快照**：扩展 `benchmark-check.ts` 支持 `firefox` 浏览器，并在 `benchmarks/` 下增加 `firefox.json`，与 accuracy 的三浏览器覆盖对齐。
2. **Hash → POST 迁移**：当 `corpusResults` 超过 10 条语料或增加更多元数据时，改用本地 POST 侧通道回传报告，避免 URL 长度天花板。
3. **增加测量阶段分解**：`corpusResults` 中的 `measureMs` 是黑盒，建议未来在 `profilePrepare` 中进一步拆分 `Intl.Segmenter` 耗时 vs `canvas.measureText` 耗时，以便更精准定位 Safari/Chrome 差异根因。
4. **CI 可复现性**：考虑使用 Docker + 固定字体包的方式在 Linux 上运行合成语料（`synthetic-long-breakable-runs`）子集，至少保证纯 Latin 路径的基准具备 CI 可复现性。
5. **版本注释**：在 JSON 中增加 `chromeVersion` 或 `platform` 字段，使历史快照具备更强的可追溯性。
