# Status 目录研究文档

## 场景与职责

`status/` 目录是 Pretext 项目的**机器可读状态仪表板**存储位置，用于集中管理和展示浏览器准确性测试与性能基准测试的综合结果。它是项目质量保障体系的核心数据枢纽，为 CI/CD 流程、性能回归检测和跨浏览器兼容性验证提供权威数据源。

### 核心定位

- **单一可信源 (Single Source of Truth)**：`status/dashboard.json` 聚合了所有关键指标
- **回归检测门控**：作为自动化测试的判断基准，检测算法变更是否引入准确性或性能回归
- **跨浏览器对比**：统一存储 Chrome、Safari、Firefox 的测试结果，支持横向对比
- **历史追踪**：通过 JSON 快照实现测试结果的时间序列追踪

### 与其他目录的关系

```
status/
    dashboard.json          ← 聚合视图（本目录唯一文件）
accuracy/
    chrome.json             ← 原始准确性数据（Chrome）
    safari.json             ← 原始准确性数据（Safari）
    firefox.json            ← 原始准确性数据（Firefox）
benchmarks/
    chrome.json             ← 原始性能数据（Chrome）
    safari.json             ← 原始性能数据（Safari）
corpora/
    dashboard.json          ← 语料库仪表板（独立生成）
    representative.json     ← 代表性锚点数据
    chrome-sampled.json     ← Chrome 采样扫描
    chrome-step10.json      ← Chrome 步进扫描
```

---

## 功能点目的

### 1. 浏览器准确性聚合 (`browserAccuracy`)

**目的**：汇总跨浏览器的文本布局准确性测试结果

**数据来源**：
- `accuracy/chrome.json` - Chrome 浏览器 7680 项测试（4字体 × 8字号 × 8宽度 × 30文本）
- `accuracy/safari.json` - Safari 浏览器同等规模测试
- `accuracy/firefox.json` - Firefox 浏览器同等规模测试

**关键指标**：
- `total`: 总测试用例数（当前 7680）
- `matchCount`: 完全匹配数（当前 7680/7680，100%）
- `mismatchCount`: 不匹配数（当前 0）

**业务价值**：
- 作为回归门控，任何不匹配都会阻断发布流程
- 验证 `layout()` 算法在目标 CSS 配置下的浏览器一致性

### 2. 性能基准聚合 (`benchmarks`)

**目的**：追踪核心 API 的性能表现，对比 DOM 测量方案

**测试分层**：

| 层级 | 描述 | 关键指标 |
|------|------|----------|
| `topLevel` | 基础 API 性能 | `prepare()` 冷启动, `layout()` 热路径 |
| `richShared` | 富文本 API（共享语料） | `layoutWithLines()`, `walkLineRanges()`, `layoutNextLine()` |
| `richPreWrap` | pre-wrap 模式压力测试 | 硬断行块处理性能 |
| `richLong` | 长文本压力测试 | 阿拉伯语长文本布局 |
| `longFormCorpusStress` | 真实语料库压力测试 | 分阶段耗时（分析/测量/准备/布局） |

**性能对比基准**（Chrome）：
- `layout()` 热路径：~0.16ms/500文本批次
- DOM 批量读取：~3.7ms（比 layout 慢 23×）
- DOM 交错读写：~41.7ms（比 layout 慢 260×）

### 3. 数据来源追踪 (`sources`)

**目的**：建立数据血缘关系，确保可追溯性

```json
{
  "accuracy": {
    "chrome": "accuracy/chrome.json",
    "safari": "accuracy/safari.json",
    "firefox": "accuracy/firefox.json"
  },
  "benchmarks": {
    "chrome": "benchmarks/chrome.json",
    "safari": "benchmarks/safari.json"
  },
  "corpora": "corpora/dashboard.json"
}
```

---

## 具体技术实现

