# editorial-engine.ts 研究文档

## 场景与职责

`editorial-engine.ts` 是 Pretext 库最复杂的交互式演示页面，展示了一个完整的编辑排版引擎能力。该页面实现了一个动态、多列的杂志风格排版布局，包含以下核心特性：

1. **多列文本流**：支持 1-3 列自适应布局，文本在列之间无缝衔接
2. **动态障碍物避让**：文本围绕移动的圆形障碍物（发光球体）自动重新排版
3. **首字下沉（Drop Cap）**：第一列开头的大号首字母装饰
4. **引言块（Pull Quotes）**：浮动在列中的引用文本框
5. **自适应标题**：通过二分搜索动态调整标题字号以适应可用空间
6. **交互式拖拽**：用户可以拖拽球体或暂停其动画

该文件是 `dynamic-layout.ts` 的演进版本，展示了 Pretext 在复杂排版场景下的性能优势——60fps 的实时文本重排，零 DOM 测量操作。

## 功能点目的

### 1. 实时动态布局演示
通过动画球体作为障碍物，展示 Pretext 如何在每一帧快速计算文本布局，实现 CSS Shapes 无法做到的复杂环绕效果。

### 2. 多列文本流
实现杂志风格的多列布局，文本从一列自然流动到下一列，使用 `layoutNextLine` API 实现逐行布局。

### 3. 障碍物避让算法
计算每行文本可用的水平区间（slots），处理圆形和矩形障碍物，支持文本在障碍物两侧同时环绕。

### 4. 自适应排版
- **标题适配**：二分搜索最优字号，确保标题不换行
- **响应式布局**：根据视口宽度自动调整列数、间距和球体数量

### 5. 交互系统
- 鼠标/触摸拖拽球体
- 点击暂停/恢复球体动画
- 文本选择模式切换

## 具体技术实现

### 核心数据结构

```typescript
// 定位的行（包含位置信息）
type PositionedLine = {
  x: number      // 水平位置
  y: number      // 垂直位置
  width: number  // 行宽度
  text: string   // 行文本内容
}

// 文本投影（完整布局状态）
type TextProjection = {
  headlineLeft: number
  headlineTop: number
  headlineFont: string
  headlineLineHeight: number
  headlineLines: PositionedLine[]
  bodyFont: string
  bodyLineHeight: number
  bodyLines: PositionedLine[]
  pullquoteFont: string
  pullquoteLineHeight: number
  pullquoteLines: PositionedLine[]
}

// 圆形障碍物
type CircleObstacle = {
  cx: number, cy: number, r: number  // 圆心和半径
  hPad: number, vPad: number          // 水平和垂直内边距
}

// 矩形障碍物
type RectObstacle = { x: number, y: number, w: number, h: number }

// 引言块配置
type PullquotePlacement = {
  colIdx: number   // 所在列索引
  yFrac: number    // 垂直位置（百分比）
  wFrac: number    // 宽度（百分比）
  side: 'left' | 'right'
}
```

### 关键算法流程

#### 1. 障碍物区间计算
```typescript
function circleIntervalForBand(
  cx: number, cy: number, r: number,
  bandTop: number, bandBottom: number,
  hPad: number, vPad: number
): Interval | null
```
计算圆形障碍物与文本行垂直带的交集，返回被阻挡的水平区间。

数学原理：
- 如果垂直带与圆无交集，返回 null
- 计算垂直距离 `minDy`
- 使用勾股定理计算最大水平偏移 `maxDx = sqrt(r² - minDy²)`
- 返回被阻挡的区间 `[cx - maxDx - hPad, cx + maxDx + hPad]`

#### 2. 可用区间计算（槽位切割）
```typescript
function carveTextLineSlots(base: Interval, blocked: Interval[]): Interval[]
```
从基础区间中减去所有被阻挡的区间，得到文本可以放置的槽位。

算法步骤：
1. 初始化槽位列表为 `[base]`
2. 遍历每个被阻挡的区间：
   - 对于每个现有槽位，检查是否与阻挡区间重叠
   - 如果重叠，将槽位分割为左侧和/或右侧剩余部分
3. 过滤掉宽度小于 `MIN_SLOT_WIDTH` (50px) 的槽位

