# 研究报告：scripts/probe-check.ts

## 场景与职责

`scripts/probe-check.ts` 是 Pretext 项目中最细粒度的“单文本浏览器神谕”诊断脚本。与批量扫描脚本（如 `accuracy-check.ts`、`corpus-sweep.ts`）不同，它针对**一段用户指定的文本**、在**单一宽度**下，将 Pretext 的预测结果与真实浏览器（Chrome 或 Safari）的 DOM 排版进行逐行对比，输出高度、行数、首行差异（`firstBreakMismatch`）以及提取器敏感度（`extractorSensitivity`）等详细信息。它是开发者调试特定排版分歧时的首选工具，对应 `package.json` 中的 `probe-check` 和 `probe-check:safari` 命令。

## 功能点目的

1. **单文本精确诊断**：当批量扫描发现某个宽度或某段文本存在差异时，使用 `probe-check` 进行显微镜式分析，定位首次断行分歧的具体偏移和原因猜测。
2. **提取器方法交叉验证**：支持通过 `--method=span` 或 `--method=range` 切换浏览器行提取策略，并自动报告两种方法的结果差异（`extractorSensitivity`），帮助区分“算法 bug”与“DOM 提取器伪影”。
3. **白空间模式切换**：支持 `--whiteSpace=pre-wrap` 参数，使其也能诊断非默认的 pre-wrap 场景。
4. **快速反馈循环**：无需运行大规模浏览器扫描，仅需数秒即可给出单点对比结果，适合在算法迭代过程中反复调用。

## 具体技术实现

### 关键流程

脚本采用 Top-level await 结构：

1. **参数解析**：
   - `--text`（必填）：待测文本。
   - `--width`：容器宽度（默认 600）。
   - `--font`：CSS font 字符串（默认 `18px serif`）。
   - `--lineHeight`：行高（默认 32）。
   - `--dir`：`ltr` 或 `rtl`（默认 `ltr`）。
   - `--lang`：语言标签（RTL 默认 `ar`，否则 `en`）。
   - `--method`：`span` 或 `range`，控制浏览器行提取方式。
   - `--whiteSpace`：传入 `pre-wrap` 则启用 pre-wrap 模式，否则为 `normal`。
   - `--browser`：浏览器选择，默认从环境变量 `PROBE_CHECK_BROWSER` 读取，否则为 `chrome`。
   - `--port`：本地服务器端口，默认从 `PROBE_CHECK_PORT` 读取或随机分配。
2. **浏览器锁与会话**：
   - `acquireBrowserAutomationLock(browser)` 获取文件锁。
   - `createBrowserSession(browser)` 创建 AppleScript 驱动的浏览器会话。
3. **页面临时服务器**：
   - `ensurePageServer(port, '/probe', process.cwd())` 启动或复用本地 Bun 服务器。
4. **构造诊断 URL**：
   - 基础路径为 `/probe`，将所有参数编码进 query string，附加唯一 `requestId`。
5. **加载报告**：
   - `loadHashReport<ProbeReport>(session, url, requestId, browser)` 导航并轮询 URL hash 中的 JSON 报告。
6. **打印报告** (`printReport`)：
   - 输出 diff px、行数对比、浏览器提取方法。
   - 若存在 `alternateBrowserLineMethod`，输出备用方法及其行数对比。
   - 若存在 `firstBreakMismatch`，输出断行所在行号、原因猜测、双方偏移、文本、渲染文本、上下文以及多种宽度测量值（ours sum/dom/full vs browser dom/full）。
7. **清理**：`finally` 中关闭会话、杀服务器、释放锁。

### 数据结构

```typescript
type ProbeReport = {
  status: 'ready' | 'error'
  requestId?: string
  whiteSpace?: 'normal' | 'pre-wrap'
  browserLineMethod?: 'range' | 'span'
  width?: number
  predictedHeight?: number
  actualHeight?: number
  diffPx?: number
  predictedLineCount?: number
  browserLineCount?: number
  firstBreakMismatch?: {
    line: number
    oursStart: number
    browserStart: number
    oursEnd: number
    browserEnd: number
    oursText: string
    browserText: string
    oursRenderedText: string
    browserRenderedText: string
    oursContext: string
    browserContext: string
    deltaText: string
    reasonGuess: string
    oursSumWidth: number
    oursDomWidth: number
    oursFullWidth: number
    browserDomWidth: number
    browserFullWidth: number
  } | null
  alternateBrowserLineMethod?: 'range' | 'span'
  alternateBrowserLineCount?: number
  alternateFirstBreakMismatch?: object | null
  extractorSensitivity?: string | null
  message?: string
}
```

