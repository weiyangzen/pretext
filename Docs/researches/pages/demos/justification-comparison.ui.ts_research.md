# justification-comparison.ui.ts 研究文档

## 场景与职责

`justification-comparison.ui.ts` 是 Justification Comparison 演示的视图层模块，负责所有 DOM 操作和渲染逻辑：

1. **DOM 元素管理**：缓存和创建 DOM 元素
2. **Canvas 渲染**：绘制 Pretext 布局结果到 Canvas
3. **CSS 栏渲染**：设置 DOM 文本内容
4. **河流指示器**：可视化展示词间距问题区域
5. **指标面板更新**：显示排版质量数据

该模块是纯视图层，无业务逻辑，可被控制器（`.ts`）调用。

## 功能点目的

### 1. Canvas 渲染

将 Pretext 布局结果绘制到 Canvas，支持：
- 左对齐（ragged）文本
- 两端对齐（justified）文本
- 河流高亮（红色背景）

### 2. CSS 栏渲染

- 设置 DOM 文本内容
- 同步列宽
- 更新控制控件状态

### 3. 河流指示器可视化

两种实现方式：
- **Canvas 栏**：直接在 Canvas 上绘制彩色矩形
- **CSS 栏**：使用绝对定位的 div 覆盖层

### 4. 指标面板

显示每栏的排版质量数据：
- 行数
- 平均词间距偏差
- 最大词间距偏差
- 河流空间数量

## 具体技术实现

### DOM 缓存结构

```typescript
export type DomCache = {
  // 控制元素
  slider: HTMLInputElement
  showIndicators: HTMLInputElement
  widthValue: HTMLElement
  
  // 布局元素
  columns: HTMLElement[]
  cssCol: HTMLElement
  cssRiverOverlay: HTMLElement
  cssParagraphs: HTMLParagraphElement[]
  cssRange: Range
  cssRiverMarks: HTMLDivElement[]
  
  // Canvas
  hyphenCanvas: CanvasSurface
  optimalCanvas: CanvasSurface
  
  // 指标面板
  metrics: {
    css: MetricPanelCache
    hyphen: MetricPanelCache
    optimal: MetricPanelCache
  }
}
```

### Canvas 渲染流程

```typescript
function paintCanvasColumn(
  surface: CanvasSurface,
  frame: CanvasColumnFrame,
  showIndicators: boolean,
  normalSpaceWidth: number
): void
```

步骤：
1. 设置 Canvas 尺寸（考虑 DPR）
2. 填充白色背景
3. 设置裁剪区域
4. 设置字体
5. 遍历段落和行，调用 `paintLine`

### 行绘制

```typescript
function paintLine(
  ctx: CanvasRenderingContext2D,
  line: PositionedLine,
  showIndicators: boolean,
  normalSpaceWidth: number
): void
```

根据 `line.spacing.kind` 分支处理：

**A. 左对齐 (ragged)**：
```typescript
case 'ragged':
case 'overflow':
  ctx.fillStyle = '#2a2520'
  for (const segment of line.segments) {
    if (segment.kind === 'space') {
      x += segment.width
    } else {
      ctx.fillText(segment.text, x, line.y)
      x += segment.width
    }
  }
```

**B. 两端对齐 (justified)**：
```typescript
case 'justified':
  for (const segment of line.segments) {
    if (segment.kind === 'space') {
      // 绘制河流指示器
      if (showIndicators && line.spacing.isRiver) {
        ctx.fillStyle = toRgba(indicator)
        ctx.fillRect(x + 1, line.y, line.spacing.width - 2, LINE_HEIGHT)
      }
      x += line.spacing.width  // 使用调整后的词间距
    } else {
      ctx.fillStyle = '#2a2520'
      ctx.fillText(segment.text, x, line.y)
      x += segment.width
    }
  }
```

### CSS 栏河流指示器

```typescript
export function syncCssRiverOverlay(
  dom: DomCache,
  controls: DemoControls,
  normalSpaceWidth: number
): void
```

实现原理：
1. 获取 CSS 栏的边界矩形（`getBoundingClientRect`）
2. 遍历每个段落的文本节点
3. 对每个空格字符：
   - 创建 `Range` 包围该空格
   - 调用 `getClientRects()` 获取实际渲染位置
   - 计算相对于覆盖层的坐标
   - 如果空格宽度超过阈值，创建彩色标记

