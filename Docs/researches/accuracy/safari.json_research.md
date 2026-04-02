# accuracy/safari.json 研究文档

## 概述

`safari.json` 是 Pretext 项目在 Safari 浏览器下的**准确性测试快照文件**，记录了文本布局引擎在 Safari 环境下的预测结果与浏览器实际渲染结果的对比数据。该文件与 `accuracy/chrome.json` 和 `accuracy/firefox.json` 共同构成跨浏览器准确性验证体系。

---

## 1. 场景与职责

### 1.1 核心职责

`safari.json` 承担以下关键职责：

1. **浏览器行为基线**: 作为 Safari 浏览器文本渲染行为的参考基准，用于验证 Pretext 布局引擎在 WebKit 环境下的准确性
2. **回归检测**: 通过对比历史快照，检测代码变更是否引入 Safari 特定的布局回归
3. **跨浏览器一致性验证**: 与 Chrome (Blink) 和 Firefox (Gecko) 快照对比，识别浏览器间差异
4. **CI/CD 质量门禁**: 作为自动化测试的预期结果，确保发布版本在 Safari 下的准确性

### 1.2 使用场景

| 场景 | 描述 | 触发命令 |
|------|------|----------|
| 本地开发验证 | 开发者验证 Safari 准确性 | `bun run accuracy-check:safari` |
| 快照更新 | 更新基准数据 | `bun run accuracy-snapshot:safari` |
| 回归测试 | 检测代码变更影响 | `bun run accuracy-check:safari` |
| 发布前检查 | 确保 Safari 兼容性 | 集成到发布流程 |

### 1.3 环境特征

根据文件内容，测试执行环境特征：

```json
{
  "userAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/26.3.1 Safari/605.1.15",
  "devicePixelRatio": 2,
  "viewport": {
    "innerWidth": 1512,
    "innerHeight": 769,
    "outerWidth": 1512,
    "outerHeight": 859,
    "visualViewportScale": 1
  },
  "screen": {
    "width": 1512,
    "height": 982,
    "availWidth": 1512,
    "availHeight": 859,
    "colorDepth": 24,
    "pixelDepth": 24
  }
}
```

- **平台**: macOS (Intel)
- **DPR**: 2 (Retina 显示屏)
- **视口**: 1512×769 逻辑像素
- **颜色深度**: 24-bit

---

## 2. 功能点目的

### 2.1 数据结构定义

`safari.json` 采用分层 JSON 结构：

```typescript
interface AccuracyReport {
  status: 'ready' | 'error';
  environment: EnvironmentFingerprint;
  total: number;           // 总测试用例数 (7680)
  matchCount: number;      // 匹配数
  mismatchCount: number;   // 不匹配数
  mismatches: Mismatch[];  // 详细不匹配记录
  rows: AccuracyRow[];     // 完整测试结果矩阵
}

interface AccuracyRow {
  label: string;           // 测试用例标签
  font: string;            // 字体配置
  fontSize: number;        // 字号
  lineHeight: number;      // 行高
  width: number;           // 容器宽度
  actual: number;          // 浏览器实际高度 (px)
  predicted: number;       // Pretext 预测高度 (px)
  diff: number;            // 差值 (predicted - actual)
}
```

### 2.2 测试矩阵维度

测试覆盖以下维度组合：

| 维度 | 取值 | 数量 |
|------|------|------|
| 字体 | Helvetica Neue, Georgia, Verdana, Courier New | 4 |
| 字号 | 12, 14, 15, 16, 18, 20, 24, 28 (px) | 8 |
| 容器宽度 | 150, 200, 250, 300, 350, 400, 500, 600 (px) | 8 |
| 测试文本 | 30 种语言/场景文本 | 30 |

**总计**: 4 × 8 × 8 × 30 = **7,680 条测试记录**

### 2.3 测试文本覆盖

