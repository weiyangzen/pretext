# Corpora 目录研究文档

## 1. 场景与职责

### 1.1 目录定位

`corpora/` 是 Pretext 项目的**多语言长文本语料库目录**，作为浏览器布局准确性测试的基准数据集（canary datasets）。它与 `accuracy/`（浏览器精度测试）、`benchmarks/`（性能基准）共同构成项目的三大测试基础设施。

### 1.2 核心职责

| 职责 | 说明 |
|------|------|
| **多语言覆盖** | 提供 17 种语言的真实长文本，覆盖拉丁、阿拉伯、希伯来、CJK、东南亚等书写系统 |
| **回归测试** | 作为浏览器布局算法的参照标准，检测引擎与浏览器实际渲染的差异 |
| **字体矩阵测试** | 支持同一语料在不同字体下的表现对比 |
| **细粒度扫描** | 支持 300-900px 宽度的逐步扫描，发现边缘情况 |

### 1.3 语料分类

```
productShaped/          # 产品形态文本（混合脚本、URL、表情符号等）
  └── mixed-app-text    # 合成语料，测试复杂场景

longForm/               # 长文本文学语料
  ├── en-gatsby-opening # 英文（盖茨比开篇）
  ├── ja-rashomon       # 日文（罗生门）
  ├── ja-kumo-no-ito    # 日文（蜘蛛之丝）
  ├── ko-unsu-joh-eun-nal # 韩文
  ├── zh-zhufu          # 中文（祝福）
  ├── zh-guxiang        # 中文（故乡）
  ├── th-nithan-vetal-* # 泰文（2篇）
  ├── my-*              # 缅甸文（2篇）
  ├── km-*              # 高棉文
  ├── ar-*              # 阿拉伯文（2篇）
  ├── hi-eidgah         # 印地文
  ├── he-masaot-*       # 希伯来文
  └── ur-chughd         # 乌尔都文
```

---

## 2. 功能点目的

### 2.1 核心功能矩阵

| 功能 | 目的 | 触发方式 |
|------|------|----------|
| `corpus-check` | 单语料单宽度精确检查 | `bun run corpus-check --id=<id> <width>` |
| `corpus-sweep` | 多宽度扫描 | `bun run corpus-sweep --id=<id> --start=300 --end=900 --step=10` |
| `corpus-taxonomy` | 分类诊断不匹配原因 | `bun run corpus-taxonomy --id=<id> <widths...>` |
| `corpus-font-matrix` | 字体变体对比 | `bun run corpus-font-matrix --id=<id> --samples=5` |
| `corpus-representative` | 生成锚点快照 | `bun run corpus-representative` |
| `corpus-status` | 生成仪表板 | `bun run corpus-status` |

### 2.2 诊断深度层级

```
light (默认)    → 仅比较高度/行数
full (--diagnose) → 逐行文本对比 + 断点分析
```

### 2.3 不匹配分类体系（Taxonomy）

定义在 `corpora/TAXONOMY.md`，脚本 `corpus-taxonomy.ts` 实现自动分类：

| 类别 | 含义 | 处理策略 |
|------|------|----------|
| `corpus-dirty` | 语料本身不干净（换行符、导航等） | 清理或拒绝语料 |
| `normalization` | 空白字符处理差异 | 修复预处理逻辑 |
| `boundary-discovery` | 分段边界错误 | 调整预处理规则 |
| `glue-policy` | 粘连策略问题（标点附着） | 修改粘连规则 |
| `edge-fit` | 边缘适配差异（<1px） | 检查浏览器容差 |
| `shaping-context` | 字形塑形上下文影响 | 接受近似或增强模型 |
| `font-mismatch` | 字体解析差异 | 验证字体栈 |
| `diagnostic-sensitivity` | 探测方法敏感 | 交叉验证提取器 |

---

## 3. 具体技术实现

### 3.1 数据文件结构

