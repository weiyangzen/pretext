# corpus.ts 研究文档

## 场景与职责

`pages/corpus.ts` 是 Pretext 库的多语言语料库压力测试引擎。它提供深度诊断能力，用于分析长文本、复杂脚本（阿拉伯语、乌尔都语、泰语等）在特定宽度下的布局行为。与 `accuracy.ts` 的批量扫描不同，此模块专注于：

- **行级精确对比**：逐行对比 Pretext 预测与浏览器实际渲染
- **断点诊断**：定位首个不匹配断点的精确位置和原因
- **提取器敏感性分析**：对比 span-probe 和 range 两种提取方法的结果
- **宽度漂移检测**：检测段宽累加与整行测量的差异

该模块支持 17 种语言的语料库，是验证复杂脚本布局准确性的关键工具。

## 功能点目的

### 1. 语料库管理

支持 17 种语言的语料库，元数据来自 `corpora/sources.json`：

```typescript
type CorpusMeta = {
  id: string                    // 如 'ja-rashomon'
  language: string              // ISO 语言代码
  direction?: 'ltr' | 'rtl'     // 文本方向
  title: string                 // 显示标题
  font_family?: string          // 推荐字体
  font_size_px?: number         // 推荐字号
  line_height_px?: number       // 推荐行高
  default_width?: number        // 默认宽度
  min_width?: number            // 最小宽度
  max_width?: number            // 最大宽度
}
```

### 2. 双模式浏览器行提取

实现两种浏览器行提取方法，并自动选择最可靠的：

**Span-Probe 方法** (`getBrowserLinesFromSpans`)：
- 为每个诊断单元创建 `<span>`
- 通过 `span.getBoundingClientRect()` 检测换行
- 优点：对保留空格和硬断行更精确
- 限制：东南亚文字可能受 span 影响

**Range 方法** (`getBrowserLinesFromRange`)：
- 使用 `document.createRange()` 选择文本范围
- 通过 `range.getClientRects()` 检测位置变化
- 优点：不改变 DOM 结构
- 限制：Safari 在 URL 查询文本上可能过度推进

**自动选择逻辑**：
```typescript
const requiresRangeProbe = rangeProbeScriptRe.test(normalizedText)  // 东南亚文字
const probeReliable = !requiresRangeProbe && direction !== 'rtl' && 
                      Math.abs(probeHeight - normalizedHeight) <= threshold
const browserResult = forcedMethod ?? (probeReliable ? probeResult : rangeResult)
```

### 3. 深度断点诊断

当发现行不匹配时，生成详细的断点诊断报告：

```typescript
type CorpusBreakMismatch = {
  line: number                  // 不匹配行号
  oursStart/browserStart: number   // 行起始偏移
  oursEnd/browserEnd: number       // 行结束偏移
  oursText/browserText: string     // 行内容
  oursContext/browserContext: string  // 断点上下文（±32字符）
  deltaText: string             // 差异文本
  reasonGuess: string           // 原因推测
  oursSumWidth: number          // 段宽累加
  oursDomWidth: number          // DOM 测量
  oursFullWidth: number         // Canvas 整行测量
  oursSegments: Array<{...}>    // 行内段详情
}
```

### 4. 宽度漂移分析

检测段宽累加与整行测量的差异：

```typescript
const drift = (ours.sumWidth ?? ours.fullWidth) - ours.fullWidth
if (Math.abs(drift) > 0.05) {
  maxDriftLine = {
    line: i + 1,
    drift,
    text: ours.contentText,
    sumWidth: ours.sumWidth ?? ours.fullWidth,
    fullWidth: ours.fullWidth,
    domWidth: measureDomTextWidth(document, ours.contentText, font, direction),
    pairAdjustedWidth: measurePairAdjustedWidth(segments, font, direction),
    segments,
  }
}
```

### 5. 批量宽度扫描

支持通过 `?widths=a,b,c` 参数进行批量扫描：

