# pages/probe.ts 研究文档

## 场景与职责

`probe.ts` 是 Pretext 项目的**文本探针（Text Probe）**诊断工具的核心逻辑实现。它提供一个轻量级、参数化的测试框架，用于对比 Pretext 库的布局预测与浏览器实际渲染结果，特别适用于：

1. **针对性调试**：快速验证特定文本片段、特定宽度下的布局行为
2. **多语言测试**：支持阿拉伯语（RTL）、英语（LTR）等多种语言
3. **提取器对比**：同时测试 Range 和 Span 两种浏览器行提取方法
4. **自动化集成**：通过 URL 参数驱动，供浏览器自动化脚本调用

与 `gatsby.ts` 的完整语料测试相比，`probe.ts` 更轻量、更灵活，适合**假设验证**和**回归测试**。

## 功能点目的

### 1. 参数化测试配置
通过 URL 参数完全控制测试行为：
- `text`：测试文本内容
- `width`：容器宽度（默认 600px，最小 100px）
- `font`：字体配置（默认 `18px serif`）
- `lineHeight`：行高（默认 32px）
- `dir`：文本方向（`ltr` 或 `rtl`）
- `lang`：语言代码（自动根据方向推断）
- `method`：浏览器行提取方法（`range` 或 `span`）
- `whiteSpace`：空白处理模式（`normal` 或 `pre-wrap`）
- `verbose`：详细输出模式

### 2. 双方法浏览器行提取
- **Range 方法**：使用 `document.createRange()` 和 `getClientRects()` 提取行信息
- **Span 方法**：使用 `<span>` 元素包裹每个单元，通过 `getBoundingClientRect()` 检测行切换
- **提取器敏感性检测**：自动识别哪种方法更适合当前浏览器/文本组合

### 3. 详细差异分析
- 首个断行不匹配（first break mismatch）定位
- 差异原因分类（溢出、段和漂移、断行策略差异等）
- 上下文格式化显示（前后 24 字符）

### 4. 报告发布
- 通过 `window.__PROBE_REPORT__` 暴露结果
- 通过 URL hash 发布导航报告
- 支持 `requestId` 追踪

## 具体技术实现

### 核心数据结构

```typescript
// 探针报告
type ProbeReport = {
  status: 'ready' | 'error'
  requestId?: string
  text?: string
  whiteSpace?: 'normal' | 'pre-wrap'
  width?: number
  contentWidth?: number
  font?: string
  lineHeight?: number
  direction?: string
  browserLineMethod?: 'range' | 'span'
  predictedHeight?: number
  actualHeight?: number
  diffPx?: number
  predictedLineCount?: number
  browserLineCount?: number
  firstBreakMismatch?: ProbeBreakMismatch | null
  alternateBrowserLineMethod?: 'range' | 'span'
  alternateBrowserLineCount?: number
  alternateFirstBreakMismatch?: ProbeBreakMismatch | null
  extractorSensitivity?: string | null  // 提取器敏感性
  ourLines?: ProbeLineSummary[]
  browserLines?: ProbeLineSummary[]
  alternateBrowserLines?: ProbeLineSummary[]
}

// 断行不匹配详情
type ProbeBreakMismatch = {
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
}
```

### 关键流程

#### 1. 初始化流程
```
init()
  ├── 发布导航阶段 'measuring'
  ├── 配置 DOM 元素（book, diagnosticDiv）
  ├── 准备文本（prepareWithSegments）
  ├── Pretext 布局计算（layout）
  ├── 浏览器行提取（主方法）
  ├── 浏览器行提取（备用方法）
  ├── 对比分析（getFirstBreakMismatch）
  ├── 提取器敏感性检测
  └── 发布报告
```

#### 2. Span 方法浏览器行提取
```typescript
function getBrowserLinesFromSpans(prepared, measuredFont, dir): ProbeLine[] {
  // 1. 清空诊断容器
  diagnosticDiv.textContent = ''
  
  // 2. 为每个诊断单元创建 span
  for (const unit of units) {
    const span = document.createElement('span')
    span.textContent = unit.text
    diagnosticDiv.appendChild(span)
    spans.push(span)
  }
  
  // 3. 遍历检测行切换（基于 top 坐标变化）
  for (let i = 0; i < units.length; i++) {
    const rect = spans[i]!.getBoundingClientRect()
    const top = rect.width > 0 || rect.height > 0 ? rect.top : lastTop
    
    if (top !== null && lastTop !== null && top > lastTop + 0.5) {
      // 检测到新行
      pushLine()
      // 开始新行
    }
  }
  
  // 4. 恢复原始文本
  diagnosticDiv.textContent = text
  return lines
}
```

