# representative.json 研究文档

## 场景与职责

`corpora/representative.json` 是 Pretext 项目的**代表性语料锚点快照文件**，用于记录在 Chrome 和 Safari 浏览器下对关键语料进行**锚点宽度测试**的详细结果。该文件是跨浏览器一致性验证的核心数据源，提供了最精细的测试环境指纹和逐行测量数据。

### 核心职责

1. **跨浏览器锚点验证**：在 300px、600px、800px 三个关键宽度下验证多浏览器一致性
2. **环境指纹记录**：详细记录测试时的浏览器版本、视口、屏幕等环境信息
3. **基准数据提供**：作为 `corpora/dashboard.json` 的 `chromeAnchors` 和 `safariAnchors` 数据源
4. **回归检测基准**：为长期稳定性追踪提供可对比的历史基准

### 锚点宽度选择 rationale

| 宽度 | 代表场景 | 测试目的 |
|------|----------|----------|
| 300px | 移动端窄屏 | 测试极端窄宽度的断行行为 |
| 600px | 平板/小桌面 | 测试中等宽度的典型布局 |
| 800px | 桌面端 | 测试较宽松宽度的布局表现 |

### 覆盖语料

10 个代表性语料（与 `corpus-representative.ts` 中的 `CORPUS_IDS` 一致）：

```typescript
const CORPUS_IDS = [
  'mixed-app-text',           // 产品形态：混合脚本
  'en-gatsby-opening',        // 英文长文本
  'ja-kumo-no-ito',           // 日文
  'ja-rashomon',              // 日文
  'zh-guxiang',               // 中文
  'zh-zhufu',                 // 中文
  'th-nithan-vetal-story-1',  // 泰文
  'my-cunning-heron-teacher', // 缅甸文
  'my-bad-deeds-return-to-you-teacher', // 缅甸文
  'ur-chughd',                // 乌尔都文（Nastaliq 字体）
] as const
```

---

## 功能点目的

### 1. 详细测量记录

```typescript
interface RepresentativeRow {
  corpusId: string;        // 语料标识
  title: string;           // 语料标题
  language: string;        // 语言代码
  direction: 'ltr' | 'rtl'; // 文本方向
  width: number;           // 容器宽度
  contentWidth: number;    // 内容宽度（width - padding*2）
  font: string;            // 实际使用的字体
  lineHeight: number;      // 行高
  predictedHeight: number; // Pretext 预测高度
  actualHeight: number;    // 浏览器实际高度
  diffPx: number;          // 高度差异
  predictedLineCount: number;  // 预测行数
  browserLineCount: number;    // 浏览器行数
}
```

### 2. 环境指纹记录

```typescript
interface EnvironmentFingerprint {
  userAgent: string;           // 完整 UA 字符串
  devicePixelRatio: number;    // 设备像素比
  viewport: {
    innerWidth: number;        // 视口宽度
    innerHeight: number;       // 视口高度
    outerWidth: number;        // 窗口宽度
    outerHeight: number;       // 窗口高度
    visualViewportScale: number | null;  // 缩放比例
  };
  screen: {
    width: number;             // 屏幕宽度
    height: number;            // 屏幕高度
    availWidth: number;        // 可用宽度
    availHeight: number;       // 可用高度
    colorDepth: number;        // 颜色深度
    pixelDepth: number;        // 像素深度
  };
}
```

### 3. 双浏览器快照

```typescript
interface CorpusRepresentativeSnapshot {
  generatedAt: string;       // ISO 8601 时间戳
  corpora: string[];         // 测试的语料列表
  widths: number[];          // [300, 600, 800]
  browsers: {
    chrome?: BrowserSnapshot;
    safari?: BrowserSnapshot;
  };
}
```

---

## 具体技术实现

### 生成流程

