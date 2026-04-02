# pages/gatsby.ts 研究文档

## 场景与职责

`gatsby.ts` 是 Pretext 项目中用于浏览器端文本布局准确性验证的核心诊断页面脚本。它基于《了不起的盖茨比》英文语料（`corpora/en-gatsby-opening.txt`），提供一个交互式的可视化工具，用于对比 Pretext 库的布局预测与浏览器实际渲染结果之间的差异。

该文件是项目质量保证体系的关键组成部分，主要用于：
1. **人工验证**：开发者可通过滑块调整容器宽度，实时观察 Pretext 预测高度与浏览器实际高度的差异
2. **自动化测试**：支持批量宽度扫描（sweep）模式，通过 URL 参数驱动，供浏览器自动化脚本调用
3. **差异诊断**：在 `report=1` 模式下，详细分析首行不匹配（first mismatch）和断行差异（break mismatch）

## 功能点目的

### 1. 实时布局对比
- 通过滑块控制容器宽度（默认 200-1200px）
- 同时显示 Pretext 计算高度和浏览器 DOM 实际高度
- 计算并显示高度差异（像素级）

### 2. 批量扫描模式（Sweep Mode）
- 通过 `?widths=300,400,500` 参数指定多个测试宽度
- 一次性验证多个宽度下的布局准确性
- 输出统计信息：精确匹配数/总测试数

### 3. 详细诊断报告（Diagnostic Mode）
- 当 `?report=1` 启用时，逐行对比 Pretext 与浏览器的断行结果
- 识别首个不匹配的行号和内容差异
- 分析断行边界的详细上下文（前后文、字符偏移、段信息）

### 4. 浏览器行提取
- 使用 `Range.getClientRects()` 技术从 DOM 中提取浏览器实际断行
- 支持诊断单元（diagnostic units）级别的粒度分析

## 具体技术实现

### 核心数据结构

```typescript
// 报告类型定义
type GatsbyReport = {
  status: 'ready' | 'error'
  width?: number
  contentWidth?: number
  predictedHeight?: number      // Pretext 预测高度
  actualHeight?: number         // 浏览器实际高度
  diffPx?: number               // 高度差异
  predictedLineCount?: number   // 预测行数
  browserLineCount?: number     // 浏览器行数
  mismatchCount?: number        // 不匹配行数
  firstMismatch?: GatsbyLineMismatch | null
  firstBreakMismatch?: GatsbyBreakMismatch | null
  rows?: GatsbySweepRow[]       // 批量扫描结果
}

// 断行不匹配详情
type GatsbyBreakMismatch = {
  line: number
  start: number
  oursEnd: number               // Pretext 断点
  browserEnd: number            // 浏览器断点
  deltaText: string             // 差异文本
  oursContext: string           // Pretext 上下文
  browserContext: string        // 浏览器上下文
  reasonGuess: string           // 差异原因推测
  segmentWindow: GatsbyBreakSegment[]  // 周边段窗口
}
```

### 关键流程

#### 1. 初始化流程
```
init()
  ├── 等待字体加载 (document.fonts.ready)
  ├── 发布导航阶段 'measuring'
  ├── 如果有 requestedWidths → runSweep()
  └── 否则 → setWidth(initialWidth)
```

#### 2. 单宽度测量流程
```
measureWidth(width)
  ├── 设置滑块值和标签
  ├── Pretext 布局计算 (layout())
  ├── DOM 高度测量 (getBoundingClientRect())
  ├── 构建报告 (buildReport())
  ├── 更新统计信息
  └── 发布报告 (setReport())
```

#### 3. 浏览器行提取流程
```
getBrowserLines(preparedText, div)
  ├── 获取诊断单元 (getDiagnosticUnits)
  ├── 遍历每个单元
  │   ├── 使用 Range 获取位置信息
  │   ├── 检测行切换（top 坐标变化 > 0.5px）
  │   └── 累积行内容
  └── 返回 DiagnosticLine[]
```

#### 4. 断行不匹配分析流程
```
getFirstBreakMismatch(contentWidth, ourLines, browserLines)
  ├── 逐行对比 (start 和 contentEnd)
  ├── 发现不匹配时：
  │   ├── 计算差异文本 (deltaText)
  │   ├── 格式化上下文 (formatBreakContext)
  │   ├── 分类差异原因 (classifyBreakMismatch)
  │   ├── 描述边界位置 (describeBoundary)
  │   └── 获取周边段窗口 (getSegmentWindow)
  └── 返回 GatsbyBreakMismatch
```

### 关键算法

#### 空白规范化（Whitespace Normalization）
```typescript
function normalizeWhitespaceNormal(input: string): string {
  if (!needsWhitespaceNormalizationRe.test(input)) return input
  let normalized = input.replace(collapsibleWhitespaceRunRe, ' ')
  // 移除首尾空格
  if (normalized.charCodeAt(0) === 0x20) normalized = normalized.slice(1)
  if (normalized.length > 0 && normalized.charCodeAt(normalized.length - 1) === 0x20) {
    normalized = normalized.slice(0, -1)
  }
  return normalized
}
```

