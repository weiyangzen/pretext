# pages/report-utils.ts 研究文档

## 场景与职责

`report-utils.ts` 是 Pretext 项目浏览器端诊断页面的**导航状态报告工具库**。它提供了一组轻量级函数，用于在浏览器自动化测试期间通过 URL hash 发布页面状态和测试结果。

该模块是浏览器自动化基础设施的关键组成部分，主要用于：
1. **阶段追踪**：报告页面加载阶段（loading → measuring → posting）
2. **结果传递**：将测试结果通过 URL hash 传递回自动化脚本
3. **状态同步**：支持 `requestId` 追踪，关联请求与响应

## 功能点目的

### 1. 导航阶段发布
- `publishNavigationPhase(phase, requestId)`：报告当前执行阶段
- 阶段定义：`'loading'` → `'measuring'` → `'posting'`
- 帮助自动化脚本检测页面状态和超时诊断

### 2. 报告发布
- `publishNavigationReport(report)`：将测试结果发布到 URL hash
- 支持任意可序列化的报告对象
- 自动化脚本通过 `readNavigationReportText` 读取结果

### 3. 报告清除
- `clearNavigationReport()`：清除 URL hash
- 在测试开始前调用，确保干净状态

## 具体技术实现

### 核心函数

#### 1. 发布导航阶段
```typescript
export function publishNavigationPhase(phase: NavigationPhase, requestId?: string): void {
  replaceNavigationHash(buildNavigationPhaseHash(phase, requestId))
}
```

**实现细节**：
- 调用 `buildNavigationPhaseHash` 构建 hash 字符串
- 使用 `history.replaceState` 更新 URL（不添加历史记录）

#### 2. 发布导航报告
```typescript
export function publishNavigationReport(report: unknown): void {
  replaceNavigationHash(buildNavigationReportHash(report))
}
```

**实现细节**：
- 接受任意类型的报告对象
- 通过 `JSON.stringify` 序列化
- 使用 URLSearchParams 编码

#### 3. 替换导航 Hash
```typescript
function replaceNavigationHash(hash: string): void {
  history.replaceState(null, '', `${location.pathname}${location.search}${hash}`)
}
```

**关键特性**：
- 使用 `history.replaceState` 而非 `location.hash`
- 不添加浏览器历史记录
- 保留 pathname 和 search 部分

### 数据结构

#### NavigationPhase 类型
```typescript
export type NavigationPhase = 'loading' | 'measuring' | 'posting'
```

| 阶段 | 含义 |
|------|------|
| `loading` | 页面加载中，等待资源 |
| `measuring` | 正在进行布局测量 |
| `posting` | 正在提交报告（POST 到端点） |

#### NavigationPhaseState
```typescript
type NavigationPhaseState = {
  phase: NavigationPhase
  requestId?: string
}
```

### Hash 构建格式

#### 阶段 Hash
```
#phase=measuring&requestId=abc123
```

#### 报告 Hash
```
#report={"status":"ready","width":600,...}
```

## 关键代码路径与文件引用

### 导入依赖
```typescript
import {
  buildNavigationPhaseHash,
  buildNavigationReportHash,
  type NavigationPhase,
} from '../shared/navigation-state.ts'
```

### 依赖模块

| 模块 | 路径 | 用途 |
|------|------|------|
| `navigation-state.ts` | `../shared/navigation-state.ts` | Hash 构建和解析函数 |

### 使用方

| 模块 | 路径 | 用途 |
|------|------|------|
| `gatsby.ts` | `./gatsby.ts` | 发布阶段和报告 |
| `probe.ts` | `./probe.ts` | 发布阶段和报告 |

### 调用示例

#### gatsby.ts 中的使用
```typescript
import { clearNavigationReport, publishNavigationPhase, publishNavigationReport } from './report-utils.ts'

// 初始化
clearNavigationReport()
publishNavigationPhase('loading', requestId)

// 测量开始
publishNavigationPhase('measuring', requestId)

// 测量完成，发布报告
publishNavigationReport(report)
```

#### probe.ts 中的使用
```typescript
import { clearNavigationReport, publishNavigationPhase, publishNavigationReport } from './report-utils.ts'

// 类似的使用模式
clearNavigationReport()
publishNavigationPhase('loading', requestId)
// ...
publishNavigationPhase('measuring', requestId)
// ...
publishNavigationReport(report)
```

