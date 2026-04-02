# shared/ 目录研究报告

## 概述

`shared/navigation-state.ts` 是 Pretext 项目中唯一一个位于 `shared/` 目录的模块。该模块作为**浏览器自动化测试基础设施**的核心组件，负责在浏览器页面（pages/）和 Node.js 自动化脚本（scripts/）之间建立双向通信协议。

---

## 一、场景与职责

### 1.1 核心场景

该模块解决的核心问题是：**浏览器自动化测试中的跨进程状态同步与报告传输**。

在 Pretext 项目中，大量的准确性测试、语料库测试和基准测试需要在真实浏览器（Chrome、Safari、Firefox）中执行。测试页面（如 `pages/accuracy.ts`、`pages/corpus.ts`、`pages/probe.ts`）需要在浏览器内执行文本布局计算，然后将结果报告回 Node.js 自动化脚本。由于浏览器环境的沙盒限制和跨域安全策略，需要一种可靠的机制来传递这些报告。

### 1.2 主要职责

| 职责 | 说明 |
|------|------|
| **导航阶段追踪** | 定义并编码页面加载的三个阶段：`loading` → `measuring` → `posting` |
| **报告序列化** | 将复杂的 JavaScript 对象序列化为 URL hash 参数，供自动化脚本读取 |
| **双向解析** | 提供从 URL hash 解析阶段状态和报告内容的工具函数 |
| **请求关联** | 通过 `requestId` 机制确保报告与请求的一一对应，防止交叉污染 |

---

## 二、功能点目的

### 2.1 NavigationPhase 类型系统

```typescript
export type NavigationPhase = 'loading' | 'measuring' | 'posting'
```

这三个阶段精确描述了测试页面的生命周期：

- **`loading`**：页面初始化，准备测试环境
- **`measuring`**：执行实际的文本测量和布局计算
- **`posting`**：正在通过 side channel（POST 请求）发送大型报告

**目的**：让自动化脚本能够精确追踪页面状态，在超时或失败时提供诊断信息（如 "last phase: measuring"）。

### 2.2 Hash-based 报告协议

**设计决策**：使用 URL hash（而非 LocalStorage、postMessage 或 WebSocket）作为传输机制的原因：

1. **无跨域限制**：hash 变化不触发跨域安全检查
2. **可观察性**：自动化脚本可以通过 AppleScript（Safari/Chrome）或 BiDi（Firefox）直接读取 `location.href`
3. **原子性**：hash 更新是原子操作，不会出现半写状态
4. **兼容性**：所有浏览器都支持，无需额外权限

### 2.3 双重传输策略

模块支持两种报告传输模式：

| 模式 | 函数 | 适用场景 |
|------|------|----------|
| **Hash 传输** | `buildNavigationReportHash` / `readNavigationReportText` | 小型报告（< 几 KB），直接嵌入 URL |
| **POST Side Channel** | `buildNavigationPhaseHash` + HTTP POST | 大型报告（语料库批量测试结果），避免 URL 长度限制 |

---

## 三、具体技术实现

### 3.1 核心数据结构

```typescript
// 内部状态表示
type NavigationPhaseState = {
  phase: NavigationPhase
  requestId?: string  // 可选的请求标识符，用于关联请求与响应
}
```

### 3.2 URL Hash 编码协议

#### 阶段状态编码（用于进度通知）

```
#phase=measuring&requestId=1234567890-abc123
```

**编码函数**：`buildNavigationPhaseHash(phase, requestId?)`

#### 报告数据编码（用于结果传输）

```
#report={"status":"ready","matchCount":100,"mismatchCount":0}
```

**编码函数**：`buildNavigationReportHash(report)`

**关键实现细节**：
- 使用 `URLSearchParams` 进行规范的 URL 编码
- 报告 JSON 直接作为 `report` 参数值嵌入
- 不处理 URL 长度限制，由调用方决定使用哪种传输模式

### 3.3 解析流程

```typescript
function getHashParams(urlOrHash: string): URLSearchParams {
  const hashStart = urlOrHash.indexOf('#')
  const hash = hashStart === -1 ? urlOrHash : urlOrHash.slice(hashStart + 1)
  return new URLSearchParams(hash)
}
```