#### 差异原因分类
```typescript
function classifyBreakMismatch(contentWidth, ours, browser): string {
  const longer = ours.contentEnd >= browser.contentEnd ? ours : browser
  const overflow = longer.fullWidth - contentWidth
  
  if (Math.abs(overflow) <= 0.05) {
    return `${longerLabel} keeps text with only ${overflow.toFixed(3)}px overflow`
  }
  
  const oursDrift = (ours.sumWidth ?? ours.fullWidth) - ours.fullWidth
  if (Math.abs(oursDrift) > 0.05) {
    return `our segment sum drifts from full-string width by ${oursDrift.toFixed(3)}px`
  }
  
  // ... 其他情况
}
```

## 关键代码路径与文件引用

### 导入依赖
```typescript
// 核心布局 API
import { layout, layoutWithLines, prepareWithSegments } from '../src/layout.ts'

// 诊断工具函数
import { formatBreakContext, getDiagnosticUnits, getLineContent, measureCanvasTextWidth } from './diagnostic-utils.ts'

// 导航报告工具
import { clearNavigationReport, publishNavigationPhase, publishNavigationReport } from './report-utils.ts'
```

### 配置常量
```typescript
const FONT = '16px Georgia, "Times New Roman", serif'
const LINE_HEIGHT = 26
const PADDING = 40
const DEFAULT_WIDTH = 600
```

### 报告发布机制
- **Hash 报告**：小数据通过 URL hash 传递（`#report=...`）
- **POST 端点**：大数据通过 `reportEndpoint` POST 到本地服务器
- **导航阶段**：通过 `publishNavigationPhase` 报告加载阶段（loading/measuring/posting）

## 依赖与外部交互

### 内部模块依赖

| 模块 | 用途 |
|------|------|
| `../src/layout.ts` | 核心布局 API：`prepareWithSegments`, `layout`, `layoutWithLines` |
| `./diagnostic-utils.ts` | 诊断工具：段提取、行内容获取、Canvas 测量 |
| `./report-utils.ts` | 导航状态报告：阶段发布、报告发布 |
| `../shared/navigation-state.ts` | 导航状态类型定义（通过 report-utils 间接使用） |
| `../corpora/en-gatsby-opening.txt` | 测试语料文本 |

### DOM 元素依赖
```typescript
const book = document.getElementById('book')!           // 主显示区域
const slider = document.getElementById('slider')!       // 宽度滑块
const valLabel = document.getElementById('val')!        // 宽度标签
const stats = document.getElementById('stats')!         // 统计信息
```

### URL 参数接口
| 参数 | 类型 | 说明 |
|------|------|------|
| `report` | `1` | 启用详细诊断模式 |
| `diagnostic` | `'full'` \| `'none'` | 诊断详细程度 |
| `requestId` | string | 请求追踪 ID |
| `reportEndpoint` | URL | POST 报告的目标端点 |
| `widths` | `300,400,500` | 批量扫描宽度列表 |
| `width` | number | 初始宽度 |

### 浏览器 API 使用
- `document.fonts.ready`：等待字体加载完成
- `performance.now()`：高精度计时
- `Range.getClientRects()` / `Range.getBoundingClientRect()`：提取浏览器行信息
- `Intl.Segmenter`：字素分割（用于边界描述）
- `CanvasRenderingContext2D.measureText()`：Canvas 文本测量

## 风险、边界与改进建议

### 当前风险

1. **字体加载时序**
   - 依赖 `document.fonts.ready` 确保字体加载完成
   - 在不支持 `document.fonts` 的浏览器中可能产生测量偏差
   - **缓解**：代码已处理不支持的情况，直接调用 `init()`

2. **行检测阈值**
   - 使用 `0.5px` 作为行切换阈值：`rectTop > lastTop + 0.5`
   - 在极高 DPI 或特殊缩放情况下可能误判
   - **建议**：考虑基于设备像素比的动态阈值

3. **Range API 限制**
   - Safari 的 Range 提取在某些场景（如 URL 查询字符串）可能过度推进
   - **缓解**：`probe.ts` 提供了 `span` 方法作为替代，但 `gatsby.ts` 主要使用 Range

4. **空白规范化假设**
   - 硬编码 CSS `white-space: normal` 行为
   - 不支持 `pre-wrap` 模式的测试

### 边界情况

1. **空文本**：通过 `createEmptyPrepared` 处理，返回空结果
2. **极窄宽度**：依赖 `layoutWithLines` 的 grapheme 级断行处理
3. **CJK 文本**：依赖 `diagnostic-utils.ts` 中的诊断单元处理

### 改进建议

1. **增强 extractor 选择**
   - 参考 `probe.ts` 实现双方法（Range/Span）对比
   - 自动检测哪种方法更适合当前浏览器/文本组合

2. **性能优化**
   - 批量扫描时，可考虑复用 DOM 元素而非重复创建
   - 大型语料（如完整 Gatsby 文本）可能需要分页处理

3. **扩展性**
   - 支持 `whiteSpace: 'pre-wrap'` 模式测试
   - 支持 RTL 文本的专用诊断
   - 支持自定义语料（不仅限于 Gatsby）

4. **报告增强**
   - 添加可视化差异高亮（如行级颜色标记）
   - 支持导出 CSV/JSON 格式的完整报告
   - 添加趋势分析（多个宽度的差异模式）

5. **代码结构**
   - 将诊断逻辑提取为独立模块，便于复用
   - 添加单元测试覆盖核心诊断算法
