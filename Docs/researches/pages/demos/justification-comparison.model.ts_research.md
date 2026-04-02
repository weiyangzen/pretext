# justification-comparison.model.ts 研究文档

## 场景与职责

`justification-comparison.model.ts` 是 Justification Comparison 演示的核心算法模块，实现了三种文本对齐算法的完整布局引擎：

1. **Greedy 算法**（贪婪）：逐行填充，到达最大宽度即断行
2. **Greedy + Hyphenation**：在贪婪基础上添加软连字符支持
3. **Knuth-Plass 算法**：全局优化，最小化整段排版 badness

该模块是纯计算层，无 DOM 操作，可被测试或服务器端渲染复用。

## 功能点目的

### 1. 算法实现与对比

| 算法 | 复杂度 | 质量 | 适用场景 |
|-----|-------|------|---------|
| Greedy | O(n) | 低 | 浏览器默认、快速预览 |
| Greedy + Hyphenation | O(n) | 中 | 改善短行问题 |
| Knuth-Plass | O(n²) | 高 | 专业排版、印刷质量 |

### 2. 排版质量度量

计算并暴露以下指标：
- **平均偏差** (avgDeviation)：词间距相对于标准空格的偏离程度
- **最大偏差** (maxDeviation)：最极端的词间距偏离
- **河流计数** (riverCount)：超过阈值的过宽词间距数量
- **行数** (lineCount)：段落占用的行数

### 3. 河流检测

识别排版中的"河流"现象——垂直方向上连续出现的过宽词间距，这会干扰阅读流畅性。

## 具体技术实现

### 核心常量

```typescript
const HUGE_BADNESS = 1e8           // 不可行解的惩罚值
const SOFT_HYPHEN = '\u00AD'       // Unicode 软连字符
const SHORT_LINE_RATIO = 0.6       // 短行阈值（行宽 < 60% 最大宽度）
const RIVER_THRESHOLD = 1.5        // 河流阈值（词间距 > 1.5 倍标准空格）
const INFEASIBLE_SPACE_RATIO = 0.4 // 不可行空格比率
const OVERFLOW_SPACE_RATIO = 0.2   // 溢出处理比率
const TIGHT_SPACE_RATIO = 0.65     // 过紧空格阈值
```

### 数据结构

#### 行段 (LineSegment)
```typescript
export type LineSegment =
  | { kind: 'text'; text: string; width: number }
  | { kind: 'space'; width: number }
```

#### 测量行 (MeasuredLine)
```typescript
export type MeasuredLine = {
  segments: LineSegment[]      // 行内容段
  wordWidth: number            // 单词总宽度
  spaceCount: number           // 空格数量
  naturalWidth: number         // 自然宽度（无伸展）
  maxWidth: number             // 最大可用宽度
  ending: 'paragraph-end' | 'wrap'  // 行结束类型
  trailingMarker: 'none' | 'soft-hyphen'  // 尾随标记
}
```

#### 定位行 (PositionedLine)
```typescript
export type PositionedLine = MeasuredLine & {
  y: number                   // 垂直位置
  spacing: LineSpacing        // 间距策略
}
```

#### 行间距策略
```typescript
export type LineSpacing =
  | { kind: 'ragged' }         // 左对齐（短行或末行）
  | { kind: 'overflow' }       // 溢出（单词过长）
  | { kind: 'justified'; width: number; isRiver: boolean }  // 两端对齐
```

### 算法详解

#### 1. 连字符化预处理

```typescript
function hyphenateParagraphText(paragraph: string): string
```
流程：
1. 按空白字符分割段落
2. 对每个单词调用 `hyphenateWord`
3. 使用软连字符 `\u00AD` 连接音节

```typescript
function hyphenateWord(word: string): string[]
```
优先级：
1. 精确词典匹配 (`HYPHEN_EXCEPTIONS`)
2. 前缀匹配 (`PREFIXES`)
3. 后缀匹配 (`SUFFIXES`)
4. 无匹配返回原词

#### 2. Greedy 布局

```typescript
function layoutParagraphGreedy(
  prepared: PreparedTextWithSegments,
  maxWidth: number,
  hyphenWidth: number
): MeasuredLine[]
```

