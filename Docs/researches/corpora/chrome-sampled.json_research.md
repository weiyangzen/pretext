# chrome-sampled.json 研究文档

## 场景与职责

`chrome-sampled.json` 是 Pretext 项目的**语料库采样精度快照文件**，用于记录在 Chrome 浏览器下对多语言长文本语料进行**稀疏采样测试**的结果。该文件是项目质量保障体系的核心组成部分，用于追踪文本布局引擎在不同语言、不同文本类型下的准确性表现。

### 核心职责

1. **稀疏采样验证**：在 300-900px 宽度范围内，以 9 个采样点（步长非均匀分布）快速验证语料库精度
2. **回归检测**：作为 CI/CD 流程中的基准数据，检测文本布局引擎的回归问题
3. **多语言覆盖**：覆盖 17 种语料，包括英语、日语、韩语、中文、泰语、缅甸语、高棉语、阿拉伯语、印地语、希伯来语、乌尔都语等
4. **快速健康检查**：相比 `chrome-step10.json` 的密集测试，提供更快速的语料健康度评估

### 语料分类

| 类别 | 语料ID | 说明 |
|------|--------|------|
| 产品形态文本 | `mixed-app-text` | 混合脚本应用文本（URL、引号、RTL、Emoji等） |
| 英文长文本 | `en-gatsby-opening` | 《了不起的盖茨比》开篇 |
| 日文 | `ja-rashomon`, `ja-kumo-no-ito` | 《罗生门》《蜘蛛之丝》 |
| 中文 | `zh-zhufu`, `zh-guxiang` | 《祝福》《故乡》 |
| 韩文 | `ko-unsu-joh-eun-nal` | 《幸运的一天》 |
| 泰文 | `th-nithan-vetal-story-1`, `th-nithan-vetal-story-7` | 泰语民间故事 |
| 缅甸文 | `my-cunning-heron-teacher`, `my-bad-deeds-return-to-you-teacher` | 缅甸语故事 |
| 高棉文 | `km-prachum-reuang-preng-khmer-volume-7-stories-1-10` | 柬埔寨民间故事 |
| 阿拉伯文 | `ar-risalat-al-ghufran-part-1`, `ar-al-bukhala` | 《宽恕书》《吝啬鬼》 |
| 印地文 | `hi-eidgah` | 《伊德加赫》 |
| 希伯来文 | `he-masaot-binyamin-metudela` | 便雅悯游记 |
| 乌尔都文 | `ur-chughd` | 《雏鸡》 |

---

## 功能点目的

### 1. 精度指标记录

每个语料条目记录以下关键指标：

```typescript
interface SweepSummary {
  corpusId: string;        // 语料唯一标识
  language: string;        // 语言代码 (en, ja, zh, th, my, km, ar, hi, he, ur, mul)
  title: string;           // 语料标题
  browser: 'chrome';       // 浏览器类型
  start: number;           // 测试起始宽度 (300)
  end: number;             // 测试结束宽度 (900)
  step: number;            // 步长 (10)
  samples: number;         // 采样数 (9)
  widthCount: number;      // 实际测试宽度数量
  exactCount: number;      // 完全匹配的宽度数量
  mismatches: SweepMismatch[];  // 不匹配记录
}
```

### 2. 不匹配分析

```typescript
interface SweepMismatch {
  width: number;           // 发生不匹配的宽度
  diffPx: number;          // 高度差异（像素）
  predictedHeight: number; // 预测高度
  actualHeight: number;    // 实际浏览器高度
  predictedLineCount: number;  // 预测行数
  browserLineCount: number;    // 浏览器实际行数
}
```

### 3. 采样策略

- **采样范围**：300px 到 900px（覆盖移动端到桌面端常见宽度）
- **采样点**：9 个均匀分布的宽度点
- **容差标准**：行高差异在 1 行以内视为可接受（diffPx ≈ ±lineHeight）

---

## 具体技术实现

### 数据生成流程

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  corpus-sweep.ts │ --> │  pages/corpus.ts  │ --> │ chrome-sampled  │
│   (测试脚本)      │     │  (浏览器测试页面)  │     │   .json         │
└─────────────────┘     └──────────────────┘     └─────────────────┘
        │                        │                       │
        ▼                        ▼                       ▼
   加载 sources.json      使用 Pretext 引擎        写入采样结果
   遍历语料列表           与浏览器 DOM 对比
   生成测试宽度序列
```

### 关键代码路径

**1. 测试脚本入口** (`scripts/corpus-sweep.ts`)

```typescript
// 第 121-123 行：加载语料源数据
async function loadSources(): Promise<CorpusMeta[]> {
  return await Bun.file('corpora/sources.json').json()
}