### 协议与命令

- **页面服务**：`bun --port=<port> --no-hmr pages/*.html`
- **浏览器控制**：AppleScript（Chrome/Safari）；Firefox 在类型层面被 `BrowserKind` 限制为 `chrome | safari`，但 `browser-automation.ts` 已具备 Firefox BiDi 支持，只是 `probe-check.ts` 的 `parseBrowser` 显式拒绝 `firefox`。
- **报告传输**：URL hash（`#report=...`），由 `pages/probe.ts` 中的 `publishNavigationReport` 写入，由 `browser-automation.ts` 的 `loadHashReport` 轮询读取。

## 关键代码路径与文件引用

- **脚本自身**：`scripts/probe-check.ts`（154 行）。
- **浏览器自动化**：`scripts/browser-automation.ts`（提供锁、会话、服务器、报告加载）。
- **诊断页面**：`pages/probe.html` + `pages/probe.ts`（核心对比逻辑，生成 `ProbeReport`）。
- **诊断工具**：`pages/diagnostic-utils.ts`（`getDiagnosticUnits`、`formatBreakContext`、`measureCanvasTextWidth`、`measureDomTextWidth` 等）。
- **导航状态**：`shared/navigation-state.ts` + `pages/report-utils.ts`（hash 报告协议）。
- **调用方**：`package.json` 中的 `probe-check`、`probe-check:safari` 命令。

## 依赖与外部交互

### 内部依赖

- `node:child_process`（`ChildProcess` 类型）
- `scripts/browser-automation.ts`
- `pages/probe.ts`（运行时页面逻辑）

### 外部依赖

- **Bun**：脚本通过 `bun run` 执行；底层自动化使用 `Bun.spawnSync`。
- **macOS + AppleScript**：Chrome/Safari 自动化依赖 `osascript`。
- **本地浏览器**：Chrome 和/或 Safari 必须已安装。
- **文件锁**：`$TMPDIR/pretext-browser-automation-locks/<browser>.lock`。

## 风险、边界与改进建议

### 风险与边界

1. **显式排除 Firefox**：`parseBrowser` 函数直接拒绝 `firefox`，导致无法使用已实现的 Firefox BiDi 自动化进行单点诊断；这与 `accuracy-check.ts` 支持 Firefox 形成能力落差。
2. **无批量模式**：虽然设计初衷就是单文本诊断，但在处理大量可疑样本时，需要外层脚本（如 shell loop）反复调用，每次都要重新获取浏览器锁、启动/复用服务器， overhead 较高。
3. **AppleScript 前台干扰**：`createBrowserSession` 默认不在前台运行（`foreground` 未传），但 Safari/Chrome 的 AppleScript 仍可能短暂窃取焦点，影响开发者正在进行其他操作。
4. **URL 长度限制**：`text` 参数通过 query string 传递，若文本极长（如整篇小说），可能触及浏览器或 shell 的 URL 长度上限；大型 corpus 诊断应使用 POST side channel（`report-server.ts` + `loadPostedReport`），而 `probe-check` 未走该路径。
5. **方法敏感度误报**：`extractorSensitivity` 提示“span mismatch disappears with range”时，开发者需要人工判断是提取器问题还是真实算法差异；脚本本身不自动做最终裁决。

### 改进建议

1. **开放 Firefox 支持**：在 `parseBrowser` 中接受 `firefox`，并验证 `pages/probe.ts` 在 Firefox headless 下的 DOM 测量稳定性，从而与 `accuracy-check.ts` 保持三浏览器覆盖一致。
2. **增加 `--output=json` 模式**：当前输出为人眼优化的多行文本。增加机器可读 JSON 输出，便于被其他脚本或 CI 解析，形成自动化诊断流水线。
3. **支持多宽度批量探测**：增加 `--widths=200,300,400` 参数，在单脚本生命周期内循环多个宽度，避免重复锁/服务器开销。
4. **长文本文件输入**：支持 `--text-file=path/to/sample.txt`，避免超长文本在命令行中转义困难。
5. **与 `pre-wrap-check` 对齐增强**：`pre-wrap-check.ts` 目前缺少 `firstBreakMismatch` 级细节；可考虑将 `probe-check.ts` 的报告打印逻辑提取为共享模块，供所有诊断脚本复用。