#### `sources.json` - 语料元数据
```typescript
type CorpusMeta = {
  id: string              // 唯一标识
  language: string        // BCP 47 语言标签
  direction: 'ltr' | 'rtl'
  title: string           // 显示标题
  source_url: string      // 来源 URL
  output: string          // 本地 txt 路径
  font_family: string     // 默认字体栈
  font_size_px: number
  line_height_px: number
  default_width: number   // 默认测试宽度
  min/max_width: number   // 测试范围
  characters: number      // 字符数统计
  lines: number           // 行数统计
  acquired_via: string    // 获取方式 (parse/extract/copy/synthetic)
}
```

#### `representative.json` - 锚点测试结果
```typescript
type RepresentativeRow = {
  corpusId: string
  title: string
  language: string
  direction: string
  width: number           // 测试宽度
  contentWidth: number    // 内容宽度（减去 padding）
  font: string
  lineHeight: number
  predictedHeight: number // Pretext 预测高度
  actualHeight: number    // 浏览器实际高度
  diffPx: number          // 差异像素
  predictedLineCount: number
  browserLineCount: number
}
```

#### `chrome-sampled.json` / `chrome-step10.json` - 扫描结果
```typescript
type SweepSummary = {
  corpusId: string
  language: string
  title: string
  browser: 'chrome' | 'safari'
  start/end/step: number  // 扫描参数
  samples: number | null  // 采样数（null 表示全量）
  widthCount: number      // 总测试宽度数
  exactCount: number      // 完全匹配数
  mismatches: SweepMismatch[]
}
```

#### `dashboard.json` - 聚合仪表板
聚合所有测试结果，包含：
- `browserRegressionGate`: 浏览器回归门控统计
- `productShaped`: 产品形态语料状态
- `longForm`: 长文本语料状态
- `fineSweepNotes`: 细粒度扫描备注
- `fontMatrixNotes`: 字体矩阵测试结果

### 3.2 关键脚本实现

#### `scripts/corpus-check.ts` - 单点检查

**核心流程：**
```
1. 解析命令行参数 (--id, --browser, --method, --font, --lineHeight, --sliceStart/End)
2. 加载 sources.json 获取语料元数据
3. 获取浏览器自动化锁 (acquireBrowserAutomationLock)
4. 启动临时页面服务器 (ensurePageServer)
5. 构建测试 URL: /corpus?id=<id>&width=<w>&report=1&diagnostic=<mode>
6. 浏览器导航并等待报告 (loadHashReport)
7. 解析报告并打印结果
8. 清理：关闭浏览器、停止服务器、释放锁
```

**报告数据结构：**
```typescript
type CorpusReport = {
  status: 'ready' | 'error'
  environment: EnvironmentFingerprint  // UA、DPR、视口等
  corpusId: string
  width: number
  predictedHeight: number
  actualHeight: number
  diffPx: number
  predictedLineCount: number
  browserLineCount: number
  browserLineMethod: 'span-probe' | 'range'  // 行提取方法
  firstBreakMismatch: CorpusBreakMismatch | null  // 首个断点差异
  maxLineWidthDrift: number  // 最大行宽漂移
  extractorSensitivity: string | null  // 提取器敏感性提示
}
```

#### `scripts/corpus-sweep.ts` - 宽度扫描

**采样策略：**
```typescript
function getSweepWidths(meta, options) {
  if (options.samples !== null) {
    // 均匀采样：在 [min, max] 区间等距取 samples 个点
    for (let i = 0; i < samples; i++) {
      const ratio = i / (samples - 1)
      widths.push(Math.round(min + (max - min) * ratio))
    }
  } else {
    // 步进扫描：从 start 到 end，每隔 step 一个点
    for (let w = min; w <= max; w += step) widths.push(w)
  }
}
```

**批量报告传输：**
- 使用 `report-server.ts` 启动本地 HTTP 服务端点
- 页面通过 POST 将大批量结果发送到该端点
- 避免 URL hash 长度限制