```typescript
for (let charIndex = 0; charIndex < text.length; charIndex++) {
  if (text[charIndex] !== ' ') continue
  
  dom.cssRange.setStart(textNode, charIndex)
  dom.cssRange.setEnd(textNode, charIndex + 1)
  const rects = dom.cssRange.getClientRects()
  if (rects.length !== 1) continue
  
  const rect = rects[0]!
  const indicator = getRiverIndicator(rect.width, normalSpaceWidth)
  if (indicator === null) continue
  
  // 创建或更新标记元素
  riverMarks.push({
    left: rect.left - overlayRect.left,
    top: rect.top - overlayRect.top,
    width: rect.width,
    color: toRgba(indicator),
  })
}
```

### 对象池模式

河流标记使用对象池避免频繁创建/销毁：

```typescript
function ensureRiverMarkCount(
  marks: HTMLDivElement[],
  overlay: HTMLElement,
  count: number
): void {
  while (marks.length < count) {
    const mark = document.createElement('div')
    mark.style.position = 'absolute'
    mark.style.pointerEvents = 'none'
    mark.style.height = `${LINE_HEIGHT}px`
    overlay.appendChild(mark)
    marks.push(mark)
  }
}

function hideUnusedRiverMarks(marks: HTMLDivElement[], fromIndex: number): void {
  for (let i = fromIndex; i < marks.length; i++) {
    marks[i]!.style.display = 'none'
  }
}
```

### 指标面板

```typescript
function createMetricPanel(container: HTMLElement): MetricPanelCache
```

创建四行指标：
```
Lines           [值]
Avg deviation   [值]  (颜色编码)
Max deviation   [值]  (颜色编码)
River spaces    [值]  (颜色编码)
```

质量等级颜色：
```typescript
function qualityClass(avgDeviation: number): 'good' | 'ok' | 'bad' {
  if (avgDeviation < 0.15) return 'good'  // 绿色
  if (avgDeviation < 0.35) return 'ok'    // 黄色
  return 'bad'                             // 红色
}
```

### DPR 处理

```typescript
function setupCanvas(surface: CanvasSurface, width: number, height: number): void {
  const dpr = devicePixelRatio || 1
  surface.element.width = width * dpr      // 实际像素尺寸
  surface.element.height = height * dpr
  surface.element.style.width = `${width}px`   // CSS 尺寸
  surface.element.style.height = `${height}px`
  surface.ctx.setTransform(dpr, 0, 0, dpr, 0, 0)  // 缩放上下文
}
```

确保在高 DPR 屏幕（Retina）上渲染清晰。

## 关键代码路径与文件引用

### 依赖导入

```typescript
import { FONT, LINE_HEIGHT, PAD, PARA_GAP, PARAGRAPHS } from './justification-comparison.data.ts'
import {
  type CanvasColumnFrame,
  getRiverIndicator,
  type DemoControls,
  type DemoFrame,
  type PositionedLine,
  type QualityMetrics,
  type RiverIndicator,
} from './justification-comparison.model.ts'
```

### 导出 API

```typescript
// DOM 缓存创建
export function createDomCache(): DomCache

// 帧渲染
export function renderFrame(
  dom: DomCache,
  frame: DemoFrame,
  normalSpaceWidth: number
): void

// CSS 河流覆盖层同步
export function syncCssRiverOverlay(
  dom: DomCache,
  controls: DemoControls,
  normalSpaceWidth: number
): void
```

### 调用链

```
renderFrame()
  ├── applyControls()           // 更新滑块和开关
  ├── applyColumnWidths()       // 设置列宽
  ├── updateMetricPanel() × 3   // 更新指标面板
  ├── paintCanvasColumn() × 2   // 绘制 Canvas
  │   ├── setupCanvas()         // 设置 DPR
  │   └── paintLine() × n       // 绘制每行
  └── syncCssRiverOverlay()     // 同步 CSS 河流指示器
      └── 使用 Range API 测量空格
```

## 依赖与外部交互

### 外部依赖

