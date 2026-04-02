# accuracy/firefox.json 研究文档

## 概述

`accuracy/firefox.json` 是 Pretext 项目中用于存储 Firefox 浏览器精度测试结果的核心数据文件。该文件作为浏览器自动化测试的检入快照（checked-in snapshot），记录了在 Firefox 浏览器环境下，Pretext 文本布局库与浏览器原生 DOM 布局行为的一致性验证数据。

---

## 一、场景与职责

### 1.1 核心场景

| 场景 | 描述 |
|------|------|
| **浏览器兼容性验证** | 验证 Pretext 库在 Firefox 浏览器中的布局预测精度 |
| **回归测试基准** | 作为持续集成中的回归测试基准数据 |
| **跨浏览器对比** | 与 `chrome.json`、`safari.json` 形成三浏览器对比矩阵 |
| **算法调优参考** | 为文本布局算法的 Firefox 特定调优提供数据支持 |

### 1.2 文件职责

- **精度记录**：记录 7680 组测试用例的预测值与实际 DOM 测量值
- **环境指纹**：捕获测试执行时的浏览器环境信息（User-Agent、DPR、视口等）
- **差异追踪**：记录 `actual`（浏览器实际高度）与 `predicted`（库预测高度）的差异
- **状态指示**：通过 `status` 字段指示测试结果状态（`ready` 或 `error`）

### 1.3 测试矩阵规模

根据 `src/test-data.ts` 定义，测试覆盖：

- **4 种字体**：`"Helvetica Neue"`、`Georgia`、`Verdana`、`"Courier New"`
- **8 种字号**：12, 14, 15, 16, 18, 20, 24, 28 px
- **8 种容器宽度**：150, 200, 250, 300, 350, 400, 500, 600 px
- **30 种文本用例**：涵盖拉丁文、阿拉伯文、希伯来文、CJK、泰文、表情符号、混合方向等

总测试组合数：4 × 8 × 8 × 30 = **7,680** 组

---

## 二、功能点目的

### 2.1 数据结构目的

```typescript
// 根级结构
{
  "status": "ready",           // 测试执行状态
  "environment": { ... },      // 浏览器环境指纹
  "total": 7680,               // 总测试数
  "matchCount": 7680,          // 匹配数
  "mismatchCount": 0,          // 不匹配数
  "mismatches": [],            // 详细不匹配记录
  "rows": [ ... ]              // 完整测试行数据
}
```

### 2.2 行级数据结构

```typescript
{
  "label": "Latin update",     // 文本用例标签（来自 TEXTS）
  "font": "\"Helvetica Neue\", Helvetica, Arial, sans-serif",
  "fontSize": 12,              // 字体大小（px）
  "lineHeight": 14,            // 行高（px，计算为 fontSize * 1.2 后取整）
  "width": 150,                // 容器宽度（px）
  "actual": 70,                // 浏览器 DOM 实际测量高度（px）
  "predicted": 70,             // Pretext 库预测高度（px）
  "diff": 0                    // 差异值（predicted - actual）
}
```

### 2.3 环境指纹目的

```typescript
{
  "userAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:148.0) Gecko/20100101 Firefox/148.0",
  "devicePixelRatio": 1,       // 设备像素比（Firefox 在标准分辨率下为 1）
  "viewport": { ... },         // 视口尺寸
  "screen": { ... }            // 屏幕信息
}
```

环境指纹用于：
- **可复现性**：确保测试结果与特定浏览器版本绑定
- **问题诊断**：当测试失败时，快速定位浏览器环境差异
- **跨平台对比**：识别不同 DPR 或视口设置下的行为差异

---

## 三、具体技术实现

### 3.1 数据生成流程

```
┌─────────────────┐
│  accuracy-check │  bun run accuracy-check:firefox
│    (script)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ browser-automation│  Firefox BiDi 协议控制
│    (script)     │  --headless --remote-debugging-port
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  /accuracy page │  pages/accuracy.ts
│   (browser)     │  执行 runSweep()
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   DOM 测量      │  getBoundingClientRect()
│  (actual 值)    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Pretext 布局   │  prepare() + layout()
│ (predicted 值)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  差异计算       │  Math.abs(actual - predicted) >= 1
│  (mismatch 判定)│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ firefox.json    │  写入文件（--full 模式）
│   (snapshot)    │
└─────────────────┘
```

### 3.2 关键代码路径

#### 3.2.1 测试执行入口

**文件**：`scripts/accuracy-check.ts`