// 第 170-195 行：生成采样宽度
function getSweepWidths(meta: CorpusMeta, options: SweepOptions): number[] {
  if (options.samples !== null) {
    // 采样模式：均匀分布 n 个采样点
    const sampled = new Set<number>()
    for (let i = 0; i < samples; i++) {
      const ratio = i / (samples - 1)
      const width = Math.round(min + (max - min) * ratio)
      sampled.add(width)
    }
    return [...sampled].sort((a, b) => a - b)
  }
}
```

**2. 浏览器测试页面** (`pages/corpus.ts`)

```typescript
// 第 945-981 行：执行宽度扫描
function runSweep(widths: number[]): void {
  const rows = widths.map(width => {
    const report = measureWidth(width, { publish: false, updateStats: false })
    return toSweepRow(report)
  })
  const exactCount = rows.filter(row => Math.round(row.diffPx) === 0).length
  // 生成报告...
}

// 第 843-904 行：单宽度测量
function measureWidth(width: number, options): CorpusReport | null {
  const predicted = layout(prepared, contentWidth, lineHeight)  // Pretext 预测
  const actualHeight = book.getBoundingClientRect().height       // 浏览器实际高度
  // 对比并生成报告...
}
```

**3. 输出写入** (`scripts/corpus-sweep.ts` 第 354-357 行)

```typescript
if (options.output !== null) {
  writeFileSync(options.output, JSON.stringify(summaries, null, 2))
  console.log(`wrote ${options.output}`)
}
```

### 数据结构详解

当前快照数据示例（节选）：

```json
{
  "corpusId": "mixed-app-text",
  "language": "mul",
  "title": "Mixed app text",
  "browser": "chrome",
  "start": 300,
  "end": 900,
  "step": 10,
  "samples": 9,
  "widthCount": 9,
  "exactCount": 9,
  "mismatches": []
}
```

**关键字段说明**：

- `samples: 9`：表示在 300-900px 范围内均匀采样 9 个点
- `widthCount`: 实际测试的宽度数量（去重后）
- `exactCount`: 预测高度与实际高度完全匹配的宽度数量
- `mismatches`: 不匹配的详细记录，包含像素级差异

---

## 关键代码路径与文件引用

### 生成路径

| 文件 | 职责 | 关键函数/行号 |
|------|------|--------------|
| `scripts/corpus-sweep.ts` | 测试编排与结果收集 | `loadSources()` (L121), `getSweepWidths()` (L170), `runDetailedDiagnose()` (L230) |
| `pages/corpus.ts` | 浏览器端测试执行 | `runSweep()` (L945), `measureWidth()` (L843), `toSweepRow()` (L906) |
| `src/layout.ts` | Pretext 布局引擎 | `layout()`, `layoutWithLines()`, `prepareWithSegments()` |
| `scripts/browser-automation.ts` | 浏览器自动化 | `createBrowserSession()`, `loadPostedReport()` |
| `scripts/report-server.ts` | 测试结果接收 | `startPostedReportServer()` |

### 消费路径

| 文件 | 职责 | 引用方式 |
|------|------|----------|
| `scripts/corpus-status.ts` | 生成仪表盘 | `loadJson<SweepSummary[]>('corpora/chrome-sampled.json')` (L277) |
| `corpora/dashboard.json` | 聚合展示 | 被合并到 `chromeSampled` 字段中 |
| `DEVELOPMENT.md` | 开发文档 | 作为 "Current Sources Of Truth" 引用 |

### 命令行调用

```bash
# 生成 chrome-sampled.json（9个采样点）
bun run corpus-sweep --id=mixed-app-text --samples=9 --browser=chrome --output=corpora/chrome-sampled.json