#### `scripts/corpus-taxonomy.ts` - 分类诊断

**分类逻辑：**
```typescript
function classifyTaxonomy(row: CorpusSweepRow): TaxonomyCategory {
  const mismatch = row.firstBreakMismatch
  const reason = mismatch?.reasonGuess ?? ''
  
  if (reason.includes('only') && reason.includes('overflow')) {
    return 'edge-fit'
  }
  if (reason.includes('segment sum drifts')) {
    return 'shaping-context'
  }
  if (row.browserLineMethod === 'span-probe' && 
      (row.maxLineWidthDrift ?? 0) === 0 && 
      mismatch === null) {
    return 'diagnostic-sensitivity'
  }
  if (mismatch != null && quoteOrPunctuationRe.test(mismatch.deltaText)) {
    return 'glue-policy'
  }
  if (mismatch != null) {
    return 'boundary-discovery'
  }
  return 'unknown'
}
```

#### `scripts/corpus-font-matrix.ts` - 字体矩阵

**字体变体配置（FONT_MATRIX）：**
```typescript
const FONT_MATRIX: Record<string, FontVariant[]> = {
  'ja-rashomon': [
    { id: 'default', label: 'Hiragino Mincho ProN', 
      font: '20px "Hiragino Mincho ProN", ...', lineHeight: 32 },
    { id: 'hiragino-sans', label: 'Hiragino Sans',
      font: '20px "Hiragino Sans", ...', lineHeight: 32 }
  ],
  'zh-zhufu': [
    { id: 'default', label: 'Songti SC', ... },
    { id: 'pingfang-sc', label: 'PingFang SC', ... }
  ],
  // ... 其他语料
}
```

### 3.3 页面实现 (`pages/corpus.ts`)

**诊断行提取双模式：**

1. **Span Probe 模式** (`getBrowserLinesFromSpans`)
   - 将每个诊断单元（diagnostic unit）包装为 `<span>`
   - 通过 `getBoundingClientRect()` 获取每个 span 的位置
   - 检测 `rect.top` 变化判断换行
   - 限制：对东南亚脚本（泰、老、高棉、缅甸）和 RTL 不可靠

2. **Range 模式** (`getBrowserLinesFromRange`)
   - 使用 `document.createRange()` 设置起始/结束偏移
   - 通过 `range.getClientRects()` 获取位置
   - 更准确但性能较低

**自动选择逻辑：**
```typescript
const requiresRangeProbe = rangeProbeScriptRe.test(normalizedText)
// rangeProbeScriptRe = /[\u0E00-\u0E7F\u0E80-\u0EFF\u1000-\u109F\u1780-\u17FF]/u
// 匹配泰文、老挝文、缅甸文、高棉文

const probeReliable = !requiresRangeProbe && 
                      direction !== 'rtl' &&
                      Math.abs(probeHeight - normalizedHeight) <= Math.max(1, lineHeight / 2)
```