```typescript
// 第 70 行：浏览器选择
const browser = (process.env['ACCURACY_CHECK_BROWSER'] ?? 'chrome').toLowerCase() as BrowserKind

// 第 193-204 行：Firefox 特殊处理（代理服务器）
if (browser === 'firefox') {
  const bunPort = await getAvailablePort()
  const bunBaseUrl = `http://localhost:${bunPort}/accuracy`
  serverProcess = spawn('/bin/zsh', ['-lc', `bun --port=${bunPort} --no-hmr pages/*.html`], ...)
  await waitForServer(bunBaseUrl)
  const proxy = await startProxyServer(`http://[::1]:${bunPort}`)
  baseUrl = proxy.baseUrl
}
```

Firefox 需要代理服务器的原因是：Firefox 的 IPv6 回环地址解析与 Chrome/Safari 存在差异，通过代理确保稳定的 `127.0.0.1` 访问。

#### 3.2.2 浏览器自动化

**文件**：`scripts/browser-automation.ts`

```typescript
// 第 357-411 行：Firefox BiDi 会话初始化
async function initializeFirefoxSession(): Promise<FirefoxSessionState> {
  const bidiPort = await getAvailablePort()
  const profileDir = mkdtempSync(join(tmpdir(), 'pretext-firefox-'))
  const firefoxProcess = spawn('/Applications/Firefox.app/Contents/MacOS/firefox', [
    '--headless',
    '--new-instance',
    '--profile', profileDir,
    '--remote-debugging-port', String(bidiPort),
    'about:blank',
  ])
  // ... WebSocket 连接 BiDi 协议
}
```

#### 3.2.3 页面端测试逻辑

**文件**：`pages/accuracy.ts`

```typescript
// 第 192-277 行：核心测试循环 runSweep()
function runSweep(): { total: number, mismatches: Mismatch[], rows: AccuracyRow[] } {
  for (const fontFamily of FONTS) {
    for (const fontSize of SIZES) {
      const font = `${fontSize}px ${fontFamily}`
      const lineHeight = Math.round(fontSize * 1.2)
      clearCache()  // 每种字号清除缓存
      
      for (const maxWidth of WIDTHS) {
        // 创建 DOM 元素并测量
        const div = document.createElement('div')
        div.style.font = font
        div.style.width = `${maxWidth}px`
        div.style.wordWrap = 'break-word'
        div.style.overflowWrap = 'break-word'
        
        for (const { label, text } of TEXTS) {
          const actual = div.getBoundingClientRect().height
          const prepared = prepareWithSegments(text, font)
          const predicted = layout(prepared, maxWidth, lineHeight).height
          
          // 差异判定阈值：1px
          if (Math.abs(actual - predicted) >= 1) {
            mismatches.push({ label, font, fontSize, lineHeight, width: maxWidth, actual, predicted, diff: predicted - actual, text })
          }
        }
      }
    }
  }
}
```

#### 3.2.4 布局引擎核心

**文件**：`src/layout.ts`

```typescript
// 第 472-474 行：prepare() - 文本分析与测量
export function prepare(text: string, font: string, options?: PrepareOptions): PreparedText {
  return prepareInternal(text, font, false, options) as PreparedText
}

// 第 495-501 行：layout() - 纯算术布局
export function layout(prepared: PreparedText, maxWidth: number, lineHeight: number): LayoutResult {
  const lineCount = countPreparedLines(getInternalPrepared(prepared), maxWidth)
  return { lineCount, height: lineCount * lineHeight }
}
```

#### 3.2.5 行断行算法

**文件**：`src/line-break.ts`

```typescript
// 第 166-171 行：行数计算入口
export function countPreparedLines(prepared: PreparedLineBreakData, maxWidth: number): number {
  if (prepared.simpleLineWalkFastPath) {
    return countPreparedLinesSimple(prepared, maxWidth)
  }
  return walkPreparedLines(prepared, maxWidth)
}

// 第 186 行：行拟合容差（浏览器特定）
const lineFitEpsilon = engineProfile.lineFitEpsilon
// Firefox 使用默认值 0.005（Chromium/Gecko 标准）
```

### 3.3 数据结构详解

#### 3.3.1 测试用例定义

**文件**：`src/test-data.ts`

```typescript
export const TEXTS = [
  { label: 'Latin update', text: "Just tried the new update..." },
  { label: 'Arabic', text: "هذا النص باللغة العربية..." },
  { label: 'Chinese', text: "这是一段中文文本..." },
  { label: 'Emoji mixed', text: "The quick 🦊 jumped..." },
  // ... 共 30 组
] as const

