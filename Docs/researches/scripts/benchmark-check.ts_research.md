# benchmark-check.ts 研究文档

## 场景与职责

`scripts/benchmark-check.ts` 是 Pretext 项目的**性能基准浏览器采集脚本**。它自动化启动浏览器访问 `/benchmark` 页面，收集 `prepare()`、`layout()`、富文本 API（`layoutWithLines` / `walkLineRanges` / `layoutNextLine`）以及 DOM 批量的耗时数据，输出中位数对比结果，并可生成 `benchmarks/*.json` 快照。

## 功能点目的

1. **性能回归监测**：定期验证核心 API（prepare/layout）与 DOM 读写相比的速度优势。
2. **多维度基准**：覆盖短文本批量、pre-wrap 块 stress、阿拉伯长文本 stress、多语言长文本语料 stress。
3. **浏览器快照**：支持 Chrome（默认）与 Safari，生成可检入的 JSON 性能快照。
4. **前台运行保障**： benchmark 对焦点敏感，脚本显式要求 `foreground: true` 以避免后台标签节流。

## 具体技术实现

### 关键数据结构

```ts
type BenchmarkResult = { label: string; ms: number; desc: string };

type CorpusBenchmarkResult = {
  id: string; label: string; font: string; chars: number;
  analysisSegments: number; segments: number; breakableSegments: number;
  width: number; lineCount: number;
  analysisMs: number; measureMs: number; prepareMs: number; layoutMs: number;
};

type BenchmarkReport = {
  status: 'ready' | 'error';
  requestId?: string;
  results?: BenchmarkResult[];
  richResults?: BenchmarkResult[];
  richPreWrapResults?: BenchmarkResult[];
  richLongResults?: BenchmarkResult[];
  corpusResults?: CorpusBenchmarkResult[];
  message?: string;
};
```

### 核心流程

1. **参数解析**
   - `--browser=` 或 `BENCHMARK_CHECK_BROWSER` 环境变量选择浏览器（仅支持 `chrome` / `safari`）。
   - `--port=` 或 `BENCHMARK_CHECK_PORT` 固定页服务端口。
   - `--output=` 指定 JSON 落盘路径。

2. **浏览器锁与会话**
   - `acquireBrowserAutomationLock(browser)` 获取单实例锁。
   - `createBrowserSession(browser, { foreground: true })` 创建前台会话，确保浏览器标签处于活跃状态，避免后台节流影响计时。

3. **页服务启动**
   - `ensurePageServer(port, '/benchmark', process.cwd())` 启动临时 Bun server 服务 `pages/benchmark.ts`。

4. **报告采集**
   - 构造 URL：`${baseUrl}/benchmark?report=1&requestId=xxx`
   - 使用 `loadHashReport<BenchmarkReport>` 轮询 URL hash 获取报告（benchmark 数据量适中，无需 POST side-channel）。

5. **结果输出**
   - `printReport` 按类别打印：
     - Top-level batch（prepare / layout / DOM batch / DOM interleaved）
     - Rich line APIs（shared / pre-wrap / Arabic long-form）
     - Long-form corpus stress（各语料的 analysis / measure / prepare / layout）
   - 若指定 `--output`，完整 JSON 落盘。
   - 报告状态为 `error` 时以退出码 `1` 结束进程。

### 协议与命令

- 页服务端：本地 HTTP，`/benchmark?report=1&requestId=xxx`
- 浏览器导航：AppleScript（Chrome/Safari）或 BiDi（Firefox，但本脚本不支持 Firefox benchmark）。
- 进程管理：`serverProcess?.kill()` 在 `finally` 中清理。

## 关键代码路径与文件引用

- 本文件：`scripts/benchmark-check.ts`
- 浏览器自动化基座：`scripts/browser-automation.ts`
- 被测页面：`pages/benchmark.ts`（执行全部性能测量与 DOM 对比）
- 核心库 API：`src/layout.ts`（`prepare`, `layout`, `layoutWithLines`, `walkLineRanges`, `layoutNextLine`, `profilePrepare`）
- 输出基准：`benchmarks/chrome.json`, `benchmarks/safari.json`

## 依赖与外部交互

- **Node/Bun 运行时**：`node:child_process`, `node:fs`。
- **本地浏览器**：Chrome / Safari（macOS AppleScript）。
- **环境变量**：`BENCHMARK_CHECK_BROWSER`, `BENCHMARK_CHECK_PORT`, `BENCHMARK_CHECK_TIMEOUT_MS`（通过 `loadHashReport` 的默认超时隐式使用）。
- **上游页面**：`pages/benchmark.ts` 中的 `performance.now()` 测量结果决定报告数值。

## 风险、边界与改进建议

1. **浏览器限制**：显式拒绝 Firefox（`parseBrowser` 中抛错），因为 benchmark 需要前台稳定计时，而 Firefox BiDi 会话当前未实现 `foreground` 语义。若未来需要 Firefox 基准，需为 BiDi 增加窗口焦点控制或接受后台数据。
2. **AppleScript 前台副作用**：`foreground: true` 会激活浏览器窗口，干扰开发者当前工作。建议在 CI/无人值守环境运行，或探索 macOS 的 `open -g` / headless Chrome 方案。
3. **页服务端口冲突**：`getAvailablePort` 随机分配；若系统临时端口耗尽会失败。可允许 `--port=0` 让操作系统分配，但当前实现是脚本层随机。
4. **报告仅走 hash**：benchmark 报告未使用 POST server，若未来增加更多 corpus 或采样点导致 JSON 体积膨胀，可能触及 URL hash 长度限制（约 2~8KB 因浏览器而异）。建议在数据量增长前迁移到 `loadPostedReport`。
5. **退出码单一**：仅在 `report.status === 'error'` 时设 `process.exitCode = 1`。若页服务启动失败抛异常，进程会以非零退出，但终端输出可能混有 AppleScript 错误与 Node 堆栈，排障成本高。可增加错误分类与重试逻辑。
