# 研究报告：scripts/report-server.ts

## 场景与职责

`scripts/report-server.ts` 是 Pretext 浏览器自动化基础设施中的**本地 POST 接收端**。当浏览器诊断页面（如 `pages/accuracy.ts`、`pages/benchmark.ts`、corpus 相关页面）需要回传大规模 JSON 报告时，URL hash（`#report=...`）可能因数据量过大而触及长度限制或导致导航卡顿。`report-server.ts` 在本地启动一个轻量级 HTTP 服务器，通过 CORS 允许浏览器页面以 `POST` 方式将报告发送到 `http://127.0.0.1:<port>/report`，从而解决大体积报告的传输问题。AGENTS.md 中明确提到该设计意图：“large batched reports should use the local POST side channel instead of stuffing every row into `#report=...`”。

## 功能点目的

1. **大报告传输**：为 accuracy sweep、benchmark、corpus sweep 等产生数千至上万行数据的页面提供可靠的报告回传通道。
2. **请求级隔离**：通过 `expectedRequestId` 参数确保只接收与当前自动化会话匹配的请求，避免旧报告或并发请求的交叉污染。
3. **超时控制**：提供可配置的超时等待（默认 120 秒），在浏览器页面长时间无响应时主动报错。
4. **生命周期管理**：与 `browser-automation.ts` 的 `loadPostedReport` 配合使用，报告到达或超时后服务器可被立即关闭。

## 具体技术实现

### 关键流程

模块仅导出一个异步工厂函数 `startPostedReportServer<T>`：

1. **端口分配**：调用 `getAvailablePort()`（来自 `browser-automation.ts`）获取随机可用端口。
2. **Promise 占位**：创建 `reportPromise`，并暴露其 `resolve` 和 `reject` 句柄供后续 HTTP handler 使用。
3. **HTTP 服务器创建**：使用 Node.js 原生 `node:http` 的 `createServer`：
   - **CORS 预检**：对 `OPTIONS` 请求返回 `204`，并设置 `Access-Control-Allow-Origin: *` 和 `Access-Control-Allow-Methods: POST, OPTIONS`。
   - **POST 处理**：读取 UTF-8 请求体，解析为 JSON，检查 `report.requestId === expectedRequestId`，匹配则 `resolveReport(report)`；无论匹配与否都返回 `204`（避免浏览器重试或报错）。
   - **错误处理**：JSON 解析失败返回 `400` 并附带错误消息；非 POST/OPTIONS 请求返回 `404`。
4. **启动监听**：绑定到 `127.0.0.1:<port>`，返回包含以下字段的对象：
   - `endpoint: string` — 完整的 POST URL（如 `http://127.0.0.1:54321/report`）。
   - `waitForReport(timeoutMs?)` — 等待报告到达的异步函数，支持 `null` 表示无限等待，默认 120,000 ms。
   - `close()` — 关闭服务器，并在报告尚未到达时 `reject` `reportPromise`。

### 数据结构

```typescript
export async function startPostedReportServer<T extends { requestId?: string }>(
  expectedRequestId: string
): Promise<{
  endpoint: string
  waitForReport: (timeoutMs?: number | null) => Promise<T>
  close: () => void
}>
```

- 泛型 `T` 要求至少包含可选的 `requestId?: string`。
- `waitForReport` 内部使用 `Promise.race` 的等价实现：若指定了超时，则启动 `setTimeout` 在到期后 reject。

### 协议细节

- **传输协议**：HTTP/1.1 POST。
- **CORS**：极简配置，仅允许 `*` 来源和 `POST, OPTIONS` 方法，满足本地 `file://` 或 `localhost` 页面的跨域需求。
- **请求体**：纯文本 JSON，`Content-Type` 未做严格校验（浏览器默认发送 `text/plain;charset=UTF-8` 或 `application/json` 均可）。
- **响应码**：
  - `204 No Content`：成功接收或预检通过。
  - `400 Bad Request`：JSON 解析失败。
  - `404 Not Found`：方法不支持。

