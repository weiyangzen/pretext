# Benchmarks 目录研究文档

## 1. 场景与职责

### 1.1 定位与目标

`benchmarks/` 目录是 Pretext 文本布局库的性能基准数据存储目录，用于：

1. **性能回归检测**：通过检入的 JSON 快照对比，检测代码变更对性能的影响
2. **浏览器性能对比**：记录 Chrome 和 Safari 两大主流浏览器下的性能基线
3. **算法优化验证**：量化优化措施（如缓存策略、快速路径）的实际效果
4. **DOM 性能对比**：证明 canvas-based 文本测量相对于传统 DOM 测量的性能优势

### 1.2 使用场景

| 场景 | 说明 |
|------|------|
| 开发阶段 | 运行 `bun run benchmark-check` 验证本地变更是否引入性能回归 |
| 发布前 | 对比 benchmarks/chrome.json 和 benchmarks/safari.json 确认性能稳定 |
| CI/CD | （当前未配置自动化 benchmark，需手动执行） |
| 性能调优 | 通过 dashboard.json 聚合视图识别热点路径 |

### 1.3 与相关目录的关系

```
benchmarks/           # 性能基准数据（本目录）
├── chrome.json       # Chrome 浏览器性能快照
└── safari.json       # Safari 浏览器性能快照

src/                  # 被测源码
├── layout.ts         # 核心布局 API（prepare/layout）
├── line-break.ts     # 行断算法实现
├── measurement.ts    # Canvas 测量与缓存
└── analysis.ts       # 文本分析与分段

pages/                # 浏览器测试页面
└── benchmark.ts      # 性能测试执行页面

scripts/              # 自动化脚本
└── benchmark-check.ts    # Node.js 端 benchmark 驱动
└── browser-automation.ts # 浏览器自动化基础
└── status-dashboard.ts   # 聚合 dashboard 生成

status/               # 状态聚合
└── dashboard.json    # 包含 benchmarks 数据的统一视图
```

---

## 2. 功能点目的

### 2.1 核心 Benchmark 类别

#### 2.1.1 Top-Level Batch Benchmark (`results`)

**目的**：验证核心 API 的基础性能指标

| 测试项 | 说明 | 典型值（Chrome） |
|--------|------|-----------------|
| `prepare()` | 500 段文本的冷启动测量批次 | ~18.6ms |
| `layout()` | 热路径吞吐量（每 500 段） | ~0.16ms |
| `DOM batch` | DOM 批量写入后统一读取 | ~3.7ms |
| `DOM interleaved` | DOM 写入+读取交替（最坏情况） | ~41.7ms |

**关键结论**：
- `layout()` 比 DOM interleaved 快约 **260 倍**
- `layout()` 比 DOM batch 快约 **23 倍**

#### 2.1.2 Rich Line APIs Benchmark (`richResults`)

**目的**：比较三种富文本行 API 的性能差异

| API | 功能 | Chrome | Safari |
|-----|------|--------|--------|
| `layoutWithLines()` | 物化每行文本 | 0.056ms | 0.05ms |
| `walkLineRanges()` | 仅几何信息，无字符串 | 0.027ms | 0.025ms |
| `layoutNextLine()` | 流式逐行布局 | 0.072ms | 0.075ms |

**设计意图**：
- `walkLineRanges()` 最快（无字符串物化），用于 shrinkwrap 和聚合几何计算
- `layoutNextLine()` 用于自定义布局的流式处理
- `layoutWithLines()` 用于需要完整行文本的场景

#### 2.1.3 Pre-wrap Chunk Stress (`richPreWrapResults`)

**目的**：测试 `whiteSpace: 'pre-wrap'` 模式下的硬断行处理性能

- 12 个生成文本，每个含 320 个硬断块
- 包含制表符、空行、尾部空格等边界情况
- 用于编辑器/输入框场景的性能验证

#### 2.1.4 Arabic Long-form Stress (`richLongResults`)

**目的**：富路径最坏情况压力测试

- 8 份阿拉伯长文本（ar-risalat-al-ghufran-part-1）
- 跨 240/300/360px 三种宽度
- 验证复杂脚本（阿拉伯语从右到左、连字）下的性能

#### 2.1.5 Long-form Corpus Stress (`corpusResults`)

**目的**：按语料库细分 prepare 阶段的性能剖析

| 指标 | 说明 |
|------|------|
| `analysisMs` | 文本分析阶段（分段、归一化） |
| `measureMs` | Canvas 测量阶段 |
| `prepareMs` | 总 prepare 时间 |
| `layoutMs` | 热路径 layout 时间 |

