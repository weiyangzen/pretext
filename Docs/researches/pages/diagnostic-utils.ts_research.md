# diagnostic-utils.ts 研究文档

## 场景与职责

`pages/diagnostic-utils.ts` 是 Pretext 浏览器测试页面的共享诊断工具库。它提供了一系列底层工具函数，用于：

- **诊断单元生成**：将预处理后的文本分割为可用于浏览器行提取的细粒度单元
- **行内容处理**：处理行尾空白字符的修剪逻辑
- **断点上下文格式化**：生成断点周围的文本上下文，便于调试
- **文本宽度测量**：提供 Canvas 和 DOM 两种宽度测量方法

该模块被 `accuracy.ts`、`corpus.ts`、`gatsby.ts` 等多个测试页面共享，是诊断基础设施的核心组件。

## 功能点目的

### 1. 诊断单元生成 (`getDiagnosticUnits`)

将 `PreparedTextWithSegments` 转换为可用于浏览器行提取的诊断单元：

```typescript
export type DiagnosticUnit = {
  text: string   // 单元文本
  start: number  // 在原始文本中的起始偏移
  end: number    // 在原始文本中的结束偏移
}
```

**处理逻辑**：
- 对于可断行段（`breakableWidths !== null`）：拆分为字素（grapheme）级别
- 对于不可断行段：保持完整段作为单个单元

**目的**：
- 允许通过 Range API 精确定位每个单元在 DOM 中的渲染位置
- 通过检测单元位置的垂直变化识别换行点

### 2. 行内容处理 (`getLineContent`)

处理行尾空白字符：

```typescript
export function getLineContent(text: string, end: number): { text: string, end: number } {
  const trimmed = text.trimEnd()
  return {
    text: trimmed,
    end: end - (text.length - trimmed.length),  // 调整结束偏移
  }
}
```

**目的**：
- CSS 的 `white-space: normal` 会折叠行尾空白
- 需要调整结束偏移以匹配实际可见内容

### 3. 断点上下文格式化 (`formatBreakContext`)

生成断点周围的文本上下文，便于调试：

```typescript
export function formatBreakContext(text: string, breakOffset: number, radius = 32): string {
  const start = Math.max(0, breakOffset - radius)
  const end = Math.min(text.length, breakOffset + radius)
  return `${start > 0 ? '…' : ''}${text.slice(start, breakOffset)}|${text.slice(breakOffset, end)}${end < text.length ? '…' : ''}`
}
// 示例输出: "…hello wor|ld, how are…"
```

### 4. 文本宽度测量

提供两种宽度测量方法，用于对比分析：

**Canvas 测量** (`measureCanvasTextWidth`)：
```typescript
export function measureCanvasTextWidth(ctx: CanvasRenderingContext2D, text: string, font: string): number {
  ctx.font = font
  return ctx.measureText(text).width
}
```

**DOM 测量** (`measureDomTextWidth`)：
```typescript
export function measureDomTextWidth(doc: Document, text: string, font: string, direction: string): number {
  const span = doc.createElement('span')
  span.style.cssText = `
    position: absolute; visibility: hidden; white-space: pre;
    font: ${font}; direction: ${direction}; unicode-bidi: plaintext;
  `
  span.textContent = text
  doc.body.appendChild(span)
  const width = span.getBoundingClientRect().width
  doc.body.removeChild(span)
  return width
}
```

**用途对比**：

| 方法 | 优点 | 缺点 | 用途 |
|------|------|------|------|
| Canvas | 快速、无 DOM 开销 | 可能与 DOM 有差异（emoji） | 快速测量、段宽累加 |
| DOM | 与浏览器一致 | 需要创建/销毁元素 | 验证、漂移检测 |

## 具体技术实现

### 字素分割

使用 `Intl.Segmenter` 进行字素级分割：

```typescript
const diagnosticGraphemeSegmenter = new Intl.Segmenter(undefined, { granularity: 'grapheme' })

for (const grapheme of diagnosticGraphemeSegmenter.segment(segment)) {
  units.push({
    text: grapheme.segment,
    start: offset + grapheme.index,
    end: offset + grapheme.index + grapheme.segment.length
  })
}
```

### 单元生成算法

```
输入: PreparedTextWithSegments
输出: DiagnosticUnit[]

for each segment in prepared.segments:
  if prepared.breakableWidths[i] !== null:
    // 可断行段：拆分为字素
    for each grapheme in segment:
      create unit with grapheme text and offset
  else:
    // 不可断行段：保持完整
    create unit with full segment text and offset
```

## 关键代码路径与文件引用

### 依赖图

```
diagnostic-utils.ts
└── ../src/layout.ts
    └── PreparedTextWithSegments (类型)
```

### 消费者

| 文件 | 使用的函数 | 用途 |
|------|-----------|------|
| `accuracy.ts` | `getDiagnosticUnits` | 浏览器行提取 |
| `corpus.ts` | `getDiagnosticUnits`, `formatBreakContext`, `getLineContent`, `measureCanvasTextWidth`, `measureDomTextWidth` | 深度诊断 |
| `gatsby.ts` | `getDiagnosticUnits`, `formatBreakContext`, `getLineContent`, `measureCanvasTextWidth` | Gatsby 测试诊断 |

## 依赖与外部交互

### 浏览器 API

| API | 用途 |
|-----|------|
| `Intl.Segmenter` | 字素级文本分割 |
| `CanvasRenderingContext2D.measureText()` | Canvas 宽度测量 |
| `document.createElement('span')` | DOM 测量容器创建 |
| `span.getBoundingClientRect()` | DOM 宽度测量 |

### 类型依赖

```typescript
import type { PreparedTextWithSegments } from '../src/layout.ts'
```

## 风险、边界与改进建议

### 已知风险

1. **字素分割性能**：
   - `Intl.Segmenter` 在长文本上可能有性能开销
   - 当前在诊断路径使用，非热路径

2. **DOM 测量开销**：
   - 每次测量需要创建/销毁 DOM 元素
   - 频繁调用可能影响性能

3. **Canvas/DOM 差异**：
   - emoji 在 Canvas 和 DOM 中宽度可能不同
   - 需要 `measurement.ts` 中的 emoji 修正

### 边界情况处理

| 场景 | 处理 |
|------|------|
| 空文本 | `trimEnd()` 安全处理 |
| 全空白文本 | `trimEnd()` 返回空字符串 |
| 超长文本 | `formatBreakContext` 使用半径限制 |
| RTL 文本 | DOM 测量使用 `direction` 参数 |

### 改进建议

1. **性能优化**：
   - DOM 测量添加缓存机制
   - 批量测量减少 DOM 操作

2. **功能扩展**：
   - 添加高度测量函数
   - 添加行盒模型测量（含 padding/border）

3. **精度提升**：
   - 集成更多测量方法（如 OffscreenCanvas）
   - 添加亚像素精度选项