**容错设计**：
- 接受完整 URL 或纯 hash 字符串
- 无 hash 时返回空参数对象
- 阶段值校验：只有 `'loading' | 'measuring' | 'posting'` 被识别为有效阶段

### 3.4 调用时序图

```
Node.js 脚本                    浏览器页面
    |                               |
    |--- 导航到测试页面 ------------>|
    |                               |
    |<-- #phase=loading -----------|
    |                               |
    |<-- #phase=measuring ---------|
    |                               |
    |         [执行测量]             |
    |                               |
    |<-- #report={...results...} ---|
    |                               |
    |--- 解析报告，验证 requestId ---|
    |                               |
```

对于大型报告（使用 POST side channel）：

```
Node.js 脚本                    浏览器页面              Report Server
    |                               |                       |
    |--- 导航到测试页面 ------------>|                       |
    |                               |                       |
    |<-- #phase=loading ------------------------------------|
    |                               |                       |
    |<-- #phase=posting ------------------------------------|
    |                               |                       |
    |                               |--- POST /report ------>|
    |                               |   body: JSON          |
    |                               |                       |
    |<-- #report={summary} ---------------------------------|
    |                               |                       |
    |--- 从 Report Server 获取完整报告 ----------------------|
    |                               |                       |
```

---

## 四、关键代码路径与文件引用

### 4.1 导出 API

| 导出 | 类型 | 使用者 | 用途 |
|------|------|--------|------|
| `NavigationPhase` | Type | `browser-automation.ts`, `pages/*.ts` | 类型安全地引用阶段值 |
| `buildNavigationPhaseHash` | Function | `pages/report-utils.ts` | 页面发布阶段更新 |
| `buildNavigationReportHash` | Function | `pages/report-utils.ts` | 页面发布最终报告 |
| `readNavigationPhaseState` | Function | `browser-automation.ts` | 脚本读取页面阶段 |
| `readNavigationReportText` | Function | `browser-automation.ts` | 脚本读取页面报告 |

### 4.2 调用链分析

#### 调用方（Pages）→ shared

```
pages/accuracy.ts
  └── import { publishNavigationPhase, publishNavigationReport } from './report-utils.ts'
      └── pages/report-utils.ts
            └── import { buildNavigationPhaseHash, buildNavigationReportHash } from '../shared/navigation-state.ts'

pages/corpus.ts
  └── 同上

pages/probe.ts
  └── 同上

pages/gatsby.ts
  └── 同上

pages/benchmark.ts
  └── 同上
```

#### 调用方（Scripts）→ shared

```
scripts/browser-automation.ts
  └── import { readNavigationPhaseState, readNavigationReportText, type NavigationPhase } from '../shared/navigation-state.ts'
      └── 被以下脚本使用：
          ├── scripts/accuracy-check.ts
          ├── scripts/corpus-check.ts
          ├── scripts/corpus-sweep.ts
          ├── scripts/probe-check.ts
          ├── scripts/pre-wrap-check.ts
          ├── scripts/benchmark-check.ts
          ├── scripts/gatsby-check.ts
          └── scripts/gatsby-sweep.ts
```

### 4.3 关键代码片段

**`shared/navigation-state.ts` 完整实现**（42 行）：

```typescript
export type NavigationPhase = 'loading' | 'measuring' | 'posting'

type NavigationPhaseState = {
  phase: NavigationPhase
  requestId?: string
}

function getHashParams(urlOrHash: string): URLSearchParams {
  const hashStart = urlOrHash.indexOf('#')
  const hash = hashStart === -1 ? urlOrHash : urlOrHash.slice(hashStart + 1)
  return new URLSearchParams(hash)
}

export function buildNavigationPhaseHash(phase: NavigationPhase, requestId?: string): string {
  const params = new URLSearchParams()
  params.set('phase', phase)
  if (requestId !== undefined) {
    params.set('requestId', requestId)
  }
  return `#${params.toString()}`
}

export function buildNavigationReportHash(report: unknown): string {
  const params = new URLSearchParams()
  params.set('report', JSON.stringify(report))
  return `#${params.toString()}`
}

export function readNavigationReportText(urlOrHash: string): string {
  return getHashParams(urlOrHash).get('report') ?? ''
}

