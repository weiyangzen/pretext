# navigation-state.ts 深度研究文档

## 概述

`shared/navigation-state.ts` 是 Pretext 项目中负责**浏览器自动化测试通信协议**的核心共享模块。它定义了浏览器页面与自动化脚本之间的状态传递机制，通过 URL hash 实现跨进程/跨环境的轻量级状态同步与报告传输。

---

## 场景与职责

### 核心场景

1. **浏览器自动化测试状态追踪**
   - 在自动化测试过程中，浏览器页面需要向控制脚本报告当前执行阶段
   - 支持三个阶段：`loading`（加载中）、`measuring`（测量中）、`posting`（报告发送中）

2. **跨域/跨进程报告传输**
   - 浏览器页面通过 URL hash 将测试报告传递给自动化脚本
   - 支持两种传输模式：
     - **Hash 模式**：小型报告直接编码在 URL hash 中
     - **POST 侧通道模式**：大型报告通过 HTTP POST 发送到本地临时服务器

3. **请求-响应关联**
   - 通过 `requestId` 将页面请求与报告响应关联，确保异步通信的可靠性

### 职责边界

| 职责 | 说明 |
|------|------|
| 状态编码/解码 | 将导航阶段和报告数据编码为 URL hash 格式，或反向解析 |
| 协议标准化 | 定义统一的通信协议，被 pages/ 和 scripts/ 共同使用 |
| 轻量级传输 | 仅处理小型控制数据；大型数据通过侧通道传输 |

---

## 功能点目的

### 1. 导航阶段状态管理

**目的**：让自动化脚本能够实时感知页面执行状态，用于超时诊断和进度跟踪。

```typescript
export type NavigationPhase = 'loading' | 'measuring' | 'posting'
```

- `loading`：页面初始加载阶段，字体加载、资源准备
- `measuring`：核心测量阶段，执行文本布局计算和浏览器对比
- `posting`：报告发送阶段，通过 POST 侧通道传输大型报告

**使用场景**：
- `scripts/browser-automation.ts` 中的 `loadHashReport` 和 `loadPostedReport` 通过轮询 URL 读取阶段状态
- 超时错误信息中包含最后观察到的阶段，帮助诊断问题（如 "last phase: posting" 表明传输阶段可能出现问题）

### 2. URL Hash 编码/解码

**目的**：提供一种无需服务器、跨域安全的浏览器-脚本通信机制。

**编码功能**：
- `buildNavigationPhaseHash(phase, requestId?)`：构建阶段状态 hash
- `buildNavigationReportHash(report)`：构建报告数据 hash

**解码功能**：
- `readNavigationPhaseState(urlOrHash)`：解析当前导航阶段
- `readNavigationReportText(urlOrHash)`：提取报告 JSON 文本

### 3. 报告传输协议

**目的**：支持两种报告传输策略，平衡可靠性和性能。

| 模式 | 适用场景 | 实现位置 |
|------|----------|----------|
| Hash 模式 | 小型报告（默认） | `buildNavigationReportHash` / `readNavigationReportText` |
| POST 侧通道 | 大型报告（如完整语料扫描） | `scripts/report-server.ts` + `publishNavigationPhase('posting')` |

---

## 具体技术实现

### 关键数据结构

```typescript
// 导航阶段类型
type NavigationPhase = 'loading' | 'measuring' | 'posting'

// 阶段状态对象
type NavigationPhaseState = {
  phase: NavigationPhase
  requestId?: string  // 可选的请求标识，用于请求-响应关联
}
```

### 核心函数实现

#### 1. Hash 参数解析器

```typescript
function getHashParams(urlOrHash: string): URLSearchParams {
  const hashStart = urlOrHash.indexOf('#')
  const hash = hashStart === -1 ? urlOrHash : urlOrHash.slice(hashStart + 1)
  return new URLSearchParams(hash)
}
```

**设计要点**：
- 兼容完整 URL 和纯 hash 字符串输入
- 使用浏览器原生 `URLSearchParams` 解析，无需额外依赖

#### 2. 阶段状态编码

```typescript
export function buildNavigationPhaseHash(phase: NavigationPhase, requestId?: string): string {
  const params = new URLSearchParams()
  params.set('phase', phase)
  if (requestId !== undefined) {
    params.set('requestId', requestId)
  }
  return `#${params.toString()}`
}
```

**输出示例**：
- `#phase=loading&requestId=1234567890-abc123`
- `#phase=measuring`

#### 3. 阶段状态解码

```typescript
export function readNavigationPhaseState(urlOrHash: string): NavigationPhaseState | null {
  const params = getHashParams(urlOrHash)
  const phase = params.get('phase')
  // 严格校验 phase 值，防止无效状态
  if (phase !== 'loading' && phase !== 'measuring' && phase !== 'posting') {
    return null
  }
  const requestId = params.get('requestId') ?? undefined
  return requestId === undefined ? { phase } : { phase, requestId }
}
```

