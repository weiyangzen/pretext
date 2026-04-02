# dashboard.json 研究文档

## 场景与职责

`corpora/dashboard.json` 是 Pretext 项目的**语料库状态仪表盘文件**，是长文本语料测试结果的**机器可读聚合视图**。它整合了来自 `chrome-sampled.json`、`chrome-step10.json`、`representative.json` 以及浏览器精度测试的多源数据，为开发者和自动化工具提供统一的状态查询接口。

### 核心职责

1. **数据聚合**：将分散的 JSON 快照文件整合为统一的仪表盘视图
2. **状态可视化**：提供产品形态文本和长文本语料的健康状态概览
3. **回归检测支持**：为 CI/CD 流程提供机器可读的质量门禁数据
4. **多维度对比**：支持 Chrome/Safari 双浏览器的锚点对比

### 仪表盘架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     corpora/dashboard.json                       │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │  browserRegression │  │  productShaped   │  │   longForm      │ │
│  │     Gate         │  │   (产品形态文本)   │  │   (长文本语料)   │ │
│  │                  │  │                  │  │                  │ │
│  │  Chrome/Safari/  │  │  mixed-app-text  │  │  en-gatsby...   │ │
│  │  Firefox 精度    │  │                  │  │  ja-rashomon    │ │
│  │                  │  │  混合脚本/URL/    │  │  zh-zhufu       │ │
│  │  7680/7680 match │  │  Emoji/RTL       │  │  ...            │ │
│  │                  │  │                  │  │                  │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                │
│  ┌─────────────────┐  ┌─────────────────┐                      │
│  │  fineSweepNotes  │  │  fontMatrixNotes │                      │
│  │  (精细扫描备注)   │  │  (字体矩阵备注)   │                      │
│  │                  │  │                  │                      │
│  │  步长=1的扫描    │  │  不同字体表现     │                      │
│  │  结果摘要        │  │  对比分析        │                      │
│  └─────────────────┘  └─────────────────┘                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 功能点目的

### 1. 浏览器回归门禁 (browserRegressionGate)

```typescript
interface BrowserRegressionGate {
  chrome: {
    total: 7680;        // 总测试用例数
    matchCount: 7680;   // 匹配数
    mismatchCount: 0;   // 不匹配数
  };
  safari: { /* 同上 */ };
  firefox: { /* 同上 */ };
}
```

**用途**：
- CI/CD 质量门禁：任何非零 `mismatchCount` 触发构建失败
- 快速健康检查：7680 个测试用例全部通过表示核心算法稳定

### 2. 产品形态文本 (productShaped)

```typescript
interface ProductShapedEntry {
  id: 'mixed-app-text';
  title: 'Mixed app text';
  language: 'mul';  // 多语言
  chromeAnchors: {
    exactWidths: [300, 600, 800];  // 精确匹配的锚点宽度
    mismatches: [];                // 锚点处的不匹配
  };
  safariAnchors: { /* 同上 */ };
  chromeSampled: {
    exactCount: 9;
    widthCount: 9;
  };
  chromeStep10: {
    exactCount: 60;
    widthCount: 61;
  };
  notes: 'remaining Chrome-only `710px` miss is SHY / extractor-sensitive';
}
```

**用途**：
- 监控产品级文本场景（URL、Emoji、RTL 等）的准确性
- 追踪特定宽度的已知问题（如 710px 的软连字符问题）

### 3. 长文本语料 (longForm)

包含 16 个长文本语料的详细状态，每个条目包含：
- 多浏览器锚点测试结果
- 稀疏和密集采样的精度统计
- 人工备注（误差模式分析）

### 4. 精细扫描备注 (fineSweepNotes)

记录步长=1px 的精细扫描结果：

```typescript
interface FineSweepNote {
  corpusId: string;
  result: 
    | { kind: 'exact_count', exactCount: number, widthCount: number }
    | { kind: 'not_fully_mapped', label: string };
  notes: string;  // 人工分析备注
}
```

### 5. 字体矩阵备注 (fontMatrixNotes)

记录同一语料在不同字体下的表现差异：

```typescript
interface FontMatrixNote {
  corpusId: string;
  status: string;  // 如 "clean on sampled matrix"
  notes: string;   // 如 "`Hiragino Mincho ProN` had `+32px` at `450px`"
}
```

---

## 具体技术实现

### 数据生成流程