### 数据生成流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        数据生成流水线                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │ accuracy-    │    │ benchmark-   │    │ corpus-      │       │
│  │ check.ts     │    │ check.ts     │    │ sweep.ts     │       │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘       │
│         │                   │                   │               │
│         ▼                   ▼                   ▼               │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │ accuracy/    │    │ benchmarks/  │    │ corpora/     │       │
│  │ *.json       │    │ *.json       │    │ *.json       │       │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘       │
│         │                   │                   │               │
│         └───────────────────┼───────────────────┘               │
│                             │                                   │
│                             ▼                                   │
│                    ┌─────────────────┐                          │
│                    │ status-dashboard│                          │
│                    │ .ts             │                          │
│                    └────────┬────────┘                          │
│                             │                                   │
│                             ▼                                   │
│                    ┌─────────────────┐                          │
│                    │ status/         │                          │
│                    │ dashboard.json  │                          │
│                    └─────────────────┘                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 关键脚本实现

#### `scripts/status-dashboard.ts`

**职责**：聚合原始 JSON 快照为统一仪表板

**核心逻辑**：
```typescript
// 类型定义
interface AccuracyReport {
  total?: number
  matchCount?: number
  mismatchCount?: number
}

interface BenchmarkResult {
  label: string
  ms: number
  desc: string
}

interface CorpusBenchmarkResult {
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

// 数据聚合流程
1. 加载 accuracy/*.json → summarizeAccuracy() → browserAccuracy
2. 加载 benchmarks/*.json → indexResults() → 分层性能指标
3. 生成时间戳与数据来源映射
4. 写入 status/dashboard.json
```

**关键函数**：
- `summarizeAccuracy(report)`: 提取准确性摘要
- `indexResults(results)`: 将结果数组转为键值索引，便于查询

#### `scripts/corpus-status.ts`

**职责**：生成语料库专用仪表板（`corpora/dashboard.json`）

**特色功能**：
- **产品形态追踪** (`PRODUCT_SHAPED`): 混合应用文本等特殊场景
- **长文本语料追踪** (`LONG_FORM`): 17 种语言的深度语料库
- **细粒度扫描备注** (`FINE_SWEEP_NOTES`): step=1 精确扫描的已知问题记录
- **字体矩阵备注** (`FONT_MATRIX_NOTES`): 不同字体下的表现差异

**元数据结构**：
```typescript
interface CorpusDashboardMeta {
  id: string
  notes: string  // 人工维护的观察笔记
}

interface FineSweepNote {
  corpusId: string
  result: { kind: 'exact_count', ... } | { kind: 'not_fully_mapped', ... }
  notes: string
}

interface FontMatrixNote {
  corpusId: string
  status: string  // 'clean', 'has small field', 'real font split' 等
  notes: string
}
```

### 数据文件格式

#### `status/dashboard.json` 结构

```json
{
  "generatedAt": "2026-04-01T18:21:50.671Z",
  "sources": {
    "accuracy": { "chrome": "...", "safari": "...", "firefox": "..." },
    "benchmarks": { "chrome": "...", "safari": "..." },
    "corpora": "corpora/dashboard.json"
  },
  "browserAccuracy": {
    "chrome": { "total": 7680, "matchCount": 7680, "mismatchCount": 0 },
    "safari": { "total": 7680, "matchCount": 7680, "mismatchCount": 0 },
    "firefox": { "total": 7680, "matchCount": 7680, "mismatchCount": 0 }
  },
  "benchmarks": {
    "chrome": {
      "topLevel": { "Our library: layout()": { "ms": 0.16, "desc": "..." }, ... },
      "richShared": { ... },
      "richPreWrap": { ... },
      "richLong": { ... },
      "longFormCorpusStress": [ /* 13 个语料库详细指标 */ ]
    },
    "safari": { /* 同结构 */ }
  }
}
```

#### 长文本语料压力测试指标

每个语料库条目包含：
- **规模指标**: `chars`, `analysisSegments`, `segments`, `breakableSegments`, `lineCount`
- **性能指标**: `analysisMs`, `measureMs`, `prepareMs`, `layoutMs`
- **配置上下文**: `font`, `width`