export const SIZES = [12, 14, 15, 16, 18, 20, 24, 28] as const
export const WIDTHS = [150, 200, 250, 300, 350, 400, 500, 600] as const
```

#### 3.3.2 不匹配阈值

```typescript
// pages/accuracy.ts 第 239 行
if (Math.abs(actual - predicted) >= 1) {
  // 记录为 mismatch
}
```

阈值设定为 **1px**，原因：
- 浏览器 DOM 测量存在亚像素精度（sub-pixel）
- 不同浏览器的舍入策略可能产生 0.5px 级差异
- 1px 是视觉上可接受的最大容差

---

## 四、关键代码路径与文件引用

### 4.1 调用链图谱

```
firefox.json
    ▲
    │ 写入（--full 模式）
    │
scripts/accuracy-check.ts
    │
    ├─► scripts/browser-automation.ts ──► Firefox BiDi 协议
    │                                         │
    │                                         ▼
    │                                    Firefox 浏览器
    │                                         │
    │                                    导航到 /accuracy
    │                                         │
    └─────────────────────────────────────────┘
                                              │
                                              ▼
                                         pages/accuracy.ts
                                              │
                                              ├─► src/layout.ts
                                              │       ├─► src/analysis.ts（文本分析）
                                              │       ├─► src/measurement.ts（canvas 测量）
                                              │       └─► src/line-break.ts（断行算法）
                                              │
                                              └─► DOM API（getBoundingClientRect）
```

### 4.2 核心文件清单

| 文件路径 | 职责 | 关键函数/类 |
|---------|------|-----------|
| `accuracy/firefox.json` | 数据存储 | JSON Schema: AccuracyReport |
| `scripts/accuracy-check.ts` | 测试编排 | `loadBrowserReport()`, `printReport()` |
| `scripts/browser-automation.ts` | 浏览器控制 | `createFirefoxSession()`, `loadHashReport()` |
| `pages/accuracy.ts` | 页面端测试 | `runSweep()`, `getBrowserLines()` |
| `src/layout.ts` | 布局引擎 | `prepare()`, `layout()`, `layoutWithLines()` |
| `src/line-break.ts` | 断行算法 | `countPreparedLines()`, `walkPreparedLines()` |
| `src/analysis.ts` | 文本分析 | `analyzeText()`, 分词与归一化 |
| `src/measurement.ts` | 文本测量 | `getSegmentMetrics()`, emoji 校正 |
| `src/test-data.ts` | 测试数据 | `TEXTS`, `SIZES`, `WIDTHS` |
| `pages/diagnostic-utils.ts` | 诊断工具 | `getDiagnosticUnits()` |

### 4.3 状态仪表盘集成

**文件**：`scripts/status-dashboard.ts`

```typescript
// 第 70-72 行：加载三浏览器精度数据
const chromeAccuracy = await loadJson<AccuracyReport>('accuracy/chrome.json')
const safariAccuracy = await loadJson<AccuracyReport>('accuracy/safari.json')
const firefoxAccuracy = await loadJson<AccuracyReport>('accuracy/firefox.json')

// 第 90-94 行：生成仪表盘摘要
browserAccuracy: {
  chrome: summarizeAccuracy(chromeAccuracy),
  safari: summarizeAccuracy(safariAccuracy),
  firefox: summarizeAccuracy(firefoxAccuracy),
}
```

输出至 `status/dashboard.json`，供项目状态监控使用。

---

## 五、依赖与外部交互

### 5.1 外部依赖

| 依赖类型 | 具体依赖 | 用途 |
|---------|---------|------|
| **浏览器** | Firefox 148+ | 测试目标浏览器 |
| **协议** | WebDriver BiDi | Firefox 自动化控制 |
| **运行时** | Bun | 脚本执行与页面服务 |
| **系统** | macOS | AppleScript 控制 Safari/Chrome（Firefox 使用 BiDi） |
| **网络** | localhost 代理 | Firefox 回环地址兼容 |

### 5.2 内部依赖

```
firefox.json
├── 被依赖：scripts/status-dashboard.ts（仪表盘生成）
├── 被依赖：DEVELOPMENT.md（文档引用）
├── 被依赖：AGENTS.md（代理指南引用）
└── 生成依赖：scripts/accuracy-check.ts（写入）
    └── 依赖：scripts/browser-automation.ts
        └── 依赖：pages/accuracy.ts
            └── 依赖：src/layout.ts
                └── 依赖：src/line-break.ts, src/analysis.ts, src/measurement.ts