#### 3. Range 方法浏览器行提取
```typescript
function getBrowserLinesFromRange(prepared, measuredFont, dir): ProbeLine[] {
  // 1. 获取文本节点
  const textNode = diagnosticDiv.firstChild
  if (!(textNode instanceof Text)) return lines
  
  // 2. 创建 Range
  const range = document.createRange()
  
  // 3. 遍历诊断单元
  for (const unit of units) {
    range.setStart(textNode, unit.start)
    range.setEnd(textNode, unit.end)
    const rects = range.getClientRects()
    const top = rects.length > 0 ? rects[0]!.top : lastTop
    
    // 4. 检测行切换
    if (top !== null && lastTop !== null && top > lastTop + 0.5) {
      pushLine()
    }
  }
  
  return lines
}
```

#### 4. 差异原因分类
```typescript
function classifyBreakMismatch(contentWidth, ours, browser): string {
  const longer = ours.contentEnd >= browser.contentEnd ? ours : browser
  const longerLabel = longer === ours ? 'ours' : 'browser'
  const overflow = longer.fullWidth - contentWidth
  
  // 情况 1：微小溢出（容差内）
  if (Math.abs(overflow) <= 0.05) {
    return `${longerLabel} keeps text with only ${overflow.toFixed(3)}px overflow`
  }
  
  // 情况 2：段和漂移
  const oursDrift = (ours.sumWidth ?? ours.fullWidth) - ours.fullWidth
  if (Math.abs(oursDrift) > 0.05) {
    return `our segment sum drifts from full-string width by ${oursDrift.toFixed(3)}px`
  }
  
  // 情况 3：浏览器能容纳更长行
  if (browser.contentEnd > ours.contentEnd && browser.fullWidth <= contentWidth) {
    return 'browser fits the longer line while our break logic cuts earlier'
  }
  
  // 情况 4：标点或 shaping 上下文差异
  return 'different break opportunity around punctuation or shaping context'
}
```

### 关键算法

#### 偏移量计算（从 cursor 到文本偏移）
```typescript
function computeOffsetFromCursor(prepared, cursor): number {
  let offset = 0
  // 累加前面所有段的长度
  for (let i = 0; i < cursor.segmentIndex; i++) {
    offset += prepared.segments[i]!.length
  }
  if (cursor.graphemeIndex === 0) return offset
  
  // 在当前段内累加字素
  let graphemeIndex = 0
  for (const grapheme of graphemeSegmenter.segment(prepared.segments[cursor.segmentIndex]!)) {
    if (graphemeIndex >= cursor.graphemeIndex) break
    offset += grapheme.segment.length
    graphemeIndex++
  }
  return offset
}
```

#### 段切片测量
```typescript
function measurePreparedSlice(prepared, start, end, measuredFont): number {
  let total = 0
  let offset = 0
  
  for (let i = 0; i < prepared.segments.length; i++) {
    const segment = prepared.segments[i]!
    const nextOffset = offset + segment.length
    
    // 跳过完全在范围外的段
    if (nextOffset <= start) {
      offset = nextOffset
      continue
    }
    if (offset >= end) break
    
    // 计算重叠部分
    const overlapStart = Math.max(start, offset)
    const overlapEnd = Math.min(end, nextOffset)
    
    // 使用完整段宽度或测量子串
    if (localStart === 0 && localEnd === segment.length) {
      total += prepared.widths[i]!
    } else {
      total += measureCanvasTextWidth(diagnosticCtx, segment.slice(localStart, localEnd), measuredFont)
    }
  }
  
  return total
}
```

## 关键代码路径与文件引用

### 导入依赖
```typescript
// 核心布局 API
import { layout, layoutWithLines, prepareWithSegments } from '../src/layout.ts'

// 诊断工具函数
import {
  formatBreakContext,
  getDiagnosticUnits,
  getLineContent,
  measureCanvasTextWidth,
  measureDomTextWidth,
} from './diagnostic-utils.ts'

// 导航报告工具
import { clearNavigationReport, publishNavigationPhase, publishNavigationReport } from './report-utils.ts'
```

