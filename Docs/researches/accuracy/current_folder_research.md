# Pretext Accuracy 系统研究文档

> 研究范围：`src/**`、`scripts/**`、`shared/**`、`pages/**`、`.github/workflows/**`、package/config、以及 `accuracy/`、`benchmarks/`、`status/`、`corpora/`、`research-data/` 下的 JSON 数据文件。  
> 上下文参考：`README.md`、`DEVELOPMENT.md`、`RESEARCH.md`、`STATUS.md`、`TODO.md`、`AGENTS.md`。

---

## 1. 场景与职责

### 1.1 定位

Accuracy（浏览器精度校验）是 Pretext 的**回归门禁（regression gate）**，用于在每次引擎或测量逻辑变更后，验证库预测的行高/换行结果与真实浏览器 DOM 渲染是否一致。它不是一个日常开发调试工具，而是 CI/本地脚本级别的自动化校验层。

### 1.2 覆盖矩阵

- **字体**：4 套字体栈（Helvetica Neue、Georgia、Verdana、Courier New）
- **字号**：8 档（12/14/15/16/18/20/24/28 px）
- **行高**：`Math.round(fontSize * 1.2)`，即 14~33 px
- **容器宽度**：8 档（150/200/250/300/350/400/500/600 px）
- **文本用例**：30 条（`src/test-data.ts`），覆盖拉丁、阿拉伯、希伯来、CJK、韩文、泰文、Emoji、混合方向、边界情况（空串、空格、换行、超长单词）
- **总组合数**：4 × 8 × 8 × 30 = **7,680 条/浏览器**

### 1.3 浏览器覆盖

当前维护三套快照：

| 浏览器 | 快照文件 | 驱动方式 |
|--------|----------|----------|
| Chrome | `accuracy/chrome.json` | AppleScript（macOS） |
| Safari | `accuracy/safari.json` | AppleScript（macOS） |
| Firefox | `accuracy/firefox.json` | WebDriver BiDi over WebSocket |

### 1.4 职责边界

- **做什么**：在真实浏览器中批量渲染隐藏 DOM，比较 `getBoundingClientRect().height`（actual）与 `layout(prepared, maxWidth, lineHeight).height`（predicted）。
- **不做什么**：不测试性能（由 `benchmarks/` 负责），不测试长语料全文（由 `corpora/` 负责），不测试 pre-wrap 以外的 CSS 配置。

---

## 2. 功能点目的

### 2.1 核心目标

Pretext 的核心价值是**用纯 JS 算术替代 DOM 测量**（避免同步回流）。Accuracy 系统的存在意义是：

1. **证明 canvas 分段求和 + 预处理规则足以逼近浏览器排版**；
2. **在引擎热路径（`src/line-break.ts`、`src/layout.ts`）变更时发出回归警报**；
3. **为不同浏览器提供可检入的“已知良好”基线（checked-in snapshots）**。

### 2.2 判定标准

- 通过条件：`|actual - predicted| < 1 px`
- 失败时输出：
  - 高度差异（diff）
  - 逐行文本对比（`ours` vs `browser`）
  - 环境指纹（DPR、viewport、UA）

### 2.3 与周边系统的关系

```
accuracy/               ← 原始快照（chrome.json / safari.json / firefox.json）
    ↓
scripts/status-dashboard.ts  ← 聚合为 status/dashboard.json
    ↓
STATUS.md               ← 给人看的入口指引
```

此外，`scripts/corpus-status.ts` 也会读取 accuracy 快照，用于在语料大盘中展示浏览器精度概况。

---

## 3. 具体技术实现

### 3.1 整体架构

Accuracy 系统由**三层**组成：

1. **页内测量层**（`pages/accuracy.ts` + `pages/accuracy.html`）
2. **浏览器自动化层**（`scripts/accuracy-check.ts` + `scripts/browser-automation.ts`）
3. **报告传输层**（`shared/navigation-state.ts` + `pages/report-utils.ts` + `scripts/report-server.ts`）

### 3.2 页内测量层（`pages/accuracy.ts`）