覆盖语料库：
- CJK：日语（ja-*）、韩语（ko-*）、中文（zh-*）
- 东南亚：泰语（th-*）、缅甸语（my-*）、高棉语（km-*）
- 阿拉伯系：乌尔都语（ur-*）、阿拉伯语（ar-*）
- 印度系：印地语（hi-*）
- 合成：长可断行（URL、缓存键等）

---

## 3. 具体技术实现

### 3.1 数据结构定义

#### 3.1.1 BenchmarkReport（页面端）

```typescript
// pages/benchmark.ts
interface BenchmarkReport {
  status: 'ready' | 'error'
  requestId?: string
  results?: BenchmarkResult[]           // Top-level
  richResults?: BenchmarkResult[]       // Shared corpus
  richPreWrapResults?: BenchmarkResult[] // Pre-wrap stress
  richLongResults?: BenchmarkResult[]   // Arabic long-form
  corpusResults?: CorpusBenchmarkResult[] // Corpus stress
  message?: string
}

interface BenchmarkResult {
  label: string   // 如 "Our library: layout()"
  ms: number      // 中位数毫秒
  desc: string    // 测试描述
}

interface CorpusBenchmarkResult {
  id: string              // 语料库标识
  label: string           // 可读标签
  font: string            // 测试字体
  chars: number           // 字符数
  analysisSegments: number
  segments: number        // 最终段数
  breakableSegments: number
  width: number           // 测试宽度
  lineCount: number       // 输出行数
  analysisMs: number      // 分析耗时
  measureMs: number       // 测量耗时
  prepareMs: number       // 总 prepare 耗时
  layoutMs: number        // layout 耗时
}
```

#### 3.1.2 JSON 快照格式

```json
{
  "status": "ready",
  "results": [...],
  "richResults": [...],
  "richPreWrapResults": [...],
  "richLongResults": [...],
  "corpusResults": [...],
  "requestId": "1775067664577-0uk4bqilf4yi"
}
```

### 3.2 关键流程

#### 3.2.1 浏览器端 Benchmark 执行流程

```
pages/benchmark.ts
├── run()
│   ├── 创建 DOM 容器（用于 DOM 对比测试）
│   ├── 准备 500 段测试文本
│   ├── 
│   ├── 1. prepare() Benchmark
│   │   └── bench(() => { clearCache(); prepare() })
│   ├── 
│   ├── 2. layout() Benchmark
│   │   └── bench(() => { layout() })
│   ├── 
│   ├── 3. DOM batch Benchmark
│   │   └── 批量设置 width → 批量读取 height
│   ├── 
│   ├── 4. DOM interleaved Benchmark
│   │   └── 逐个设置 width → 立即读取 height
│   ├── 
│   ├── 5. Rich APIs Benchmark
│   │   ├── prepareWithSegments() 准备 60 段文本
│   │   └── buildRichBenchmarks() 测试三种 API
│   ├── 
│   ├── 6. Pre-wrap Stress
│   │   └── 生成 12 个含硬断的文本
│   ├── 
│   ├── 7. Arabic Long-form Stress
│   │   └── 8 份阿拉伯文本 × 3 种宽度
│   ├── 
│   ├── 8. Corpus Stress
│   │   └── buildCorpusBenchmarks()
│   │       ├── profilePrepare() 分析+测量阶段
│   │       └── bench() layout 阶段
│   └── 
│   └── 渲染结果表格
│
└── bench(fn, sampleRepeats, warmup, runs)
    ├── 执行 warmup 轮
    ├── 执行 runs 轮，每轮执行 sampleRepeats 次
    └── 返回中位数时间
```

#### 3.2.2 Node.js 端自动化流程

```
scripts/benchmark-check.ts
├── 解析参数 (--browser, --port, --output)
├── acquireBrowserAutomationLock() 获取浏览器锁
├── createBrowserSession() 创建浏览器会话
├── ensurePageServer() 启动页面服务
├── 
├── loadHashReport() 导航并等待结果
│   ├── session.navigate(url + '?report=1&requestId=xxx')
│   ├── 轮询 URL hash 等待 report
│   └── 解析返回的 BenchmarkReport
├── 
├── printReport() 打印结果
├── 如指定 --output，写入 JSON 文件
└── 清理（关闭浏览器、释放锁）
```

### 3.3 关键算法与优化

#### 3.3.1 统计方法：中位数而非平均值

```typescript
function median(times: number[]): number {
  const sorted = [...times].sort((a, b) => a - b)
  const mid = Math.floor(sorted.length / 2)
  return sorted.length % 2 === 0 
    ? (sorted[mid - 1]! + sorted[mid]!) / 2 
    : sorted[mid]!
}
```

