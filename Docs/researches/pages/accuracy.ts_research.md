# accuracy.ts 研究文档

## 场景与职责

`pages/accuracy.ts` 是 Pretext 库的核心精度测试引擎。它执行系统化的浏览器兼容性验证，通过对比 Pretext 库的预测结果与浏览器 DOM 的实际渲染行为，量化算法准确性。该模块是项目质量保证体系的技术基石，支持自动化 CI/CD 流程中的回归检测。

核心职责：
- **多维度扫描**：跨 4 种字体族、8 种字号、8 种容器宽度、55 组测试文本的完整矩阵测试
- **高度精度验证**：对比 `layout()` 预测高度与 `div.getBoundingClientRect().height`
- **行级诊断**：不匹配时提取浏览器实际行内容与 Pretext 预测行内容对比
- **环境指纹采集**：记录 User-Agent、DPR、视口尺寸等环境信息

## 功能点目的

### 1. 系统化精度扫描 (`runSweep`)

遍历预定义的字体、字号、宽度、文本组合矩阵，执行批量测试：

```typescript
for (const fontFamily of FONTS) {
  for (const fontSize of SIZES) {
    for (const maxWidth of WIDTHS) {
      for (const { text } of TEXTS) {
        // 对比 actual vs predicted
      }
    }
  }
}
```

### 2. 浏览器行提取 (`getBrowserLines`)

使用 `Range.getClientRects()` 精确提取浏览器实际断行位置：

```typescript
const range = document.createRange()
for (const unit of units) {
  range.setStart(textNode, unit.start)
  range.setEnd(textNode, unit.end)
  const rects = range.getClientRects()
  // 通过 rect.top 变化检测换行
}
```

### 3. 诊断报告生成

生成结构化报告，包含：
- 环境指纹（EnvironmentFingerprint）
- 匹配/不匹配统计
- 不匹配项的详细差异（行级对比）
- 可选的完整行数据（`?full=1`）

### 4. 自动化集成

支持通过 URL hash 与外部自动化框架通信：
- `publishNavigationPhase`: 报告加载/测量/提交阶段
- `publishNavigationReport`: 通过 hash 或 POST 提交结果

## 具体技术实现

### 核心数据结构

```typescript
type Mismatch = {
  label: string        // 测试文本标签
  font: string         // 字体族
  fontSize: number     // 字号
  lineHeight: number   // 行高
  width: number        // 容器宽度
  actual: number       // DOM 实际高度
  predicted: number    // Pretext 预测高度
  diff: number         // 差异值
  text: string         // 原始文本
  diagnosticLines?: string[]  // 行级差异详情
}

type AccuracyReport = {
  status: 'ready' | 'error'
  environment?: EnvironmentFingerprint
  total?: number
  matchCount?: number
  mismatchCount?: number
  mismatches?: Mismatch[]
  rows?: AccuracyRow[]  // 完整数据（可选）
}
```

### 测试矩阵常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `FONTS` | 4 种字体族 | Helvetica Neue, Georgia, Verdana, Courier New |
| `SIZES` | [12,14,15,16,18,20,24,28] | 常见字号 |
| `WIDTHS` | [150,200,250,300,350,400,500,600] | 容器宽度 |
| `TEXTS` | 55 组 | 来自 `src/test-data.ts` |

**总测试用例数**: 4 × 8 × 8 × 55 = **14,080**

### 行级诊断算法

当高度不匹配时，执行深度诊断：

1. 使用 `layoutWithLines()` 获取 Pretext 预测的行内容
2. 使用 `getBrowserLines()` 通过 Range API 提取浏览器实际行
3. 逐行对比，生成差异描述：
   ```
   L3 ours="The quick brown fox jumps over the..." browser="The quick brown fox jumps over the..."
   ```

### 环境指纹采集

```typescript
type EnvironmentFingerprint = {
  userAgent: string
  devicePixelRatio: number
  viewport: { innerWidth, innerHeight, outerWidth, outerHeight, visualViewportScale }
  screen: { width, height, availWidth, availHeight, colorDepth, pixelDepth }
}
```

## 关键代码路径与文件引用

### 依赖图

```
accuracy.ts
├── ../src/layout.ts
│   ├── clearCache()
│   ├── layout()              // 热路径：纯算术计算高度
│   ├── layoutWithLines()     // 富路径：返回行内容
│   └── prepareWithSegments() // 预处理：分词+测量
├── ./diagnostic-utils.ts
│   └── getDiagnosticUnits()  // 诊断单元生成
├── ./report-utils.ts
│   ├── clearNavigationReport()
│   ├── publishNavigationPhase()
│   └── publishNavigationReport()
└── ../src/test-data.ts
    ├── TEXTS (55组测试文本)
    ├── SIZES (8种字号)
    └── WIDTHS (8种宽度)
```

### 调用链

```
render()
└── runSweep()
    ├── prepareWithSegments()  // 每个文本一次
    ├── layout()               // 热路径高度计算
    └── 不匹配时:
        ├── getBrowserLines()  // Range API 提取
        └── layoutWithLines()  // 获取预测行
```

## 依赖与外部交互

### 浏览器 API

| API | 用途 |
|-----|------|
| `document.createElement` | 创建隐藏测试容器 |
| `div.getBoundingClientRect()` | 获取 DOM 实际高度 |
| `document.createRange()` | 精确文本范围选择 |
| `range.getClientRects()` | 获取范围的渲染矩形 |
| `performance.now()` | 高精度计时（可选） |
| `fetch()` | POST 大型报告到 reportEndpoint |

### URL 参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `requestId` | string | 自动化测试请求标识 |
| `full` | '1' | 包含完整 rows 数据 |
| `reportEndpoint` | URL | 大型报告的 POST 端点 |

### 全局对象

```typescript
interface Window {
  __ACCURACY_REPORT__?: AccuracyReport  // 报告挂载点
}
```

## 风险、边界与改进建议

### 已知风险

1. **Range API 精度**：
   - Safari 在东南亚文字（泰/老/高棉/缅甸）上可能过度推进
   - RTL 文本的 Range 提取需要特殊处理

2. **字体加载时序**：
   - 依赖 `document.fonts.ready`，但某些浏览器实现不完整
   - 系统字体回退可能导致跨平台差异

3. **测量噪声**：
   - 亚像素渲染差异（特别是 Safari）
   - 行高舍入差异

### 边界情况处理

| 场景 | 处理策略 |
|------|----------|
| 空文本 | `TEXTS` 包含空字符串测试用例 |
| 超长单词 | `overflow-wrap: break-word` + 字素级断行 |
| CJK 禁则 | 在 `analysis.ts` 中处理 kinsoku 规则 |
| 软连字符 | 行尾显示 `-`，不计入文本长度 |

### 改进建议

1. **并行化**：当前串行执行，可考虑分片并行（需注意 DOM 操作同步性）
2. **增量测试**：支持仅测试变更的字体/尺寸组合
3. **视觉回归**：集成截图对比，检测非高度相关的渲染差异
4. **性能基准**：记录每次测试的执行时间，检测性能回归
5. **提取器可配置**：允许通过 URL 参数选择 span-probe/range 提取方法

### 相关配置

- 行适配容差：`0.005`（Chromium/Gecko），`1/64`（Safari/WebKit）
- 不匹配阈值：`Math.abs(actual - predicted) >= 1`（像素）