#### 3.2.1 隐藏 DOM 构造

```ts
const container = document.createElement('div')
container.style.cssText = 'position:absolute;top:-9999px;left:-9999px;visibility:hidden'
```

对每个 (font, fontSize, maxWidth) 组合，批量创建 `<div>` 并设置：

- `font`、`lineHeight`、`width`
- `wordWrap: 'break-word'`
- `overflowWrap: 'break-word'`
- `textContent = text`

#### 3.2.2 双重 API 调用

- 快路径：`layout(prepared, maxWidth, lineHeight).height` —— 仅返回行数/高度，无字符串分配。
- 富路径（仅 mismatch 时）：`layoutWithLines(prepared, maxWidth, lineHeight)` —— 生成逐行文本，用于诊断。

#### 3.2.3 逐行诊断提取（`getBrowserLines`）

当高度差 ≥ 1 px 时，使用 `Range.getClientRects()` 按**诊断单元（DiagnosticUnit）**逐行提取浏览器真实文本：

- 单元来源：`getDiagnosticUnits(prepared)`（`pages/diagnostic-utils.ts`）
- 对 `breakableWidths !== null` 的 segment，按 grapheme 拆分；否则按整 segment 提取。
- 通过比较连续 `rect.top` 差异（> 0.5 px）判断换行边界。

> 注意：该提取器是 Range-based，对 Safari 的 preserved-space / URL query 场景存在已知偏差（AGENTS.md 有专门提示）。

### 3.3 浏览器自动化层

#### 3.3.1 `scripts/accuracy-check.ts`

入口脚本，职责：

1. 获取浏览器自动化锁（`acquireBrowserAutomationLock`），防止并发冲突；
2. 启动本地临时 Bun 页面服务器（`ensurePageServer`）；
3. Firefox 特殊处理：需经 `localhost` → `127.0.0.1` 代理转发（Firefox 对 IPv6/localhost 解析有差异）；
4. 构造带 `requestId` 的 URL，导航浏览器；
5. 接收报告并可选写入 `--output` 文件。

环境变量：

- `ACCURACY_CHECK_BROWSER`：默认 `chrome`，可选 `safari`、`firefox`
- `ACCURACY_CHECK_TIMEOUT_MS`：Safari 默认 240s，其他 120s
- `ACCURACY_CHECK_PORT`：指定本地页面服务器端口

命令行参数：

- `--full`：请求完整 7,680 条 rows（默认只传 mismatches）
- `--output=path`：将 JSON 报告写入文件（用于刷新快照）

#### 3.3.2 `scripts/browser-automation.ts`

**浏览器会话抽象**

```ts
type BrowserSession = {
  navigate: (url: string) => MaybePromise<void>
  readLocationUrl: () => MaybePromise<string>
  close: () => void
}
```

- **Chrome / Safari**：通过 `osascript` 调用 AppleScript 操作浏览器窗口/标签页。
  - `foreground?: boolean` 控制是否 `activate`（benchmark 用 foreground，accuracy 默认 background）。
- **Firefox**：启动 `/Applications/Firefox.app/Contents/MacOS/firefox` headless 实例，开启 `--remote-debugging-port`，通过 **WebDriver BiDi** WebSocket 通信。

**自动化锁机制**

```ts
const LOCK_DIR = join(process.env['TMPDIR'] ?? '/tmp', 'pretext-browser-automation-locks')
```

- 每个浏览器一个 `.lock` 文件，内写 `{ pid, startedAt }`。
- 自修复：若锁文件存在但对应进程已死，则自动删除并重新获取。
- 超时：默认 120s。

### 3.4 报告传输层

#### 3.4.1 双通道设计

- **小报告**：通过 URL hash 传输（`#report=...`），由 `shared/navigation-state.ts` 编解码。
- **大报告**：当 `--full` 时，启用本地 POST 服务器（`scripts/report-server.ts`），页面通过 `fetch` 侧传，避免 hash 长度爆炸。

#### 3.4.2 导航阶段状态（Navigation Phase）

用于调试超时问题：