算法步骤：
1. 从光标 `{ segmentIndex: 0, graphemeIndex: 0 }` 开始
2. 调用 `layoutNextLine` 获取下一行内容
3. 将布局结果转换为 `MeasuredLine`
4. 更新光标，重复直到文本耗尽

```typescript
function buildMeasuredLineFromLayoutResult(
  prepared: PreparedTextWithSegments,
  start: LayoutCursor,
  end: LayoutCursor,
  maxWidth: number,
  hyphenWidth: number
): MeasuredLine
```

处理软连字符：
- 如果行尾是软连字符，设置 `trailingMarker = 'soft-hyphen'`
- 在 `segments` 中添加可见的 `'-'` 字符（使用 `hyphenWidth`）

#### 3. Knuth-Plass 算法

```typescript
function layoutParagraphOptimal(
  prepared: PreparedTextWithSegments,
  maxWidth: number,
  resources: DemoResources
): MeasuredLine[]
```

核心步骤：

**A. 构建断点候选列表**
```typescript
const breakCandidates: BreakCandidate[] = [
  { segIndex: 0, kind: 'start' },           // 段落开始
  { segIndex: n, kind: 'space' },           // 空格后
  { segIndex: n, kind: 'soft-hyphen' },     // 软连字符后
  { segIndex: len, kind: 'end' },           // 段落结束
]
```

**B. 动态规划求解**
```typescript
const dp: number[] = new Array(candidateCount).fill(Infinity)
const previous: number[] = new Array(candidateCount).fill(-1)
dp[0] = 0  // 起始点 badness 为 0

for (let to = 1; to < candidateCount; to++) {
  for (let from = to - 1; from >= 0; from--) {
    // 计算从 from 到 to 的 badness
    const lineStats = getLineStatsFromBreakCandidates(...)
    const totalBadness = dp[from] + lineBadness(lineStats, ...)
    if (totalBadness < dp[to]) {
      dp[to] = totalBadness
      previous[to] = from
    }
  }
}
```

**C. 回溯重建路径**
```typescript
const breakIndices: number[] = []
let current = candidateCount - 1
while (current > 0) {
  breakIndices.push(current)
  current = previous[current]!
}
breakIndices.reverse()
```

#### 4. Badness 计算

```typescript
function lineBadness(
  lineStats: LineStats,
  maxWidth: number,
  normalSpaceWidth: number,
  isLastLine: boolean
): number
```

末行特殊处理：
```typescript
if (isLastLine) {
  if (lineStats.wordWidth > maxWidth) return HUGE_BADNESS
  return 0  // 末行不惩罚（左对齐）
}
```

常规行计算：
```typescript
const justifiedSpace = (maxWidth - wordWidth) / spaceCount
const ratio = (justifiedSpace - normalSpaceWidth) / normalSpaceWidth
const badness = Math.abs(ratio) ** 3 * 1000

// 额外惩罚
const riverPenalty = justifiedSpace > threshold ? ... : 0
const tightPenalty = justifiedSpace < tightThreshold ? ... : 0
const hyphenPenalty = hasHyphen ? 50 : 0

return badness + riverPenalty + tightPenalty + hyphenPenalty
```

### 质量指标计算

```typescript
function computeMetrics(
  paragraphs: MeasuredLine[][],
  normalSpaceWidth: number
): QualityMetrics
```

遍历所有行：
1. 计算每行的实际词间距
2. 与标准空格比较，计算偏差
3. 统计超过河流阈值的行数

### 河流指示器

```typescript
export function getRiverIndicator(
  spaceWidth: number,
  normalSpaceWidth: number
): RiverIndicator | null
```

返回 RGBA 颜色，强度与空格宽度成正比：
```typescript
const intensity = Math.min(1, (spaceWidth / normalSpaceWidth - 1.5) / 1.5)
return {
  red: Math.round(220 + intensity * 35),
  green: Math.round(180 - intensity * 80),
  blue: Math.round(180 - intensity * 80),
  alpha: 0.25 + intensity * 0.35,
}
```

## 关键代码路径与文件引用

### 依赖导入