export function readNavigationPhaseState(urlOrHash: string): NavigationPhaseState | null {
  const params = getHashParams(urlOrHash)
  const phase = params.get('phase')
  if (phase !== 'loading' && phase !== 'measuring' && phase !== 'posting') {
    return null
  }

  const requestId = params.get('requestId') ?? undefined
  return requestId === undefined ? { phase } : { phase, requestId }
}
```

---

## 五、依赖与外部交互

### 5.1 模块依赖图

```
shared/navigation-state.ts
    │
    ├── 被导入于 ──> pages/report-utils.ts
    │                  └── 被导入于 ──> pages/accuracy.ts
    │                  └── 被导入于 ──> pages/corpus.ts
    │                  └── 被导入于 ──> pages/probe.ts
    │                  └── 被导入于 ──> pages/gatsby.ts
    │                  └── 被导入于 ──> pages/benchmark.ts
    │
    └── 被导入于 ──> scripts/browser-automation.ts
                       └── 被导入于 ──> scripts/accuracy-check.ts
                       └── 被导入于 ──> scripts/corpus-check.ts
                       └── 被导入于 ──> scripts/corpus-sweep.ts
                       └── 被导入于 ──> scripts/probe-check.ts
                       └── 被导入于 ──> scripts/pre-wrap-check.ts
                       └── 被导入于 ──> scripts/benchmark-check.ts
                       └── 被导入于 ──> scripts/gatsby-check.ts
                       └── 被导入于 ──> scripts/gatsby-sweep.ts
```

### 5.2 运行时交互

#### 浏览器端运行时

- **环境**：浏览器 JavaScript 运行时
- **全局依赖**：`history.replaceState`, `location.pathname`, `location.search`
- **无外部库依赖**：纯原生 API 实现

#### Node.js 端运行时

- **环境**：Node.js + Bun 运行时
- **获取 URL 的方式**：
  - Safari/Chrome：通过 AppleScript 读取 `location.href`
  - Firefox：通过 WebSocket BiDi 协议执行 `script.evaluate` 获取 `location.href`

### 5.3 与 browser-automation.ts 的协作

`scripts/browser-automation.ts` 中的关键使用模式：

```typescript
// 读取当前阶段以提供超时诊断
async function readLastNavigationPhase(
  session: BrowserSession,
  expectedRequestId: string,
): Promise<NavigationPhase | null> {
  const currentUrl = await session.readLocationUrl()
  const phaseState = readNavigationPhaseState(currentUrl)
  if (phaseState === null) return null
  if (phaseState.requestId !== undefined && phaseState.requestId !== expectedRequestId) {
    return null
  }
  return phaseState.phase
}

