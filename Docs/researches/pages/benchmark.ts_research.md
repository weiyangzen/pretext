# benchmark.ts 研究文档

## 场景与职责

`pages/benchmark.ts` 是 Pretext 库的综合性能基准测试引擎。它执行多维度的性能测量，量化 Pretext 相对于 DOM 原生布局的性能优势，并评估不同 API 变体（热路径 vs 富路径）的性能特征。该模块是项目性能主张的技术基础，也是 CI/CD 流程中检测性能回归的关键工具。

核心测试维度：
- **基础 API 性能**：`prepare()` 冷路径 vs `layout()` 热路径
- **DOM 对比**：Batch vs Interleaved 写读模式
- **富 API 对比**：`layoutWithLines()` vs `walkLineRanges()` vs `layoutNextLine()`
- **Pre-wrap 模式**：硬断行、制表符、保留空白的性能
- **长文本压力**：阿拉伯语长文本的重复布局
- **多语言语料**：13 种语言的完整语料库压力测试
- **CJK vs Latin 扩展性**：字符数从 50 到 1000 的扩展性测试

## 功能点目的

### 1. 基础 API 基准 (`prepare()` / `layout()`)

测量核心两阶段 API 的性能：

```typescript
// prepare(): 冷路径，每文本一次
const tPrepare = bench(() => {
  clearCache()
  for (let i = 0; i < COUNT; i++) {
    prepare(texts[i]!, FONT)
  }
})

// layout(): 热路径，每次调整大小时调用
const tLayout = bench(repeatIndex => {
  const maxWidth = LAYOUT_SAMPLE_WIDTHS[repeatIndex % LAYOUT_SAMPLE_WIDTHS.length]!
  for (let i = 0; i < COUNT; i++) {
    const result = layout(prepared[i]!, maxWidth, LINE_HEIGHT)
    sum += result.height + result.lineCount
  }
}, LAYOUT_SAMPLE_REPEATS)  // 200 次重复
```

### 2. DOM 对比基准

创建真实 DOM 元素进行对比：

```typescript
// DOM Batch: 写入全部，然后读取全部
const tBatch = bench(() => {
  for (let i = 0; i < COUNT; i++) divs[i]!.style.width = `${WIDTH_AFTER}px`
  for (let i = 0; i < COUNT; i++) sum += divs[i]!.getBoundingClientRect().height
  for (let i = 0; i < COUNT; i++) divs[i]!.style.width = `${WIDTH_BEFORE}px`
})

// DOM Interleaved: 每个元素写后立即读（强制布局抖动）
const tInterleaved = bench(() => {
  for (let i = 0; i < COUNT; i++) {
    divs[i]!.style.width = `${WIDTH_AFTER}px`
    sum += divs[i]!.getBoundingClientRect().height  // 强制同步布局
  }
})
```

### 3. 富 API 基准 (`buildRichBenchmarks`)

对比三种富路径 API：

| API | 特点 | 典型用途 |
|-----|------|----------|
| `layoutWithLines()` | 物化行文本字符串 | 自定义渲染器 |
| `walkLineRanges()` | 仅几何信息，无字符串 | Shrinkwrap 计算 |
| `layoutNextLine()` | 流式逐行布局 | 虚拟滚动 |

### 4. 多语言语料基准 (`buildCorpusBenchmarks`)

测试 13 种语言的完整语料库：

```typescript
const CORPORA = [
  { id: 'ja-kumo-no-ito', label: 'Japanese prose', ... },
  { id: 'ko-unsu-joh-eun-nal', label: 'Korean prose', ... },
  { id: 'zh-zhufu', label: 'Chinese prose', ... },
  { id: 'th-nithan-vetal-story-1', label: 'Thai prose', ... },
  { id: 'ur-chughd', label: 'Urdu prose', ... },
  // ... 共 13 种
]
```

每个语料测量：
- `analysisMs`: 文本分析阶段
- `measureMs`: Canvas 测量阶段
- `prepareMs`: 完整准备（冷路径）
- `layoutMs`: 热路径布局（200 次重复）

### 5. CJK vs Latin 扩展性测试

测试字符数从 50 到 1000 的扩展性：

```typescript
const charSizes = [50, 100, 200, 500, 1000]
for (const n of charSizes) {
  // prepare (冷路径)
  // layout (1000 次重复取平均)
}
```

## 具体技术实现

### 基准测试框架

```typescript
function bench(
  fn: (repeatIndex: number) => void,
  sampleRepeats = 1,    // 每次运行的重复次数
  warmup = WARMUP,      // 预热次数 (2)
  runs = RUNS,          // 测量次数 (10)
): number {
  // 预热
  for (let i = 0; i < warmup; i++) runRepeated()
  
  // 测量
  const times: number[] = []
  for (let i = 0; i < runs; i++) {
    const t0 = performance.now()
    runRepeated()
    times.push((performance.now() - t0) / sampleRepeats)
  }
  
  return median(times)
}
```

