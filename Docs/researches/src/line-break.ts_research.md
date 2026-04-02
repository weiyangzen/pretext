# src/line-break.ts 研究文档

## 场景与职责

`line-break.ts` 是 Pretext 库的核心换行算法模块，负责将经过预处理的文本数据（`PreparedLineBreakData`）在指定最大宽度下进行行分割计算。该模块实现了与浏览器 CSS `white-space: normal` 和 `overflow-wrap: break-word` 行为一致的换行逻辑。

**核心职责：**
1. 实现高性能的纯算术换行计算（无 DOM 读取、无 Canvas 调用）
2. 支持两种模式：简单快速路径（`simpleLineWalkFastPath`）和完整功能路径
3. 处理各种换行机会点：空格、制表符、零宽断点、软连字符等
4. 支持在字素（grapheme）级别进行长单词断行
5. 提供行计数、行遍历、单行布局等多种 API

**在架构中的位置：**
- 被 `layout.ts` 调用，是 `layout()` / `layoutWithLines()` / `layoutNextLine()` 的底层实现
- 依赖 `measurement.ts` 提供的引擎配置（`EngineProfile`）
- 依赖 `analysis.ts` 定义的段类型（`SegmentBreakKind`）

---

## 功能点目的

### 1. 行计数与行遍历

| 函数 | 目的 |
|------|------|
| `countPreparedLines()` | 快速计算给定宽度下的行数，用于 `layout()` 热路径 |
| `walkPreparedLines()` | 遍历所有行并可选回调每行信息，用于 `layoutWithLines()` |
| `layoutNextLineRange()` | 从指定位置计算下一行范围，用于流式布局 `layoutNextLine()` |

### 2. 换行机会点处理

支持八种段中断类型（`SegmentBreakKind`）：
- `'text'` - 普通文本，非断点
- `'space'` - 可折叠空格，断点后不显示
- `'preserved-space'` - 保留空格（pre-wrap 模式）
- `'tab'` - 制表符，按 tab stop 对齐
- `'glue'` - 非断行胶合（NBSP, NNBSP, WJ 等）
- `'zero-width-break'` - 零宽断点（ZWSP）
- `'soft-hyphen'` - 软连字符，断行时显示连字符
- `'hard-break'` - 强制换行（pre-wrap 模式的 `\n`）

### 3. 长单词字素级断行

当单词宽度超过最大宽度时，在字素边界处断行：
- 使用预计算的 `breakableWidths`（各字素宽度数组）
- 支持 `breakablePrefixWidths` 前缀累积宽度（Safari 兼容）
- 软连字符断行时添加 `discretionaryHyphenWidth`

### 4. 双路径架构

**简单路径（`walkPreparedLinesSimple`）：**
- 适用于无特殊段类型的普通文本
- 减少分支判断，提高性能
- 不支持 tab、软连字符、强制换行等

**完整路径（`walkPreparedLines`）：**
- 支持所有段类型和复杂场景
- 处理 chunk（硬换行分隔的文本块）
- 支持软连字符的优先/延迟断行策略

---

## 具体技术实现

### 关键数据结构

```typescript
// 行断点游标，精确到段和字素索引
type LineBreakCursor = {
  segmentIndex: number
  graphemeIndex: number
}

// 预处理后的换行数据（由 layout.ts 准备）
type PreparedLineBreakData = {
  widths: number[]                    // 各段宽度
  lineEndFitAdvances: number[]        // 行尾适配进宽（空格为0）
  lineEndPaintAdvances: number[]      // 行尾绘制宽度（保留空格宽度）
  kinds: SegmentBreakKind[]           // 段类型
  simpleLineWalkFastPath: boolean     // 是否可用简单路径
  breakableWidths: (number[] | null)[]      // 可断行段的字素宽度
  breakablePrefixWidths: (number[] | null)[] // 前缀累积宽度
  discretionaryHyphenWidth: number    // 软连字符断行时添加的宽度
  tabStopAdvance: number              // 制表位步进
  chunks: {                           // 硬换行分隔的块
    startSegmentIndex: number
    endSegmentIndex: number
    consumedEndSegmentIndex: number
  }[]
}

// 内部布局行（输出）
type InternalLayoutLine = {
  startSegmentIndex: number
  startGraphemeIndex: number
  endSegmentIndex: number
  endGraphemeIndex: number
  width: number
}
```

### 核心算法流程

#### 1. 简单路径行遍历（`walkPreparedLinesSimple`）

```
初始化: lineW=0, hasContent=false, pendingBreak=-1

对于每个段 i:
  如果无内容:
    跳过前导可折叠段（空格、ZWSP、软连字符）
    
  如果段宽 > maxWidth 且有 breakableWidths:
    // 长单词字素级断行
    逐字素尝试添加，溢出时换行
  
  如果 lineW + width > maxWidth + epsilon:
    如果有待断点 pendingBreak:
      回退到 pendingBreak 处换行
    否则:
      强制换行（当前段移至下一行）
  
  否则:
    添加段到当前行
    如果可以断行后: 更新 pendingBreak
```

#### 2. 完整路径行遍历（`walkPreparedLines`）

按 chunk 处理（支持 pre-wrap 硬换行）：

```
对于每个 chunk:
  重置行状态
  对于 chunk 内的每个段:
    处理软连字符特殊逻辑
    处理 tab 特殊宽度计算
    尝试添加段:
      如果溢出:
        检查软连字符优先断行（Safari 策略）
        检查软连字符延续断行
        检查当前段是否可断行
        检查 pendingBreak 是否可用
        检查长单词字素断行
        否则强制换行
    更新 pendingBreak
  如果有内容: 发射最后一行
```

#### 3. 软连字符断行算法（`fitSoftHyphenBreak`）