```
┌─────────────────────────────────────────────────────────────────┐
│               representative.json 生成流程                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 初始化                                                       │
│     └── bun run corpus-representative                           │
│                              │                                  │
│                              ▼                                  │
│  2. 启动测试服务器                                              │
│     └── ensurePageServer(port, '/corpus')                       │
│                              │                                  │
│                              ▼                                  │
│  3. 遍历浏览器 [chrome, safari]                                  │
│     │                                                           │
│     ├── 获取浏览器自动化锁                                       │
│     ├── 创建浏览器会话                                           │
│     │                                                           │
│     │  遍历 CORPUS_IDS (10 个语料)                               │
│     │  │                                                        │
│     │  ├── 生成 requestId                                       │
│     │  ├── 构建测试 URL                                         │
│     │  │   └── /corpus?id={id}&widths=300,600,800&report=1      │
│     │  ├── 启动报告服务器                                        │
│     │  ├── 浏览器导航到 URL                                      │
│     │  ├── pages/corpus.ts 执行测试                             │
│     │  │   ├── prepareWithSegments()                            │
│     │  │   ├── layout() vs DOM 对比                             │
│     │  │   └── 生成报告                                         │
│     │  ├── 接收报告                                             │
│     │  └── 转换为 RepresentativeRow                             │
│     │                                                           │
│     └── 收集环境指纹                                            │
│                              │                                  │
│                              ▼                                  │
│  4. 组装快照                                                    │
│     └── { generatedAt, corpora, widths, browsers }              │
│                              │                                  │
│                              ▼                                  │
│  5. 写入文件                                                    │
│     └── corpora/representative.json                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 关键代码实现

**1. 主脚本** (`scripts/corpus-representative.ts`)

```typescript
// 第 84-97 行：配置常量
const CORPUS_IDS = [
  'mixed-app-text', 'en-gatsby-opening', 'ja-kumo-no-ito',
  'ja-rashomon', 'zh-guxiang', 'zh-zhufu',
  'th-nithan-vetal-story-1', 'my-cunning-heron-teacher',
  'my-bad-deeds-return-to-you-teacher', 'ur-chughd',
] as const

const WIDTHS = [300, 600, 800] as const

// 第 177-231 行：主测试循环
for (const browser of browsers) {
  const lock = await acquireBrowserAutomationLock(browser)
  const session = createBrowserSession(browser)
  
  try {
    const rows: RepresentativeRow[] = []
    let environment: BrowserSnapshot['environment'] | null = null
    
    for (const corpusId of CORPUS_IDS) {
      const requestId = `${Date.now()}-${browser}-${corpusId}-${Math.random().toString(36).slice(2)}`
      const reportServer = await startPostedReportServer<CorpusSweepReport>(requestId)
      
      const url = `${baseUrl}?id=${encodeURIComponent(corpusId)}` +
                  `&widths=${encodeURIComponent(WIDTHS.join(','))}` +
                  `&report=1` +
                  `&diagnostic=light` +  // 轻量级诊断
                  `&requestId=${encodeURIComponent(requestId)}` +
                  `&reportEndpoint=${encodeURIComponent(reportServer.endpoint)}`
      
      const report = await loadPostedReport(session, url, ...)
      
      if (report.environment !== undefined) {
        environment = report.environment  // 记录环境指纹
      }
      
      for (const row of report.rows!) {
        rows.push(toRepresentativeRow(report, row))
      }
    }
    
    snapshot.browsers[browser] = { environment: environment!, rows }
  } finally {
    session.close()
    lock.release()
  }
}
```

**2. 行数据转换** (`scripts/corpus-representative.ts` 第 121-151 行)

```typescript
function toRepresentativeRow(
  report: CorpusSweepReport,
  row: CorpusSweepRow,
): RepresentativeRow {
  // 验证必要字段存在
  if (
    report.corpusId === undefined ||
    report.title === undefined ||
    report.language === undefined ||
    report.direction === undefined ||
    report.font === undefined ||
    report.lineHeight === undefined
  ) {
    throw new Error('Corpus report was missing representative snapshot fields')
  }
  
  return {
    corpusId: report.corpusId,
    title: report.title,
    language: report.language,
    direction: report.direction,
    width: row.width,
    contentWidth: row.contentWidth,
    font: report.font,
    lineHeight: report.lineHeight,
    predictedHeight: row.predictedHeight,
    actualHeight: row.actualHeight,
    diffPx: row.diffPx,
    predictedLineCount: row.predictedLineCount,
    browserLineCount: row.browserLineCount,
  }
}
```

**3. 浏览器端批量测试** (`pages/corpus.ts`)

```typescript
// 第 945-981 行：runSweep 函数处理多宽度测试
function runSweep(widths: number[]): void {
  const rows = widths.map(width => {
    const report = measureWidth(width, { publish: false, updateStats: false })
    return toSweepRow(report)
  })
  
  setReport(withRequestId({
    status: 'ready',
    environment: getEnvironmentFingerprint(),  // 记录环境
    // ... 其他字段
    rows,
  }))
}
```

### 数据示例

**Chrome 环境指纹**（节选）：

```json
{
  "userAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36...",
  "devicePixelRatio": 2,
  "viewport": {
    "innerWidth": 1512,
    "innerHeight": 762,
    "outerWidth": 1512,
    "outerHeight": 849,
    "visualViewportScale": 1
  },
  "screen": {
    "width": 1512,
    "height": 982,
    "availWidth": 1512,
    "availHeight": 849,
    "colorDepth": 30,
    "pixelDepth": 30
  }
}
```

**单条测量记录**（`mixed-app-text` @ 300px）：

```json
{
  "corpusId": "mixed-app-text",
  "title": "Mixed app text",
  "language": "mul",
  "direction": "ltr",
  "width": 300,
  "contentWidth": 220,
  "font": "18px \"Helvetica Neue\", Arial, \"Apple SD Gothic Neo\", ...",
  "lineHeight": 30,
  "predictedHeight": 1400,
  "actualHeight": 1400,
  "diffPx": 0,
  "predictedLineCount": 44,
  "browserLineCount": 44
}
```

---

## 关键代码路径与文件引用

### 生成路径

| 文件 | 职责 | 关键行号 |
|------|------|----------|
| `scripts/corpus-representative.ts` | 测试编排与结果收集 | L84-97 (配置), L177-231 (主循环), L121-151 (行转换) |
| `pages/corpus.ts` | 浏览器端测试执行 | L945-981 (runSweep), L288-307 (环境指纹) |
| `scripts/browser-automation.ts` | 浏览器自动化 | `createBrowserSession()`, `loadPostedReport()` |
| `scripts/report-server.ts` | 报告接收 | `startPostedReportServer()` |

### 消费路径

| 文件 | 职责 | 引用方式 |
|------|------|----------|
| `scripts/corpus-status.ts` | 生成仪表盘 | `loadJson<RepresentativeSnapshot>('corpora/representative.json')` (L276) |
| `corpora/dashboard.json` | 状态聚合 | 提取为 `chromeAnchors` 和 `safariAnchors` |
| `DEVELOPMENT.md` | 开发文档 | 作为 "checked-in representative corpus anchor rows" 引用 |

### 命令行调用

```bash
# 生成 representative.json（Chrome + Safari）
bun run corpus-representative

