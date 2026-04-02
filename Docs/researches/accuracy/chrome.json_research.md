# accuracy/chrome.json 研究文档

## 概述

`accuracy/chrome.json` 是 Pretext 文本布局库的核心浏览器准确性测试快照文件，记录了在 Google Chrome 浏览器中执行的全面文本布局准确性测试结果。该文件作为**基准真值（Ground Truth）**被检入版本控制，用于持续验证库的计算结果与浏览器实际渲染行为的一致性。

---

## 1. 场景与职责

### 1.1 核心职责

| 职责 | 说明 |
|------|------|
**准确性验证基准** | 作为 Chrome 浏览器下文本布局计算的"黄金标准"，验证 `layout()` 和 `layoutWithLines()` API 的输出与浏览器 DOM 实际渲染高度的一致性
**回归检测** | 通过对比新旧快照，检测代码变更是否引入了布局计算偏差
**跨浏览器对标** | 与 `accuracy/safari.json`、`accuracy/firefox.json` 形成三浏览器覆盖矩阵，识别引擎特定行为差异
**Dashboard 数据源** | 被 `status/dashboard.json` 聚合，作为项目健康度的核心指标

### 1.2 使用场景

1. **开发阶段**: 运行 `bun run accuracy-check` 验证本地修改是否破坏 Chrome 兼容性
2. **发布阶段**: 运行 `bun run accuracy-snapshot` 刷新快照，确保基准与最新实现同步
3. **CI/CD**: 理论上可用于自动化验证（当前由 GitHub Pages workflow 覆盖构建流程）
4. **问题诊断**: 当 Safari/Firefox 出现不匹配时，以 Chrome 结果作为参考判断是否为引擎差异

---

## 2. 功能点目的

### 2.1 测试覆盖矩阵

文件包含 **7,680 个测试用例**，覆盖以下维度：

```
4 字体族 × 8 字号 × 8 容器宽度 × 30 文本样本 = 7,680 组合
```

| 维度 | 具体值 | 目的 |
|------|--------|------|
**字体族** | `"Helvetica Neue"`, `Georgia`, `Verdana`, `"Courier New"` | 覆盖无衬线、衬线、等宽字体类别 |
**字号** | 12, 14, 15, 16, 18, 20, 24, 28 px | 覆盖正文到标题常见尺寸 |
**容器宽度** | 150, 200, 250, 300, 350, 400, 500, 600 px | 覆盖窄列到宽容器场景 |
**文本样本** | 30 个标签化文本（见 `src/test-data.ts`） | 覆盖多语言、多脚本、边界情况 |

### 2.2 文本样本分类

源自 `src/test-data.ts` 的 30 个测试文本：

| 类别 | 标签示例 | 测试重点 |
|------|----------|----------|
**拉丁文本** | `Latin update`, `Latin short`, `Latin punctuation` | 基础英文排版、标点粘连、连字符 |
**RTL 语言** | `Arabic`, `Arabic short`, `Hebrew`, `Hebrew short` | 从右到左布局、双向文本 |
**混合文本** | `Mixed en+ar`, `Mixed en+he`, `Mixed report` | LTR/RTL 边界处理 |
**CJK 文本** | `Chinese`, `Chinese short`, `Japanese`, `Korean` | 中日韩字符、禁则处理（Kinsoku） |
**东南亚** | `Thai` | 泰语无空格分词 |
**Emoji** | `Emoji mixed`, `Emoji dense` | 彩色表情符号宽度校正 |
**边界情况** | `Empty`, `Whitespace`, `Long word`, `Newlines` | 空文本、长单词、硬换行 |

### 2.3 验证指标

每个测试用例记录以下字段：

```typescript
interface AccuracyRow {
  label: string;        // 文本样本标签（如 "Latin update"）
  font: string;         // 字体族（如 "Helvetica Neue"）
  fontSize: number;     // 字号（px）
  lineHeight: number;   // 行高（px，通常为 fontSize * 1.2）
  width: number;        // 容器最大宽度（px）
  actual: number;       // 浏览器 DOM 实际渲染高度（px）
  predicted: number;    // Pretext 库预测高度（px）
  diff: number;         // 差异（predicted - actual）
}
```