**原因**：排除异常值（如 GC 暂停、浏览器节流）的影响

#### 3.3.2 采样重复策略

| 测试类型 | 采样重复 | 原因 |
|---------|---------|------|
| prepare() | 1× | 本身就是冷启动，无需内部重复 |
| layout() | 200× | 亚毫秒级操作，需要重复获得可测量时间 |
| DOM batch | 1× | 本身耗时较长，单次即可测量 |
| Rich APIs | 40× | 中等粒度，平衡精度与耗时 |
| Pre-wrap | 20× | 复杂场景，适度重复 |

#### 3.3.3 浏览器锁机制

```typescript
// scripts/browser-automation.ts
const LOCK_DIR = join(process.env['TMPDIR'] ?? '/tmp', 'pretext-browser-automation-locks')

async function acquireBrowserAutomationLock(browser, timeoutMs = 120_000)
```

- 文件锁路径：`/tmp/pretext-browser-automation-locks/{chrome|safari}.lock`
- 自修复：检测到锁的进程已死亡时自动清理
- 超时：默认 2 分钟

### 3.4 报告传输协议

#### 3.4.1 Hash-based 报告传输

```
页面端                              Node.js 端
--------                            -----------
1. 执行 benchmark
2. 结果 → JSON
3. 编码为 URL hash
   #report={encoded_json}            
4. history.replaceState()           
                                    5. 轮询读取 location.href
                                    6. 解析 hash 获取报告
```

相关代码：
- `shared/navigation-state.ts`：hash 编解码
- `pages/report-utils.ts`：页面端发布
- `scripts/browser-automation.ts`：Node.js 端读取

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件映射

| 文件 | 职责 | 关键函数/类型 |
|------|------|--------------|
| `benchmarks/chrome.json` | Chrome 性能快照 | BenchmarkReport JSON |
| `benchmarks/safari.json` | Safari 性能快照 | BenchmarkReport JSON |
| `pages/benchmark.ts` | 浏览器端 benchmark 执行 | `run()`, `bench()`, `buildRichBenchmarks()` |
| `pages/benchmark.html` | Benchmark 页面模板 | 样式定义 |
| `scripts/benchmark-check.ts` | Node.js 端驱动 | `printReport()`, CLI 参数解析 |
| `scripts/browser-automation.ts` | 浏览器自动化 | `createBrowserSession()`, `loadHashReport()` |
| `scripts/status-dashboard.ts` | Dashboard 生成 | 聚合 benchmarks 到 dashboard |
| `shared/navigation-state.ts` | 导航状态协议 | `NavigationPhase`, `buildNavigationReportHash()` |
| `pages/report-utils.ts` | 报告发布工具 | `publishNavigationReport()` |

### 4.2 被测源码文件

| 文件 | 被测功能 | Benchmark 关联 |
|------|---------|---------------|
| `src/layout.ts` | `prepare()`, `layout()`, `layoutWithLines()`, `walkLineRanges()`, `layoutNextLine()` | 全部 |
| `src/line-break.ts` | `countPreparedLines()`, `walkPreparedLines()`, `layoutNextLineRange()` | layout 性能 |
| `src/measurement.ts` | `getSegmentMetrics()`, `getEngineProfile()` | prepare 性能 |
| `src/analysis.ts` | `analyzeText()`, `normalizeWhitespaceNormal()` | analysisMs |
| `src/test-data.ts` | `TEXTS` | 共享测试语料 |

### 4.3 关键代码片段

#### 4.3.1 layout() 热路径（pages/benchmark.ts:523-531）

```typescript
const tLayout = bench(repeatIndex => {
  const maxWidth = LAYOUT_SAMPLE_WIDTHS[repeatIndex % LAYOUT_SAMPLE_WIDTHS.length]!
  let sum = 0
  for (let i = 0; i < COUNT; i++) {
    const result = layout(prepared[i]!, maxWidth, LINE_HEIGHT)
    sum += result.height + result.lineCount
  }
  topLayoutSink += sum + repeatIndex
}, LAYOUT_SAMPLE_REPEATS)
```

#### 4.3.2 Corpus 性能剖析（pages/benchmark.ts:319-373）

```typescript
function buildCorpusBenchmarks(): CorpusBenchmarkResult[] {
  for (const corpus of CORPORA) {
    // 阶段 1：分析
    const profile = profilePrepare(corpus.text, corpus.font)
    analysisSamples.push(profile.analysisMs)
    
    // 阶段 2：测量
    measureSamples.push(profile.measureMs)
    prepareSamples.push(profile.totalMs)
    
    // 阶段 3：layout
    const layoutMs = bench(repeatIndex => {
      const width = corpus.sampleWidths[repeatIndex % corpus.sampleWidths.length]!
      const result = layout(prepared, width, corpus.lineHeight)
      corpusLayoutSink += result.height + result.lineCount + repeatIndex
    }, CORPUS_LAYOUT_SAMPLE_REPEATS)
  }
}
```