```typescript
import {
  prepareWithSegments,
  layoutNextLine,
  type LayoutCursor,
  type PreparedTextWithSegments,
} from '../../src/layout.ts'
import {
  FONT,
  HYPHEN_EXCEPTIONS,
  LINE_HEIGHT,
  PAD,
  PARA_GAP,
  PARAGRAPHS,
  PREFIXES,
  SUFFIXES,
} from './justification-comparison.data.ts'
```

### 导出 API

```typescript
// 资源创建
export function createDemoResources(): DemoResources

// 帧构建
export function buildDemoFrame(
  resources: DemoResources,
  controls: DemoControls
): DemoFrame

// 河流指示器
export function getRiverIndicator(
  spaceWidth: number,
  normalSpaceWidth: number
): RiverIndicator | null
```

### 核心类型导出

```typescript
export type DemoControls = { colWidth: number; showIndicators: boolean }
export type DemoResources = { /* ... */ }
export type DemoFrame = { css: {...}; hyphen: CanvasColumnFrame; optimal: CanvasColumnFrame }
export type MeasuredLine = { /* ... */ }
export type PositionedLine = { /* ... */ }
export type QualityMetrics = { avgDeviation: number; maxDeviation: number; riverCount: number; lineCount: number }
```

## 依赖与外部交互

### 外部依赖

| 依赖 | 用途 |
|------|------|
| `../../src/layout.ts` | Pretext 核心 API：prepareWithSegments, layoutNextLine |
| `justification-comparison.data.ts` | 文本数据、连字符词典、排版常量 |

### Canvas 测量

```typescript
const measureCanvas = document.createElement('canvas')
const measureCtx = measureCanvas.getContext('2d')
measureCtx.font = FONT
const normalSpaceWidth = measureCtx.measureText(' ').width
const hyphenWidth = measureCtx.measureText('-').width
```

## 风险、边界与改进建议

### 已知风险

1. **Knuth-Plass 复杂度**
   - O(n²) 复杂度在处理长段落时可能成为瓶颈
   - 当前实现使用 `if (lineStats.naturalWidth > maxWidth * 2) break` 提前剪枝

2. **连字符词典不完整**
   - 仅覆盖演示文本中的词汇
   - 未覆盖的单词无法断行，可能导致过宽词间距

3. **Badness 参数调优**
   - 惩罚值（riverPenalty: 5000, tightPenalty: 3000, hyphenPenalty: 50）是经验值
   - 不同语言/字体可能需要调整

### 边界情况处理

| 边界情况 | 处理方式 |
|---------|---------|
| 单词长度 > 最大宽度 | 在字素边界强制断行（由 `layoutNextLine` 处理） |
| 无空格行 | 使用 `slack * slack * 10` 惩罚 |
| 空段落 | 返回空数组 `[]` |
| 动态规划无解 | 理论上不会发生（至少可以逐字断行） |

### 改进建议

1. **性能优化**
   - 实现断点图剪枝（只保留可行断点）
   - 使用 A* 或 Dijkstra 替代完整 DP
   - 缓存段落布局结果（如果宽度未变）

2. **算法增强**
   - 添加字间距（letter-spacing）调整
   - 实现悬挂标点（hanging punctuation）
   - 支持多栏布局的跨栏优化

3. **连字符改进**
   - 集成 libhyphen 或类似库
   - 支持自定义例外词典
   - 支持多语言 pattern 文件

4. **更多指标**
   - 添加相邻行词间距差异度量
   - 计算排版密度（灰度）
   - 添加连字符频率统计

5. **可视化增强**
   - 显示断点图（force graph）
   - 高亮选中的断行路径
   - 显示每行的 badness 分解

6. **参数可调**
   - 暴露 badness 权重为 UI 控件
   - 允许用户调整河流阈值
   - 支持自定义连字符惩罚

### 数学细节

**Badness 公式推导**：

TeX 的 badness 公式为：
```
badness = 100 * |r|³
where r = (actual_space - ideal_space) / ideal_space
```

本实现使用：
```
badness = |r|³ * 1000
```

差异在于缩放因子，不影响相对排序。

**动态规划状态转移**：

```
dp[i] = min(dp[j] + cost(j, i)) for all j < i
where cost(j, i) = badness of line from break j to break i
```

时间复杂度：O(m²) where m = number of break candidates
空间复杂度：O(m)