// 在超时错误消息中包含最后阶段
function getTimeoutMessage(
  browser: BrowserKind,
  target: 'report' | 'posted report',
  lastPhase: NavigationPhase | null,
  observedUrl: string | null = null,
): string {
  if (lastPhase === null) {
    const locationLabel = observedUrl === null ? '' : `; last URL: ${observedUrl}`
    return `Timed out waiting for ${target} from ${browser} (no navigation feedback${locationLabel})`
  }
  return `Timed out waiting for ${target} from ${browser} (last phase: ${lastPhase})`
}
```

---

## 六、风险、边界与改进建议

### 6.1 已知风险

| 风险 | 严重程度 | 说明 |
|------|----------|------|
| **URL 长度限制** | 中 | 大型报告（如完整语料库扫描）可能超过浏览器 URL 长度限制（~2KB-64KB 不等）。项目已通过 POST side channel 模式缓解 |
| **JSON 序列化失败** | 低 | 报告对象包含循环引用或 BigInt 时 `JSON.stringify` 会失败。当前报告结构简单，风险较低 |
| **Hash 冲突** | 低 | 如果页面同时使用 hash 进行路由，可能产生冲突。当前测试页面无路由需求 |
| **编码兼容性** | 低 | `URLSearchParams` 对 Unicode 的处理在不同浏览器中可能有细微差异。当前主要使用 ASCII 键名，风险较低 |

### 6.2 边界情况

1. **空 hash 处理**：`readNavigationPhaseState('')` 返回 `null`
2. **无效 phase 值**：`readNavigationPhaseState('#phase=invalid')` 返回 `null`
3. **部分 hash**：`readNavigationPhaseState('phase=loading')` 正常工作（无前导 #）
4. **多余参数**：`readNavigationPhaseState('#phase=loading&foo=bar')` 忽略未知参数

### 6.3 改进建议

#### 短期（维护性改进）

1. **添加类型守卫**：
   ```typescript
   export function isNavigationPhase(value: unknown): value is NavigationPhase {
     return value === 'loading' || value === 'measuring' || value === 'posting'
   }
   ```

2. **文档化 URL 长度限制**：
   在 `buildNavigationReportHash` 函数注释中明确建议：报告大于 4KB 时应使用 POST side channel

3. **添加报告大小检查（开发模式）**：
   ```typescript
   export function buildNavigationReportHash(report: unknown): string {
     const json = JSON.stringify(report)
     if (process.env.NODE_ENV === 'development' && json.length > 4096) {
       console.warn('[navigation-state] Report exceeds 4KB, consider using POST side channel')
     }
     // ...
   }
   ```

#### 中期（功能扩展）

1. **支持压缩编码**：
   对于中等大小报告（4KB-32KB），可考虑使用 `lz-string` 等库进行 URL-safe 压缩，减少传输开销

2. **添加校验和**：
   为报告添加 CRC32 或简单校验和，检测 URL 截断或传输错误：
   ```typescript
   params.set('report', JSON.stringify(report))
   params.set('chksum', simpleChecksum(params.get('report')!))
   ```

3. **支持分片传输**：
   对于超大型报告，支持分片 hash 协议：
   ```
   #phase=chunk&index=0&total=3&data=...
   #phase=chunk&index=1&total=3&data=...
   ```

#### 长期（架构演进）

1. **考虑 WebSocket/BiDi 标准化**：
   Firefox 已使用 WebSocket BiDi 协议，Safari/Chrome 未来可能支持。可设计一个统一的传输抽象，逐步替代 hash-based 协议

2. **Service Worker 拦截**：
   使用 Service Worker 拦截 `fetch('/__pretext_report__')` 请求，将报告从页面直接发送到 Node.js 脚本，完全绕过 URL 限制

### 6.4 测试建议

当前项目未对 `shared/navigation-state.ts` 进行单元测试，建议添加：

```typescript
// src/navigation-state.test.ts（建议新建）
describe('navigation-state', () => {
  describe('buildNavigationPhaseHash', () => {
    it('encodes phase without requestId', () => {
      expect(buildNavigationPhaseHash('loading')).toBe('#phase=loading')
    })
    it('encodes phase with requestId', () => {
      expect(buildNavigationPhaseHash('measuring', 'req-123')).toBe('#phase=measuring&requestId=req-123')
    })
  })

  describe('readNavigationPhaseState', () => {
    it('parses valid phase', () => {
      expect(readNavigationPhaseState('#phase=posting')).toEqual({ phase: 'posting' })
    })
    it('returns null for invalid phase', () => {
      expect(readNavigationPhaseState('#phase=invalid')).toBeNull()
    })
    it('returns null for empty hash', () => {
      expect(readNavigationPhaseState('')).toBeNull()
    })
  })

  describe('round-trip', () => {
    it('preserves data through encode/decode', () => {
      const original = { status: 'ready', count: 42 }
      const hash = buildNavigationReportHash(original)
      const json = readNavigationReportText(hash)
      expect(JSON.parse(json)).toEqual(original)
    })
  })
})
```

---

## 七、总结

`shared/navigation-state.ts` 是 Pretext 浏览器自动化测试基础设施的**关键纽带**。尽管只有 42 行代码，它承担了跨进程通信协议的核心职责：

1. **简洁性**：使用 URL hash 作为传输机制，无需额外依赖
2. **可靠性**：三阶段状态机 + requestId 关联，确保测试可追踪
3. **扩展性**：支持 hash 和 POST 两种传输模式，适应不同报告大小

该模块的设计体现了 Pretext 项目的工程哲学：**最小化依赖，最大化可观察性**。所有使用场景都经过生产环境验证（Chrome、Safari、Firefox 的自动化测试），是当前测试流程的稳定基石。