```
┌─────────────────────────────────────────────────────────────────┐
│                  dashboard.json 生成流程                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  输入文件                                                        │
│  ├── accuracy/chrome.json ──┐                                  │
│  ├── accuracy/safari.json ──┼──> browserRegressionGate         │
│  └── accuracy/firefox.json ─┘                                  │
│                                                                 │
│  ├── corpora/representative.json ──┐                           │
│  │                                  ├──> productShaped         │
│  ├── corpora/chrome-sampled.json ──┤      longForm             │
│  └── corpora/chrome-step10.json ───┘      chromeAnchors        │
│                                           safariAnchors        │
│                                                                 │
│  硬编码元数据                                                    │
│  ├── PRODUCT_SHAPED ──────────────┐                            │
│  ├── LONG_FORM ───────────────────┼──> 分类和备注              │
│  ├── FINE_SWEEP_NOTES ────────────┤                            │
│  └── FONT_MATRIX_NOTES ───────────┘                            │
│                                                                 │
│  输出：corpora/dashboard.json                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 关键代码实现

**1. 主脚本** (`scripts/corpus-status.ts`)

```typescript
// 第 59-131 行：硬编码语料分类和备注
const PRODUCT_SHAPED: CorpusDashboardMeta[] = [
  {
    id: 'mixed-app-text',
    notes: 'remaining Chrome-only `710px` miss is SHY / extractor-sensitive...',
  },
]

const LONG_FORM: CorpusDashboardMeta[] = [
  { id: 'en-gatsby-opening', notes: 'legacy Gatsby long-form canary...' },
  { id: 'ja-kumo-no-ito', notes: 'second Japanese canary...' },
  // ... 16 个语料
]

const FINE_SWEEP_NOTES: FineSweepNote[] = [
  {
    corpusId: 'ar-risalat-al-ghufran-part-1',
    result: { kind: 'exact_count', exactCount: 594, widthCount: 601 },
    notes: 'remaining misses are one-line positive edge-fit cases',
  },
  // ...
]

const FONT_MATRIX_NOTES: FontMatrixNote[] = [
  {
    corpusId: 'ja-kumo-no-ito',
    status: 'sampled matrix has a small field',
    notes: '`Hiragino Mincho ProN` had `+32px` at `450px`...',
  },
  // ...
]
```

**2. 数据加载与索引** (`scripts/corpus-status.ts` 第 230-286 行)

```typescript
// 加载所有源数据
const representative = await loadJson<RepresentativeSnapshot>('corpora/representative.json')
const chromeSampled = await loadJson<SweepSummary[]>('corpora/chrome-sampled.json')
const chromeStep10 = await loadJson<SweepSummary[]>('corpora/chrome-step10.json')
const chromeAccuracy = await loadJson<AccuracySnapshot>('accuracy/chrome.json')
const safariAccuracy = await loadJson<AccuracySnapshot>('accuracy/safari.json')
const firefoxAccuracy = await loadJson<AccuracySnapshot>('accuracy/firefox.json')