**通过标准**: `|actual - predicted| < 1px`（见 `pages/accuracy.ts:239`）

---

## 3. 具体技术实现

### 3.1 数据生成流程

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│ accuracy-check  │────▶│  pages/accuracy  │────▶│  Chrome Browser │
│    (Node/Bun)   │     │    (浏览器页面)   │     │  (AppleScript)  │
└─────────────────┘     └──────────────────┘     └─────────────────┘
         │                       │                        │
         │                       ▼                        │
         │              ┌──────────────────┐              │
         │              │  执行 runSweep()  │              │
         │              │  - 创建隐藏 DOM   │              │
         │              │  - 调用 layout()  │              │
         │              │  - 对比高度差异   │              │
         │              └──────────────────┘              │
         │                       │                        │
         ▼                       ▼                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                      报告收集（Hash/POST）                        │
│  - 小报告: URL hash (#report=...)                               │
│  - 大报告: POST 到本地 report-server.ts                         │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │  accuracy/chrome.json │
                         │  （本文件）            │
                         └─────────────────────┘
```

### 3.2 关键数据结构

#### 3.2.1 文件根结构

```json
{
  "status": "ready",           // "ready" | "error"
  "environment": {              // 浏览器环境指纹
    "userAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...",
    "devicePixelRatio": 2,
    "viewport": { /* innerWidth, innerHeight, ... */ },
    "screen": { /* width, height, colorDepth, ... */ }
  },
  "total": 7680,               // 总测试数
  "matchCount": 7680,          // 通过数
  "mismatchCount": 0,          // 失败数
  "mismatches": [],            // 详细失败信息（仅在失败时填充）
  "rows": [ /* ...7680 个 AccuracyRow... */ ],
  "requestId": "1775067639950-5r3dl2"  // 请求追踪 ID
}
```

#### 3.2.2 不匹配记录结构（mismatches 数组元素）

```typescript
interface AccuracyMismatch extends AccuracyRow {
  text: string;              // 完整原始文本
  diagnosticLines?: string[]; // 逐行诊断信息
}
```

诊断信息格式示例：
```
L1 ours="Hello world" browser="Hello world"
L2 ours="foo bar" browser="foo  bar" (same content, different count?)
```

### 3.3 核心算法流程

#### 3.3.1 浏览器端测量逻辑（pages/accuracy.ts:192-277）

```typescript
function runSweep() {
  for (const fontFamily of FONTS) {
    for (const fontSize of SIZES) {
      clearCache();  // 每个字号清除缓存，避免交叉污染
      
      for (const maxWidth of WIDTHS) {
        // 1. 创建 DOM 元素并设置样式
        const div = document.createElement('div');
        div.style.font = font;
        div.style.width = `${maxWidth}px`;
        div.style.wordWrap = 'break-word';
        div.style.overflowWrap = 'break-word';
        
        // 2. 浏览器实际渲染高度
        const actual = div.getBoundingClientRect().height;
        
        // 3. Pretext 预测高度
        const prepared = prepareWithSegments(text, font);
        const predicted = layout(prepared, maxWidth, lineHeight).height;
        
        // 4. 对比（允许 1px 误差）
        if (Math.abs(actual - predicted) >= 1) {
          // 记录不匹配，包含逐行诊断
        }
      }
    }
  }
}
```

#### 3.3.2 逐行诊断生成（getBrowserLines）

使用 `Range.getClientRects()` 提取浏览器实际断行位置：

```typescript
function getBrowserLines(prepared: PreparedTextWithSegments, div: HTMLDivElement): string[] {
  const units = getDiagnosticUnits(prepared);  // 获取诊断单元（segment/grapheme）
  const range = document.createRange();
  
  for (const unit of units) {
    range.setStart(textNode, unit.start);
    range.setEnd(textNode, unit.end);
    const rects = range.getClientRects();
    // 通过 rect.top 变化检测换行
  }
}
```

### 3.4 引擎特定处理

#### 3.4.1 行适应容差（lineFitEpsilon）

`src/measurement.ts:94-99` 定义了不同引擎的容差：

```typescript
const engineProfile = {
  lineFitEpsilon: isSafari ? 1 / 64 : 0.005,  // Safari 更宽松
  // ...
};
```

Chrome 使用 `0.005`（约 1/200），这是基于阿拉伯语细宽度场域的实证调优结果。

#### 3.4.2 Emoji 宽度校正

`src/measurement.ts:123-151` 实现了针对 Chrome/Firefox 的 emoji 宽度校正：

```typescript
function getEmojiCorrection(font: string, fontSize: number): number {
  // 在 macOS 上，canvas 测量的 emoji 宽度比 DOM 实际渲染宽
  // 通过创建 span 元素对比测量，计算校正偏移量
  const canvasW = ctx.measureText('\u{1F600}').width;
  const domW = span.getBoundingClientRect().width;
  return canvasW - domW;  // 通常为 0 或一个固定偏移
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 数据生产链路

| 文件 | 角色 | 关键函数/代码 |
|------|------|---------------|
| `scripts/accuracy-check.ts` | 自动化入口 | `loadBrowserReport()`, `printReport()` |
| `scripts/browser-automation.ts` | 浏览器控制 | `createChromeSession()`, `loadHashReport()` |
| `pages/accuracy.ts` | 浏览器端测试页 | `runSweep()`, `getBrowserLines()` |
| `pages/diagnostic-utils.ts` | 诊断辅助 | `getDiagnosticUnits()` |
| `pages/report-utils.ts` | 报告发布 | `publishReport()`, `publishNavigationPhase()` |
| `shared/navigation-state.ts` | 状态协议 | `buildNavigationReportHash()`, `readNavigationPhaseState()` |
| `scripts/report-server.ts` | 大报告接收 | `startPostedReportServer()` |

### 4.2 核心算法链路

| 文件 | 角色 | 关键函数/代码 |
|------|------|---------------|
| `src/layout.ts` | 主 API | `prepare()`, `layout()`, `layoutWithLines()` |
| `src/analysis.ts` | 文本分析 | `analyzeText()`, `normalizeWhitespaceNormal()` |
| `src/measurement.ts` | 测量引擎 | `getSegmentMetrics()`, `getEmojiCorrection()` |
| `src/line-break.ts` | 断行算法 | `countPreparedLines()`, `walkPreparedLines()` |
| `src/bidi.ts` | 双向文本 | `computeSegmentLevels()` |
| `src/test-data.ts` | 测试数据 | `TEXTS`, `SIZES`, `WIDTHS` |

### 4.3 数据消费链路

| 文件 | 角色 | 关键代码 |
|------|------|----------|
| `scripts/status-dashboard.ts` | Dashboard 生成 | `summarizeAccuracy(chromeAccuracy)` |
| `status/dashboard.json` | 聚合展示 | `browserAccuracy.chrome` |

---

## 5. 依赖与外部交互

### 5.1 运行时依赖

```
浏览器端（pages/accuracy.ts）:
├── DOM API
│   ├── document.createElement()      // 创建测试容器
│   ├── getBoundingClientRect()       // 获取实际渲染高度
│   ├── document.createRange()        // 逐行诊断
│   └── Range.getClientRects()        // 获取文本矩形
├── Canvas API
│   ├── OffscreenCanvas / HTMLCanvasElement
│   └── CanvasRenderingContext2D.measureText()  // 文本宽度测量
└── Intl API
    ├── Intl.Segmenter (granularity: 'word')    // 分词
    └── Intl.Segmenter (granularity: 'grapheme') // 字素分割

Node/Bun 端（scripts/accuracy-check.ts）:
├── AppleScript（macOS）
│   └── osascript 控制 Chrome/Safari 导航
├── WebSocket（Firefox）
│   └── Firefox BiDi 协议通信
└── HTTP Server
    └── Bun 临时页面服务器
```

### 5.2 外部系统交互

| 系统 | 交互方式 | 目的 |
|------|----------|------|
**Chrome 浏览器** | AppleScript（macOS） | 自动化导航到测试页面并提取结果 |
**临时 HTTP 服务** | Bun 内置服务器 | 提供 `pages/accuracy.ts` 页面 |
**Report Server** | 本地 HTTP POST | 接收大体积报告（避免 URL hash 限制） |
**文件系统** | `writeFileSync` | 写入 JSON 快照 |

### 5.3 锁机制

`scripts/browser-automation.ts:167-216` 实现了单浏览器锁：

```typescript
// 防止并行运行多个浏览器自动化任务
const LOCK_DIR = join(process.env['TMPDIR'] ?? '/tmp', 'pretext-browser-automation-locks');

export async function acquireBrowserAutomationLock(browser: BrowserKind, timeoutMs = 120_000) {
  // 使用文件锁（wx 标志创建）+ PID 检查实现自愈合
}
```

---

## 6. 风险、边界与改进建议

### 6.1 当前风险

| 风险类别 | 具体描述 | 影响等级 |
|----------|----------|----------|
**平台绑定** | 当前自动化仅支持 macOS（AppleScript 控制 Chrome/Safari） | 中 |
**字体依赖** | 测试结果依赖系统字体（Helvetica Neue, Georgia 等） | 中 |
**DPR 敏感** | 在 `devicePixelRatio=2`（Retina）下生成，未验证其他 DPR | 低 |
**单点失效** | 7,680 个用例全部通过，但无法保证覆盖所有边界 | 低 |
**版本漂移** | Chrome 升级可能改变渲染行为，需定期刷新快照 | 中 |

### 6.2 已知边界情况

1. **system-ui 字体**: 明确不支持（`AGENTS.md` 警告），canvas 和 DOM 解析不同字体变体
2. **Safari Range 提取**: 在 URL query 文本上可能过度推进（需用 `--method=span` 交叉验证）
3. **阿拉伯语细宽度**: Chrome 使用 `0.005` 容差，Safari 使用 `1/64`（约 0.0156）
4. **软连字符（Soft Hyphen）**: 当前在 `layoutWithLines()` 中显示尾部 `-`，但行类型无单独元数据标志

### 6.3 改进建议

#### 6.3.1 短期（维护性）

| 建议 | 优先级 | 实现思路 |
|------|--------|----------|
**添加生成时间戳** | 高 | 在 JSON 根添加 `generatedAt` 字段，便于追踪快照新鲜度 |
**分离环境元数据** | 中 | 将 `environment` 拆分到独立文件，减少主文件体积 |
**压缩存储** | 低 | 使用 JSON 行格式或 gzip 压缩，当前 76,828 行过大 |

#### 6.3.2 中期（功能性）

| 建议 | 优先级 | 实现思路 |
|------|--------|----------|
**Linux/Windows 支持** | 中 | 使用 Chrome DevTools Protocol (CDP) 替代 AppleScript |
**增量测试** | 中 | 仅测试变更影响的字体/字号组合，缩短 CI 时间 |
**视觉回归** | 低 | 结合 Playwright 截图对比，捕获亚像素差异 |

#### 6.3.3 长期（架构性）

| 建议 | 优先级 | 实现思路 |
|------|--------|----------|
**运行时校准** | 低 | 将 `lineFitEpsilon` 从硬编码改为运行时校准，类似 emoji 校正 |
**富断行元数据** | 低 | 在 `LayoutLine` 类型中添加 `softHyphenBroken` 标志 |
**多 DPR 测试矩阵** | 低 | 在 1x、2x、3x DPR 下分别生成快照 |

### 6.4 监控建议

1. **Dashboard 告警**: 当 `matchCount/total < 100%` 时触发通知
2. **定期刷新**: 建议每月运行 `bun run accuracy-snapshot` 并审查差异
3. **跨浏览器对比**: 建立 Chrome/Safari/Firefox 差异白名单，区分引擎差异与实现缺陷

---

## 附录：文件统计

| 属性 | 值 |
|------|-----|
**文件路径** | `accuracy/chrome.json` |
**总行数** | 76,828 行 |
**总测试用例** | 7,680 个 |
**通过数** | 7,680 (100%) |
**失败数** | 0 |
**字体族数** | 4 |
**字号档位** | 8 (12~28px) |
**宽度档位** | 8 (150~600px) |
**文本样本数** | 30 |
**最后 requestId** | `1775067639950-5r3dl2` |

---

## 参考文档

- `AGENTS.md` - 项目代理开发指南
- `DEVELOPMENT.md` - 开发命令参考
- `RESEARCH.md` - 文本布局研究日志
- `STATUS.md` - 项目状态概览
- `src/layout.ts` - 核心布局 API
- `pages/accuracy.ts` - 浏览器端测试实现
