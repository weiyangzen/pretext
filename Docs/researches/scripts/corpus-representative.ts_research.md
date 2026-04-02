# corpus-representative.ts 研究文档

## 场景与职责

`scripts/corpus-representative.ts` 是 Pretext 项目的**语料代表性快照生成脚本**。它按固定的语料 ID 列表与固定宽度锚点（300/600/800），在 Chrome 和 Safari 中批量运行 `/corpus` 页面，收集每个语料的高度预测差异与行数，最终生成 `corpora/representative.json`——这是 corpus dashboard 和长期回归监测的核心检入基准。

## 功能点目的

1. **生成可检入的代表性快照**：为每个浏览器输出结构化的 `CorpusRepresentativeSnapshot`，包含环境指纹和所有语料在所有锚点宽度下的结果行。
2. **固定锚点回归监测**：宽度锚点 `[300, 600, 800]` 是长期跟踪的“金丝雀”点，任何引擎改动导致这些点漂移都会被快照差异暴露。
3. **跨浏览器覆盖**：默认遍历 `chrome` 和 `safari`（通过 `--browser=all` 或单独指定）。
4. **自动化流水线集成**：`package.json` 中的 `"corpus-status:refresh"` 脚本链将其作为第一步，后续接 `corpus-sweep` 和 `corpus-status`。

## 具体技术实现

### 关键数据结构

```ts
type RepresentativeRow = {
  corpusId: string; title: string; language: string; direction: string;
  width: number; contentWidth: number; font: string; lineHeight: number;
  predictedHeight: number; actualHeight: number; diffPx: number;
  predictedLineCount: number; browserLineCount: number;
};

type BrowserSnapshot = {
  environment: { userAgent; devicePixelRatio; viewport; screen };
  rows: RepresentativeRow[];
};

type CorpusRepresentativeSnapshot = {
  generatedAt: string; corpora: string[]; widths: number[];
  browsers: Partial<Record<BrowserKind, BrowserSnapshot>>;
};
```

### 固定常量

```ts
const CORPUS_IDS = [
  'mixed-app-text', 'en-gatsby-opening', 'ja-kumo-no-ito', 'ja-rashomon',
  'zh-guxiang', 'zh-zhufu', 'th-nithan-vetal-story-1',
  'my-cunning-heron-teacher', 'my-bad-deeds-return-to-you-teacher', 'ur-chughd',
] as const;
const WIDTHS = [300, 600, 800] as const;
```

### 核心流程

1. **参数解析**
   - `--browser=`：`all`（默认）、`chrome`、`safari`。
   - `--port=` / `CORPUS_CHECK_PORT`。
   - `--timeout=` / `CORPUS_CHECK_TIMEOUT_MS`（默认 180s）。
   - `--output=`：默认 `corpora/representative.json`。

2. **页服务启动**
   - `ensurePageServer(port, '/corpus', cwd)` 启动一个共享的临时 Bun server。注意：页服务进程在所有浏览器间复用，不在每个浏览器循环中重启。

3. **逐浏览器采集**
   对 `browsers` 数组中的每个浏览器：
   - `acquireBrowserAutomationLock(browser)` 获取锁。
   - `createBrowserSession(browser)` 创建会话。
   - 初始化 `rows: RepresentativeRow[]` 和 `environment` 引用。

4. **逐语料批量测量**
   对每个 `CORPUS_IDS` 中的语料：
   - 生成 `requestId`。
   - 启动 `startPostedReportServer<CorpusSweepReport>(requestId)`。
   - 构造 URL：
     ```
     /corpus?id=<corpusId>&widths=300,600,800&report=1&diagnostic=light&requestId=...&reportEndpoint=...
     ```
   - `loadPostedReport` 等待 POST 报告。
   - 校验报告状态，提取 `environment`（首次非空即保存），将 `report.rows` 映射为 `RepresentativeRow` 并追加到 `rows`。
   - 关闭该语料的 `reportServer`。

5. **快照组装与落盘**
   - 将 `environment` 和 `rows` 写入 `snapshot.browsers[browser]`。
   - 所有浏览器完成后，`mkdirSync` 确保目录存在，`writeFileSync` 输出 JSON。

6. **清理**
   - 每个浏览器的 `finally` 中关闭会话并释放锁。
   - 最外层 `finally` 中 `serverProcess?.kill()`。

## 关键代码路径与文件引用

- 本文件：`scripts/corpus-representative.ts`
- 浏览器自动化基座：`scripts/browser-automation.ts`
- POST 报告服务器：`scripts/report-server.ts`
- 被测页面：`pages/corpus.ts`
- 语料元数据：`corpora/sources.json`（用于 `toRepresentativeRow` 中的 title/language/direction/font/lineHeight 校验，但文本内容实际由 `pages/corpus.ts` 的硬编码 `loadText` 映射导入）
- 输出基准：`corpora/representative.json`
- 调用链：`package.json` 中 `"corpus-representative"` 和 `"corpus-status:refresh"`

## 依赖与外部交互

- **Node/Bun 运行时**：`node:fs`, `node:path`, `node:child_process`。
- **本地浏览器**：Chrome / Safari（AppleScript）。
- **环境变量**：`CORPUS_CHECK_PORT`, `CORPUS_CHECK_TIMEOUT_MS`。
- **上游页面**：`pages/corpus.ts` 的 `runSweep` 批量测量与 `toSweepRow` 数据转换。

## 风险、边界与改进建议

1. **语料列表硬编码**：`CORPUS_IDS` 与 `pages/corpus.ts` 中的 `loadText` switch-case、`corpora/sources.json` 需要保持同步。若新增语料但未加入 `CORPUS_IDS`，代表性快照不会覆盖它，可能导致回归漏检。建议从 `sources.json` 动态过滤（如排除未稳定语料）或增加启动校验。
2. **诊断级别固定为 `light`**：`diagnostic=light` 不返回 `firstBreakMismatch` 等深度字段，因此 representative 快照只包含高度/行数差异，不含断点根因。这是设计上的折中（减少数据量与运行时间），但定位问题时仍需重新跑 `corpus-check --diagnose`。
3. **单页服务跨浏览器复用**：虽然高效，但若前一个浏览器退出后端口仍被系统 TIME_WAIT 持有，后一个浏览器启动时 `ensurePageServer` 可能误判为“已有服务”而返回 `process: null`，导致实际无服务。当前 `resolveBaseUrl` 会真实探测 HTTP 可达性，可降低该风险。
4. **无 Firefox 支持**：与多数 corpus 脚本一致，未支持 Firefox。若未来需要三浏览器 representative，需扩展 `--browser` 解析和 `CORPUS_IDS` 的测试覆盖。
5. **输出路径默认覆盖**：默认直接写 `corpora/representative.json`，无备份机制。建议在 `corpus-status:refresh` 链中先 git stash 或生成 `.bak`，防止误覆盖后无法恢复。