### 关键常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `COUNT` | 500 | 每批次文本数 |
| `WARMUP` | 2 | 预热运行次数 |
| `RUNS` | 10 | 测量运行次数 |
| `LAYOUT_SAMPLE_REPEATS` | 200 | layout() 重复次数 |
| `LAYOUT_SAMPLE_WIDTHS` | [200,250,300,350,400] | 循环宽度 |
| `RICH_COUNT` | 60 | 富 API 测试文本数 |
| `CORPUS_LAYOUT_SAMPLE_REPEATS` | 200 | 语料布局重复次数 |

### 防优化技术

防止 JavaScript 引擎优化消除无副作用代码：

```typescript
let topLayoutSink = 0
// ...
topLayoutSink += sum + repeatIndex
// ...
root.dataset['topLayoutSink'] = String(topLayoutSink)
console.log('benchmark sinks', { topLayoutSink, ... })
```

### 报告数据结构

```typescript
type BenchmarkResult = { label: string, ms: number, desc: string }

type CorpusBenchmarkResult = {
  id: string
  label: string
  font: string
  chars: number
  analysisSegments: number
  segments: number
  breakableSegments: number
  width: number
  lineCount: number
  analysisMs: number
  measureMs: number
  prepareMs: number
  layoutMs: number
}

type BenchmarkReport = {
  status: 'ready' | 'error'
  requestId?: string
  results?: BenchmarkResult[]
  richResults?: BenchmarkResult[]
  richPreWrapResults?: BenchmarkResult[]
  richLongResults?: BenchmarkResult[]
  corpusResults?: CorpusBenchmarkResult[]
}
```

## 关键代码路径与文件引用

### 依赖图

```
benchmark.ts
├── ../src/layout.ts
│   ├── prepare() / prepareWithSegments()
│   ├── layout()
│   ├── layoutNextLine()
│   ├── layoutWithLines()
│   ├── walkLineRanges()
│   ├── clearCache()
│   └── profilePrepare()
├── ./report-utils.ts
│   ├── clearNavigationReport()
│   ├── publishNavigationPhase()
│   └── publishNavigationReport()
├── ../src/test-data.ts
│   └── TEXTS
└── ../corpora/*.txt (13 个语料文件)
```

### 执行流程

```
run()
├── 创建 DOM 容器和 500 个 div
├── 预热并强制初始布局
├── 预准备 500 个 PreparedText
├── benchmark prepare()
├── benchmark layout()
├── benchmark DOM batch
├── benchmark DOM interleaved
├── 清理 DOM 容器
├── benchmark 富 APIs (shared corpus)
├── benchmark 富 APIs (pre-wrap)
├── benchmark 富 APIs (Arabic long-form)
├── benchmark 多语言语料
├── CJK vs Latin 扩展性测试
└── 渲染结果表格
```

## 依赖与外部交互

### 浏览器 API

| API | 用途 |
|-----|------|
| `performance.now()` | 高精度计时 |
| `requestAnimationFrame()` | 状态更新帧同步 |
| `document.createElement()` | 创建测试 DOM |
| `div.getBoundingClientRect()` | DOM 高度测量 |
| `document.fonts.ready` | 等待字体加载 |

### URL 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `report` | '1' | 启用报告模式 |
| `requestId` | string | 自动化测试标识 |

### 全局对象

```typescript
interface Window {
  __BENCHMARK_REPORT__?: BenchmarkReport
}
```

## 风险、边界与改进建议

### 已知风险

1. **后台标签页节流**：浏览器会节流后台标签的计时器，导致结果失真
2. **垃圾回收干扰**：长测试期间可能触发 GC 暂停
3. **热路径假设**：`layout()` 的 200 次重复可能不完全代表真实使用模式

### 方法论限制

| 问题 | 当前处理 | 潜在改进 |
|------|----------|----------|
| 亚毫秒精度 | 重复 200 次取平均 | 使用 `performance.measure()` |
| JIT 编译影响 | 2 次预热 | 更多预热或代码预热 |
| 内存压力 | 未测量 | 添加 `performance.memory` 监控 |
| 帧率影响 | 未测量 | 添加 RAF 帧时间监控 |

### 改进建议

1. **统计增强**：
   - 添加标准差、变异系数
   - 检测并标记异常值
   - 计算统计显著性（vs DOM）

2. **场景扩展**：
   - 动态内容更新（插入/删除文本）
   - 响应式布局（连续宽度变化）
   - 真实 React/Vue 组件集成

3. **可视化**：
   - 实时性能图表
   - 历史趋势对比
   - 火焰图集成

4. **自动化**：
   - 与 CI 集成，自动检测 >10% 性能回归
   - 多浏览器并行测试
   - 结果上传到性能监控服务

### 相关配置

- 基准更新触发条件：修改 `src/analysis.ts`, `src/measurement.ts`, `src/line-break.ts`, `src/layout.ts`, `src/bidi.ts` 或 `pages/benchmark.ts` 时需刷新 `benchmarks/*.json`