```typescript
function runSweep(widths: number[]): void {
  const rows = widths.map(width => {
    const report = measureWidth(width, { publish: false, updateStats: false })
    return toSweepRow(report)
  })
  const exactCount = rows.filter(row => Math.round(row.diffPx) === 0).length
  // 生成汇总报告
}
```

## 具体技术实现

### 诊断单元生成

```typescript
function getDiagnosticUnits(prepared: PreparedTextWithSegments): DiagnosticUnit[] {
  const units: DiagnosticUnit[] = []
  let offset = 0

  for (let i = 0; i < prepared.segments.length; i++) {
    const segment = prepared.segments[i]!
    if (prepared.breakableWidths[i] !== null) {
      // 可断行段：拆分为字素
      for (const grapheme of diagnosticGraphemeSegmenter.segment(segment)) {
        units.push({ text: grapheme.segment, start, end })
      }
    } else {
      // 不可断行段：保持完整
      units.push({ text: segment, start: offset, end: offset + segment.length })
    }
    offset += segment.length
  }
  return units
}
```

### 光标偏移计算

将段索引+字素索引转换为文本偏移：

```typescript
function getCursorOffset(prepared: PreparedTextWithSegments, segmentIndex: number, graphemeIndex: number): number {
  let offset = 0
  for (let i = 0; i < segmentIndex; i++) {
    offset += prepared.segments[i]!.length
  }
  if (graphemeIndex === 0) return offset
  
  let localOffset = 0
  let localGraphemeIndex = 0
  for (const grapheme of diagnosticGraphemeSegmenter.segment(prepared.segments[segmentIndex]!)) {
    if (localGraphemeIndex === graphemeIndex) break
    localOffset += grapheme.segment.length
    localGraphemeIndex++
  }
  return offset + localOffset
}
```

### 断点原因分类

```typescript
function classifyBreakMismatch(contentWidth: number, ours: DiagnosticLine, browser: DiagnosticLine): string {
  const longer = ours.contentEnd >= browser.contentEnd ? ours : browser
  const overflow = longer.fullWidth - contentWidth
  
  if (Math.abs(overflow) <= 0.05) {
    return `${longerLabel} keeps text with only ${overflow.toFixed(3)}px overflow`
  }
  
  const oursDrift = (ours.sumWidth ?? ours.fullWidth) - ours.fullWidth
  if (Math.abs(oursDrift) > 0.05) {
    return `our segment sum drifts from full-string width by ${oursDrift.toFixed(3)}px`
  }
  
  if (browser.contentEnd > ours.contentEnd && browser.fullWidth <= contentWidth) {
    return 'browser fits the longer line while our break logic cuts earlier'
  }
  
  return 'different break opportunity around punctuation or shaping context'
}
```

### 成对调整宽度测量

检测字距调整（kerning）影响：

```typescript
function measurePairAdjustedWidth(segments: Segment[], font: string, direction: string): number {
  let total = 0
  for (const segment of segments) {
    total += segment.domWidth
  }
  // 加上相邻段的字距调整
  for (let i = 1; i < segments.length; i++) {
    const prev = segments[i - 1]!
    const next = segments[i]!
    total += measureDomTextWidth(document, prev.text + next.text, font, direction) 
           - prev.domWidth - next.domWidth
  }
  return total
}
```

## 关键代码路径与文件引用

### 依赖图

```
corpus.ts
├── ../src/layout.ts
│   ├── layout()
│   ├── layoutWithLines()
│   └── prepareWithSegments()
├── ./diagnostic-utils.ts
│   ├── formatBreakContext()
│   ├── getDiagnosticUnits()
│   ├── getLineContent()
│   ├── measureCanvasTextWidth()
│   └── measureDomTextWidth()
├── ./report-utils.ts
│   ├── clearNavigationReport()
│   ├── publishNavigationPhase()
│   └── publishNavigationReport()
├── ../corpora/sources.json        // 语料元数据
└── ../corpora/*.txt (17 个语料文件)
```

### 执行流程

