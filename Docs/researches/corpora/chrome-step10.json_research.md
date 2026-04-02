# chrome-step10.json 研究文档

## 场景与职责

`chrome-step10.json` 是 Pretext 项目的**高密度语料库精度快照文件**，用于记录在 Chrome 浏览器下对多语言长文本语料进行**密集步进测试**（step=10px）的结果。相比 `chrome-sampled.json` 的稀疏采样，该文件提供了更精细的宽度覆盖，用于深入分析文本布局引擎的边界行为和边缘案例。

### 核心职责

1. **密集宽度覆盖**：在 300-900px 范围内，以 10px 为步长进行 61 个宽度点的全面测试
2. **边界行为分析**：捕捉稀疏采样可能遗漏的特定宽度下的布局异常
3. **误差模式识别**：通过大量数据点识别系统性的偏差模式（如特定宽度范围的持续正/负偏差）
4. **算法调优依据**：为文本布局算法的容差调整和断行策略优化提供数据支撑

### 与 chrome-sampled.json 的对比

| 特性 | chrome-sampled.json | chrome-step10.json |
|------|---------------------|---------------------|
| 采样策略 | 稀疏均匀采样（9点） | 密集步进（61点） |
| 用途 | 快速健康检查 | 深度边界分析 |
| 运行时间 | 快（约 1-2 分钟） | 慢（约 10-15 分钟） |
| 数据粒度 | 粗粒度 | 细粒度 |
| 主要消费方 | CI 快速验证 | 算法调优、问题诊断 |

---

## 功能点目的

### 1. 密集宽度测试

```typescript
interface SweepSummary {
  corpusId: string;        // 语料唯一标识
  language: string;        // 语言代码
  title: string;           // 语料标题
  browser: 'chrome';       // 浏览器类型
  start: 300;              // 起始宽度
  end: 900;                // 结束宽度
  step: 10;                // 步长：10px
  samples: null;           // 密集模式下为 null（表示全量步进）
  widthCount: 61;          // (900-300)/10 + 1 = 61
  exactCount: number;      // 完全匹配的宽度数量
  mismatches: SweepMismatch[];  // 详细不匹配记录
}
```

### 2. 误差模式分析

通过密集数据可以识别：

- **系统性偏差**：如 `ur-chughd` 在窄宽度（300-450px）持续出现 -38px 到 -190px 的负偏差
- **边缘拟合问题**：如 `zh-zhufu` 在特定宽度出现 ±32px 的单行偏差
- **字体相关偏差**：不同字体导致的不同误差模式

### 3. 诊断支持

为 `corpus-check --diagnose` 提供背景数据：

```bash
# 当 chrome-step10.json 显示某宽度有问题时
bun run corpus-check --id=ur-chughd --diagnose 300 320 340
```

---

## 具体技术实现

### 宽度生成算法

**1. 步进模式** (`scripts/corpus-sweep.ts` 第 190-194 行)

```typescript
function getSweepWidths(meta: CorpusMeta, options: SweepOptions): number[] {
  // 当 options.samples === null 时，使用步进模式
  const widths: number[] = []
  for (let width = min; width <= max; width += options.step) {
    widths.push(width)
  }
  return widths
}
```

**2. 宽度计算**

```
范围：300px - 900px
步长：10px
数量：(900 - 300) / 10 + 1 = 61 个宽度点

具体序列：[300, 310, 320, ..., 880, 890, 900]
```

### 数据收集流程