```ts
type NavigationPhase = 'loading' | 'measuring' | 'posting'
```

页面在关键阶段调用 `publishNavigationPhase(phase, requestId)`，将状态写入 hash。自动化脚本在超时前会读取最后阶段，输出类似：

```
Timed out waiting for report from safari (last phase: posting)
```

`posting` 超时通常意味着侧传服务器或 CORS 问题，而非文本引擎本身卡死。

### 3.5 核心引擎与精度策略

Accuracy 的“预测端”依赖以下关键实现：

#### 3.5.1 测量阶段（`src/measurement.ts`）

- **Canvas 上下文**：优先 `OffscreenCanvas`，回退 `document.createElement('canvas')`。
- **缓存结构**：`Map<font, Map<segment, SegmentMetrics>>`
- **Emoji 修正**：自动检测 canvas 与 DOM 的 emoji 宽度差异（macOS Apple Color Emoji 在 <24px 时 canvas 会过宽），修正值按 grapheme 数扣除。
- **引擎画像（EngineProfile）**：
  - `lineFitEpsilon`：Safari `1/64`，Chromium/Gecko `0.005`
  - `carryCJKAfterClosingQuote`：仅 Chromium 启用
  - `preferPrefixWidthsForBreakableRuns` / `preferEarlySoftHyphenBreak`：仅 Safari 启用

#### 3.5.2 分析阶段（`src/analysis.ts`）

- **标准化**：默认 `white-space: normal` 行为（折叠空白、去头尾空格）；可选 `pre-wrap` 模式。
- **分词**：`Intl.Segmenter(granularity: 'word')`
- **后处理流水线**（按顺序）：
  1. 标点合并（left-sticky punctuation 并入前词）
  2. CJK 禁则（kinsoku）合并
  3. 阿拉伯无空格标点合并
  4. URL 合并（`https://...` 与 query string 分两段）
  5. 数字/时间范围合并
  6. Glue（NBSP/NNBSP/WJ）连接文本合并
  7. 转义引号簇合并（`\"word\"`）

#### 3.5.3 换行阶段（`src/line-break.ts`）

- **快路径**：`simpleLineWalkFastPath` 为 true 时（无非空格特殊 segment），走简化版 `walkPreparedLinesSimple`，避免 chunk 和 fit/paint advance 的复杂 bookkeeping。
- **富路径**：支持 hard-break chunks、soft hyphen、tab stop、grapheme 级断行（`overflow-wrap: break-word`）。
- **断行逻辑**：
  - 尾部空格/零宽断点/soft hyphen 允许“悬挂”出线宽（不触发换行）。
  - 超长 word-like segment 在 grapheme 边界强制折断。

---

## 4. 关键代码路径与文件引用

### 4.1 数据定义与配置

| 文件 | 作用 |
|------|------|
| `src/test-data.ts` | 30 条跨语言测试文本、8 档字号、8 档宽度 |
| `package.json` | `accuracy-check`、`accuracy-snapshot` 等 npm scripts |
| `accuracy/chrome.json` | Chrome 7,680 行全量快照（含 environment 指纹） |
| `accuracy/safari.json` | Safari 快照 |
| `accuracy/firefox.json` | Firefox 快照 |
| `status/dashboard.json` | 聚合三浏览器 accuracy 与 benchmarks 的机器可读大盘 |

### 4.2 页内执行路径

```
pages/accuracy.html
    └── pages/accuracy.ts
            ├── src/layout.ts      (prepareWithSegments / layout / layoutWithLines)
            ├── src/test-data.ts   (TEXTS / SIZES / WIDTHS)
            ├── pages/diagnostic-utils.ts   (getDiagnosticUnits)
            └── pages/report-utils.ts       (publishNavigationPhase / publishNavigationReport)
```

### 4.3 自动化执行路径

```
scripts/accuracy-check.ts
    ├── scripts/browser-automation.ts
    │       ├── createBrowserSession('chrome' | 'safari' | 'firefox')
    │       ├── acquireBrowserAutomationLock
    │       ├── ensurePageServer
    │       ├── loadHashReport
    │       └── loadPostedReport
    ├── scripts/report-server.ts
    └── shared/navigation-state.ts
```