**安全设计**：
- 严格的阶段值白名单校验，拒绝非法值
- 返回 `null` 表示无有效状态，而非抛出异常

#### 4. 报告数据编码

```typescript
export function buildNavigationReportHash(report: unknown): string {
  const params = new URLSearchParams()
  params.set('report', JSON.stringify(report))
  return `#${params.toString()}`
}
```

**限制**：
- 报告数据通过 JSON 序列化后存入 hash
- 受 URL 长度限制（通常 2KB-8KB），适合小型报告

#### 5. 报告数据解码

```typescript
export function readNavigationReportText(urlOrHash: string): string {
  return getHashParams(urlOrHash).get('report') ?? ''
}
```

**返回值**：
- 报告 JSON 字符串（需调用方自行解析）
- 空字符串表示无报告数据

---

## 关键代码路径与文件引用

### 调用关系图

```
shared/navigation-state.ts (协议定义)
    │
    ├── pages/report-utils.ts (浏览器端封装)
    │       ├── publishNavigationPhase() → buildNavigationPhaseHash()
    │       ├── publishNavigationReport() → buildNavigationReportHash()
    │       └── clearNavigationReport()
    │
    └── scripts/browser-automation.ts (脚本端封装)
            ├── loadHashReport() → readNavigationPhaseState(), readNavigationReportText()
            ├── loadPostedReport() → readNavigationPhaseState()
            └── readLastNavigationPhase() → readNavigationPhaseState()
