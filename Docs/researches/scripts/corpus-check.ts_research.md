# corpus-check.ts 研究文档

## 场景与职责

`scripts/corpus-check.ts` 是 Pretext 项目的**长文本语料单点/窄幅诊断脚本**。它针对 `corpora/sources.json` 中定义的某一语料（corpus），在真实浏览器中加载 `/corpus` 页面，对比 Pretext 预测高度与浏览器 DOM 实际高度，输出像素级差异、行数差异以及可选的逐行断点诊断。用于定位特定语料在特定宽度下的布局不一致根因。

## 功能点目的

1. **单语料窄幅诊断**：支持一次检测一个 corpus 的一个或多个宽度（命令行位置参数传入宽度列表）。
2. **高度与行数对账**：报告 `predictedHeight` / `actualHeight`、行数 `predictedLineCount` / `browserLineCount`。
3. **深度断点诊断**：`--diagnose` 开启 full diagnostic 模式，页面会返回 `firstBreakMismatch`、`maxLineWidthDrift`、`extractorSensitivity` 等字段，帮助判断是 edge-fit、shaping drift 还是 extractor 敏感性问题。
4. **覆盖参数覆盖**：支持 `--font=`、`--lineHeight=`、`--method=`（span/range）、`--sliceStart=` / `--sliceEnd=` 等覆盖，便于快速验证假设。
5. **环境指纹采集**：报告包含浏览器 DPR、viewport、screen 等环境信息，便于复现问题。

## 具体技术实现

### 关键数据结构

```ts
type CorpusMeta = {
  id: string; language: string; title: string;
  min_width?: number; max_width?: number; default_width?: number;
};

type CorpusOverrideOptions = {
  font: string | null; lineHeight: number | null;
  method: 'span' | 'range' | null;
  sliceStart: number | null; sliceEnd: number | null;
};

type CorpusReport = {
  status: 'ready' | 'error';
  environment?: { userAgent; devicePixelRatio; viewport; screen };
  corpusId?: string; sliceStart?: number | null; sliceEnd?: number | null;
  title?: string; width?: number;
  predictedHeight?: number; actualHeight?: number; diffPx?: number;
  predictedLineCount?: number; browserLineCount?: number;
  browserLineMethod?: 'span-probe' | 'range';
  alternateBrowserLineMethod?: 'span-probe' | 'range';
  alternateBrowserLineCount?: number;
  probeHeight?: number; normalizedHeight?: number;
  mismatchCount?: number;
  firstMismatch?: { line: number; ours: string; browser: string } | null;
  firstBreakMismatch?: { line: number; oursStart; oursEnd; browserStart; browserEnd; oursText; browserText; deltaText; reasonGuess; oursSumWidth; oursDomWidth; oursFullWidth; browserDomWidth; browserFullWidth; oursSegments: [...] } | null;
  alternateFirstBreakMismatch?: object | null;
  extractorSensitivity?: string | null;
  maxLineWidthDrift?: number;
  maxDriftLine?: { line; drift; text; sumWidth; fullWidth; domWidth; pairAdjustedWidth; segments } | null;
  message?: string;
};
```

### 核心流程

1. **参数解析**
   - `--id=`：指定 corpus ID（必需）。
   - `--browser=` / `CORPUS_CHECK_BROWSER`：选择 `chrome`（默认）或 `safari`。
   - `--port=` / `CORPUS_CHECK_PORT`：固定页服务端口。
   - `--timeout=` / `CORPUS_CHECK_TIMEOUT_MS`：默认 180 秒。
   - `--diagnose`：在 URL 中附加 `diagnostic=full`（否则 `diagnostic=light`）。
   - `--font=`, `--lineHeight=`, `--method=span|rang`, `--sliceStart=`, `--sliceEnd=`：覆盖参数。
   - 位置参数：宽度列表；若未提供，使用 corpus meta 的 `min_width` / `default_width` / `max_width` 三个锚点。

