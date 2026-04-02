# accuracy-check.ts 研究文档

## 场景与职责

`scripts/accuracy-check.ts` 是 Pretext 项目的**浏览器精度回归检测入口脚本**。它负责在真实浏览器（Chrome / Safari / Firefox）中自动化运行 `/accuracy` 页面，对 `src/test-data.ts` 中的共享语料执行“预测高度 vs 实际 DOM 高度”的批量对账，输出匹配率、不匹配明细，并可写入 JSON 快照。该脚本是 CI/本地验证布局引擎与浏览器行为一致性的核心工具。

## 功能点目的

1. **端到端精度扫测**：驱动浏览器访问本地 `pages/accuracy.ts` 页面，遍历多字体 × 多字号 × 多宽度 × 多文本的组合矩阵。
2. **浏览器兼容性覆盖**：支持 Chrome（默认）、Safari、Firefox；Firefox 因 IPv6/localhost 兼容问题需额外启动代理服务器。
3. **报告采集**：通过 URL hash 或 POST side-channel（`--full` 模式）获取页面回传的 `AccuracyReport`。
4. **快照落盘**：支持 `--output=` 将完整报告写入 JSON，用于更新 `accuracy/*.json` 检入基准。
5. **不匹配可视化**：在终端打印每个不匹配的 label、font、width、actual/predicted、diff 及诊断行。

## 具体技术实现

### 关键数据结构

```ts
type AccuracyMismatch = {
  label: string; font: string; fontSize: number; lineHeight: number;
  width: number; actual: number; predicted: number; diff: number;
  text: string; diagnosticLines?: string[];
};

type AccuracyReport = {
  status: 'ready' | 'error';
  environment?: { userAgent; devicePixelRatio; viewport; screen };
  total?: number; matchCount?: number; mismatchCount?: number;
  mismatches?: AccuracyMismatch[]; rows?: AccuracyRow[]; message?: string;
};
```

### 核心流程

1. **参数解析**
   - 浏览器由 `ACCURACY_CHECK_BROWSER` 环境变量或默认 `chrome` 决定。
   - `--full` 开关控制是否请求完整 rows（会启用 `startPostedReportServer` POST 回传，避免 hash 长度爆炸）。
   - `--output=` 指定落盘路径。
   - `ACCURACY_CHECK_PORT` 可固定页服务端口号；`ACCURACY_CHECK_TIMEOUT_MS` 控制等待报告超时（Safari 默认 240s，其他 120s）。

2. **服务启动**
   - **非 Firefox**：调用 `ensurePageServer(port, '/accuracy', cwd)` 启动临时 Bun server（`bun --port=${port} --no-hmr pages/*.html`）。
   - **Firefox**：Bun server 绑定 `localhost` 后，额外启动一个本地 HTTP proxy（`startProxyServer`），将 `127.0.0.1` 代理到 `[::1]`，解决 Firefox 对 IPv6 localhost 的兼容问题。

3. **浏览器会话与锁**
   - `acquireBrowserAutomationLock(browser)`：获取文件锁，防止多进程同时操作同一浏览器实例。
   - `createBrowserSession(browser)`：创建浏览器会话（Chrome/Safari 用 AppleScript；Firefox 用 BiDi WebSocket）。

4. **报告加载**
   - 普通模式：`loadHashReport(session, url, requestId, browser, timeoutMs)` 轮询 URL hash 中的 report。
   - `--full` 模式：页面通过 `fetch` POST 到 `reportServer.endpoint`，脚本用 `loadPostedReport` 等待 side-channel 数据。

5. **结果输出**
   - `printReport`：终端打印匹配率、环境指纹、每条 mismatch 的 diff 与诊断。
   - 若指定 `--output`，以 JSON 格式落盘。

### 协议与命令

- 页服务端：本地 HTTP，`/accuracy?report=1&requestId=xxx[&full=1][&reportEndpoint=...]`
- 代理协议：Node `http.createServer` 透传请求到目标 origin，过滤 `transfer-encoding` 响应头。
- 进程管理：`spawn('/bin/zsh', ['-lc', 'bun ...'])` 启动页服务；退出时 `SIGTERM` 清理。

## 关键代码路径与文件引用

- 本文件：`scripts/accuracy-check.ts`
- 浏览器自动化基座：`scripts/browser-automation.ts`（`acquireBrowserAutomationLock`, `createBrowserSession`, `ensurePageServer`, `getAvailablePort`, `loadHashReport`, `loadPostedReport`）
- POST 报告服务器：`scripts/report-server.ts`（`startPostedReportServer`）
- 被测页面：`pages/accuracy.ts`（执行实际 canvas/DOM 对比扫测）
- 共享语料与尺寸矩阵：`src/test-data.ts`（`TEXTS`, `SIZES`, `WIDTHS`）
- 输出基准：`accuracy/chrome.json`, `accuracy/safari.json`, `accuracy/firefox.json`

## 依赖与外部交互

- **Node/Bun 运行时**：使用 `node:child_process`, `node:fs`, `node:http`, `node:path`。
- **本地浏览器**：
  - Chrome / Safari 通过 macOS `osascript` 控制。
  - Firefox 通过 `/Applications/Firefox.app/Contents/MacOS/firefox` 的 `--remote-debugging-port` + BiDi WebSocket。
- **环境变量**：`ACCURACY_CHECK_BROWSER`, `ACCURACY_CHECK_PORT`, `ACCURACY_CHECK_TIMEOUT_MS`。
- **上游页面**：依赖 `pages/accuracy.ts` 正确解析 query params 并回传 report。

## 风险、边界与改进建议

1. **Firefox 代理脆弱性**：`startProxyServer` 对 `transfer-encoding` 做简单过滤，若上游返回 chunked + trailer 可能丢失语义。建议未来改用固定 `127.0.0.1` 监听规避 IPv6 问题，或让 `ensurePageServer` 直接绑定 IPv4。
2. **AppleScript 前台干扰**：Safari/Chrome 的 AppleScript 导航会短暂抢占前台。`accuracy-check` 未传 `foreground: true`，但窗口创建仍可能闪动。建议在无头/后台模式成熟后迁移。
3. **超时硬编码**：Safari 240s / 其他 120s 是基于经验值；若新增更大数据集可能仍需上调。可考虑按 `TEXTS.length × SIZES.length × WIDTHS.length × FONTS.length` 动态估算。
4. **报告体积**：`--full` 模式通过 POST server 解决 hash 长度限制，但 POST server 端口由 `getAvailablePort()` 随机分配，若防火墙异常可能失败。可增加 `--report-port` 固定端口。
5. **进程泄漏**：`finally` 中先关 proxy 再杀 Bun server；若 proxy 仍有连接挂起，`serverProcess.kill('SIGTERM')` 可能不等待优雅退出。建议增加 `serverProcess` 的 `kill('SIGKILL')` 兜底。