# 仅生成特定浏览器
bun run corpus-representative --browser=chrome
bun run corpus-representative --browser=safari
bun run corpus-representative --browser=all

# 指定输出路径
bun run corpus-representative --output=corpora/representative.json

# 指定端口
bun run corpus-representative --port=3210
```

---

## 依赖与外部交互

### 文件依赖图

```
corpora/representative.json
│
├─ 读取 ──> corpora/sources.json          (语料元数据)
│            └── 通过 pages/corpus.ts 间接使用
│
├─ 依赖 ──> pages/corpus.ts               (浏览器测试页面)
│            ├─> src/layout.ts            (Pretext 引擎)
│            └─> pages/diagnostic-utils.ts (诊断工具)
│
├─ 依赖 ──> scripts/browser-automation.ts (浏览器自动化)
│            ├─> Chrome CDP
│            └─> AppleScript (Safari)
│
├─ 依赖 ──> scripts/report-server.ts      (结果接收服务)
│
└─ 消费 ──> scripts/corpus-status.ts      (生成 dashboard.json)
             └─> corpora/dashboard.json
```

### 浏览器自动化详情

**Chrome 控制**（`scripts/browser-automation.ts`）：
- 使用 Chrome DevTools Protocol (CDP)
- 通过 `--remote-debugging-port=9222` 连接
- 自动获取自动化锁防止冲突

**Safari 控制**（`scripts/browser-automation.ts`）：
- 使用 AppleScript 控制 Safari
- 通过 `osascript` 执行 JavaScript
- 需要 macOS 环境

### 数据流

```
┌─────────────────────────────────────────────────────────────────┐
│                    representative.json 数据流                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  输入                                                            │
│  ├── CORPUS_IDS (硬编码 10 个语料)                               │
│  ├── WIDTHS (硬编码 [300, 600, 800])                             │
│  └── sources.json (字体/行高配置)                                │
│                              │                                  │
│                              ▼                                  │
│  处理                                                            │
│  ├── Chrome 会话 ──> 10 语料 × 3 宽度 = 30 条记录                │
│  ├── Safari 会话 ──> 10 语料 × 3 宽度 = 30 条记录                │
│  └── 环境指纹采集                                                │
│                              │                                  │
│                              ▼                                  │
│  输出                                                            │
│  └── representative.json                                        │
│      ├── generatedAt                                            │
│      ├── corpora: [...]                                         │
│      ├── widths: [300, 600, 800]                                │
│      └── browsers: { chrome: {...}, safari: {...} }             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 风险、边界与改进建议

### 已知风险

#### 1. 硬编码语料列表

**问题**：`CORPUS_IDS` 在 `scripts/corpus-representative.ts` 中硬编码，与 `sources.json` 可能不同步。

