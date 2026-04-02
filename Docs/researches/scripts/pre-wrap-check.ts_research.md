# 研究报告：scripts/pre-wrap-check.ts

## 场景与职责

`scripts/pre-wrap-check.ts` 是 Pretext 项目中专门用于验证 `{ whiteSpace: 'pre-wrap' }` 模式与真实浏览器行为一致性的“浏览器神谕”扫描脚本。该脚本针对一组精心设计的预置测试用例，在自动化控制的浏览器（Chrome / Safari，Firefox 当前显式不支持）中打开诊断页面 `/probe`，将 Pretext 的预测结果与 DOM 实际排版结果进行逐行对比，最终输出差异报告。它是 `package.json` 中 `pre-wrap-check` 命令的实现，也是 AGENTS.md 中提到的“small permanent browser-oracle sweep for the non-default `{ whiteSpace: 'pre-wrap' }` mode”。

## 功能点目的

1. **pre-wrap 模式回归防护**：确保在 `white-space: pre-wrap` 下，空格、制表符（`\t`）、硬换行（`\n`）以及 CRLF 的保留与折叠行为与浏览器一致。
2. **边界场景覆盖**：测试悬挂空格（hanging spaces）、连续空格、前导空格、尾随空格、空行、仅空格行、制表位对齐、RTL 缩进、混合脚本缩进等边缘情况。
3. **跨浏览器一致性扫描**：支持在 Chrome 和 Safari 上批量运行，捕获引擎间差异。
4. **精确度断言**：不仅比较总高度，还比较行数（`predictedLineCount` vs `browserLineCount`），任何差异都会导致脚本以非零退出码失败。

## 具体技术实现

### 关键流程

脚本采用 Top-level await 顺序执行：

1. **参数解析**：
   - `--port`：指定本地页面临时服务器端口（默认随机）。
   - `--browser`：逗号分隔的浏览器列表（默认 `chrome,safari`）。
   - `--timeout`：单浏览器超时（默认 60,000 ms）。
2. **浏览器锁与会话**：对每个浏览器：
   - 调用 `acquireBrowserAutomationLock(browser)` 获取文件锁，防止并发自动化冲突。
   - 调用 `createBrowserSession(browser)` 创建 AppleScript（Chrome/Safari）或 BiDi（Firefox，但当前直接报错）会话。
3. **页面临时服务器**：
   - 调用 `ensurePageServer(port, '/probe', process.cwd())` 启动本地 Bun 服务器（`bun --port=<port> --no-hmr pages/*.html`），若已有可复用服务器则跳过。
4. **用例循环**：遍历硬编码的 `ORACLE_CASES`（15 个测试用例）：
   - 为每个用例生成唯一 `requestId`。
   - 调用 `buildProbeUrl()` 构造 `/probe` URL，强制附加 `whiteSpace=pre-wrap` 和 `method=span`。
   - 调用 `loadHashReport<ProbeReport>(session, url, requestId, browser, timeoutMs)` 导航页面并轮询 URL hash 中的报告。
   - `printCaseResult()` 输出单行结果（diff px、line count、height、extractor sensitivity）。
   - `reportIsExact()` 断言 `diffPx === 0` 且行数相等且高度相等；任一不匹配则该浏览器判定为失败。
5. **清理**：`finally` 块中关闭浏览器会话、杀掉临时服务器进程、释放锁。
6. **退出码**：若任一浏览器失败，设置 `process.exitCode = 1`。

### 数据结构

```typescript
type ProbeReport = {
  status: 'ready' | 'error'
  requestId?: string
  browserLineMethod?: 'range' | 'span'
  width?: number
  predictedHeight?: number
  actualHeight?: number
  diffPx?: number
  predictedLineCount?: number
  browserLineCount?: number
  extractorSensitivity?: string | null
  message?: string
}

type OracleCase = {
  label: string
  text: string
  width: number
  font: string
  lineHeight: number
  dir?: 'ltr' | 'rtl'
  lang?: string
}
```

`ORACLE_CASES` 包含 15 个用例，覆盖：
- 空格悬挂 / 保留空格段（`foo    bar` 在 126px 宽度下）
- 硬换行、双换行、尾随换行、CRLF 规范化
- 换行后的前导空格、换行前的空格与制表符
- 默认制表位与双制表位、换行后的制表符
- 混合脚本缩进（AGI + 中文 + 阿拉伯语 + emoji）
- RTL 缩进（阿拉伯语）

### 协议与命令

