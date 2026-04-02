# browser-automation.ts 研究文档

## 场景与职责

`scripts/browser-automation.ts` 是 Pretext 项目**所有浏览器自动化脚机的共享基座**。它封装了浏览器会话管理（Chrome / Safari / Firefox）、单实例文件锁、临时页服务器生命周期、可用端口探测，以及两种报告拉取机制（URL hash 轮询 / POST side-channel）。所有 accuracy、benchmark、corpus 类脚本均依赖此文件。

## 功能点目的

1. **跨浏览器统一抽象**：为上层脚本提供统一的 `BrowserSession` 接口（`navigate`, `readLocationUrl`, `close`），屏蔽 Chrome/Safari 的 AppleScript 实现与 Firefox 的 BiDi WebSocket 实现差异。
2. **单实例互斥锁**：通过文件锁防止多个进程同时操作同一浏览器（避免标签混乱、数据竞争）。
3. **页服务器按需启动**：检测端口是否已有存活服务，避免重复启动；无服务时自动 `bun --no-hmr` 启动。
4. **报告可靠传输**：
   - **Hash Report**：小数据通过 URL hash 回传，轮询读取。
   - **Posted Report**：大数据通过本地临时 HTTP server 接收 POST，解决 hash 长度限制。
5. **导航状态可观测**：在 URL hash 中嵌入 `phase`（loading / measuring / posting），超时错误时可定位卡在哪一步。

## 具体技术实现

### 关键数据结构

```ts
export type BrowserKind = 'chrome' | 'safari' | 'firefox';
export type BrowserSession = {
  navigate: (url: string) => MaybePromise<void>;
  readLocationUrl: () => MaybePromise<string>;
  close: () => void;
};
export type BrowserSessionOptions = { foreground?: boolean };
export type PageServer = { baseUrl: string; process: ChildProcess | null };
export type BrowserAutomationLock = { release: () => void };
```

### 浏览器会话实现

#### Safari / Chrome（AppleScript）

- **创建窗口/标签**：
  - Safari：`make new document` → 返回 `window id`。
  - Chrome：`make new tab at end of tabs of targetWindow` → 返回 `windowId,tabId`。
- **前台控制**：`options.foreground === true` 时，脚本先 `activate` 应用，并将窗口/标签置前。
- **后台安全**：`runBackgroundAppleScript` 在运行前记录当前最前应用，运行后恢复，减少干扰。
- **导航**：直接设置目标标签的 `URL`。
- **读位置**：读取当前标签的 `URL`。
- **关闭**：关闭指定窗口/标签；失败静默处理。

#### Firefox（BiDi WebSocket）

- **启动**：`spawn` Firefox 二进制，带 `--headless --new-instance --profile <tmp> --remote-debugging-port <port> about:blank`。
- **连接**：`connectFirefoxBidi(port)` 通过 WebSocket `ws://127.0.0.1:${port}/session` 连接。
- **协议握手**：
  1. `session.new`
  2. `browsingContext.getTree` 获取根 `context`
- **导航**：`browsingContext.navigate`（wait: 'none'）。
- **读位置**：`script.evaluate` 执行 `location.href`。
- **关闭**：关闭 WebSocket，SIGTERM 进程，删除临时 profile 目录。

### 文件锁机制

- 锁目录：`${TMPDIR}/pretext-browser-automation-locks/`
- 锁文件：`${browser}.lock`
- 创建方式：`openSync(lockPath, 'wx')` 原子独占创建；写入 `{ pid, startedAt }` JSON。
- **自愈合**：若锁文件已存在，读取 metadata 中的 pid；若进程已死（`process.kill(pid, 0)` 失败且非 EPERM），则删除锁文件并重试。
- 超时：默认 120 秒；超时抛 `Timed out waiting for ${browser} automation lock`。

### 页服务器管理

```ts
export async function ensurePageServer(port, pathname, cwd): Promise<PageServer>
```

- 先尝试 `LOOPBACK_BASES`（`127.0.0.1`, `localhost`, `[::1]`）探测该端口是否已有可达服务。
- 若已有服务，返回 `{ baseUrl, process: null }`。
- 若无服务，启动 `bun --port=${port} --no-hmr pages/*.html`，并在 20 秒内轮询直到可达。