```
┌─────────────────────────────────────────────────────────────────┐
│                     chrome-step10.json 生成流程                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 初始化                                                      │
│     └── bun run corpus-sweep --all --step=10 --browser=chrome   │
│                              │                                  │
│                              ▼                                  │
│  2. 加载语料元数据                                              │
│     └── sources.json (17 个语料)                                │
│                              │                                  │
│                              ▼                                  │
│  3. 为每个语料生成 61 个宽度点                                   │
│     └── [300, 310, ..., 900]                                    │
│                              │                                  │
│                              ▼                                  │
│  4. 浏览器自动化测试                                            │
│     ├── 启动临时 HTTP 服务器                                     │
│     ├── 导航到 pages/corpus.ts                                  │
│     ├── 对每个宽度执行 layout() vs DOM 对比                     │
│     └── 收集结果                                                │
│                              │                                  │
│                              ▼                                  │
│  5. 生成报告                                                    │
│     └── chrome-step10.json                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 关键代码路径

**1. 测试执行** (`scripts/corpus-sweep.ts`)

```typescript
// 第 292-351 行：主测试循环
for (const meta of targets) {
  const widths = getSweepWidths(meta, options)  // 生成 61 个宽度
  const requestId = `${Date.now()}-${meta.id}-${Math.random().toString(36).slice(2)}`
  
  // 构建测试 URL
  let url = `${baseUrl}?id=${encodeURIComponent(meta.id)}` +
            `&widths=${encodeURIComponent(widths.join(','))}` +  // 61 个宽度
            `&report=1` +
            `&requestId=${encodeURIComponent(requestId)}`
  
  // 执行浏览器测试并收集报告
  const report = await loadPostedReport(session, url, ...)
  
  // 汇总结果
  const summary: SweepSummary = {
    corpusId: meta.id,
    // ...
    widthCount: rows.length,      // 61
    exactCount,                   // 精确匹配数
    mismatches,                   // 不匹配详情
  }
}
```

**2. 浏览器端批量测试** (`pages/corpus.ts` 第 945-981 行)

```typescript
function runSweep(widths: number[]): void {
  // widths 包含 61 个宽度值
  const rows = widths.map(width => {
    const report = measureWidth(width, { publish: false, updateStats: false })
    return toSweepRow(report)
  })
  
  const exactCount = rows.filter(row => Math.round(row.diffPx) === 0).length
  
  // 生成批量报告
  setReport(withRequestId({
    status: 'ready',
    widths,
    widthCount: rows.length,    // 61
    exactCount,                 // 如 50/61
    rows,                       // 61 行详细数据
  }))
}
```

### 数据示例分析

以 `en-gatsby-opening` 为例：

```json
{
  "corpusId": "en-gatsby-opening",
  "language": "en",
  "title": "The Great Gatsby opening",
  "browser": "chrome",
  "start": 300,
  "end": 900,
  "step": 10,
  "samples": null,
  "widthCount": 61,
  "exactCount": 50,
  "mismatches": [
    { "width": 300, "diffPx": -26, "predictedHeight": 243882, "actualHeight": 243908, ... },
    { "width": 320, "diffPx": -26, ... },
    { "width": 370, "diffPx": -78, ... },
    // ... 共 11 个不匹配
  ]
}
```

**关键发现**：
- 在 300-370px 窄宽度范围存在系统性 -26px 到 -78px 的负偏差
- 430px 处出现 +26px 的正偏差（异常翻转）
- 580px 后趋于稳定

---

## 关键代码路径与文件引用

### 生成路径

| 文件 | 职责 | 关键行号 |
|------|------|----------|
| `scripts/corpus-sweep.ts` | 测试编排 | L66-81 (SweepOptions), L170-195 (getSweepWidths), L292-351 (主循环) |
| `pages/corpus.ts` | 浏览器端执行 | L945-981 (runSweep), L843-904 (measureWidth) |
| `src/layout.ts` | 布局引擎 | `layout()`, `prepareWithSegments()` |
| `scripts/browser-automation.ts` | 浏览器控制 | `loadPostedReport()` |

### 消费路径

| 文件 | 职责 | 引用方式 |
|------|------|----------|
| `scripts/corpus-status.ts` | 生成仪表盘 | `loadJson<SweepSummary[]>('corpora/chrome-step10.json')` (L278) |
| `corpora/dashboard.json` | 状态展示 | 合并为 `chromeStep10` 字段 |
| `AGENTS.md` | 开发指南 | 作为 "checked-in Chrome corpus sweep snapshots" 引用 |

### 命令行调用

```bash
# 生成 chrome-step10.json
bun run corpus-sweep --all --step=10 --browser=chrome --output=corpora/chrome-step10.json

# 单语料密集测试
bun run corpus-sweep --id=ur-chughd --step=10 --browser=chrome

# 带诊断的密集测试
bun run corpus-sweep --id=zh-zhufu --step=10 --browser=chrome --diagnose
```

---

## 依赖与外部交互

### 文件依赖图

```
corpora/chrome-step10.json
│
├─ 读取 ──> corpora/sources.json          (语料元数据)
│
├─ 依赖 ──> pages/corpus.ts               (浏览器测试页面)
│            ├─> src/layout.ts            (Pretext 引擎)
│            └─> pages/diagnostic-utils.ts (诊断工具)
│
├─ 依赖 ──> scripts/browser-automation.ts (浏览器自动化)
│            └─> AppleScript (macOS Safari 控制)
│
├─ 依赖 ──> scripts/report-server.ts      (结果接收服务)
│
├─ 生成 ──> corpora/dashboard.json        (via corpus-status.ts)
│
└─ 引用 ──> status/dashboard.json         (主状态仪表盘)
```

### 运行时依赖

| 依赖 | 用途 | 配置 |
|------|------|------|
| Chrome | 测试目标浏览器 | 环境变量 `CORPUS_CHECK_BROWSER=chrome` |
| Bun | JavaScript 运行时 | `bun run` |
| 临时 HTTP 服务器 | 提供测试页面 | 动态端口 (默认 0 表示自动分配) |
| CDP | Chrome DevTools Protocol | 端口 9222 |

### 数据流

```
┌─────────────────────────────────────────────────────────────────┐
│                        生成阶段                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   sources.json ──┐                                              │
│                  │                                              │
│                  ▼                                              │
│   corpus-sweep.ts ──> 生成 61 宽度点 ──> 浏览器自动化测试        │
│                  │                          │                   │
│                  │                          ▼                   │
│                  │              pages/corpus.ts                 │
│                  │              ├── layout() 预测               │
│                  │              └── DOM 实际测量                │
│                  │                          │                   │
│                  │                          ▼                   │
│                  └────────────── 收集结果 ──> chrome-step10.json │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        消费阶段                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   chrome-step10.json ──> corpus-status.ts                       │
│                              │                                  │
│                              ├── 计算 exactCount/widthCount     │
│                              ├── 识别误差模式                    │
│                              └── 生成 dashboard.json            │
│                                                                 │
│   chrome-step10.json ──> 人工分析                               │
│                              │                                  │
│                              ├── 识别问题宽度                    │
│                              └── 指导 corpus-check --diagnose   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 风险、边界与改进建议