```

### 页面端使用路径

| 文件 | 使用方式 | 关键调用 |
|------|----------|----------|
| `pages/accuracy.ts` | 导入 `report-utils.ts` | `publishNavigationPhase('loading')`, `publishNavigationPhase('measuring')`, `publishReport()` |
| `pages/probe.ts` | 导入 `report-utils.ts` | `publishNavigationPhase('loading')`, `publishNavigationPhase('measuring')`, `publishReport()` |
| `pages/corpus.ts` | 导入 `report-utils.ts` | `publishNavigationPhase('loading')`, `publishNavigationPhase('measuring')`, `setReport()` |
| `pages/gatsby.ts` | 导入 `report-utils.ts` | `publishNavigationPhase('loading')`, `publishNavigationPhase('measuring')`, `setReport()` |
| `pages/benchmark.ts` | 导入 `report-utils.ts` | `publishNavigationPhase('loading')`, `publishNavigationPhase('measuring')`, `setReport()` |

### 脚本端使用路径

| 文件 | 使用方式 | 关键调用 |
|------|----------|----------|
| `scripts/browser-automation.ts` | 直接导入 | `readNavigationPhaseState`, `readNavigationReportText` |
| `scripts/accuracy-check.ts` | 通过 `browser-automation.ts` | `loadHashReport`, `loadPostedReport` |
| `scripts/corpus-check.ts` | 通过 `browser-automation.ts` | `loadHashReport` |
| `scripts/probe-check.ts` | 通过 `browser-automation.ts` | `loadHashReport` |
| `scripts/gatsby-check.ts` | 通过 `browser-automation.ts` | `loadHashReport` |
| `scripts/corpus-sweep.ts` | 通过 `browser-automation.ts` | `loadHashReport` |

---

## 依赖与外部交互

### 模块依赖

```typescript
// 无外部依赖
// 仅使用浏览器/Node.js 内置 API：URLSearchParams
```

### 运行时依赖

| 环境 | 依赖 API |
|------|----------|
| 浏览器 | `URLSearchParams`, `history.replaceState`, `location.href` |
| Node.js | `URLSearchParams` (global) |

### 协议交互流程

#### Hash 报告模式（小型报告）

```
1. 脚本端：session.navigate(url + '&report=1&requestId=xxx')
2. 页面端：执行测量逻辑
3. 页面端：publishNavigationPhase('measuring', requestId)
4. 页面端：publishNavigationReport(report) → URL hash 更新为 #report={json}
5. 脚本端：轮询 readLocationUrl()，检测到 report 参数后解析
```

#### POST 侧通道模式（大型报告）

```
1. 脚本端：启动临时 HTTP 服务器 (report-server.ts)
2. 脚本端：session.navigate(url + '&reportEndpoint=http://localhost:xxx/report')
3. 页面端：publishNavigationPhase('measuring', requestId)
4. 页面端：publishNavigationPhase('posting', requestId)
5. 页面端：fetch(reportEndpoint, { method: 'POST', body: reportJson })
6. 页面端：publishNavigationReport(navigationReport) // 仅确认信息
7. 脚本端：waitForReport() 解析 POST 数据
```

---

## 风险、边界与改进建议

### 已知风险

#### 1. URL 长度限制

**风险**：`buildNavigationReportHash` 将完整报告 JSON 存入 URL hash，当报告数据过大时可能超出浏览器 URL 长度限制（约 2KB-8KB，因浏览器而异）。

**缓解措施**：
- 项目已实施 POST 侧通道模式（`scripts/report-server.ts`）用于大型报告
- 页面端通过 `reportEndpoint` 参数检测是否使用侧通道

**代码体现**（`pages/accuracy.ts:121-140`）：
```typescript
function publishReport(report: AccuracyReport): void {
  if (reportEndpoint !== null) {
    // 大型报告使用 POST 侧通道
    publishNavigationPhase('posting', requestId)
    void (async () => {
      try {
        await fetch(reportEndpoint, { method: 'POST', body: reportJson })
        publishNavigationReport(toNavigationReport(report)) // 仅发送确认
      } catch { /* ... */ }
    })()
    return
  }
  publishNavigationReport(toNavigationReport(report)) // 小型报告直接走 hash
}
```

#### 2. 阶段状态竞争

**风险**：`posting` 阶段与 `measuring` 阶段之间可能存在竞态，如果 POST 请求失败，脚本端可能永远等待。

**缓解措施**：
- `scripts/browser-automation.ts` 中的 `loadPostedReport` 实现了超时机制（默认 60 秒）
- 超时错误包含最后观察到的阶段，便于诊断

#### 3. 多请求 ID 混淆

**风险**：如果浏览器页面被意外导航或刷新，可能残留旧的 `requestId`，导致脚本端接受错误的报告。

**缓解措施**：
- `readNavigationPhaseState` 检查 `requestId` 匹配
- 不匹配时返回 `null`，脚本端继续轮询

### 边界条件

| 边界 | 行为 |
|------|------|
| 无 hash | `readNavigationPhaseState` 返回 `null` |
| 非法 phase 值 | `readNavigationPhaseState` 返回 `null` |
| 无 report 参数 | `readNavigationReportText` 返回空字符串 |
| requestId 缺失 | 阶段对象不包含该字段 |
| URL 包含多个 # | 使用第一个 # 后的内容作为 hash |

### 改进建议

#### 1. 增加报告大小检测

在 `buildNavigationReportHash` 中添加大小检测，当序列化后的报告超过阈值时抛出警告或自动降级：

```typescript
export function buildNavigationReportHash(report: unknown): string {
  const json = JSON.stringify(report)
  if (json.length > 1500) {
    console.warn('Report may exceed URL length limits, consider using POST side channel')
  }
  // ...
}
```

#### 2. 增加阶段超时元数据

扩展 `NavigationPhaseState` 包含时间戳，帮助诊断超时问题：

```typescript
type NavigationPhaseState = {
  phase: NavigationPhase
  requestId?: string
  timestamp?: number  // 阶段进入时间
}
```

#### 3. 支持更多传输协议

考虑支持 `BroadcastChannel` 或 `SharedWorker` 用于同源场景下的更高效通信（需要评估浏览器兼容性）。

#### 4. 增加类型安全的报告解析

当前 `readNavigationReportText` 返回原始字符串，调用方需自行解析。可考虑增加泛型解析函数：

```typescript
export function parseNavigationReport<T>(urlOrHash: string): T | null {
  const text = readNavigationReportText(urlOrHash)
  if (!text) return null
  try {
    return JSON.parse(text) as T
  } catch {
    return null
  }
}
```

#### 5. 文档化协议规范

当前协议知识分散在代码注释中。建议创建独立的 `PROTOCOL.md` 文档，详细说明：
- Hash 参数规范
- 阶段转换图
- 错误处理约定
- 版本兼容性策略

---

## 附录：相关文件清单

### 直接依赖文件

| 文件 | 角色 |
|------|------|
| `shared/navigation-state.ts` | 本研究目标，协议定义 |
| `pages/report-utils.ts` | 浏览器端封装 |
| `scripts/browser-automation.ts` | 脚本端封装，核心消费者 |
| `scripts/report-server.ts` | POST 侧通道服务器 |

### 间接使用文件（页面）

- `pages/accuracy.ts` - 精度测试页面
- `pages/probe.ts` - 文本探测页面
- `pages/corpus.ts` - 语料库测试页面
- `pages/gatsby.ts` - Gatsby 文本测试页面
- `pages/benchmark.ts` - 性能基准测试页面

### 间接使用文件（脚本）

- `scripts/accuracy-check.ts` - 精度检查脚本
- `scripts/corpus-check.ts` - 语料库检查脚本
- `scripts/probe-check.ts` - 探测检查脚本
- `scripts/gatsby-check.ts` - Gatsby 检查脚本
- `scripts/corpus-sweep.ts` - 语料库扫描脚本
- `scripts/pre-wrap-check.ts` - pre-wrap 模式检查脚本