```typescript
function fitSoftHyphenBreak(
  graphemeWidths: number[],
  initialWidth: number,
  maxWidth: number,
  lineFitEpsilon: number,
  discretionaryHyphenWidth: number,
  cumulativeWidths: boolean,
): { fitCount: number, fittedWidth: number }
```

逻辑：
- 从当前行宽开始，逐字素尝试添加
- 每添加一个字素，检查加上 `discretionaryHyphenWidth` 后是否溢出
- 返回能容纳的字素数量和最终行宽
- 用于软连字符后的长单词延续断行

#### 4. 制表符处理（`getTabAdvance`）

```typescript
function getTabAdvance(lineWidth: number, tabStopAdvance: number): number {
  if (tabStopAdvance <= 0) return 0
  const remainder = lineWidth % tabStopAdvance
  if (Math.abs(remainder) <= 1e-6) return tabStopAdvance
  return tabStopAdvance - remainder
}
```

- 计算到下一个 tab stop 的距离
- 默认 tab stop 为 8 个空格宽度

### 关键代码路径

#### 行起始规范化（`normalizeLineStart` / `normalizeLineStartWithChunk`）

```typescript
// 简单路径：跳过前导可折叠段
function normalizeSimpleLineStartSegmentIndex(...)

// 完整路径：考虑 chunk 边界
function normalizeLineStartWithChunk(...)
```

作用：在从指定游标开始布局时，跳过前导空格/ZWSP/软连字符，确保行起始位置正确。

#### 可断行段宽度获取（`getBreakableAdvance`）

```typescript
function getBreakableAdvance(
  graphemeWidths: number[],
  graphemePrefixWidths: number[] | null,
  graphemeIndex: number,
  preferPrefixWidths: boolean,
): number
```

- 根据引擎配置选择使用直接宽度或前缀差分宽度
- Safari 使用前缀宽度策略（`preferPrefixWidthsForBreakableRuns: true`）

#### 软连字符延续断行（`continueSoftHyphenBreakableSegment`）

复杂逻辑处理：
1. 检查前一个断点是否为软连字符
2. 尝试在当前段内继续填充字素
3. 如果全部容纳：清除 pendingBreak，继续
4. 如果部分容纳：发射带连字符的行，剩余部分开启新行

---

## 依赖与外部交互

### 导入依赖

```typescript
import type { SegmentBreakKind } from './analysis.js'
import { getEngineProfile } from './measurement.js'
```

### 被调用方

| 调用方 | 调用函数 | 用途 |
|--------|----------|------|
| `layout.ts` | `countPreparedLines` | `layout()` 计算行数 |
| `layout.ts` | `walkPreparedLines` | `layoutWithLines()` 遍历行 |
| `layout.ts` | `layoutNextLineRange` | `layoutNextLine()` 流式布局 |

### 引擎配置依赖

```typescript
const engineProfile = getEngineProfile()
// lineFitEpsilon: 行宽容差（Safari 1/64，其他 0.005）
// preferPrefixWidthsForBreakableRuns: Safari 使用前缀宽度
// preferEarlySoftHyphenBreak: Safari 优先在软连字符处断行
// carryCJKAfterClosingQuote: Chromium 在引号后携带 CJK
```

---

## 风险、边界与改进建议

### 已知风险

1. **行宽容差敏感**
   - Safari 使用 `1/64`（约 0.0156）的容差，其他浏览器使用 `0.005`
   - 容差选择影响跨浏览器一致性，AGENTS.md 提到这是待决策问题

2. **软连字符复杂逻辑**
   - `continueSoftHyphenBreakableSegment` 和 `maybeFinishAtSoftHyphen` 逻辑复杂
   - 涉及多个状态变量（`pendingBreakKind`, `pendingBreakFitWidth` 等）
   - 容易在边界条件下出现不一致

3. **双路径维护成本**
   - 简单路径和完整路径逻辑相似但细节不同
   - 修改时需要同步更新两处，容易遗漏

4. **Chunk 边界处理**
   - `normalizeLineStartWithChunk` 使用二分查找定位 chunk
   - 边界条件下（空 chunk、越界访问）需要仔细验证

### 边界情况

| 场景 | 处理 |
|------|------|
| 空文本 | 直接返回 0 行 |
| 全空格文本 | 简单路径跳过所有段，返回 0 行；pre-wrap 模式保留 |
| 超长单词 | 在字素边界断行，可能每行一个字素 |
| 软连字符在段首 | 特殊处理，不显示连字符 |
| Tab 在空行 | 按 tab stop 计算宽度 |
| 连续硬换行 | 生成空行（pre-wrap 模式） |

### 改进建议

1. **统一路径**
   - 考虑通过编译时或运行时代码生成减少双路径维护成本
   - 或者提取更细粒度的可复用函数

2. **软连字符重构**
   - 将软连字符逻辑提取为独立的状态机
   - 增加更多边界条件的单元测试

3. **性能优化**
   - `walkPreparedLines` 中的 chunk 遍历可以进一步优化
   - 考虑使用迭代器模式减少回调开销

4. **文档完善**
   - 增加更多内联注释说明复杂逻辑
   - 绘制状态转换图说明软连字符处理

5. **容差校准**
   - 根据 AGENTS.md 建议，考虑运行时校准替代硬编码容差
   - 与 emoji 校正机制类似，通过实际测量确定

### 测试覆盖

测试主要位于 `src/layout.test.ts`，覆盖：
- 基本换行逻辑
- 软连字符处理
- pre-wrap 模式
- 长单词字素断行
- `layoutNextLine` 流式布局
- `walkLineRanges` 非物化遍历

建议增加：
- 极端宽度（接近 0 或极大值）
- 复杂混合文本（CJK + RTL + emoji）
- 并发/重入场景