### 已知问题与风险

#### 1. 运行时间过长

**问题**：61 个宽度点 × 17 个语料 = 1037 次布局测试，运行时间约 10-15 分钟。

**影响**：
- 不适合 CI 快速验证
- 开发迭代效率低

**缓解**：`chrome-sampled.json`（9点）用于快速检查，step10 用于深度分析。

#### 2. 数据量膨胀

**当前大小**：约 22KB（962 行），随着语料增加会持续膨胀。

**风险**：
- Git 仓库体积增长
- JSON 解析内存开销

#### 3. 浏览器版本锁定

**问题**：不同 Chrome 版本的字体渲染差异可能导致数据不一致。

**当前状态**：
- 记录在 `representative.json` 的 `environment` 字段
- 但未在 `chrome-step10.json` 中内联记录

### 边界条件

| 边界 | 当前值 | 说明 |
|------|--------|------|
| 宽度范围 | 300-900px | 固定范围，不支持自定义 |
| 步长 | 10px | 固定步长，不支持 5px 或 1px |
| 浏览器 | Chrome only | 无 Safari/Firefox 对应文件 |
| 语料数量 | 17 | 与 sources.json 同步 |
| 超时 | 180s | 通过 `CORPUS_CHECK_TIMEOUT_MS` 配置 |

### 改进建议

#### 1. 自适应步长

```typescript
// 建议：在已知问题区域使用更细粒度
function getAdaptiveStepWidths(meta: CorpusMeta, baseStep: number): number[] {
  const knownIssues = getKnownIssueWidths(meta.id)  // 从历史数据加载
  const baseWidths = getStepWidths(300, 900, baseStep)  // 10px 步长
  
  // 在问题区域插入 5px 步长
  const extraWidths = knownIssues.flatMap(w => [
    w - 5, w - 2, w, w + 2, w + 5
  ])
  
  return [...new Set([...baseWidths, ...extraWidths])].sort((a, b) => a - b)
}
```

#### 2. 增量更新

```bash
# 建议：仅更新变化的部分
bun run corpus-sweep --incremental \
  --baseline=corpora/chrome-step10.json \
  --output=corpora/chrome-step10.json
```

#### 3. 多浏览器支持

```bash
# 建议：生成其他浏览器的 step10 文件
bun run corpus-sweep --all --step=10 --browser=safari --output=corpora/safari-step10.json
bun run corpus-sweep --all --step=10 --browser=firefox --output=corpora/firefox-step10.json
```

#### 4. 数据压缩

```typescript
// 建议：使用更紧凑的格式
// 当前：每个 mismatch 存储完整对象
// 优化：仅存储变化的字段
interface CompactMismatch {
  w: number;    // width
  d: number;    // diffPx
  p?: number;   // predictedHeight (仅当需要时)
  a?: number;   // actualHeight (仅当需要时)
}
```

#### 5. 环境信息内联

```json
{
  "generatedAt": "2026-03-31T10:07:49.046Z",
  "environment": {
    "browser": "Chrome/146.0.0.0",
    "os": "macOS 14.5",
    "dpr": 2
  },
  "summaries": [...]
}
```

#### 6. 误差模式自动分类

```typescript
// 建议：自动生成误差模式报告
function classifyMismatchPattern(mismatches: SweepMismatch[]): string {
  const negativeWidths = mismatches.filter(m => m.diffPx < 0).map(m => m.width)
  const positiveWidths = mismatches.filter(m => m.diffPx > 0).map(m => m.width)
  
  if (negativeWidths.length > positiveWidths.length * 2) {
    return 'systematic-negative'
  }
  if (positiveWidths.length > negativeWidths.length * 2) {
    return 'systematic-positive'
  }
  return 'mixed'
}
```

### 维护 checklist

- [ ] 每次修改核心布局算法后重新生成
- [ ] 每季度验证一次（浏览器版本更新后）
- [ ] 与 `chrome-sampled.json` 定期对比，确保采样代表性
- [ ] 监控文件大小，必要时实施压缩策略
- [ ] 跟踪已知问题宽度，用于自适应步长优化