**当前列表**（10 个）：
```typescript
['mixed-app-text', 'en-gatsby-opening', 'ja-kumo-no-ito', 'ja-rashomon',
 'zh-guxiang', 'zh-zhufu', 'th-nithan-vetal-story-1', 'my-cunning-heron-teacher',
 'my-bad-deeds-return-to-you-teacher', 'ur-chughd']
```

**缺失语料**（7 个未包含）：
- `ko-unsu-joh-eun-nal`（韩文）
- `th-nithan-vetal-story-7`（泰文）
- `km-prachum-reuang-preng-khmer-volume-7-stories-1-10`（高棉文）
- `ar-risalat-al-ghufran-part-1`（阿拉伯文）
- `ar-al-bukhala`（阿拉伯文）
- `hi-eidgah`（印地文）
- `he-masaot-binyamin-metudela`（希伯来文）

**原因**：这些语料在 `corpus-representative.ts` 开发时被认为优先级较低或数据较干净。

#### 2. Safari 自动化可靠性

**问题**：Safari 的 AppleScript 自动化比 Chrome CDP 更脆弱。

**已知问题**：
- 需要 macOS 环境
- 可能受系统弹窗干扰
- 速度较慢

#### 3. 环境指纹差异

**问题**：Chrome 和 Safari 的测试环境可能不同（即使同一台机器）。

**示例**：
```json
// Chrome
{ "devicePixelRatio": 2, "viewport": { "innerHeight": 762, ... } }

// Safari
{ "devicePixelRatio": 2, "viewport": { "innerHeight": 759, ... } }  // 差异 3px
```

### 边界条件

| 边界 | 当前值 | 说明 |
|------|--------|------|
| 语料数量 | 10 | 硬编码，非全部 17 个 |
| 锚点宽度 | [300, 600, 800] | 固定，不可配置 |
| 浏览器 | Chrome + Safari | 无 Firefox |
| 诊断级别 | light | 非 full，减少开销 |
| 超时 | 180s | 通过环境变量配置 |

### 改进建议

#### 1. 动态语料列表

```typescript
// 建议：从 sources.json 加载，但允许过滤
async function getRepresentativeCorpora(): Promise<string[]> {
  const sources = await loadJson<CorpusSource[]>('corpora/sources.json')
  
  // 优先选择有历史问题的语料
  const priorityCorpora = [
    'mixed-app-text',      // 产品形态
    'ur-chughd',           // Nastaliq 字体
    'zh-zhufu',            // 中文
    'ja-rashomon',         // 日文
  ]
  
  // 补充其他语料至最多 12 个
  const others = sources
    .map(s => s.id)
    .filter(id => !priorityCorpora.includes(id))
    .slice(0, 12 - priorityCorpora.length)
  
  return [...priorityCorpora, ...others]
}
```

#### 2. 可配置锚点宽度

```bash
# 建议：命令行参数支持自定义宽度
bun run corpus-representative --widths=320,768,1024
```

#### 3. Firefox 支持

```typescript
// 建议：增加 Firefox 支持
const BROWSERS: BrowserKind[] = ['chrome', 'safari', 'firefox']

// 使用 geckodriver 或 Playwright 控制 Firefox
```

#### 4. 增量更新

```bash
# 建议：仅更新特定语料
bun run corpus-representative --update-only=ur-chughd,zh-zhufu
```

#### 5. 环境一致性检查

```typescript
// 建议：验证 Chrome 和 Safari 在相同条件下测试
function validateEnvironmentConsistency(
  chromeEnv: EnvironmentFingerprint,
  safariEnv: EnvironmentFingerprint,
): string[] {
  const issues: string[] = []
  
  if (Math.abs(chromeEnv.devicePixelRatio - safariEnv.devicePixelRatio) > 0.01) {
    issues.push('DPR mismatch')
  }
  
  if (chromeEnv.screen.width !== safariEnv.screen.width) {
    issues.push('Screen resolution mismatch')
  }
  
  return issues
}
```

#### 6. 数据压缩

```typescript
// 建议：使用更紧凑的存储格式
interface CompactRepresentativeRow {
  c: string;   // corpusId
  w: number;   // width
  d: number;   // diffPx
  p: number;   // predictedHeight
  a: number;   // actualHeight
  pl: number;  // predictedLineCount
  bl: number;  // browserLineCount
}
```

### 维护 checklist

- [ ] 每次添加新语料时评估是否加入 `CORPUS_IDS`
- [ ] 每次升级 Chrome/Safari 主版本后重新生成
- [ ] 验证 `corpora/dashboard.json` 中的锚点数据与 representative.json 一致
- [ ] 检查环境指纹是否有异常变化
- [ ] 对比 Chrome 和 Safari 的相同语料差异，识别浏览器特定问题
- [ ] 定期审查 `notes` 字段的时效性