### 报告拉取

#### `loadHashReport<T>`

- 导航到目标 URL 后，以 100ms 为周期轮询 `session.readLocationUrl()`。
- 解析 URL hash 中的 `report` 字段；若 `requestId` 匹配则返回解析后的 JSON。
- 同时读取 `phase` 字段更新 `lastPhase`，用于超时诊断。
- 超时消息包含最后观测到的 phase（`loading` / `measuring` / `posting`）和 URL。

#### `loadPostedReport<T>`

- 与 `loadHashReport` 并行运行：
  - 一方面轮询 hash 中的 phase 以观测进度；
  - 另一方面等待 `waitForReport()`（通常由 `startPostedReportServer` 提供）接收 POST 数据。
- 任一条件满足（POST 数据到达或 hash 中报告出现）即返回。
- 超时同样携带 `lastPhase` 诊断信息。

### 端口分配

```ts
export async function getAvailablePort(requestedPort?: number | null): Promise<number>
```

- 若传入有效数字，直接返回该端口。
- 否则创建临时 TCP server 监听 `0`，读取分配到的端口后关闭 server。

## 关键代码路径与文件引用

- 本文件：`scripts/browser-automation.ts`
- POST 报告服务器：`scripts/report-server.ts`（消费 `getAvailablePort`）
- 导航状态工具：`shared/navigation-state.ts`（`readNavigationPhaseState`, `readNavigationReportText`, `buildNavigationPhaseHash`, `buildNavigationReportHash`）
- 调用方：`scripts/accuracy-check.ts`, `scripts/benchmark-check.ts`, `scripts/corpus-check.ts`, `scripts/corpus-sweep.ts`, `scripts/corpus-font-matrix.ts`, `scripts/corpus-representative.ts`, `scripts/corpus-taxonomy.ts`, `scripts/gatsby-check.ts`, `scripts/gatsby-sweep.ts`

## 依赖与外部交互

- **Node 核心模块**：`node:child_process`, `node:fs`, `node:net`, `node:os`, `node:path`。
- **macOS 系统工具**：`osascript`（Safari/Chrome 控制必需）。
- **Firefox 二进制**：硬编码路径 `/Applications/Firefox.app/Contents/MacOS/firefox`。
- **WebSocket**：Firefox BiDi 连接使用全局 `WebSocket`（Bun/Node 环境需原生支持）。
- **文件系统**：`TMPDIR` 环境变量决定锁目录与 Firefox profile 临时目录。

## 风险、边界与改进建议

1. **AppleScript 平台锁定**：Chrome/Safari 会话完全依赖 macOS AppleScript，无法在 Linux/Windows CI 上运行。若需跨平台，应引入 Playwright/Puppeteer 作为可选后端。
2. **Firefox 路径硬编码**：`/Applications/Firefox.app/Contents/MacOS/firefox` 仅适用于 macOS 默认安装位置。建议通过 `which firefox` 或环境变量 `FIREFOX_BIN` 覆盖。
3. **锁的自愈竞态**：`isProcessAlive` 使用 `process.kill(pid, 0)`，在容器或权限受限环境可能误判；若两个进程同时发现死锁并删除，仍可能产生竞态。可考虑使用 `proper-lockfile` 等更健壮的库。
4. **BiDi 错误处理粗糙**：`getBidiStringValue` 对嵌套结果结构做简单降级，若 Firefox 未来改变 `script.evaluate` 返回格式，可能静默返回空字符串。应增加 schema 校验。
5. **页服务器探测范围**：`LOOPBACK_BASES` 包含 `[::1]`，但 `ensurePageServer` 启动的 Bun server 绑定 `127.0.0.1`，在纯 IPv4 环境 `[::1]` 探测必然失败，增加少量延迟。可精简探测列表。
6. **超时信息增强**：当前超时仅记录 `lastPhase` 和 `observedUrl`。建议增加最后一次成功收到响应的时间戳、已轮询次数，便于排查浏览器卡死或网络不通。