### 4.4 核心引擎路径

```
src/layout.ts
    ├── src/analysis.ts       (analyzeText / buildMergedSegmentation)
    ├── src/measurement.ts    (getSegmentMetrics / getEngineProfile / emoji correction)
    └── src/line-break.ts     (countPreparedLines / walkPreparedLines / layoutNextLineRange)
```

### 4.5 关键类型速查

```ts
// pages/accuracy.ts
interface AccuracyReport {
  status: 'ready' | 'error'
  environment: EnvironmentFingerprint
  total: number
  matchCount: number
  mismatchCount: number
  mismatches: Mismatch[]
  rows?: AccuracyRow[]      // --full 时存在
}

// src/layout.ts
interface LayoutResult {
  lineCount: number
  height: number
}
interface LayoutLinesResult extends LayoutResult {
  lines: LayoutLine[]       // { text, width, start, end }
}

// src/measurement.ts
interface EngineProfile {
  lineFitEpsilon: number
  carryCJKAfterClosingQuote: boolean
  preferPrefixWidthsForBreakableRuns: boolean
  preferEarlySoftHyphenBreak: boolean
}
```

---

## 5. 依赖与外部交互

### 5.1 运行时依赖

- **Bun**：页面服务器、脚本执行、JSON 解析/写入。
- **macOS + AppleScript**：Chrome/Safari 自动化必需。
- **Firefox.app**：需安装于 `/Applications/Firefox.app/Contents/MacOS/firefox`。
- **浏览器实例**： accuracy 不依赖 Playwright/Puppeteer，而是直接调用系统浏览器。

### 5.2 内部模块依赖图

```
accuracy-check.ts
    ├─► browser-automation.ts
    │       ├─► shared/navigation-state.ts
    │       └─► node:child_process / node:net / node:fs
    ├─► report-server.ts
    │       └─► node:http
    └─► node:fs / node:path

accuracy.ts (browser runtime)
    ├─► src/layout.ts
    │       ├─► src/analysis.js
    │       ├─► src/measurement.js
    │       └─► src/line-break.js
    ├─► pages/diagnostic-utils.ts
    ├─► pages/report-utils.ts
    │       └─► shared/navigation-state.ts
    └─► src/test-data.ts
```

### 5.3 外部交互协议

#### 5.3.1 Hash 报告协议

页面侧：
```ts
history.replaceState(null, '', `${location.pathname}${location.search}#report=${JSON.stringify(report)}`)
```

脚本侧：
```ts
const reportJson = readNavigationReportText(currentUrl)
const report = JSON.parse(reportJson)
```

#### 5.3.2 POST 侧传协议

- 服务器：`scripts/report-server.ts` 在随机端口启动，支持 CORS `POST/OPTIONS`。
- 端点：`http://127.0.0.1:${port}/report`
- 页面通过 `fetch(reportEndpoint, { method: 'POST', body: JSON.stringify(report) })` 发送。

### 5.4 CI 集成

`.github/workflows/pages.yml` 仅负责构建并部署 GitHub Pages 静态站点，**不直接运行 accuracy 校验**。Accuracy 目前以本地脚本 + 人工检入快照的方式运作，尚未实现无人值守的 CI accuracy gate。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 平台锁定

- **macOS 强依赖**：Chrome/Safari 自动化完全基于 AppleScript，无法在 Linux/Windows CI 上运行。
- **Firefox 硬编码路径**：`/Applications/Firefox.app/...` 也是 macOS 专属。
- **结果**：accuracy 无法在 GitHub Actions（ubuntu-latest）上自动执行，必须本地运行并手动提交快照。

#### 6.1.2 单浏览器并发锁

`browser-automation.ts` 的文件锁机制虽然支持自修复，但**不支持同一浏览器的多并发任务**。若同时跑 corpus-sweep 和 accuracy-check，后者会等待锁超时。

#### 6.1.3 提取器偏差