**示例**（阿拉伯语长文本）：
```json
{
  "id": "ar-risalat-al-ghufran-part-1",
  "label": "Arabic prose",
  "font": "20px \"Geeza Pro\", \"Noto Naskh Arabic\", \"Arial\", serif",
  "chars": 106857,
  "analysisSegments": 37603,
  "segments": 37603,
  "breakableSegments": 18745,
  "width": 300,
  "lineCount": 2643,
  "analysisMs": 29.7,
  "measureMs": 33.8,
  "prepareMs": 63.4,
  "layoutMs": 0.30
}
```

---

## 关键代码路径与文件引用

### 生成路径

| 命令 | 入口脚本 | 输出文件 | 刷新时机 |
|------|----------|----------|----------|
| `bun run status-dashboard` | `scripts/status-dashboard.ts` | `status/dashboard.json` | accuracy/benchmarks 快照变更后 |
| `bun run corpus-status` | `scripts/corpus-status.ts` | `corpora/dashboard.json` | representative/sweep 快照变更后 |
| `bun run corpus-status:refresh` | 复合命令 | 多个 corpora/*.json | 完整语料库状态刷新 |

### 核心代码文件

#### 数据生成层
- **`scripts/status-dashboard.ts`** (115行): 主仪表板生成器
  - 读取 accuracy/*.json, benchmarks/*.json
  - 聚合为统一格式
  - 写入 status/dashboard.json

- **`scripts/corpus-status.ts`** (340行): 语料库仪表板生成器
  - 维护硬编码的语料库元数据 (`PRODUCT_SHAPED`, `LONG_FORM`)
  - 生成锚点摘要、回归门控状态
  - 人工维护的观察笔记系统

#### 数据收集层
- **`scripts/accuracy-check.ts`** (259行): 浏览器准确性自动化测试
  - 启动临时页面服务器 (`ensurePageServer`)
  - 浏览器自动化 (`createBrowserSession`)
  - 报告收集 (`loadHashReport`/`loadPostedReport`)

- **`scripts/benchmark-check.ts`** (144行): 性能基准自动化测试
  - 支持 Chrome/Safari
  - 前景模式运行确保准确性
  - 分层基准测试执行

- **`scripts/corpus-sweep.ts`** (362行): 语料库宽度扫描
  - 批量宽度测试 (`--start`, `--end`, `--step`)
  - 采样模式 (`--samples`)
  - 诊断联动 (`--diagnose`)

- **`scripts/corpus-representative.ts`** (239行): 代表性锚点生成
  - 固定宽度集合 [300, 600, 800]
  - 多浏览器支持
  - 环境指纹采集

#### 浏览器自动化层
- **`scripts/browser-automation.ts`** (715行): 跨浏览器自动化核心
  - **锁机制**: `acquireBrowserAutomationLock()` - 单浏览器单进程控制
  - **会话管理**: Chrome/Safari/Firefox 三后端支持
  - **报告传输**: Hash-based + POST side-channel 双模式
  - **超时诊断**: Phase 追踪 (`loading` → `measuring` → `posting`)

- **`scripts/report-server.ts`** (85行): POST 报告接收服务
  - 临时 HTTP 服务器
  - CORS 支持
  - 请求 ID 验证

#### 页面测试层
- **`pages/accuracy.ts`** (372行): 准确性测试页面
  - 4字体 × 8字号 × 8宽度 × 30文本 矩阵测试
  - DOM 高度对比
  - 逐行诊断 (`getBrowserLines`)

- **`pages/benchmark.ts`** (762行): 性能基准页面
  - 多层基准测试执行
  - CJK vs Latin 扩展测试
  - 长文本语料库压力测试

- **`pages/corpus.ts`** (1137行): 语料库测试页面
  - 双提取器支持 (span-probe / range)
  - 诊断深度分级 (`light`/`full`)
  - 切片测试支持 (`sliceStart`/`sliceEnd`)

### 共享工具
- **`shared/navigation-state.ts`** (42行): URL Hash 状态协议
  - Phase 编码: `#phase=measuring&requestId=xxx`
  - 报告编码: `#report={"status":"ready",...}`
  - 解析工具: `readNavigationPhaseState()`, `readNavigationReportText()`

- **`pages/report-utils.ts`** (21行): 报告发布工具
  - `publishNavigationPhase()` - 阶段广播
  - `publishNavigationReport()` - 结果广播

- **`pages/diagnostic-utils.ts`** (74行): 诊断工具集
  - `getDiagnosticUnits()` - 诊断单元生成
  - `measureCanvasTextWidth()` - Canvas 测量
  - `measureDomTextWidth()` - DOM 测量

---

## 依赖与外部交互

### 内部依赖图

```
status/dashboard.json
    ↑
    ├── scripts/status-dashboard.ts ──┬── accuracy/*.json
    │                                 ├── benchmarks/*.json
    │                                 └── corpora/dashboard.json (引用)
    │
    └── corpora/dashboard.json ───────┬── corpora/representative.json
                                      ├── corpora/chrome-sampled.json
                                      ├── corpora/chrome-step10.json
                                      └── accuracy/*.json (回归门控)
```

### 外部系统交互

#### 浏览器自动化
- **Chrome**: AppleScript 控制 (`Google Chrome` 应用)
- **Safari**: AppleScript 控制 (`Safari` 应用)
- **Firefox**: WebDriver BiDi 协议 (`--remote-debugging-port`)

#### 页面服务器
- **Bun 开发服务器**: `bun --port=X --no-hmr pages/*.html`
- **端口分配**: 动态获取 (`getAvailablePort`)
- **健康检查**: `resolveBaseUrl()` 轮询 loopback 地址

#### 文件系统
- **锁目录**: `$TMPDIR/pretext-browser-automation-locks/`
- **锁文件**: `{browser}.lock` (含 PID 和启动时间元数据)
- **自愈合**: 检测死锁进程并清理

### 环境变量

| 变量 | 用途 | 默认值 |
|------|------|--------|
| `ACCURACY_CHECK_BROWSER` | 准确性测试目标浏览器 | `chrome` |
| `ACCURACY_CHECK_TIMEOUT_MS` | 准确性测试超时 | 120000 (Safari: 240000) |
| `BENCHMARK_CHECK_BROWSER` | 基准测试目标浏览器 | `chrome` |
| `CORPUS_CHECK_BROWSER` | 语料库测试目标浏览器 | `chrome` |
| `CORPUS_CHECK_TIMEOUT_MS` | 语料库测试超时 | 180000 |
| `TMPDIR` | 锁文件存储位置 | `/tmp` |

---

## 风险、边界与改进建议

### 当前风险

#### 1. 单点故障风险
- **风险**: `status/dashboard.json` 是唯一的聚合视图，文件损坏将导致状态不可读
- **缓解**: 原始数据分散在 accuracy/, benchmarks/, corpora/ 目录，可重新生成
- **建议**: 添加生成校验和或版本控制标记

#### 2. 浏览器自动化锁竞争
- **风险**: 单浏览器全局锁可能导致 CI 流水线排队
- **缓解**: 锁文件含 PID 元数据，支持死锁检测和自愈合
- **边界**: 不支持并行多浏览器测试（设计如此，避免资源争用）

#### 3. 数据新鲜度依赖人工触发
- **风险**: 仪表板不会自动更新，需手动运行 `status-dashboard` 命令
- **现状**: 由 AGENTS.md 规范约束，在相关源码变更后手动刷新
- **建议**: 考虑在 CI 中集成自动刷新（权衡：需要浏览器环境）

#### 4. Safari 测量噪声
- **风险**: Safari 基准测试结果 warmup 不一致，数值波动较大
- **现状**: STATUS.md 已明确标注 Safari 为次要基准，Chrome 为主基准
- **缓解**: 冷启动运行，避免后台标签节流

### 技术边界

#### 准确性测试边界
- **测试矩阵**: 4字体 × 8字号 × 8宽度 × 30文本 = 7680 项/浏览器
- **容差**: 高度差 < 1px 视为匹配
- **CSS 目标**: `white-space: normal`, `word-break: normal`, `overflow-wrap: break-word`
- **排除项**: `break-all`, `keep-all`, `strict`, `loose`, `anywhere` 未测试

#### 性能测试边界
- **测量精度**: 受限于 `performance.now()` (通常 0.1ms 精度)
- **采样策略**: 中位数过滤异常值，非平均值
- **内存影响**: 未测量内存占用，仅关注 CPU 时间
- **并发**: 单线程测试，未评估 Worker 场景

#### 语料库覆盖边界
- **语言覆盖**: 17 种语言的 19 个长文本语料库
- **脚本覆盖**: Latin, CJK, Arabic, Hebrew, Thai, Khmer, Myanmar, Devanagari
- **字体依赖**:  macOS 系统字体为主，跨平台可移植性受限

### 改进建议

#### 短期（维护性）

1. **添加 Schema 验证**
   ```typescript
   // 为 dashboard.json 添加 JSON Schema
   interface DashboardSchema {
     generatedAt: string  // ISO 8601
     sources: { /* 严格路径格式 */ }
     browserAccuracy: { [browser: string]: AccuracySummary }
     benchmarks: { [browser: string]: BenchmarkCategories }
   }
   ```

2. **增量更新支持**
   - 当前：全量重新生成
   - 改进：仅更新变更的浏览器/语料库子集

3. **历史趋势追踪**
   - 当前：单点快照
   - 改进：追加模式或 Git 历史分析

#### 中期（功能性）

4. **多平台支持**
   - 当前：macOS 专用（AppleScript 控制浏览器）
   - 改进：Linux 支持（Chrome DevTools Protocol）
   - 影响：CI 环境可运行准确性测试

5. **实时仪表板**
   - 当前：静态 JSON 文件
   - 改进：简单的 HTML 可视化界面
   - 参考：DEVELOPMENT.md 中的 `/demos` 页面模式

6. **差异分析工具**
   - 当前：人工对比 JSON 文件
   - 改进：`scripts/dashboard-diff.ts` 自动对比两个快照

#### 长期（架构性）

7. **数据库存储**
   - 当前：文件系统 JSON
   - 改进：SQLite 或轻量级数据库
   - 收益：查询能力、历史趋势、关联分析

8. **分布式测试**
   - 当前：单机串行
   - 改进：多机并行（Chrome/Safari/Firefox 同时运行）
   - 依赖：远程浏览器网格或容器化

9. **预测性回归检测**
   - 利用历史数据建立性能基线模型
   - 自动标记异常波动（而非固定阈值）

### 监控建议

```yaml
# 建议添加的健康检查指标
status_health:
  - dashboard_json_age: < 24h  # 文件新鲜度
  - accuracy_match_rate: == 100%  # 准确性门控
  - benchmark_regression:  # 性能回归
      layout_threshold: < 0.2ms
      prepare_threshold: < 25ms
  - corpus_coverage:  # 语料库覆盖
      min_corpora: 17
      min_exact_rate: 95%
