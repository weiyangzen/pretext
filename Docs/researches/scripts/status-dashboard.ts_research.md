# 研究报告：scripts/status-dashboard.ts

## 场景与职责

`scripts/status-dashboard.ts` 是 Pretext 项目的**状态仪表盘聚合生成脚本**。它将分散在 `accuracy/`、`benchmarks/` 目录下的浏览器快照 JSON 文件读取、汇总、索引，最终输出为一个统一的机器可读仪表盘 `status/dashboard.json`。该文件是项目当前健康状况的“单一事实来源”（single source of truth）之一，供开发者、CI 和文档快速查阅各浏览器上的精度、性能数据。对应 `package.json` 中的 `status-dashboard` 命令，也是 DEVELOPMENT.md 推荐的常规维护命令。

## 功能点目的

1. **多源数据聚合**：将 `accuracy/chrome.json`、`accuracy/safari.json`、`accuracy/firefox.json` 以及 `benchmarks/chrome.json`、`benchmarks/safari.json` 合并为统一的 JSON 结构。
2. **索引化加速查询**：将扁平的 benchmark 结果数组转换为以 `label` 为键的对象索引，方便下游页面或工具按名称快速查找指标。
3. **来源可追溯性**：在输出中保留 `sources` 字段，记录各数据片段来自哪个文件路径，便于审计和跳转。
4. **生成时间戳**：写入 `generatedAt` ISO 时间戳，明确仪表盘数据的新鲜度。

## 具体技术实现

### 关键流程

脚本采用 Top-level await 的极简顺序执行：

1. **参数解析**：
   - `--output`：指定输出路径，默认 `status/dashboard.json`。
2. **数据加载**：
   - 使用 `Bun.file(path).json()` 并行（语法上顺序，但 `await` 可轻易改为并行）读取 5 个输入文件：
     - `accuracy/chrome.json`
     - `accuracy/safari.json`
     - `accuracy/firefox.json`
     - `benchmarks/chrome.json`
     - `benchmarks/safari.json`
3. **数据转换**：
   - **Accuracy**：通过 `summarizeAccuracy()` 提取 `total`、`matchCount`、`mismatchCount`。
   - **Benchmark**：通过 `indexResults()` 将 `results`、`richResults`、`richPreWrapResults`、`richLongResults` 四个数组分别转换为 `Record<label, { ms, desc }>` 索引；`corpusResults` 保持原始数组不变。
4. **组装仪表盘对象**：
   ```typescript
   const dashboard = {
     generatedAt: new Date().toISOString(),
     sources: { accuracy: {...}, benchmarks: {...}, corpora: 'corpora/dashboard.json' },
     browserAccuracy: { chrome: {...}, safari: {...}, firefox: {...} },
     benchmarks: { chrome: {...}, safari: {...} }
   }
   ```
5. **写入文件**：
   - 使用 `mkdirSync(dirname(output), { recursive: true })` 确保目录存在。
   - 使用 `writeFileSync` 写入格式化后的 JSON（2 空格缩进 + 尾随换行）。
6. **日志输出**：打印 `wrote <output>`。

### 数据结构

#### 输入类型

```typescript
type AccuracyReport = {
  total?: number
  matchCount?: number
  mismatchCount?: number
}

type BenchmarkResult = {
  label: string
  ms: number
  desc: string
}

type CorpusBenchmarkResult = {
  id: string
  label: string
  font: string
  chars: number
  analysisSegments: number
  segments: number
  breakableSegments: number
  width: number
  lineCount: number
  analysisMs: number
  measureMs: number
  prepareMs: number
  layoutMs: number
}

type BenchmarkReport = {
  results?: BenchmarkResult[]
  richResults?: BenchmarkResult[]
  richPreWrapResults?: BenchmarkResult[]
  richLongResults?: BenchmarkResult[]
  corpusResults?: CorpusBenchmarkResult[]
}
```

#### 输出结构（`status/dashboard.json`）

- `generatedAt`: ISO 字符串
- `sources`: 各输入文件路径映射
- `browserAccuracy`: 三浏览器的 `{ total, matchCount, mismatchCount }`
- `benchmarks.chrome` / `benchmarks.safari`:
  - `topLevel`: 基础 benchmark 索引
  - `richShared`: `layoutWithLines` / `walkLineRanges` / `layoutNextLine` 共享语料 benchmark 索引
  - `richPreWrap`: pre-wrap 模式 benchmark 索引
  - `richLong`: 长文本 Arabic 语料 benchmark 索引
  - `longFormCorpusStress`: 长语料逐条性能数据数组