#### 3. 单列布局
```typescript
function layoutColumn(
  prepared: PreparedTextWithSegments,
  startCursor: LayoutCursor,
  regionX: number, regionY: number,
  regionW: number, regionH: number,
  lineHeight: number,
  circleObstacles: CircleObstacle[],
  rectObstacles: RectObstacle[],
  singleSlotOnly: boolean = false
): { lines: PositionedLine[], cursor: LayoutCursor }
```
核心布局循环：
1. 从 `startCursor` 开始逐行布局
2. 对每行计算垂直带（`bandTop` 到 `bandBottom`）
3. 收集与该带相交的所有障碍物区间
4. 计算可用槽位
5. 对每个槽位调用 `layoutNextLine` 填充文本
6. 返回填充的行和最终光标位置

#### 4. 自适应标题适配
```typescript
function fitHeadline(maxWidth: number, maxHeight: number, maxSize: number = 92): HeadlineFit
```
使用二分搜索找到最大字号，使得：
- 标题不会跨词换行（`breaksWord === false`）
- 总高度不超过 `maxHeight`

缓存机制：使用 `cachedHeadlineWidth/Height/MaxSize` 避免重复计算

#### 5. 动画循环
```typescript
function render(now: number): boolean
```
每帧执行：
1. 处理指针事件（拖拽、点击）
2. 更新球体位置（物理模拟：速度、边界反弹、球体间排斥）
3. 构建障碍物列表
4. 计算标题布局
5. 计算引言块位置
6. 布局所有列的文本
7. 同步 DOM 元素（使用对象池避免创建/销毁）
8. 返回 `stillAnimating` 控制 RAF 继续或暂停

### 物理模拟

球体运动使用简单的欧拉积分：
```typescript
orb.x += orb.vx * dt
orb.y += orb.vy * dt
```

边界碰撞检测和反弹：
```typescript
if (orb.x - radius < 0) {
  orb.x = radius
  orb.vx = Math.abs(orb.vx)  // 向右反弹
}
```

球体间排斥力（软碰撞）：
```typescript
const force = (minDist - dist) * 0.8
const nx = dx / dist  // 法线方向
const ny = dy / dist
// 应用冲量
a.vx -= nx * force * dt
b.vx += nx * force * dt
```

### DOM 对象池

为避免频繁的 DOM 操作，使用对象池缓存：
```typescript
const domCache = {
  stage: HTMLDivElement           // 根容器
  dropCap: HTMLDivElement         // 首字下沉元素
  bodyLines: HTMLSpanElement[]    // 正文行池
  headlineLines: HTMLSpanElement[] // 标题行池
  pullquoteLines: HTMLSpanElement[] // 引言行池
  pullquoteBoxes: HTMLDivElement[] // 引言框池
  orbs: HTMLDivElement[]          // 球体元素池
}
```

`syncPool` 函数管理池大小：
- 如果池太小，创建新元素
- 如果池太大，隐藏多余元素

### 文本投影比较

为避免不必要的 DOM 更新，使用 `textProjectionEqual` 比较新旧投影：
```typescript
if (!textProjectionEqual(committedTextProjection, textProjection)) {
  projectTextProjection(textProjection)
  committedTextProjection = textProjection
}
```

## 关键代码路径与文件引用

### 依赖导入
```typescript
import {
  layoutNextLine,
  layoutWithLines,
  prepareWithSegments,
  walkLineRanges,
  type LayoutCursor,
  type PreparedTextWithSegments,
} from '../../src/layout.ts'
```

### 核心 API 使用

1. **文本预处理**（初始化时）：
   ```typescript
   const preparedBody = prepareWithSegments(BODY_TEXT, BODY_FONT)
   const preparedPullquotes = PULLQUOTE_TEXTS.map(text => 
     prepareWithSegments(text, PQ_FONT)
   )
   ```

2. **首字下沉宽度测量**：
   ```typescript
   walkLineRanges(preparedDropCap, 9999, line => {
     dropCapWidth = line.width
   })
   ```

3. **标题适配**：
   ```typescript
   const prepared = prepareWithSegments(HEADLINE_TEXT, font)
   walkLineRanges(prepared, maxWidth, line => {
     if (line.end.graphemeIndex !== 0) breaksWord = true
   })
   ```

