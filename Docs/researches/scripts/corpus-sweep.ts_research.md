# corpus-sweep.ts 研究文档

## 场景与职责

`scripts/corpus-sweep.ts` 是 Pretext 项目的**语料粗粒度宽度扫测脚本**。它对单个或全部语料在指定宽度区间内进行批量高度对比，快速定位哪些宽度存在布局不一致。支持线性步进（`--step`）或均匀采样（`--samples`），并可对不匹配宽度自动调用 `corpus-check.ts --diagnose` 进行深度诊断。

## 功能点目的

1. **快速宽度扫测**：以 `step=10` 或 `samples=9` 等策略覆盖 300~900px 区间，批量获取每个宽度的 diffPx。
2. **全语料批量模式**：`--all` 开关可对 `corpora/sources.json` 中的所有语料一次性扫测。
3. **自动诊断联动**：`--diagnose` 模式下，对前 N 个（默认 6 个）不匹配宽度自动 `spawnSync` 调用 `scripts/corpus-check.ts`，输出逐行断点详情。
4. **快照落盘**：支持 `--output=` 将 `SweepSummary[]` 写入 JSON，作为 `corpora/chrome-sampled.json` 或 `corpora/chrome-step10.json` 的生成源。

## 具体技术实现

### 关键数据结构

```ts
type CorpusSweepRow = {
  width: number; contentWidth: number;
  predictedHeight: number; actualHeight: number; diffPx: number;
  predictedLineCount: number; browserLineCount: number;
};

type CorpusSweepReport = {
  status: 'ready' | 'error';
  requestId?: string; corpusId?: string; title?: string; language?: string;
  widthCount?: number; exactCount?: number; rows?: CorpusSweepRow[];
  message?: string;
};

type SweepMismatch = {
  width: number; diffPx: number;
  predictedHeight: number; actualHeight: number;
  predictedLineCount: number | null; browserLineCount: number | null;
};

type SweepSummary = {
  corpusId: string; language: string; title: string; browser: BrowserKind;
  start: number; end: number; step: number; samples: number | null;
  widthCount: number; exactCount: number; mismatches: SweepMismatch[];
};
```

### 核心流程

1. **参数解析**
   - `--id=` 或 `--all`：指定单个语料或全部语料（必需其一）。
   - `--start=`（默认 300）、`--end=`（默认 900）、`--step=`（默认 10）。
   - `--samples=`：均匀采样数；与 `step` 互斥（优先 `samples`）。
   - `--browser=` / `CORPUS_CHECK_BROWSER`：`chrome`（默认）或 `safari`。
   - `--port=` / `CORPUS_CHECK_PORT`、 `--timeout=` / `CORPUS_CHECK_TIMEOUT_MS`（默认 180s）。
   - `--font=`、`--lineHeight=`：覆盖页面默认字体与行高。
   - `--diagnose` + `--diagnose-limit=`（默认 6）：对不匹配宽度自动深度诊断。
   - `--output=`：JSON 落盘路径。

2. **宽度集合生成**
   - `getSweepWidths(meta, options)`：
     - 若 `samples` 非空，均匀采样并去重。
     - 否则按 `step` 生成等差数列，同时受 `meta.min_width` / `meta.max_width` 裁剪。

3. **服务与浏览器启动**
   - `ensurePageServer(port, '/corpus', cwd)` 启动共享页服务。
   - `acquireBrowserAutomationLock(browser)` + `createBrowserSession(browser)`。

4. **逐语料扫测**
   对每个目标语料：
   - 生成 `requestId`。
   - 构造 URL：
     ```
     /corpus?id=<corpusId>&widths=<w1,w2,...>&report=1&requestId=...
     ```
     并追加 `font` / `lineHeight` 覆盖。
   - 启动 `startPostedReportServer<CorpusSweepReport>(requestId)`。
   - `loadPostedReport` 等待 POST 报告。
   - 将 `report.rows` 转为 `SweepMismatch[]`（过滤 `Math.round(diffPx) !== 0`），计算 `exactCount`。
   - 组装 `SweepSummary` 并 `printSummary`（输出 `exactCount/widthCount exact | N nonzero` 及分桶信息）。
   - 若开启 `--diagnose`，调用 `runDetailedDiagnose(meta, mismatches, options)`：
     - 取前 `diagnoseLimit` 个 mismatch 宽度。
     - `spawnSync('bun', ['run', 'scripts/corpus-check.ts', '--id=', '--diagnose', ...widths], { env: { CORPUS_CHECK_BROWSER, CORPUS_CHECK_PORT, CORPUS_CHECK_TIMEOUT_MS } })`。
     - 将子进程 stdout/stderr 透传到当前进程；若子进程非 0 退出则抛错中断。

5. **结果落盘与清理**
   - 若 `--output` 指定，将所有 `summaries` 写入 JSON。
   - `finally` 中关闭会话、杀页服务、释放锁。

## 关键代码路径与文件引用

- 本文件：`scripts/corpus-sweep.ts`
- 浏览器自动化基座：`scripts/browser-automation.ts`
- POST 报告服务器：`scripts/report-server.ts`
- 深度诊断子进程：`scripts/corpus-check.ts`
- 被测页面：`pages/corpus.ts`（处理 `widths` 批量参数）
- 语料元数据：`corpora/sources.json`
- 输出基准：`corpora/chrome-sampled.json`（`--samples=9`）、`corpora/chrome-step10.json`（`--step=10`）
- 调用别名：`scripts/gatsby-sweep.ts`（固定 `--id=en-gatsby-opening`）

## 依赖与外部交互

- **Node/Bun 运行时**：`node:child_process`（`spawnSync` 用于诊断联动）、`node:fs`。
- **本地浏览器**：Chrome / Safari（AppleScript）。
- **环境变量**：`CORPUS_CHECK_BROWSER`, `CORPUS_CHECK_PORT`, `CORPUS_CHECK_TIMEOUT_MS`。
- **上游页面**：`pages/corpus.ts` 的 `runSweep` 批量测量逻辑。

## 风险、边界与改进建议

1. **诊断联动阻塞主流程**：`runDetailedDiagnose` 使用同步 `spawnSync`，会完全阻塞当前事件循环，且每个不匹配宽度串行执行 `corpus-check`。对于大面积漂移的语料，诊断阶段可能非常慢。建议改为异步 `spawn` 并限制并发数，或支持 `--diagnose-parallel=N`。
2. **子进程环境透传有限**：诊断子进程仅透传了 `CORPUS_CHECK_BROWSER/PORT/TIMEOUT_MS`，未透传 `--font` / `--lineHeight` 的覆盖到子进程环境。实际上 `corpus-check.ts` 支持 `--font=` 和 `--lineHeight=` 命令行参数，当前 `runDetailedDiagnose` 已正确追加这两个参数，行为正确。
3. **POST server 与页服务端口冲突**：`ensurePageServer` 和 `startPostedReportServer` 各自调用 `getAvailablePort()`。在端口极其紧张的环境，可能出现分配失败。建议增加端口分配失败重试。
4. **采样与 step 的裁剪逻辑**：`getSweepWidths` 先用 `meta.min_width` / `meta.max_width` 裁剪区间，再应用采样。若语料的 `min_width` 大于 `options.end`，结果为空数组，页面可能返回空报告。建议增加空宽度集合的前置校验并给出友好提示。
5. **`--all` 模式无进度持久化**：若 `--all` 扫测到第 N 个语料时崩溃，前面结果丢失。建议增加 `--checkpoint` 模式，每完成一个语料即追加写入临时 JSON，最后合并。