```

### 5.3 环境变量

| 变量 | 说明 | 默认值 |
|-----|------|-------|
| `ACCURACY_CHECK_BROWSER` | 目标浏览器 | `chrome` |
| `ACCURACY_CHECK_TIMEOUT_MS` | 测试超时 | `240000`（Firefox 更长） |
| `ACCURACY_CHECK_PORT` | 页面服务端口 | 自动分配 |

### 5.4 包脚本命令

**文件**：`package.json`

```json
{
  "scripts": {
    "accuracy-check:firefox": "ACCURACY_CHECK_BROWSER=firefox bun run scripts/accuracy-check.ts",
    "accuracy-snapshot:firefox": "ACCURACY_CHECK_BROWSER=firefox bun run scripts/accuracy-check.ts --full --output=accuracy/firefox.json"
  }
}
```

---

## 六、风险、边界与改进建议

### 6.1 当前风险

| 风险点 | 严重程度 | 说明 |
|-------|---------|------|
| **Firefox 版本绑定** | 中 | 当前数据基于 Firefox 148.0，新版本可能产生差异 |
| **macOS 依赖** | 中 | Firefox 自动化假设 `/Applications/Firefox.app` 路径 |
| **DPR 假设** | 低 | 测试在 DPR=1 环境执行，高 DPR 未覆盖 |
| **字体回退** | 低 | `"Helvetica Neue"` 在 Linux/Windows 可能回退 |

### 6.2 边界条件

#### 6.2.1 行拟合容差

```typescript
// src/measurement.ts 中定义的引擎配置
const engineProfile = {
  lineFitEpsilon: 0.005,  // Chromium/Gecko（包括 Firefox）
  // lineFitEpsilon: 1/64,  // Safari/WebKit
}
```

Firefox 使用 **0.005** 的容差，与 Chrome 一致，区别于 Safari 的 `1/64`。

#### 6.2.2 空不匹配记录

当前 `firefox.json` 中 `mismatches: []` 表示 **100% 精度**（7680/7680）。这是健康状态，但需注意：

- 任何算法变更都可能导致不匹配出现
- 新字体或新测试用例可能暴露边界问题

### 6.3 改进建议

#### 6.3.1 短期改进

1. **增加 DPR 覆盖**
   ```typescript
   // 建议在多 DPR 环境执行测试
   const DPR_VARIANTS = [1, 2]  // 标准 + Retina
   ```

2. **增加 Linux/Windows 环境**
   - 当前仅 macOS 数据，跨平台差异未捕获

3. **细化不匹配诊断**
   ```typescript
   // 当前 mismatch 结构可增加：
   {
     "diagnosticLines": ["L1 ours=\"...\" browser=\"...\""],
     "canvasWidths": [...],  // 各段 canvas 测量值
     "domRects": [...]       // DOM 逐行 Rect 数据
   }
   ```

#### 6.3.2 中期改进

1. **增量测试模式**
   ```bash
   # 仅测试变更的用例
   bun run accuracy-check:firefox --incremental --since=HEAD~1
   ```

2. **趋势分析**
   ```typescript
   // 对比历史快照，识别精度退化
   const trend = compareSnapshots('firefox-148.json', 'firefox-149.json')
   ```

3. **自动化刷新机制**
   ```yaml
   # .github/workflows/accuracy.yml
   - name: Refresh Firefox Snapshot
     if: github.event_name == 'schedule'
     run: bun run accuracy-snapshot:firefox
   ```

#### 6.3.3 长期改进

1. **WASM 化测量引擎**
   - 将 canvas 测量逻辑编译为 WASM，确保跨浏览器一致性

2. **机器学习辅助**
   - 利用历史不匹配数据训练模型，预测潜在问题组合

3. **实时精度监控**
   - 在演示页面集成精度检查器，用户可报告环境特定问题

### 6.4 维护检查清单

- [ ] 每月执行 `bun run accuracy-snapshot:firefox` 验证数据新鲜度
- [ ] 每次 `src/layout.ts`、`src/line-break.ts` 变更后刷新快照
- [ ] 新 Firefox 版本发布后对比差异
- [ ] 保持与 `chrome.json`、`safari.json` 的同步更新节奏

---

## 附录：JSON Schema 参考

```typescript
interface AccuracyReport {
  status: 'ready' | 'error'
  environment?: {
    userAgent: string
    devicePixelRatio: number
    viewport: { innerWidth, innerHeight, outerWidth, outerHeight, visualViewportScale }
    screen: { width, height, availWidth, availHeight, colorDepth, pixelDepth }
  }
  total?: number
  matchCount?: number
  mismatchCount?: number
  mismatches?: Array<{
    label: string
    font: string
    fontSize: number
    lineHeight: number
    width: number
    actual: number
    predicted: number
    diff: number
    text: string
    diagnosticLines?: string[]
  }>
  rows?: Array<{
    label: string
    font: string
    fontSize: number
    lineHeight: number
    width: number
    actual: number
    predicted: number
    diff: number
  }>
}
```

---

*文档生成时间：2026-04-02*
*基于文件版本：firefox.json (7680 行测试数据，100% 精度)*