4. **逐行布局**（列布局核心）：
   ```typescript
   const line = layoutNextLine(prepared, cursor, slotWidth)
   if (line === null) { textExhausted = true; break }
   lines.push({
     x: Math.round(slot.left),
     y: Math.round(lineTop),
     text: line.text,
     width: line.width,
   })
   cursor = line.end
   ```

5. **完整段落布局**（引言块）：
   ```typescript
   const pullquoteLines = layoutWithLines(prepared, pullquoteWidth - 20, PQ_LINE_HEIGHT).lines
   ```

### 响应式断点

```typescript
const NARROW_BREAKPOINT = 760  // 窄屏阈值
const columnCount = pageWidth > 1000 ? 3 : pageWidth > 640 ? 2 : 1
```

窄屏适配：
- 减少球体数量（5 → 3）
- 缩小球体尺寸（scale 0.58）
- 减小间距和边距
- 降低标题最大字号（92 → 38）

## 依赖与外部交互

### 外部依赖

| 依赖 | 用途 |
|------|------|
| `../../src/layout.ts` | Pretext 核心 API：prepareWithSegments, layoutNextLine, layoutWithLines, walkLineRanges |
| `document.fonts.ready` | 等待字体加载完成 |
| `requestAnimationFrame` | 动画循环调度 |
| `performance.now()` | 时间戳计算（用于 dt） |

### DOM 事件监听

```typescript
stage.addEventListener('pointerdown', handler)      // 开始拖拽
window.addEventListener('pointermove', handler)     // 拖拽中
window.addEventListener('pointerup', handler)       // 结束拖拽
window.addEventListener('resize', scheduleRender)   // 窗口调整
document.addEventListener('selectionchange', handler) // 文本选择
```

### CSS 类名约定

生成的 DOM 元素使用以下 CSS 类：
- `.orb` - 发光球体
- `.drop-cap` - 首字下沉
- `.line` - 正文行
- `.headline-line` - 标题行
- `.pullquote-line` - 引言行
- `.pullquote-box` - 引言框背景

## 风险、边界与改进建议

### 已知风险

1. **字体加载依赖**
   - 代码等待 `document.fonts.ready`，但如果字体加载失败或超时，布局可能基于错误字体度量
   - 建议：添加字体加载超时处理

2. **性能边界**
   - 虽然 60fps 在大多数情况下可维持，但极端情况（大量障碍物 + 极窄列宽）可能导致帧率下降
   - 当前障碍物检测是 O(n_lines × n_obstacles) 复杂度

3. **触摸交互冲突**
   - 文本选择和球体拖拽在触摸设备上存在手势冲突
   - 当前通过 `interactionMode` 状态机管理，但用户体验可能不够直观

4. **硬编码常量**
   - 大量布局常量（边距、字号、行高等）硬编码在文件中
   - 修改主题需要修改源代码

### 边界情况处理

| 边界情况 | 处理方式 |
|---------|---------|
| 文本耗尽 | `layoutNextLine` 返回 null，设置 `textExhausted = true` |
| 无可用槽位 | 跳过该行，继续下一行 |
| 球体被拖出视口 | 边界反弹约束 |
| 窗口极窄 | 降至单列，减少球体数量 |
| 文本选择激活 | 暂停动画循环，恢复原生选择行为 |

### 改进建议

1. **性能优化**
   - 使用空间分割（如四叉树）加速障碍物查询
   - 对静态布局（无动画时）使用缓存避免重复计算

2. **可配置性**
   - 将设计常量提取为配置对象或 CSS 变量
   - 支持主题切换

3. **可访问性**
   - 添加键盘导航支持（Tab 切换球体，方向键移动）
   - 为球体添加 ARIA 标签
   - 支持减少动画偏好（`prefers-reduced-motion`）

4. **代码组织**
   - 将物理模拟提取为独立模块
   - 将响应式逻辑提取为可复用的断点系统

5. **调试支持**
   - 添加可视化调试模式（显示槽位边界、光标位置等）
   - 暴露性能指标（布局耗时、FPS）

### 浏览器兼容性

- 依赖 `Intl.Segmenter`（现代浏览器支持）
- 使用 CSS `color-mix`（较新特性，但已广泛支持）
- 触摸事件处理可能需要针对旧版 iOS 调整