// 建立索引
const chromeRepresentativeByCorpus = indexRepresentativeRows(representative, 'chrome')
const safariRepresentativeByCorpus = indexRepresentativeRows(representative, 'safari')
const sampledByCorpus = indexSweepSummaries(chromeSampled)
const step10ByCorpus = indexSweepSummaries(chromeStep10)
```

**3. 仪表盘组装** (`scripts/corpus-status.ts` 第 288-336 行)

```typescript
const dashboard = {
  generatedAt: new Date().toISOString(),
  sources: {
    accuracy: { chrome: 'accuracy/chrome.json', safari: '...', firefox: '...' },
    representative: 'corpora/representative.json',
    chromeSampled: 'corpora/chrome-sampled.json',
    chromeStep10: 'corpora/chrome-step10.json',
    taxonomy: 'corpora/TAXONOMY.md',
  },
  browserRegressionGate: {
    chrome: summarizeAccuracy(chromeAccuracy),
    safari: summarizeAccuracy(safariAccuracy),
    firefox: summarizeAccuracy(firefoxAccuracy),
  },
  productShaped: PRODUCT_SHAPED.map(meta => ({
    id: meta.id,
    title: step10?.title ?? sampled?.title ?? meta.id,
    language: step10?.language ?? sampled?.language ?? '',
    chromeAnchors: summarizeAnchors(chromeRepresentativeByCorpus.get(meta.id)),
    safariAnchors: summarizeAnchors(safariRepresentativeByCorpus.get(meta.id)),
    chromeSampled: sampled ? { exactCount: sampled.exactCount, widthCount: sampled.widthCount } : null,
    chromeStep10: step10 ? { exactCount: step10.exactCount, widthCount: step10.widthCount } : null,
    notes: meta.notes,
  })),
  longForm: LONG_FORM.map(/* 同上 */),
  fineSweepNotes: FINE_SWEEP_NOTES,
  fontMatrixNotes: FONT_MATRIX_NOTES,
}
```

### 辅助函数

**1. 锚点汇总** (`scripts/corpus-status.ts` 第 255-265 行)

```typescript
function summarizeAnchors(rows: RepresentativeRow[] | undefined): AnchorSummary | null {
  if (rows === undefined || rows.length === 0) return null
  
  const sorted = [...rows].sort((a, b) => a.width - b.width)
  return {
    exactWidths: sorted.filter(row => Math.round(row.diffPx) === 0).map(row => row.width),
    mismatches: sorted
      .filter(row => Math.round(row.diffPx) !== 0)
      .map(row => ({ width: row.width, diffPx: Math.round(row.diffPx) })),
  }
}
```

**2. 精度汇总** (`scripts/corpus-status.ts` 第 267-273 行)

```typescript
function summarizeAccuracy(snapshot: AccuracySnapshot) {
  return {
    total: snapshot.total ?? 0,
    matchCount: snapshot.matchCount ?? 0,
    mismatchCount: snapshot.mismatchCount ?? 0,
  }
}
```

---

## 关键代码路径与文件引用

### 生成路径

| 文件 | 职责 | 关键行号 |
|------|------|----------|
| `scripts/corpus-status.ts` | 仪表盘生成器 | L59-131 (元数据), L230-286 (数据加载), L288-336 (组装) |
| `corpora/representative.json` | 锚点数据源 | 被 indexRepresentativeRows() 消费 |
| `corpora/chrome-sampled.json` | 稀疏采样源 | 被 indexSweepSummaries() 消费 |
| `corpora/chrome-step10.json` | 密集采样源 | 被 indexSweepSummaries() 消费 |
| `accuracy/*.json` | 精度数据源 | 被 summarizeAccuracy() 消费 |

### 消费路径

| 文件 | 职责 | 引用方式 |
|------|------|----------|
| `status/dashboard.json` | 主仪表盘 | `sources.corpora = 'corpora/dashboard.json'` |
| `corpora/STATUS.md` | 语料状态文档 | 人工阅读参考 |
| CI/CD 脚本 | 质量门禁 | 读取 `browserRegressionGate` |

### 命令行调用

```bash
# 生成 dashboard.json
bun run corpus-status

# 生成并指定输出路径
bun run corpus-status --output=corpora/dashboard.json

# 刷新（重新生成依赖的快照后）
bun run corpus-status:refresh
```

---

## 依赖与外部交互

### 文件依赖图

```
corpora/dashboard.json
│
├─ 读取 ──> accuracy/chrome.json           (浏览器精度)
├─ 读取 ──> accuracy/safari.json           (浏览器精度)
├─ 读取 ──> accuracy/firefox.json          (浏览器精度)
├─ 读取 ──> corpora/representative.json    (锚点测试)
├─ 读取 ──> corpora/chrome-sampled.json    (稀疏采样)
├─ 读取 ──> corpora/chrome-step10.json     (密集采样)
│
├─ 引用 ──> corpora/TAXONOMY.md            (分类法文档)
│
└─ 被引用 ──> status/dashboard.json        (主状态仪表盘)
```

### 数据依赖关系

```
┌─────────────────────────────────────────────────────────────────┐
│                     数据依赖层级                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Level 0: 原始测试数据                                           │
│  ├── accuracy/*.json         (浏览器精度测试)                     │
│  ├── corpora/representative.json  (锚点测试)                      │
│  ├── corpora/chrome-sampled.json  (稀疏采样)                      │
│  └── corpora/chrome-step10.json   (密集采样)                      │
│                              │                                  │
│                              ▼                                  │
│  Level 1: 语料仪表盘                                             │
│  └── corpora/dashboard.json  (本文件)                            │
│                              │                                  │
│                              ▼                                  │
│  Level 2: 主仪表盘                                               │
│  └── status/dashboard.json   (聚合所有状态)                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 运行时依赖

| 依赖 | 用途 |
|------|------|
| Bun | JavaScript 运行时 |
| Node.js fs | 文件读写 |
| 源 JSON 文件 | 数据输入（必须在生成前存在） |

---

## 风险、边界与改进建议

### 已知风险

#### 1. 硬编码元数据不同步

**问题**：`PRODUCT_SHAPED`、`LONG_FORM` 等数组在 `scripts/corpus-status.ts` 中硬编码，与 `sources.json` 可能不同步。

**风险场景**：
- 新增语料未添加到 `LONG_FORM` 数组
- 删除语料后数组中仍有残留

**当前状态**：需要人工维护，无自动校验。

#### 2. 单浏览器采样数据

**问题**：`chromeSampled` 和 `chromeStep10` 仅包含 Chrome 数据，无 Safari/Firefox 对应数据。

**影响**：无法直接对比多浏览器在密集采样下的表现。

#### 3. 时间戳漂移

**问题**：`generatedAt` 是生成时间，但依赖的源文件可能有不同的时间戳。

**示例**：
```json
{
  "generatedAt": "2026-03-31T10:07:49.046Z",
  "sources": {
    "chromeSampled": "corpora/chrome-sampled.json"  // 可能生成于不同时间
  }
}
```

### 边界条件

| 边界 | 说明 |
|------|------|
| 语料覆盖 | 仅包含硬编码的 17 个语料 |
| 浏览器覆盖 | 锚点测试支持 Chrome/Safari，采样仅 Chrome |
| 宽度锚点 | 固定 [300, 600, 800] |
| 数据新鲜度 | 依赖源文件的存在和正确性 |

### 改进建议

#### 1. 自动同步语料列表

```typescript
// 建议：从 sources.json 自动推导分类
async function deriveCategories(): Promise<{productShaped: string[], longForm: string[]}> {
  const sources = await loadJson<CorpusSource[]>('corpora/sources.json')
  
  return {
    productShaped: sources.filter(s => s.id === 'mixed-app-text').map(s => s.id),
    longForm: sources.filter(s => s.id !== 'mixed-app-text').map(s => s.id),
  }
}
```

#### 2. 源文件时间戳追踪

```typescript
// 建议：记录源文件的生成时间
const dashboard = {
  generatedAt: new Date().toISOString(),
  sources: {
    chromeSampled: {
      path: 'corpora/chrome-sampled.json',
      generatedAt: (await Bun.file('corpora/chrome-sampled.json').json()).generatedAt,
    },
    // ...
  },
}
```

#### 3. 多浏览器采样支持

```typescript
// 建议：增加 Safari 采样数据
interface CorpusDashboardEntry {
  // ...
  chromeSampled: SampledResult | null;
  safariSampled: SampledResult | null;  // 新增
  chromeStep10: Step10Result | null;
  safariStep10: Step10Result | null;    // 新增
}
```

#### 4. 增量更新支持

```bash
# 建议：仅更新变化的条目
bun run corpus-status --incremental --corpus-id=zh-zhufu
```

#### 5. 验证脚本

```typescript
// 建议：添加验证脚本
function validateDashboard(dashboard: Dashboard): string[] {
  const errors: string[] = []
  
  // 检查所有 sources.json 中的语料都有对应条目
  const sourceIds = new Set(sources.map(s => s.id))
  const dashboardIds = new Set([
    ...dashboard.productShaped.map(p => p.id),
    ...dashboard.longForm.map(l => l.id),
  ])
  
  for (const id of sourceIds) {
    if (!dashboardIds.has(id)) {
      errors.push(`Missing dashboard entry for corpus: ${id}`)
    }
  }
  
  return errors
}
```

#### 6. 可视化集成

```typescript
// 建议：生成 HTML 报告
bun run corpus-status --format=html --output=corpora/dashboard.html
```

### 维护 checklist

- [ ] 每次添加/删除语料后检查 `PRODUCT_SHAPED` 和 `LONG_FORM` 数组
- [ ] 每次更新 `chrome-sampled.json` 或 `chrome-step10.json` 后重新生成
- [ ] 验证 `browserRegressionGate` 中的计数与 `accuracy/*.json` 一致
- [ ] 检查 `notes` 字段是否仍准确描述当前状态
- [ ] 定期审查 `fineSweepNotes` 和 `fontMatrixNotes` 的时效性