- **本地页面服务**：`bun --port=<port> --no-hmr pages/*.html`（由 `browser-automation.ts` 启动）。
- **浏览器导航**：AppleScript 控制 Safari / Chrome；Firefox 当前显式抛出 `Firefox is not currently supported for pre-wrap oracle checks`。
- **报告传输**：通过 URL hash（`#report=...`）传递 JSON 报告；`loadHashReport` 轮询 URL 读取报告。
- **CORS/POST**：`pre-wrap-check` 仅使用 hash 传输，不启用 POST side channel（与 corpus sweep 等大型报告不同）。

## 关键代码路径与文件引用

- **脚本自身**：`scripts/pre-wrap-check.ts`（267 行）。
- **浏览器自动化基础设施**：`scripts/browser-automation.ts`（提供 `acquireBrowserAutomationLock`、`createBrowserSession`、`ensurePageServer`、`loadHashReport` 等）。
- **诊断页面**：`pages/probe.html` + `pages/probe.ts`（浏览器端实际执行布局、测量、对比并生成 `ProbeReport`）。
- **导航状态工具**：`shared/navigation-state.ts`（定义 `NavigationPhase`，解析 URL hash 中的 phase 和 report）。
- **页面报告工具**：`pages/report-utils.ts`（浏览器端将报告写入 URL hash）。
- **调用方**：`package.json` 中的 `"pre-wrap-check": "bun run scripts/pre-wrap-check.ts"`。

## 依赖与外部交互

### 内部依赖

- `node:child_process`（`ChildProcess` 类型）
- `scripts/browser-automation.ts`
- `pages/probe.ts`（运行时依赖，作为被测页面逻辑）

### 外部依赖

- **Bun**：脚本通过 `bun run` 执行，且 `browser-automation.ts` 使用 `Bun.spawnSync` 启动页面临时服务器。
- **macOS + AppleScript**：Chrome 和 Safari 的自动化完全依赖 `osascript`，因此只能在 macOS 上运行。
- **本地浏览器**：需要安装 Chrome 和/或 Safari。
- **文件锁目录**：`$TMPDIR/pretext-browser-automation-locks/`（或 `/tmp/...`），用于单所有者串行化。

## 风险、边界与改进建议

### 风险与边界

1. **Firefox 不支持**：代码在 `runBrowser` 中显式拒绝 Firefox，意味着 pre-wrap 在 Firefox 上的行为目前无人值守；若未来需要三浏览器一致性，必须补齐支持。
2. **提取器方法锁定为 `span`**：`buildProbeUrl` 硬编码 `method=span`。AGENTS.md 提到 Safari span extraction 在 pre-wrap 场景下比 Range 更可靠，因此这是有意为之，但也意味着无法通过命令行切换方法进行交叉验证。
3. **AppleScript 脆弱性**：Safari / Chrome 的窗口/标签管理完全依赖 AppleScript；浏览器更新或用户手动关闭窗口会导致自动化失败。
4. **单浏览器串行**：文件锁确保同一浏览器不会被两个脚本并发使用，但多浏览器之间（如同时跑 Chrome 和 Safari）无全局锁，若临时服务器端口冲突可能产生竞态（不过 `getAvailablePort` 会分配不同端口）。
5. **用例规模小且固定**：15 个用例是“compact oracle set”，无法覆盖所有 Unicode 边缘情况；AGENTS.md 明确建议保持小规模。
6. **超时信息有限**：`loadHashReport` 超时后会报告最后 phase（`loading` / `measuring` / `posting`），但 `pre-wrap-check` 不使用 POST，因此 `posting` 阶段理论上不会出现。

### 改进建议

1. **增加 `--method` 覆盖开关**：允许命令行传入 `method=range` 进行交叉验证，帮助诊断 span/range 提取器敏感差异（类似 `probe-check.ts` 已支持）。
2. **Firefox 支持**：`browser-automation.ts` 已实现基于 WebDriver BiDi 的 Firefox 会话，只需在 `pre-wrap-check.ts` 中移除 Firefox 拒绝逻辑，并验证 `/probe` 页面在 Firefox headless 下的 DOM 测量稳定性。
3. **动态用例加载**：当前用例硬编码在脚本中。可考虑从 JSON 文件加载，方便非工程师添加新的 pre-wrap 边界用例而不修改 TS 源码。
4. **与 CI 集成**：建议在 `.github/workflows` 中加入 macOS runner + `bun run pre-wrap-check` 的 nightly 或 PR gate，防止 pre-wrap 改动回退。
5. **增强错误诊断输出**：当 `reportIsExact` 失败时，当前仅打印 diff 和行数。可借鉴 `probe-check.ts` 输出 `firstBreakMismatch` 的详细断行偏移和上下文，加速根因定位。