```

---

## 附录：关键数据结构

### AccuracySummary
```typescript
{
  total: number        // 总测试数
  matchCount: number   // 通过数
  mismatchCount: number // 失败数
}
```

### BenchmarkResult
```typescript
{
  label: string  // 测试名称，如 "Our library: layout()"
  ms: number     // 中位耗时
  desc: string   // 人类可读描述
}
```

### CorpusBenchmarkResult
```typescript
{
  id: string              // 语料库标识
  label: string          // 显示名称
  font: string           // 测试字体
  chars: number          // 字符数
  analysisSegments: number   // 分析阶段段数
  segments: number       // 总段数
  breakableSegments: number  // 可断行段数
  width: number          // 测试宽度
  lineCount: number      // 行数
  analysisMs: number     // 文本分析耗时
  measureMs: number      // 测量耗时
  prepareMs: number      // 准备总耗时
  layoutMs: number       // 布局耗时
}
```

### SweepSummary
```typescript
{
  corpusId: string
  language: string
  title: string
  browser: BrowserKind
  start: number      // 扫描起始宽度
  end: number        // 扫描结束宽度
  step: number       // 步进
  samples: number | null  // 采样数（如有）
  widthCount: number // 总宽度点数
  exactCount: number // 完全匹配数
  mismatches: Array<{
    width: number
    diffPx: number
    predictedHeight: number
    actualHeight: number
    predictedLineCount: number | null
    browserLineCount: number | null
  }>
}
```