```
init()
├── loadSources()              // 加载 sources.json
├── loadCorpus(selectedMeta)
│   ├── loadText(meta)         // 导入语料文本
│   ├── updateTitle(meta)
│   ├── configureControls(meta) // 设置滑块范围
│   ├── prepareWithSegments()  // 预处理文本
│   ├── applyTextSlice()       // 应用切片参数
│   ├── setupDiagnosticDivs()  // 设置诊断容器
│   └── measureWidth() / runSweep()
│       ├── layout()           // Pretext 预测
│       ├── book.getBoundingClientRect()  // DOM 实际
│       └── addDiagnostics()   // 深度诊断（full 模式）
│           ├── getOurLines()  // Pretext 行
│           ├── getBrowserLinesFromSpans()  // Span 提取
│           ├── getBrowserLinesFromRange()  // Range 提取
│           ├── compareLines() // 行对比
│           └── getFirstBreakMismatch()  // 断点诊断
```

## 依赖与外部交互

### 浏览器 API

| API | 用途 |
|-----|------|
| `document.createElement()` | 创建诊断容器 |
| `div.getBoundingClientRect()` | 高度测量 |
| `document.createRange()` | 精确文本范围选择 |
| `range.getClientRects()` | 获取渲染位置 |
| `canvas.getContext('2d')` | Canvas 文本宽度测量 |
| `ctx.measureText()` | 段宽测量 |
| `Intl.Segmenter` | 字素分割 |
| `document.fonts.ready` | 字体加载等待 |

### URL 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `id` | string | 语料 ID |
| `width` | number | 初始宽度 |
| `widths` | number[] | 批量扫描宽度列表 |
| `diagnostic` | 'full' \| 'light' | 诊断详细程度 |
| `method` | 'span' \| 'range' | 强制提取方法 |
| `font` | string | 覆盖字体 |
| `lineHeight` | number | 覆盖行高 |
| `sliceStart/sliceEnd` | number | 文本切片 |
| `reportEndpoint` | URL | 报告提交端点 |
| `requestId` | string | 请求标识 |

### 全局对象

```typescript
interface Window {
  __CORPUS_REPORT__?: CorpusReport
  __CORPUS_DEBUG__?: {
    corpusId: string
    font: string
    lineHeight: number
    padding: number
    direction: string
    width: number
    contentWidth: number
    getNormalizedText: () => string
    layoutWithLines: (width: number) => LayoutLinesResult
  }
}
```

## 风险、边界与改进建议

### 已知风险

1. **提取器敏感性**：
   - Safari Range 在 URL 查询文本上可能过度推进
   - 东南亚文字 span-probe 可能受 span 影响
   - RTL 文本需要 range 方法

2. **大语料性能**：
   - 超大文本（25 万字符）的 Range 操作可能缓慢
   - 完整诊断模式需要大量内存

3. **字体加载**：
   - 复杂脚本（乌尔都语 Nastaliq）需要特定字体
   - 字体回退可能影响测量精度

### 边界情况处理

| 场景 | 处理策略 |
|------|----------|
| 文本切片 | `sliceStart/sliceEnd` 参数支持子文本测试 |
| 空语料 | 加载时检查并报错 |
| 宽度越界 | 钳制到 `min_width/max_width` 范围 |
| 字体覆盖 | `font` 参数覆盖 `sources.json` 配置 |

### 改进建议

1. **性能优化**：
   - 超大语料启用虚拟滚动
   - 诊断结果缓存避免重复计算
   - Web Worker  offload 复杂分析

2. **诊断增强**：
   - 添加字形级（glyph-level）分析
   - 集成 HarfBuzz 对比（如可用）
   - 可视化断点位置标记

3. **自动化**：
   - 与 CI 集成，自动检测语料回归
   - 多浏览器并行测试
   - 结果趋势分析

4. **用户体验**：
   - 并排对比视图
   - 差异高亮显示
   - 导出诊断报告为 JSON/HTML

### 相关配置

- 行适配容差：`0.005`（Chromium/Gecko），`1/64`（Safari/WebKit）
- 诊断上下文半径：32 字符
- 提取器切换阈值：`lineHeight / 2`