## 关键代码路径与文件引用

- **模块自身**：`scripts/report-server.ts`（85 行）。
- **端口工具**：`scripts/browser-automation.ts` 中的 `getAvailablePort()`。
- **消费者 1**：`scripts/browser-automation.ts` 中的 `loadPostedReport<T>()`（负责导航页面并将 `waitForReport` 与轮询逻辑结合）。
- **消费者 2（浏览器端）**：各诊断页面通过 `fetch(endpoint, { method: 'POST', body: JSON.stringify(report) })` 发送报告。具体实现分散在：
  - `pages/accuracy.ts`（accuracy sweep 页面）
  - `pages/benchmark.ts`（benchmark 页面）
  - `pages/corpus*.ts` 系列页面（corpus 扫描页面）
- **调用链示例**：
  ```
  accuracy-check.ts -> browser-automation.ts:loadPostedReport
                     -> report-server.ts:startPostedReport
                     -> 浏览器页面 POST 到 127.0.0.1:<port>/report
  ```

## 依赖与外部交互

### 内部依赖

- `node:http`（`createServer`）
- `scripts/browser-automation.ts`（`getAvailablePort`）

### 外部依赖

- **本地网络栈**：需要 `127.0.0.1` 回环地址可用，且端口不被防火墙拦截。
- **浏览器 `fetch` API**：发送端页面使用标准 `fetch` 进行 POST。
- **Node.js/Bun 运行时**：`node:http` 在 Bun 下兼容运行，脚本本身无 Bun 专属 API。

## 风险、边界与改进建议

### 风险与边界

1. **无 TLS/认证**：服务器完全开放于本地回环，任何能访问 `127.0.0.1` 的进程都可 POST 数据；在共享开发机或容器环境中存在极低的注入风险（虽然 `requestId` 做了匹配过滤）。
2. **单请求设计**：服务器在收到第一个匹配 `requestId` 的报告后即 resolve，但不会自动关闭；调用方需显式调用 `close()`。若调用方忘记关闭，端口会保持占用直到进程退出。
3. **请求体无大小限制**：当前使用字符串拼接 `body += chunk` 读取请求体，若浏览器意外发送超大 payload（如数十 MB），可能导致内存膨胀或性能问题。
4. **CORS 过于宽松**：`Access-Control-Allow-Origin: *` 在本地开发场景下足够，但理论上允许任何网页向该端口 POST；可收紧为仅允许 `localhost` 或页面实际来源。
5. **错误报告静默丢弃**：若 `requestId` 不匹配，服务器返回 `204` 但不通知调用方。这在正常流程中无害，但若浏览器因缓存或重导航发送了旧报告，调用方完全不知情。
6. **超时与关闭竞态**：`close()` 会 reject `reportPromise`，但如果此时 `waitForReport` 的 timeout 也刚好触发，可能出现双重 reject；当前实现中 `waitForReport` 使用新的 Promise 包装，竞态影响有限。

### 改进建议

1. **增加请求体大小限制**：在 `req.on('data')` 中累加 `Buffer.byteLength(chunk)`，超过阈值（如 10 MB）时主动销毁连接并 reject，防止内存滥用。
2. **收紧 CORS**：将 `access-control-allow-origin` 从 `*` 改为动态反射 `req.headers.origin`（若存在且为本地来源），减少暴露面。
3. **不匹配请求日志**：当 `requestId` 不匹配时，可在响应前打印一条 warn 日志，帮助调试浏览器重导航或并发问题。
4. **自动关闭选项**：在 `waitForReport` resolve/reject 后自动调用 `server.close()`，减少调用方遗漏关闭导致的端口泄漏。
5. **健康检查端点**：增加 `GET /health` 或 `HEAD /health`，使 `browser-automation.ts` 在将 endpoint URL 传给浏览器前能快速确认服务器已就绪，避免浏览器在服务器尚未 listen 时发起 POST。