**断点差异分析：**
```typescript
function getFirstBreakMismatch(prepared, normalizedText, contentWidth, 
                               ourLines, browserLines, font, direction): 
                               CorpusBreakMismatch | null {
  // 逐行对比，找到第一个起始或结束偏移不同的行
  // 返回详细的差异信息：文本、上下文、宽度测量、原因猜测
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 调用关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                         命令层 (CLI)                             │
├─────────────────┬─────────────────┬─────────────────────────────┤
│ corpus-check    │ corpus-sweep    │ corpus-taxonomy             │
│ corpus-sweep    │ corpus-sweep    │ corpus-font-matrix          │
└────────┬────────┴────────┬────────┴──────────────┬──────────────┘
         │                 │                       │
         ▼                 ▼                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                     浏览器自动化层                                │
│              scripts/browser-automation.ts                        │
│  - acquireBrowserAutomationLock()  # 文件锁，防止并发             │
│  - createBrowserSession()          # Chrome/Safari/Firefox      │
│  - ensurePageServer()              # Bun dev server             │
│  - loadHashReport() / loadPostedReport()  # 报告获取            │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      页面执行层                                   │
│                   pages/corpus.ts                                │
│  - 加载语料文本 (import '../corpora/xxx.txt')                    │
│  - prepareWithSegments()  # 文本预处理                           │
│  - layout() / layoutWithLines()  # Pretext 布局                  │
│  - 浏览器渲染对比 (DOM height vs predicted height)               │
│  - 诊断提取 (span-probe / range)                                 │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      核心引擎层                                   │
│              src/layout.ts, src/analysis.ts                      │
│  - prepare() / prepareWithSegments()  # 分段与归一化              │
│  - layout()  # 行布局算法                                        │
│  - Intl.Segmenter  # 词/字素分段                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 关键文件索引

| 文件 | 类型 | 职责 |
|------|------|------|
| `corpora/sources.json` | 数据 | 语料元数据注册表 |
| `corpora/representative.json` | 数据 | 锚点宽度（300/600/800）测试结果 |
| `corpora/chrome-sampled.json` | 数据 | 9点采样扫描结果 |
| `corpora/chrome-step10.json` | 数据 | 10px步进全扫描结果 |
| `corpora/dashboard.json` | 数据 | 聚合仪表板（由 corpus-status.ts 生成） |
| `corpora/TAXONOMY.md` | 文档 | 不匹配分类体系定义 |
| `scripts/corpus-check.ts` | 脚本 | 单点精确检查 |
| `scripts/corpus-sweep.ts` | 脚本 | 多宽度扫描 |
| `scripts/corpus-taxonomy.ts` | 脚本 | 分类诊断 |
| `scripts/corpus-font-matrix.ts` | 脚本 | 字体矩阵测试 |
| `scripts/corpus-representative.ts` | 脚本 | 生成 representative.json |
| `scripts/corpus-status.ts` | 脚本 | 生成 dashboard.json |
| `scripts/browser-automation.ts` | 共享 | 浏览器自动化基础设施 |
| `scripts/report-server.ts` | 共享 | 批量报告接收服务 |
| `pages/corpus.ts` | 页面 | 浏览器端测试执行 |
| `pages/diagnostic-utils.ts` | 共享 | 诊断工具函数 |
| `src/layout.ts` | 核心 | 布局引擎 API |
| `src/analysis.ts` | 核心 | 文本分析与归一化 |

---

## 5. 依赖与外部交互

### 5.1 内部依赖

```
corpora/ ←── pages/corpus.ts (通过 import 加载 txt)
     ↑
     └── scripts/corpus-*.ts (读取 sources.json, 生成 JSON 快照)
     ↑
     └── scripts/browser-automation.ts (浏览器控制)
     ↑
     └── scripts/status-dashboard.ts (读取生成仪表板)