| 依赖 | 用途 |
|------|------|
| `justification-comparison.data.ts` | 排版常量（FONT, LINE_HEIGHT 等） |
| `justification-comparison.model.ts` | 类型定义、河流指示器计算 |
| Canvas 2D API | 渲染 Pretext 布局结果 |
| Range API | 测量 CSS 栏空格位置 |
| getBoundingClientRect | 获取元素位置 |

### DOM 操作

```typescript
// 元素创建
document.createElement('div')
document.createElement('span')
document.createElement('canvas')
document.createRange()

// 属性设置
element.style.left = '10px'
element.style.width = '100px'
element.textContent = 'text'

// 元素查询
document.getElementById('id')
document.querySelectorAll('.column')

// 元素操作
parent.replaceChildren(...children)
parent.appendChild(child)
```

## 风险、边界与改进建议

### 已知风险

1. **Range API 兼容性**
   - `Range.getClientRects()` 对空格的返回在不同浏览器可能有差异
   - 某些浏览器可能返回多个矩形（跨行情况）

2. **DPR 变化**
   - 如果用户移动窗口到不同 DPR 的屏幕，Canvas 可能模糊
   - 当前只在初始化时读取 `devicePixelRatio`

3. **大量河流标记**
   - 如果文本有很多过宽空格，可能创建大量 DOM 元素
   - 虽然使用对象池，但仍可能影响性能

4. **字体不匹配**
   - Canvas 和 DOM 使用相同字体声明，但实际渲染可能略有差异
   - 特别是在字体加载完成前开始渲染

### 边界情况处理

| 边界情况 | 处理方式 |
|---------|---------|
| Range 返回多个矩形 | 跳过（`rects.length !== 1`） |
| 空格宽度 < 1px | 跳过（`rect.width < 1`） |
| 河流指示器关闭 | 隐藏所有标记（`hideUnusedRiverMarks(..., 0)`） |
| Canvas 上下文获取失败 | 抛出错误（`throw new Error(...)`） |

### 改进建议

1. **DPR 变化监听**
   ```typescript
   window.addEventListener('resize', () => {
     if (window.devicePixelRatio !== lastDpr) {
       lastDpr = window.devicePixelRatio
       scheduleRender()
     }
   })
   ```

2. **Canvas 渲染优化**
   ```typescript
   // 使用 willReadFrequently 提示（如果需要读取像素）
   const ctx = canvas.getContext('2d', { willReadFrequently: false })
   
   // 使用 offscreen canvas 进行双缓冲（复杂场景）
   ```

3. **Range API 降级**
   ```typescript
   // 如果 Range API 返回异常结果，使用估算值
   if (rects.length === 0) {
     // 使用字符索引估算位置
   }
   ```

4. **虚拟滚动（长文本）**
   ```typescript
   // 如果段落很长，只渲染可见区域
   const visibleRange = getVisibleRange()
   for (let i = visibleRange.start; i < visibleRange.end; i++) {
     paintLine(...)
   }
   ```

5. **CSS 容器查询**
   ```typescript
   // 使用 ResizeObserver 替代 window resize
   const observer = new ResizeObserver(entries => {
     scheduleRender()
   })
   observer.observe(dom.cssCol)
   ```

6. **河流标记优化**
   ```typescript
   // 合并相邻的河流标记（如果间距很小）
   function mergeAdjacentMarks(marks: RiverMark[]): RiverMark[]
   ```

7. **可访问性**
   ```typescript
   // 为 Canvas 添加 ARIA 标签
   canvas.setAttribute('role', 'img')
   canvas.setAttribute('aria-label', '排版布局可视化')
   
   // 提供文本替代
   const altText = generateAltText(frame)
   ```

8. **性能监控**
   ```typescript
   function paintCanvasColumn(...) {
     const start = performance.now()
     // ... 绘制逻辑
     const duration = performance.now() - start
     if (duration > 16) {
       console.warn(`Slow canvas render: ${duration.toFixed(2)}ms`)
     }
   }
   ```

### 维护注意事项

- 修改颜色值时，需同步更新 CSS 和 Canvas 渲染代码
- 添加新指标时，需要在 `createMetricPanel` 和 `updateMetricPanel` 中同步更新
- Canvas 字体必须与 CSS 字体完全一致，否则度量会不匹配
- 河流指示器的视觉样式（颜色、透明度）在 `getRiverIndicator`（model.ts）和 `toRgba` 中定义