## 依赖与外部交互

### 外部模块依赖

#### `../shared/navigation-state.ts`
```typescript
export type NavigationPhase = 'loading' | 'measuring' | 'posting'

export function buildNavigationPhaseHash(phase: NavigationPhase, requestId?: string): string
export function buildNavigationReportHash(report: unknown): string
export function readNavigationReportText(urlOrHash: string): string
export function readNavigationPhaseState(urlOrHash: string): NavigationPhaseState | null
```

### 浏览器 API 使用

| API | 用途 |
|-----|------|
| `history.replaceState` | 无历史记录更新 URL hash |
| `location.pathname` | 获取当前路径 |
| `location.search` | 获取查询字符串 |

### 与共享模块的关系

```
pages/
├── report-utils.ts          # 本文件（页面端发布工具）
└── ../shared/
    └── navigation-state.ts  # 共享状态管理（构建/解析）
```

`navigation-state.ts` 同时被以下模块使用：
- 浏览器端：`report-utils.ts`（发布）
- Node.js 端：自动化脚本（读取和解析）

## 风险、边界与改进建议

### 当前风险

1. **Hash 长度限制**
   - URL hash 长度受浏览器限制（通常数 KB 到数十 KB）
   - 大型报告可能超出限制，导致数据截断
   - **缓解**：大型报告使用 POST 端点（`reportEndpoint`）传输

2. **序列化错误**
   - `JSON.stringify` 可能失败（循环引用、BigInt 等）
   - 当前实现无错误处理
   - **风险**：报告发布失败，自动化脚本超时

3. **并发覆盖**
   - 多个异步操作可能同时调用 `replaceNavigationHash`
   - 后执行的调用会覆盖先调用的结果
   - **缓解**：调用方需确保顺序执行

4. **浏览器兼容性**
   - `history.replaceState` 在 IE9 及以下不支持
   - **缓解**：项目目标浏览器为现代浏览器，可接受

### 边界情况

1. **空报告**：`publishNavigationReport({})` 发布空对象
2. **undefined report**：`publishNavigationReport(undefined)` 发布 `"undefined"`
3. **特殊字符**：依赖 `URLSearchParams` 正确处理编码
4. **超大报告**：超出 hash 限制时行为未定义

### 改进建议

1. **添加错误处理**
   ```typescript
   export function publishNavigationReport(report: unknown): void {
     try {
       replaceNavigationHash(buildNavigationReportHash(report))
     } catch (error) {
       console.error('Failed to publish report:', error)
       // 发布错误状态
       replaceNavigationHash(buildNavigationReportHash({
         status: 'error',
         message: 'Failed to serialize report'
       }))
     }
   }
   ```

2. **添加大小检查**
   ```typescript
   export function publishNavigationReport(report: unknown): void {
     const hash = buildNavigationReportHash(report)
     if (hash.length > 4000) {  // 保守限制
       console.warn('Report too large for hash, consider using POST endpoint')
       // 截断或切换到 POST
     }
     replaceNavigationHash(hash)
   }
   ```

3. **队列化调用**
   ```typescript
   let pendingHash: string | null = null
   
   function replaceNavigationHash(hash: string): void {
     if (pendingHash !== null) {
       // 队列化，避免覆盖
       pendingHash = hash
       return
     }
     pendingHash = hash
     requestAnimationFrame(() => {
       history.replaceState(null, '', `${location.pathname}${location.search}${pendingHash}`)
       pendingHash = null
     })
   }
   ```

4. **添加调试模式**
   ```typescript
   const DEBUG = location.search.includes('debug=1')
   
   export function publishNavigationPhase(phase: NavigationPhase, requestId?: string): void {
     if (DEBUG) {
       console.log('[Navigation]', phase, requestId)
     }
     replaceNavigationHash(buildNavigationPhaseHash(phase, requestId))
   }
   ```

5. **考虑替代传输机制**
   - Broadcast Channel API（同源页面间通信）
   - Service Worker 消息传递
   - WebSocket（长期连接场景）

6. **类型安全增强**
   ```typescript
   // 为报告添加类型约束
   export interface NavigationReport {
     status: 'ready' | 'error'
     requestId?: string
     [key: string]: unknown
   }
   
   export function publishNavigationReport(report: NavigationReport): void {
     replaceNavigationHash(buildNavigationReportHash(report))
   }
   ```