#### 4.3.3 profilePrepare 实现（src/layout.ts:436-456）

```typescript
export function profilePrepare(text: string, font: string, options?: PrepareOptions): PrepareProfile {
  const t0 = performance.now()
  const analysis = analyzeText(text, getEngineProfile(), options?.whiteSpace)
  const t1 = performance.now()
  const prepared = measureAnalysis(analysis, font, false) as InternalPreparedText
  const t2 = performance.now()

  return {
    analysisMs: t1 - t0,
    measureMs: t2 - t1,
    totalMs: t2 - t0,
    // ...
  }
}
```

---

## 5. 依赖与外部交互

### 5.1 内部依赖图

```
benchmarks/*.json
    ▲
    │ 写入
scripts/benchmark-check.ts
    │ 调用
    ├── scripts/browser-automation.ts
    │       ├── shared/navigation-state.ts
    │       └── AppleScript (Safari/Chrome 控制)
    │
    └── pages/benchmark.ts (通过浏览器加载)
            ├── src/layout.ts
            │       ├── src/line-break.ts
            │       ├── src/measurement.ts
            │       └── src/analysis.ts
            ├── src/test-data.ts
            └── pages/report-utils.ts
                    └── shared/navigation-state.ts

scripts/status-dashboard.ts
    ├── benchmarks/chrome.json
    ├── benchmarks/safari.json
    └── status/dashboard.json (输出)
```

### 5.2 外部依赖

| 依赖 | 用途 | 位置 |
|------|------|------|
| macOS AppleScript | Safari/Chrome 自动化控制 | `scripts/browser-automation.ts` |
| WebSocket | Firefox BiDi 协议通信 | `scripts/browser-automation.ts` |
| Bun | 脚本运行时、文件操作 | 全部 scripts |
| Canvas API | 文本测量 | `src/measurement.ts` |
| `Intl.Segmenter` | 文本分段 | `src/analysis.ts`, `src/layout.ts` |

### 5.3 环境变量

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `BENCHMARK_CHECK_BROWSER` | 目标浏览器 | `chrome` |
| `BENCHMARK_CHECK_PORT` | 页面服务端口 | `0`（自动分配） |

### 5.4 命令行接口

```bash
# 基本用法
bun run benchmark-check                    # Chrome
bun run benchmark-check:safari             # Safari

# 带输出
bun run scripts/benchmark-check.ts --browser=chrome --output=benchmarks/chrome.json

# 自定义端口
bun run scripts/benchmark-check.ts --port=3210
```

---

## 6. 风险、边界与改进建议

### 6.1 当前风险

#### 6.1.1 浏览器特异性

| 风险 | 说明 | 缓解措施 |
|------|------|---------|
| Safari 自动化慢 | AppleScript 控制比 Chrome CDP 慢 | 仅用于验证，不用于频繁迭代 |
| Firefox 支持有限 | 使用 BiDi 协议，稳定性待验证 | 当前 benchmark 仅支持 Chrome/Safari |
| 字体差异 | 不同系统字体渲染差异影响测量 | 使用通用字体栈，避免 system-ui |

#### 6.1.2 测量精度

| 风险 | 说明 | 缓解措施 |
|------|------|---------|
| 低分辨率计时器 | Firefox/Safari 可能返回 0ms | 使用 `LAYOUT_SAMPLE_REPEATS=200` 内部重复 |
| 后台标签节流 | 浏览器后台运行时限流 setTimeout | `foreground: true` 强制前台运行 |
| GC 干扰 | 垃圾回收导致异常值 | 使用中位数而非平均值 |

#### 6.1.3 数据一致性

| 风险 | 说明 | 缓解措施 |
|------|------|---------|
| 锁竞争 | 多进程同时请求同一浏览器 | 文件锁机制 + 120s 超时 |
| 孤儿锁 | 进程崩溃后锁未清理 | 自修复：检测 PID 存活状态 |
| 快照过时 | 代码优化后未更新快照 | AGENTS.md 要求：关键文件变更需刷新快照 |

### 6.2 边界情况

#### 6.2.1 测试规模边界

```typescript
const COUNT = 500                    // 基础 batch 规模
const RICH_COUNT = 60                // Rich API 测试规模
const RICH_PRE_WRAP_COUNT = 12       // Pre-wrap 文本数
const RICH_PRE_WRAP_LINE_COUNT = 320 // 每文本硬断数
const RICH_LONG_REPEAT = 8           // 阿拉伯长文本重复
```