2. **服务与浏览器启动**
   - `ensurePageServer(port, '/corpus', cwd)` 启动临时页服务。
   - `acquireBrowserAutomationLock(browser)` + `createBrowserSession(browser)` 获取锁并创建会话。

3. **逐宽度测量**
   对每个目标宽度：
   - 生成唯一 `requestId`。
   - 构造 URL：
     ```
     /corpus?id=<corpusId>&width=<w>&report=1&diagnostic=<full|light>&requestId=...
     ```
     并追加覆盖参数。
   - `loadHashReport<CorpusReport>(session, url, requestId, browser, timeoutMs)` 拉取报告。
   - `printReport(report)`：终端打印 diff、高度、行数、环境、drift、break mismatch 等。
   - 若报告状态为 `error`，设 `process.exitCode = 1` 并中断后续宽度。

4. **清理**
   - `finally` 中关闭会话、杀掉页服务进程、释放锁。

### 协议与命令

- 页服务端：本地 HTTP，`/corpus?id=...&width=...&report=1&diagnostic=...&requestId=...`
- 诊断模式：依赖 `pages/corpus.ts` 中的 `addDiagnostics` 函数，使用 `getBrowserLinesFromSpans` 与 `getBrowserLinesFromRange` 双路提取浏览器行数据。
- 进程管理：`spawn` 启动 Bun server；`SIGTERM` 结束。

## 关键代码路径与文件引用

- 本文件：`scripts/corpus-check.ts`
- 浏览器自动化基座：`scripts/browser-automation.ts`
- 被测页面：`pages/corpus.ts`（执行实际 DOM 高度测量与断点诊断）
- 诊断工具：`pages/diagnostic-utils.ts`（`getDiagnosticUnits`, `formatBreakContext`, `measureCanvasTextWidth`, `measureDomTextWidth`）
- 语料元数据：`corpora/sources.json`
- 调用别名：`scripts/gatsby-check.ts`（固定 `--id=en-gatsby-opening --diagnose`）

## 依赖与外部交互

- **Node/Bun 运行时**：`node:child_process`, `node:fs`（仅 `mkdirSync/writeFileSync` 在 `--output` 时用到，但本脚本未暴露 `--output`，保留类型定义）。
- **本地浏览器**：Chrome / Safari（AppleScript）。
- **环境变量**：`CORPUS_CHECK_BROWSER`, `CORPUS_CHECK_PORT`, `CORPUS_CHECK_TIMEOUT_MS`。
- **上游页面**：`pages/corpus.ts` 的 `measureWidth` 与 `addDiagnostics`。

## 风险、边界与改进建议

1. **无 `--output` 暴露**：代码中解析了 `--output` 但未在文档或 CLI 中强调，且 `printReport` 不输出 JSON。若需要落盘，需显式加 `--output=` 参数，但 `package.json` 未提供对应示例。建议统一暴露或移除未使用字段。
2. **宽度默认值可能重复**：`getTargetWidths` 使用 `[min, default, max]`，若三者相同会去重；但若 `default` 未定义，可能得到 `[300, 300, 900]` 的去重后 `[300, 900]`，行为合理但文档未说明。
3. **诊断模式对 Safari 的干扰**：`diagnostic=full` 会触发 span-probe 与 range 双路提取。Safari 的 `Range` 在 URL query 文本上可能 over-advance（AGENTS.md 已记录），脚本层无法干预，但诊断报告中的 `extractorSensitivity` 字段可帮助识别 extractor 导致的伪阴性。
4. **slice 覆盖的边界**：`--sliceStart` / `--sliceEnd` 在 `pages/corpus.ts` 中作用于 normalized text，若切片点落在 grapheme 中间可能导致意外断行。建议增加 grapheme 边界校验。
5. **超时硬编码**：180 秒对单宽度通常足够，但若 `--diagnose` + 超长语料（如 `ar-al-bukhala` 25 万字符）可能超时。建议按文本长度或诊断级别动态调整超时，或允许命令行覆盖。