### 命令与协议

- 无网络交互，纯本地文件 IO。
- 使用 `Bun.file().json()` 读取输入，使用 `node:fs` 的同步 API 写入输出。

## 关键代码路径与文件引用

- **脚本自身**：`scripts/status-dashboard.ts`（115 行）。
- **输入数据文件**：
  - `accuracy/chrome.json`
  - `accuracy/safari.json`
  - `accuracy/firefox.json`
  - `benchmarks/chrome.json`
  - `benchmarks/safari.json`
- **输出文件**：`status/dashboard.json`（默认，可通过 `--output` 覆盖）。
- **关联脚本**：`scripts/corpus-status.ts`（生成 `corpora/dashboard.json`，`status-dashboard.ts` 仅引用其路径而不消费其内容）。
- **调用方**：`package.json` 中的 `"status-dashboard": "bun run scripts/status-dashboard.ts"`；DEVELOPMENT.md 中的工作流说明。

## 依赖与外部交互

### 内部依赖

- `node:fs`（`mkdirSync`、`writeFileSync`）
- `node:path`（`dirname`）
- `Bun.file().json()`（Bun 运行时 API）

### 外部依赖

- **Bun**：脚本使用 `Bun.file(path).json()` 读取 JSON，无法在原生 Node.js 下直接运行。
- **上游快照生成脚本**：`accuracy-check.ts`、`benchmark-check.ts` 等负责生成输入 JSON；`status-dashboard.ts` 本身不验证这些文件是否存在或是否最新。
- **corpus 仪表盘**：`corpora/dashboard.json` 由 `corpus-status.ts` 维护，`status-dashboard.ts` 只在 `sources.corpora` 中记录其路径，不读取其内容。

## 风险、边界与改进建议

### 风险与边界

1. **Bun 锁定**：`Bun.file().json()` 是 Bun 专属 API，限制了脚本在纯 Node.js CI 环境中的可移植性。
2. **输入文件缺失即崩溃**：若任一输入文件不存在或格式非法，`Bun.file().json()` 会抛出异常，脚本无容错或降级逻辑。
3. **无内容校验**：脚本假设输入 JSON 的结构与预期类型完全一致，不会检查 `accuracy` 中 `total` 是否为负数、`benchmark` 中 `ms` 是否非数字等。
4. **corpus 数据未聚合**：`corpora/dashboard.json` 的内容未被读入，导致 `status/dashboard.json` 不包含语料级别的长期趋势统计；若用户期望在统一仪表盘查看 corpus 健康度，需要额外打开 `corpora/dashboard.json`。
5. **时间戳无 TZ 信息**：`new Date().toISOString()` 生成 UTC 时间，对本地开发者直观性稍弱。
6. **无增量/缓存机制**：每次运行都全量重写输出文件，数据量小（<1 MB）时无性能问题，但未来若扩展更多指标，可考虑增量更新。

### 改进建议

1. **兼容 Node.js 读取 JSON**：将 `Bun.file(path).json()` 替换为 `node:fs/promises` 的 `readFile(path, 'utf8').then(JSON.parse)`，使脚本在 Node.js 和 Bun 双环境下可运行。
2. **输入文件存在性检查**：在加载前使用 `fs.existsSync` 或 `fs.access` 检查文件是否存在，缺失时打印友好错误并跳过该部分（而非直接崩溃）。
3. **结构校验**：引入轻量级运行时校验（如 `zod` 或手写断言），确保 `accuracy` 和 `benchmark` 字段类型正确，防止上游脚本输出格式漂移导致下游消费方异常。
4. **内联 corpus 仪表盘摘要**：读取 `corpora/dashboard.json` 并将其关键摘要（如各语料的总行数、最近更新时间）嵌入 `status/dashboard.json`，实现真正的“单一仪表盘”。
5. **增加 `--watch` 模式**：在开发或持续集成场景中，监听 `accuracy/` 和 `benchmarks/` 文件变化并自动重生成仪表盘，减少手动运行频率。
6. **输出校验与 diff**：生成后打印与旧版本的 diff 摘要（如新增/删除的 benchmark 标签、accuracy 变化），帮助开发者快速发现异常波动。