```typescript
// src/test-data.ts 中定义的测试文本
const TEXTS = [
  // 拉丁语系
  { label: 'Latin update', text: "Just tried the new update..." },
  { label: 'Latin compatibility', text: "Does anyone know if this works..." },
  { label: 'Latin short', text: "This is exactly what I was looking for..." },
  { label: 'Latin caching', text: "The key insight is that you can cache..." },
  { label: 'Latin punctuation', text: "Performance is critical for this kind..." },
  { label: 'Latin hyphenation', text: "One thing I noticed is that the line..." },
  
  // 阿拉伯语 (RTL)
  { label: 'Arabic', text: "هذا النص باللغة العربية..." },
  { label: 'Arabic short', text: "مرحبا بالعالم، هذه تجربة..." },
  
  // 希伯来语 (RTL)
  { label: 'Hebrew', text: "זהו טקסט בעברית כדי לבדוק..." },
  { label: 'Hebrew short', text: "שלום עולם, זוהי בדיקה..." },
  
  // 混合 LTR + RTL
  { label: 'Mixed en+ar', text: "The meeting is scheduled for يوم الثلاثاء..." },
  { label: 'Mixed report', text: "According to the report by محمد الأحمد..." },
  { label: 'Mixed en+he', text: "The project name is פרויקט חדש..." },
  { label: 'Mixed version', text: "Version 3.2.1 של התוכנה was released..." },
  
  // CJK
  { label: 'Chinese', text: "这是一段中文文本..." },
  { label: 'Chinese short', text: "性能测试显示..." },
  { label: 'Japanese', text: "これはテキストレイアウトライブラリのテストです..." },
  { label: 'Japanese short', text: "パフォーマンスは非常に重要です..." },
  { label: 'Korean', text: "이것은 텍스트 레이아웃 라이브러리의 테스트입니다..." },
  
  // 泰语
  { label: 'Thai', text: "นี่คือข้อความทดสอบสำหรับไลบรารีจัดวางข้อความ..." },
  
  // Emoji
  { label: 'Emoji mixed', text: "The quick 🦊 jumped over the lazy 🐕..." },
  { label: 'Emoji dense', text: "Great work! 👏👏👏 This is exactly what we needed 🎯..." },
  
  // 多脚本混合
  { label: 'Multi-script', text: "Hello مرحبا שלום 你好 こんにちは 안녕하세요 สวัสดี..." },
  { label: 'Numbers+RTL', text: "The price is $42.99 (approximately ٤٢٫٩٩ ريال..." },
  
  // 边界情况
  { label: 'Empty', text: "" },
  { label: 'Single char', text: "A" },
  { label: 'Whitespace', text: "   " },
  { label: 'Newlines', text: "Hello\nWorld\nMultiple\nLines" },
  { label: 'Long word', text: "Superlongwordwithoutanyspaces..." },
  { label: 'Long mixed', text: "In the heart of القاهرة القديمة..." },
];
```

---

## 3. 具体技术实现

### 3.1 生成流程

```
┌─────────────────────────────────────────────────────────────────┐
│                     accuracy-snapshot:safari                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  scripts/accuracy-check.ts (ACCURACY_CHECK_BROWSER=safari)      │
│  - 设置超时: 240s (Safari 比 Chrome 慢)                          │
│  - 获取浏览器自动化锁                                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  scripts/browser-automation.ts                                   │
│  - createSafariSession(): 通过 AppleScript 控制 Safari          │
│  - acquireBrowserAutomationLock(): 单浏览器串行执行              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  pages/accuracy.ts (在 Safari 中执行)                            │
│  - runSweep(): 执行完整测试矩阵                                  │
│  - 对比 DOM 实际高度 vs layout() 预测高度                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  生成 accuracy/safari.json                                      │
│  - 包含完整 rows 数组 (7680 条记录)                              │
│  - 空 mismatches 数组表示 100% 匹配                              │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 关键代码路径

#### 3.2.1 准确性检查脚本

**文件**: `scripts/accuracy-check.ts`

```typescript
// Safari 特定配置
const browser = (process.env['ACCURACY_CHECK_BROWSER'] ?? 'chrome').toLowerCase() as BrowserKind
const reportTimeoutMs = Number.parseInt(
  process.env['ACCURACY_CHECK_TIMEOUT_MS'] ?? (browser === 'safari' ? '240000' : '120000'),
  10,
)