#### 6.2.2 宽度采样边界

```typescript
const LAYOUT_SAMPLE_WIDTHS = [200, 250, 300, 350, 400]
const RICH_LAYOUT_SAMPLE_WIDTHS = [180, 220, 260]
const RICH_PRE_WRAP_SAMPLE_WIDTHS = [220, 260, 320]
const RICH_LONG_SAMPLE_WIDTHS = [240, 300, 360]
```

### 6.3 改进建议

#### 6.3.1 短期改进

1. **添加 Firefox Benchmark 支持**
   ```typescript
   // scripts/benchmark-check.ts 已定义 firefox 类型但未启用
   function parseBrowser(value: string | null): BrowserKind {
     // 当前仅支持 chrome/safari
   }
   ```

2. **添加趋势追踪**
   - 在 dashboard.json 中添加历史时间序列
   - 或添加 `benchmarks/history/` 目录存储历次结果

3. **CI 集成**
   ```yaml
   # .github/workflows/benchmark.yml
   - name: Run Benchmark
     run: |
       bun run benchmark-check --output=benchmarks/pr-result.json
       bun run scripts/benchmark-compare.ts \
         --baseline=benchmarks/chrome.json \
         --current=benchmarks/pr-result.json
   ```

#### 6.3.2 中期改进

1. **细分 Corpus 性能**
   - 当前仅记录总 analysisMs/measureMs
   - 建议细分：Intl.Segmenter 耗时、CJK 特殊处理耗时、Emoji 校正耗时

2. **内存 Benchmark**
   ```typescript
   // 新增内存测量
   const heapBefore = performance.memory?.usedJSHeapSize
   // ... 执行操作
   const heapAfter = performance.memory?.usedJSHeapSize
   ```

3. **并发 Benchmark**
   - 测试多文本并行 prepare 的性能
   - 验证 `clearCache()` 的缓存清除效果

#### 6.3.3 长期改进

1. **Methodology Review**
   > AGENTS.md 中提到："Benchmark methodology still needs review"
   
   建议：
   - 参考 Web Benchmarking 最佳实践（如 web-benchmarks 库）
   - 考虑使用 PerformanceObserver 获取更精确的计时

2. **跨平台支持**
   - 当前 AppleScript 仅支持 macOS
   - 建议添加 Linux（Chrome DevTools Protocol）和 Windows 支持

3. **可视化 Dashboard**
   - 添加 `pages/demos/benchmark-history.html`
   - 展示性能趋势图表

### 6.4 维护检查清单

当修改以下文件时，必须刷新对应 benchmark 快照：

| 文件变更 | 需刷新快照 |
|---------|-----------|
| `src/analysis.ts` | `benchmarks/chrome.json`, `benchmarks/safari.json` |
| `src/measurement.ts` | `benchmarks/chrome.json`, `benchmarks/safari.json` |
| `src/line-break.ts` | `benchmarks/chrome.json`, `benchmarks/safari.json` |
| `src/layout.ts` | `benchmarks/chrome.json`, `benchmarks/safari.json` |
| `src/bidi.ts` | `benchmarks/chrome.json`, `benchmarks/safari.json` |
| `pages/benchmark.ts` | `benchmarks/chrome.json`, `benchmarks/safari.json` |

刷新后需执行：
```bash
bun run status-dashboard  # 更新 status/dashboard.json
```

---

## 7. 附录

### 7.1 快照生成时间

| 文件 | 最后更新时间 |
|------|-------------|
| `benchmarks/chrome.json` | 2026-04-02（当前） |
| `benchmarks/safari.json` | 2026-04-02（当前） |
| `status/dashboard.json` | 2026-04-01T18:21:50.671Z |

### 7.2 相关文档

- `AGENTS.md` - 代理开发指南，包含 benchmark 方法论
- `DEVELOPMENT.md` - 开发命令参考
- `RESEARCH.md` - 算法研究日志
- `STATUS.md` - 项目状态概览

### 7.3 关键性能数字（Chrome）

| 指标 | 数值 | 单位 |
|------|------|------|
| prepare() 冷启动 | 18.6 | ms/500 texts |
| layout() 热路径 | 0.16 | ms/500 texts |
| DOM batch | 3.7 | ms/batch |
| DOM interleaved | 41.7 | ms/batch |
| layout() vs DOM interleaved | 260× | 加速比 |
| walkLineRanges() | 0.027 | ms/60 texts |
| layoutWithLines() | 0.056 | ms/60 texts |