# 或全量语料
bun run corpus-sweep --all --samples=9 --browser=chrome --output=corpora/chrome-sampled.json
```

---

## 依赖与外部交互

### 依赖文件

```
corpora/chrome-sampled.json
├── 读取 corpora/sources.json          # 语料元数据源
├── 依赖 pages/corpus.ts               # 浏览器测试页面
├── 依赖 src/layout.ts                 # Pretext 核心引擎
├── 依赖 scripts/browser-automation.ts # 浏览器自动化
├── 生成 corpora/dashboard.json        # 被 corpus-status.ts 消费
└── 被 status/dashboard.json 引用      # 主状态仪表盘
```

### 浏览器依赖

- **Chrome**：主要测试目标浏览器
- **CDP (Chrome DevTools Protocol)**：通过 `scripts/browser-automation.ts` 控制浏览器

### 运行时依赖

- **Bun**：JavaScript 运行时和包管理器
- **临时 HTTP 服务器**：用于提供测试页面（端口动态分配）

### 数据流

```
┌─────────────────────────────────────────────────────────────────┐
│                        数据生成阶段                               │
├─────────────────────────────────────────────────────────────────┤
│  sources.json ──> corpus-sweep.ts ──> Chrome ──> chrome-sampled  │
│       │                                    │           .json     │
│       │                                    │                      │
│       ▼                                    ▼                      │
│  语料元数据                          pages/corpus.ts              │
│  (字体/行高/方向)                    (布局对比测试)                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        数据消费阶段                               │
├─────────────────────────────────────────────────────────────────┤
│  chrome-sampled.json ──> corpus-status.ts ──> dashboard.json     │
│       │                                                           │
│       ├──> 计算 exactCount/widthCount                            │
│       ├──> 合并到 productShaped/longForm 分类                     │
│       └──> 生成可读的状态报告                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 风险、边界与改进建议

### 已知风险

#### 1. 采样稀疏性风险

**问题**：9 个采样点可能遗漏某些特定宽度下的布局问题。

**证据**：`chrome-step10.json`（61个宽度点）相比 `chrome-sampled.json`（9个点）发现了更多不匹配：
- `en-gatsby-opening`: sampled 8/9 exact → step10 50/61 exact
- `ur-chughd`: sampled 4/9 exact → step10 31/61 exact

**缓解**：`chrome-step10.json` 作为补充，用于详细分析。

#### 2. 浏览器版本敏感性

**问题**：不同 Chrome 版本的字体渲染可能存在亚像素差异。

**当前状态**：快照在特定 Chrome 版本（146.0.0.0）下生成，记录在 `representative.json` 的 `environment` 字段中。

#### 3. 字体可用性差异

**问题**：不同系统（macOS/Windows/Linux）的默认字体不同，影响测试结果。

**示例**：`zh-zhufu` 在 `Songti SC` 和 `PingFang SC` 下表现不同（见 `corpora/dashboard.json` fontMatrixNotes）。

### 边界条件

| 边界 | 说明 |
|------|------|
| 宽度范围 | 300-900px，超出此范围需要额外测试 |
| 语言覆盖 | 17 种语料，不包含所有 Unicode 脚本 |
| 字体回退 | 依赖系统字体，未强制指定特定字体文件 |
| 行高模式 | 仅测试 `white-space: normal`，`pre-wrap` 模式单独测试 |

### 改进建议

#### 1. 增加动态采样

```typescript
// 建议：在已知问题宽度附近增加采样密度
function getAdaptiveSweepWidths(meta: CorpusMeta, knownIssues: number[]): number[] {
  const baseWidths = getUniformSamples(300, 900, 9)
  const extraWidths = knownIssues.flatMap(w => [w - 5, w, w + 5])
  return [...new Set([...baseWidths, ...extraWidths])].sort((a, b) => a - b)
}
```

#### 2. 版本追踪增强

```json
{
  "generatedAt": "2026-03-31T10:07:49.046Z",
  "browserVersion": "Chrome/146.0.0.0",
  "osVersion": "macOS 14.5",
  "fontHash": "sha256:xxx"  // 建议：记录实际使用的字体指纹
}
```

#### 3. 自动化回归检测

```bash
# 建议：CI 流程中自动对比
bun run corpus-sweep --all --samples=9 --browser=chrome --output=/tmp/new-sampled.json
diff <(jq '.[] | {id, exactCount}' corpora/chrome-sampled.json) \
     <(jq '.[] | {id, exactCount}' /tmp/new-sampled.json)
```

#### 4. 多浏览器扩展

当前仅 Chrome 有采样快照，建议增加：
- `safari-sampled.json`
- `firefox-sampled.json`

#### 5. 与 dashboard.json 的合并优化

当前 `corpus-status.ts` 手动维护 `PRODUCT_SHAPED` 和 `LONG_FORM` 数组，建议：

```typescript
// 建议：从 sources.json 自动推导分类
const categories = {
  productShaped: sources.filter(s => s.id === 'mixed-app-text'),
  longForm: sources.filter(s => s.id !== 'mixed-app-text')
}
```

### 维护 checklist

- [ ] 每次更新 `src/layout.ts` 或 `src/analysis.ts` 后重新生成
- [ ] 每次添加新语料后更新
- [ ] 每次升级 Chrome 主版本后验证
- [ ] 与 `chrome-step10.json` 定期对比，确保采样代表性