// Safari 会话创建
const session = createBrowserSession(browser)
```

#### 3.2.2 Safari 自动化实现

**文件**: `scripts/browser-automation.ts` (第 413-476 行)

```typescript
function createSafariSession(options: BrowserSessionOptions): BrowserSession {
  // 使用 AppleScript 控制 Safari
  const scriptLines = ['tell application "Safari"']
  
  if (options.foreground === true) {
    scriptLines.unshift('tell application "Safari" to activate')
  }
  
  // 创建新文档
  scriptLines.push('set targetDocument to make new document with properties {URL:"about:blank"}')
  // ... 窗口管理
  
  return {
    navigate(url) { /* AppleScript 导航 */ },
    readLocationUrl() { /* 读取当前 URL */ },
    close() { /* 关闭窗口 */ },
  }
}
```

#### 3.2.3 准确性测试页面

**文件**: `pages/accuracy.ts`

核心测试逻辑：

```typescript
function runSweep(): { total: number, mismatches: Mismatch[], rows: AccuracyRow[] } {
  for (const fontFamily of FONTS) {
    for (const fontSize of SIZES) {
      const font = `${fontSize}px ${fontFamily}`
      const lineHeight = Math.round(fontSize * 1.2)
      clearCache()
      
      for (const maxWidth of WIDTHS) {
        for (const { text } of TEXTS) {
          // 1. 创建 DOM 元素获取实际高度
          const div = document.createElement('div')
          div.style.font = font
          div.style.lineHeight = `${lineHeight}px`
          div.style.width = `${maxWidth}px`
          div.style.wordWrap = 'break-word'
          div.style.overflowWrap = 'break-word'
          div.textContent = text
          container.appendChild(div)
          
          // 2. Pretext 预测
          const prepared = prepareWithSegments(text, font)
          const actual = divs[i]!.getBoundingClientRect().height
          const predicted = layout(prepared[i]!, maxWidth, lineHeight).height
          
          // 3. 对比 (允许 1px 误差)
          if (Math.abs(actual - predicted) >= 1) {
            // 记录 mismatch，包含行级诊断
            const browserLines = getBrowserLines(prepared[i]!, divs[i]!)
            const ourLayout = layoutWithLines(prepared[i]!, maxWidth, lineHeight)
            // ... 详细诊断
          }
        }
      }
    }
  }
}
```

#### 3.2.4 浏览器行提取诊断

**文件**: `pages/accuracy.ts` (第 159-190 行)

```typescript
function getBrowserLines(
  prepared: PreparedTextWithSegments,
  div: HTMLDivElement,
): string[] {
  const textNode = div.firstChild
  if (!(textNode instanceof Text)) return []
  
  const units = getDiagnosticUnits(prepared)
  const range = document.createRange()
  const browserLines: string[] = []
  let currentLine = ''
  let lastTop: number | null = null
  
  // 使用 Range API 逐字符测量位置
  for (const unit of units) {
    range.setStart(textNode, unit.start)
    range.setEnd(textNode, unit.end)
    const rects = range.getClientRects()
    const rectTop: number | null = rects.length > 0 ? rects[0]!.top : lastTop
    
    // 检测行边界 (Y 坐标变化 > 0.5px)
    if (rectTop !== null && lastTop !== null && rectTop > lastTop + 0.5) {
      browserLines.push(currentLine)
      currentLine = unit.text
    } else {
      currentLine += unit.text
    }
    
    if (rectTop !== null) lastTop = rectTop
  }
  
  if (currentLine) browserLines.push(currentLine)
  return browserLines
}
```

### 3.3 Safari 特定引擎配置

**文件**: `src/measurement.ts` (第 65-101 行)

```typescript
export function getEngineProfile(): EngineProfile {
  const ua = navigator.userAgent
  const vendor = navigator.vendor
  
  // Safari 检测
  const isSafari =
    vendor === 'Apple Computer, Inc.' &&
    ua.includes('Safari/') &&
    !ua.includes('Chrome/') &&
    !ua.includes('Chromium/') &&
    !ua.includes('CriOS/') &&
    !ua.includes('FxiOS/') &&
    !ua.includes('EdgiOS/')
  
  const isChromium = /* ... */
  
  return {
    // Safari 使用更大的容差 (1/64 vs 0.005)
    lineFitEpsilon: isSafari ? 1 / 64 : 0.005,
    // Chromium 特有: 引号后 CJK 字符携带
    carryCJKAfterClosingQuote: isChromium,
    // Safari 特有: 使用前缀宽度计算
    preferPrefixWidthsForBreakableRuns: isSafari,
    // Safari 特有: 软连字符提前断行
    preferEarlySoftHyphenBreak: isSafari,
  }
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 完整调用链

```
safari.json
    │
    ├── 生成者 ──────────────────────────────────────────────┐
    │                                                          │
    ├── package.json                                          │
    │   └── scripts.accuracy-snapshot:safari                  │
    │       └── "ACCURACY_CHECK_BROWSER=safari bun run ..."   │
    │                                                          │
    ├── scripts/accuracy-check.ts                             │
    │   ├── 环境变量: ACCURACY_CHECK_BROWSER=safari           │
    │   ├── 超时: 240000ms (Safari 专用)                      │
    │   ├── createBrowserSession('safari')                    │
    │   └── loadHashReport<AccuracyReport>()                  │
    │                                                          │
    ├── scripts/browser-automation.ts                         │
    │   ├── createSafariSession()                             │
    │   │   └── AppleScript 控制 Safari                       │
    │   ├── acquireBrowserAutomationLock('safari')            │
    │   └── loadHashReport()                                  │
    │                                                          │
    ├── pages/accuracy.ts                                     │
    │   ├── runSweep()                                        │
    │   │   ├── FONTS × SIZES × WIDTHS × TEXTS 矩阵           │
    │   │   ├── prepareWithSegments()                         │
    │   │   ├── layout() → 预测高度                           │
    │   │   ├── div.getBoundingClientRect().height → 实际高度 │
    │   │   └── getBrowserLines() → 行级诊断                  │
    │   └── publishReport()                                   │
    │                                                          │
    ├── src/layout.ts                                         │
    │   ├── prepare() / prepareWithSegments()                 │
    │   ├── layout()                                          │
    │   └── layoutWithLines()                                 │
    │                                                          │
    ├── src/line-break.ts                                     │
    │   ├── countPreparedLines()                              │
    │   └── walkPreparedLines()                               │
    │                                                          │
    ├── src/measurement.ts                                    │
    │   └── getEngineProfile() → Safari 特定配置              │
    │                                                          │
    └── src/analysis.ts                                       │
        └── analyzeText()                                     │
                                                                │
    ├── 消费者 ──────────────────────────────────────────────┤
        │
        ├── scripts/status-dashboard.ts
        │   └── 聚合到 status/dashboard.json
        │
        ├── DEVELOPMENT.md
        │   └── 作为 "Current Sources Of Truth" 引用
        │
        └── AGENTS.md
            └── 作为 "checked-in raw accuracy rows" 引用
```

### 4.2 核心文件清单

| 文件 | 职责 | 与 safari.json 关系 |
|------|------|---------------------|
| `accuracy/safari.json` | Safari 准确性快照 | **本研究对象** |
| `accuracy/chrome.json` | Chrome 准确性快照 | 同级对比基准 |
| `accuracy/firefox.json` | Firefox 准确性快照 | 同级对比基准 |
| `scripts/accuracy-check.ts` | 准确性检查主脚本 | 生成/验证 safari.json |
| `scripts/browser-automation.ts` | 浏览器自动化 | Safari AppleScript 控制 |
| `pages/accuracy.ts` | 准确性测试页面 | 在 Safari 中执行测试 |
| `src/layout.ts` | 核心布局引擎 | 被测试的目标系统 |
| `src/line-break.ts` | 换行算法 | 布局引擎核心 |
| `src/measurement.ts` | 测量与引擎配置 | Safari 特定配置 |
| `src/analysis.ts` | 文本分析 | 预处理文本 |
| `src/test-data.ts` | 测试数据 | 定义测试矩阵 |
| `status/dashboard.json` | 状态仪表板 | 聚合 safari.json 统计 |

---

## 5. 依赖与外部交互

### 5.1 外部系统依赖

| 依赖 | 用途 | 交互方式 |
|------|------|----------|
| Safari 浏览器 | 渲染参考 | AppleScript (osascript) |
| macOS System Events | 窗口管理 | AppleScript |
| Bun 运行时 | 脚本执行 | 子进程 spawn |
| Node.js net 模块 | 端口分配 | createServer |
| WebSocket | Firefox BiDi | ws 协议 (Firefox 专用) |

### 5.2 Safari 自动化细节

```typescript
// browser-automation.ts 中 Safari 特定实现

// 1. 创建窗口
const scriptLines = [
  'tell application "Safari"',
  'set targetDocument to make new document with properties {URL:"about:blank"}',
  'return id of front window as string',
  'end tell'
]
const windowIdRaw = runBackgroundAppleScript(scriptLines)

// 2. 导航
const navigateLines = [
  'tell application "Safari"',
  `set targetWindow to first window whose id is ${windowId}`,
  `set URL of current tab of targetWindow to ${JSON.stringify(url)}`,
  'end tell'
]

// 3. 读取 URL
const readLines = [
  'tell application "Safari"',
  `return URL of current tab of (first window whose id is ${windowId})`,
  'end tell'
]
```

### 5.3 并发控制

```typescript
// browser-automation.ts
const LOCK_DIR = join(process.env['TMPDIR'] ?? '/tmp', 'pretext-browser-automation-locks')

export async function acquireBrowserAutomationLock(
  browser: AutomationBrowserKind,
  timeoutMs = 120_000,
): Promise<BrowserAutomationLock> {
  // 文件锁机制确保单浏览器串行执行
  // Safari 锁文件: /tmp/pretext-browser-automation-locks/safari.lock
}
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 环境敏感性

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| macOS 版本差异 | 不同 macOS 版本 Safari 渲染可能有细微差异 | 固定测试环境，文档化系统版本 |
| 字体可用性 | Safari 依赖系统字体，不同 macOS 版本字体可能不同 | 使用标准 Web 安全字体 |
| DPR 变化 | Retina (DPR=2) vs 非 Retina 显示差异 | 固定 DPR=2 环境测试 |
| Safari 版本 | WebKit 版本更新可能改变渲染行为 | 版本化快照，定期更新 |

#### 6.1.2 技术限制

```typescript
// measurement.ts 中的 Safari 特定容差
lineFitEpsilon: isSafari ? 1 / 64 : 0.005  // Safari 容差更大 (~0.0156 vs 0.005)
```

- **原因**: Safari 的文本测量精度与 Chrome/Firefox 存在差异
- **风险**: 更大的容差可能掩盖真实的布局问题

#### 6.1.3 自动化脆弱性

```typescript
// accuracy-check.ts 中的 Safari 超时
const reportTimeoutMs = browser === 'safari' ? '240000' : '120000'
```

- Safari 自动化比 Chrome 慢 2 倍
- AppleScript 控制可能受系统弹窗、权限请求干扰
- 需要前台应用状态管理 (`getFrontmostApplicationName`)

### 6.2 边界条件

#### 6.2.1 测试覆盖边界

| 边界 | 当前状态 | 说明 |
|------|----------|------|
| `white-space: normal` | ✅ 完全覆盖 | 默认模式 |
| `white-space: pre-wrap` | ⚠️ 单独测试 | `scripts/pre-wrap-check.ts` |
| `word-break: break-all` | ❌ 未测试 | AGENTS.md 标记为 "untested" |
| `word-break: keep-all` | ❌ 未测试 | AGENTS.md 标记为 "untested" |
| `line-break: strict` | ❌ 未测试 | AGENTS.md 标记为 "untested" |
| `line-break: loose` | ❌ 未测试 | AGENTS.md 标记为 "untested" |
| `overflow-wrap: anywhere` | ❌ 未测试 | 与 `break-word` 有差异 |

#### 6.2.2 语言覆盖边界

当前 `safari.json` 覆盖：
- ✅ 拉丁语系 (英语)
- ✅ CJK (中日韩)
- ✅ 阿拉伯语、希伯来语 (RTL)
- ✅ 泰语
- ✅ Emoji
- ⚠️ 缅甸语、高棉语、老挝语 (仅语料库测试)
- ❌ 印地语、乌尔都语 (仅语料库测试)

### 6.3 改进建议

#### 6.3.1 短期改进

1. **增加版本元数据**
   ```json
   {
     "environment": {
       "safariVersion": "17.4",
       "webkitVersion": "605.1.15",
       "macosVersion": "14.4"
     }
   }
   ```

2. **细化不匹配诊断**
   - 当前 `mismatches` 为空数组 (100% 匹配)
   - 建议保留最近的不匹配历史用于趋势分析

3. **增加性能指标**
   ```json
   {
     "performance": {
       "totalDurationMs": 45000,
       "avgTestDurationMs": 5.86
     }
   }
   ```

#### 6.3.2 中期改进

1. **多版本 Safari 测试**
   - 使用 Safari Technology Preview
   - 对比不同 WebKit 版本差异

2. **iOS Safari 覆盖**
   - 当前仅测试 macOS Safari
   - iOS WebKit 与 macOS 存在差异

3. **可视化报告**
   ```
   pages/demos/accuracy-visualization.html
   - 热力图显示不同字体/尺寸/宽度的匹配度
   - 与 Chrome/Firefox 的对比视图
   ```

#### 6.3.3 长期改进

1. **动态容差校准**
   ```typescript
   // 替代固定的 1/64
   const lineFitEpsilon = await calibrateEpsilon(font, sampleText)
   ```

2. **机器学习辅助**
   - 训练模型预测 Safari 特定渲染行为
   - 减少硬编码的引擎差异处理

3. **标准化测试集**
   - 与 W3C 文本布局测试套件对齐
   - 贡献回 Web Platform Tests

### 6.4 监控建议

```typescript
// 建议添加的监控指标
interface AccuracyMetrics {
  // 当前已有
  total: number
  matchCount: number
  mismatchCount: number
  
  // 建议新增
  matchRate: number           // 匹配率趋势
  avgAbsDiff: number          // 平均绝对差值
  maxDiff: number             // 最大差值
  diffDistribution: number[]  // 差值分布直方图
  slowestTestMs: number       // 最慢测试耗时
}
```

---

## 附录 A: 文件统计

| 属性 | 值 |
|------|-----|
| 文件路径 | `accuracy/safari.json` |
| 总行数 | 76,828 |
| 测试记录数 | 7,680 |
| 文件大小 | ~2.5 MB |
| 匹配率 | 100% (7680/7680) |
| 不匹配数 | 0 |

## 附录 B: 相关命令速查

```bash
# Safari 准确性检查
bun run accuracy-check:safari

# 更新 Safari 快照
bun run accuracy-snapshot:safari

# 带完整行数据输出
bun run scripts/accuracy-check.ts --full --output=accuracy/safari.json

# 对比不同浏览器结果
bun run accuracy-check  # Chrome (默认)
bun run accuracy-check:safari
bun run accuracy-check:firefox
```

## 附录 C: 数据样本

```json
{
  "label": "Latin update",
  "font": "\"Helvetica Neue\", Helvetica, Arial, sans-serif",
  "fontSize": 12,
  "lineHeight": 14,
  "width": 150,
  "actual": 70,
  "predicted": 70,
  "diff": 0
}
```

---

*文档生成时间: 2026-04-02*
*基于 Pretext 项目 commit: 当前工作目录状态*