- `getBrowserLines` 使用 `Range.getClientRects()`，在 Safari 的 preserved-space 和 URL query 边界处可能过度推进（over-advance）。
- AGENTS.md 明确提醒：对 Safari URL/query 探测 miss，应先交叉验证 `--method=span`。

#### 6.1.4 字体限制

- `system-ui` 被明确标记为**不安全**（`RESEARCH.md`），因为 canvas 与 DOM 在 macOS 上可能解析到不同光学变体。accuracy 测试仅使用命名字体栈。

### 6.2 边界条件

| 边界 | 当前行为 |
|------|----------|
| 空文本 | `layout()` 返回 `{ lineCount: 0, height: 0 }`，与 DOM 一致 |
| 纯空白 | 标准化后可能为空，或保留为单个空格（pre-wrap） |
| 超长无空格单词 | 在 grapheme 边界强制折断（`overflow-wrap: break-word`） |
| Emoji | 自动 canvas→DOM 修正，但仅在 `textMayContainEmoji` 为 true 时触发 |
| Soft Hyphen (`\u00AD`) | 未断行时 invisible；断行时 trailing `-` 出现在 `layoutWithLines().lines[].text` |
| Tab (`\t`) | 仅在 `pre-wrap` 模式下保留，按 8 倍空格宽度的 tab stop 对齐 |

### 6.3 改进建议

#### 6.3.1 解耦 macOS 依赖

- **Chrome**：可迁移到 Chrome DevTools Protocol（CDP）或 Puppeteer，支持 Linux CI。
- **Safari**：可探索 `safaridriver`（WebDriver）替代 AppleScript，虽然启动更慢，但具备跨平台潜力。
- **优先级**：高。当前手动提交快照的流程增加了回归漏检风险。

#### 6.3.2 引入 diff 阈值分层

当前阈值是硬编码的 `1 px`。建议：
- 增加 `warning` 层（如 `0.5 px`），在报告中标黄但不算失败，用于提前发现累积漂移。
- 保留 `1 px` 作为硬性回归红线。

#### 6.3.3 诊断信息增强

- 当前 mismatch 仅输出逐行文本对比。可补充：
  - 触发换行的具体 segment 索引与宽度
  - canvas 测量值 vs DOM `Range` 宽度对比
  - 引擎画像（`lineFitEpsilon` 等）当时取值
- 这能加速 `line-break.ts` 或 `measurement.ts` 变更后的根因定位。

#### 6.3.4 扩展 CSS 配置覆盖

AGENTS.md 的 Open Questions 已指出：`break-all`、`keep-all`、`strict`、`loose`、`anywhere` 等配置尚未测试。Accuracy 当前仅验证默认配置（`white-space: normal` + `word-break: normal` + `overflow-wrap: break-word` + `line-break: auto`）以及窄化的 `pre-wrap`。

#### 6.3.5 自动化锁改进

- 当前锁只有“浏览器级别”粒度。若未来需要并行跑不同语料/不同端口，可考虑将锁 key 改为 `browser + port + task` 的组合，或改用进程间消息队列协调。

#### 6.3.6 减少 full-row 报告体积

7,680 条 rows 的 JSON 约 1.7 MB。虽然 POST 侧传解决了 hash 长度问题，但长期可考虑：
- 仅输出 mismatches + 抽样 rows（如每宽度/字体一条代表）
- 使用行内压缩或二进制格式（如 MessagePack）

---

## 附录：关键命令速查

```bash
# 本地运行 accuracy 校验（默认 Chrome）
bun run accuracy-check

# Safari / Firefox
bun run accuracy-check:safari
bun run accuracy-check:firefox

# 刷新检入快照（--full 输出完整 rows）
bun run accuracy-snapshot
bun run accuracy-snapshot:safari
bun run accuracy-snapshot:firefox

# 刷新状态大盘
bun run status-dashboard

# 类型检查 + lint
bun run check
```

---

*文档生成时间：2026-04-02*  
*基于仓库 commit 前的最新源码与 JSON 快照结构。*