```

### 5.2 外部依赖

| 依赖 | 用途 | 说明 |
|------|------|------|
| `bun` | 运行时 + 包管理 | 所有脚本使用 Bun API (`Bun.file`) |
| `osascript` | Safari/Chrome 自动化 | macOS AppleScript 控制浏览器 |
| `WebSocket` | Firefox BiDi 协议 | Firefox 远程调试协议 |
| `Intl.Segmenter` | 文本分段 | 浏览器原生 API，用于词/字素分段 |
| `Canvas API` | 文本测量 | `measureText()` 获取字符宽度 |
| `DOM Range` | 行提取 | `document.createRange()` 精确获取文本位置 |

### 5.3 浏览器自动化锁

**锁文件位置：** `$TMPDIR/pretext-browser-automation-locks/{chrome,safari,firefox}.lock`

**机制：**
```typescript
// 创建独占锁文件（O_EXCL），包含进程 PID 和时间戳
// 如果锁存在但进程已死，自动清理并重新获取
// 超时：120 秒
```

**必要性：**
- 防止多个脚本同时控制同一浏览器
- 避免自动化会话冲突

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| **提取器敏感性** | 某些不匹配仅在特定提取方法（span/range）下出现 | 自动检测并报告 `extractorSensitivity` |
| **字体回退差异** | 不同系统字体解析不同 | 明确指定字体栈，避免 `system-ui` |
| **DPR/缩放影响** | 设备像素比影响测量精度 | 记录环境指纹，标准化测试环境 |
| **语料污染** | 原始文本包含换行符、页眉页脚 | 手动清理，标记 `acquired_via` 方式 |
| **东南亚脚本** | Span 探测会干扰布局 | 强制使用 Range 模式 |
| **RTL 文本** | Span 探测不可靠 | 强制使用 Range 模式 |

### 6.2 边界条件

**行适配容差：**
```typescript
// Chromium/Gecko: 0.005px
// Safari/WebKit: 1/64 px (~0.0156px)
// 原因：阿拉伯文细宽度场需要更宽松的容差
```

**宽度范围限制：**
- 默认：300-900px（涵盖移动端到桌面端）
- 可覆盖：`--start`, `--end`, `--step`
- 语料级限制：`min_width`, `max_width` in sources.json

**切片支持：**
- 支持 `--sliceStart` / `--sliceEnd` 测试语料子集
- 用于定位大语料中的问题区域

### 6.3 改进建议

#### 短期（维护性）

1. **自动化锁持久化**
   - 当前：进程级内存锁
   - 建议：增加锁的持久化检查，防止崩溃后死锁

2. **语料版本控制**
   - 当前：txt 文件直接修改
   - 建议：增加校验和或版本号，检测语料变更后自动失效缓存结果

3. **错误恢复**
   - 当前：单点失败导致整个扫描终止
   - 建议：增加 `--continue-on-error` 模式，记录失败宽度后继续

#### 中期（功能性）

4. **增量扫描**
   - 当前：每次全量扫描
   - 建议：基于上次结果，仅扫描可能受影响的宽度范围

5. **并行扫描**
   - 当前：单浏览器串行
   - 建议：支持多浏览器实例并行（需更精细的锁机制）

6. **可视化报告**
   - 当前：JSON + 控制台输出
   - 建议：生成 HTML 报告，展示宽度-差异曲线、行对比高亮

#### 长期（架构性）

7. **Shaping-Aware 模型**
   - 当前问题：乌尔都文（Nastaliq）等连体字形脚本存在系统性差异
   - 建议：研究字形塑形感知（shaping-aware）的布局模型

8. **跨平台字体矩阵**
   - 当前：字体矩阵主要针对 macOS 系统字体
   - 建议：扩展支持 Windows（DirectWrite）和 Linux（Harfbuzz）字体栈

9. **CI/CD 集成**
   - 当前：本地手动运行
   - 建议：GitHub Actions 工作流，定期回归测试并更新快照

---

## 附录：快速参考

### 常用命令

```bash
# 单语料检查
bun run corpus-check --id=ja-rashomon 300 600 800

# 全量扫描（9点采样）
bun run corpus-sweep --all --samples=9 --output=corpora/chrome-sampled.json

# 细粒度扫描
bun run corpus-sweep --id=zh-zhufu --start=300 --end=900 --step=10

# 分类诊断
bun run corpus-taxonomy --id=ur-chughd 300 340 600

# 字体矩阵
bun run corpus-font-matrix --id=zh-zhufu --samples=5

# 更新所有快照
bun run corpus-status:refresh
```

### 文件生成关系

```
corpora/sources.json (手动维护)
    ↓
corpora/representative.json (corpus-representative.ts 生成)
corpora/chrome-sampled.json (corpus-sweep.ts --samples=9 生成)
corpora/chrome-step10.json (corpus-sweep.ts --step=10 生成)
    ↓
corpora/dashboard.json (corpus-status.ts 聚合生成)
```