### URL 参数解析
```typescript
const params = new URLSearchParams(location.search)
const requestId = params.get('requestId') ?? undefined
const text = params.get('text') ?? ''
const width = Math.max(100, Number.parseInt(params.get('width') ?? '600', 10))
const font = params.get('font') ?? '18px serif'
const lineHeight = Math.max(1, Number.parseInt(params.get('lineHeight') ?? '32', 10))
const direction = params.get('dir') === 'rtl' ? 'rtl' : 'ltr'
const lang = params.get('lang') ?? (direction === 'rtl' ? 'ar' : 'en')
const browserLineMethod = params.get('method') === 'span' ? 'span' : 'range'
const verbose = params.get('verbose') === '1'
const whiteSpace = params.get('whiteSpace') === 'pre-wrap' ? 'pre-wrap' : 'normal'
```

### 诊断容器配置
```typescript
const diagnosticDiv = document.createElement('div')
diagnosticDiv.style.position = 'absolute'
diagnosticDiv.style.top = '-99999px'
diagnosticDiv.style.left = '-99999px'
diagnosticDiv.style.visibility = 'hidden'
diagnosticDiv.style.whiteSpace = cssWhiteSpace
// ... 其他样式
```

## 依赖与外部交互

### 内部模块依赖

| 模块 | 用途 |
|------|------|
| `../src/layout.ts` | 核心布局 API：`prepareWithSegments`, `layout`, `layoutWithLines` |
| `./diagnostic-utils.ts` | 诊断工具：段提取、行内容获取、Canvas/DOM 测量 |
| `./report-utils.ts` | 导航状态报告：阶段发布、报告发布 |

### DOM 元素依赖
| 元素 ID | 来源 | 用途 |
|---------|------|------|
| `stats` | `probe.html` | 显示统计信息 |
| `book` | `probe.html` | 文本渲染区域 |

### 浏览器 API 使用
- `document.createRange()` / `Range.setStart()` / `Range.setEnd()`：Range 方法行提取
- `Range.getClientRects()`：获取文本片段位置信息
- `getBoundingClientRect()`：获取元素/范围位置信息
- `Intl.Segmenter`：字素级文本分割
- `CanvasRenderingContext2D.measureText()`：Canvas 文本测量

### 与 gatsby.ts 的差异

| 特性 | probe.ts | gatsby.ts |
|------|----------|-----------|
| 测试范围 | 单片段 | 完整语料 |
| 参数来源 | URL 参数 | 滑块 + URL 参数 |
| 提取器方法 | Range + Span 双方法 | 主要 Range |
| 交互性 | 无（一次性） | 实时滑块交互 |
| 批量扫描 | 不支持 | 支持 |
| RTL 支持 | 完整支持 | 有限 |
| pre-wrap 支持 | 支持 | 不支持 |

## 风险、边界与改进建议

### 当前风险

1. **提取器方法选择**
   - Range 方法在 Safari 处理 URL 查询字符串时可能过度推进
   - Span 方法在东南亚文字（泰语、老挝语等）可能扰乱行断
   - **缓解**：双方法对比 + 提取器敏感性检测

2. **字体加载时序**
   - 依赖 `document.fonts.ready` 确保字体加载
   - 自定义字体未加载完成时测量可能不准确
   - **缓解**：等待 `document.fonts.ready` 后执行初始化

3. **固定内边距**
   - 硬编码 `PADDING = 40`，与 `probe.html` 的 CSS 耦合
   - 两边不一致会导致测量偏差

4. **行检测阈值**
   - 使用 `0.5px` 作为行切换阈值
   - 在特殊缩放情况下可能误判

### 边界情况

1. **空文本**：`text=''` 时返回空结果
2. **极窄宽度**：依赖 `layoutWithLines` 的 grapheme 级断行
3. **超长文本**：无分页处理，可能内存压力
4. **特殊字符**：依赖 `Intl.Segmenter` 正确处理

### 改进建议

1. **动态阈值**
   ```typescript
   const LINE_THRESHOLD = 0.5 * (window.devicePixelRatio || 1)
   ```

2. **字体加载超时**
   ```typescript
   const fontTimeout = new Promise((_, reject) => 
     setTimeout(() => reject(new Error('Font load timeout')), 5000)
   )
   await Promise.race([document.fonts.ready, fontTimeout])
   ```

3. **配置同步机制**
   - 通过 CSS 自定义属性或数据属性共享 padding 配置
   - 运行时读取实际计算样式

4. **更多提取器方法**
   - 添加 `intersection` 方法（Intersection Observer）
   - 添加 `caret` 方法（CaretPosition API）

5. **可视化增强**
   - 添加行边界高亮显示
   - 添加差异字符标记
   - 添加宽度标尺

6. **性能优化**
   - 缓存已测量的段切片
   - 复用 DOM 元素而非重复创建

7. **错误处理**
   - 添加更详细的错误分类
   - 添加恢复建议
