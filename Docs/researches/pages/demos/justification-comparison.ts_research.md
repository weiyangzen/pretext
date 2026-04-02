# justification-comparison.ts 研究文档

## 场景与职责

`justification-comparison.ts` 是 Justification Comparison 演示的入口模块和控制器，负责：

1. **状态管理**：维护当前列宽和河流指示器开关状态
2. **事件处理**：监听滑块输入、复选框变化和窗口调整事件
3. **渲染调度**：协调模型计算和 UI 渲染，使用 requestAnimationFrame 优化性能
4. **CSS 覆盖层同步**：延迟同步 CSS 栏的河流指示器（需要等待 DOM 布局完成）

该模块采用简单的 MVC 模式，将业务逻辑委托给 `.model.ts`，DOM 操作委托给 `.ui.ts`。

## 功能点目的

### 1. 用户交互响应

- **宽度滑块**：实时调整三栏的列宽，观察不同宽度下的算法表现
- **河流开关**：切换红色高亮指示器，直观展示词间距问题区域
- **窗口调整**：响应浏览器窗口大小变化，保持布局适应性

### 2. 性能优化

- **RAF 节流**：使用 `requestAnimationFrame` 避免频繁重绘
- **延迟同步**：CSS 栏的河流指示器需要等待浏览器完成布局，延迟一帧执行
- **事件合并**：同一帧内的多个事件只触发一次渲染

### 3. 关注点分离

| 模块 | 职责 |
|-----|------|
| `.ts` (本文件) | 事件处理、状态管理、渲染调度 |
| `.model.ts` | 布局算法、数据计算 |
| `.ui.ts` | DOM 操作、Canvas 绘制 |
| `.data.ts` | 静态数据、常量 |

## 具体技术实现

### 状态结构

```typescript
type State = {
  controls: DemoControls        // 当前控制值
  events: {
    widthInput: number | null   // 待处理的宽度输入
    showIndicatorsInput: boolean | null  // 待处理的开关输入
  }
}
```

使用"待处理事件"模式允许在一帧内合并多个输入事件。

### DOM 缓存

```typescript
const dom = createDomCache()
```

在初始化时一次性获取所有 DOM 元素引用，避免运行时查询。

### 事件监听

```typescript
// 宽度滑块输入
dom.slider.addEventListener('input', () => {
  state.events.widthInput = Number.parseInt(dom.slider.value, 10)
  scheduleRender()
})

// 河流开关输入
dom.showIndicators.addEventListener('input', () => {
  state.events.showIndicatorsInput = dom.showIndicators.checked
  scheduleRender()
})

// 窗口调整
window.addEventListener('resize', scheduleRender)
```

### 渲染调度

```typescript
let scheduledRaf: number | null = null

function scheduleRender(): void {
  if (scheduledRaf !== null) return  // 已调度，跳过
  scheduledRaf = requestAnimationFrame(function renderAndSyncCssOverlay() {
    scheduledRaf = null
    render()
  })
}
```

确保每帧最多渲染一次，避免输入事件触发过多渲染。

### 渲染流程

```typescript
function render(): void {
  // 1. 读取待处理事件，更新控制状态
  let colWidth = state.controls.colWidth
  if (state.events.widthInput !== null) colWidth = state.events.widthInput
  
  let showIndicators = state.controls.showIndicators
  if (state.events.showIndicatorsInput !== null) showIndicators = state.events.showIndicatorsInput
  
  const nextControls = { colWidth, showIndicators }
  
  // 2. 计算新帧（调用模型层）
  const frame = buildDemoFrame(resources, nextControls)
  
  // 3. 更新状态
  state.controls = nextControls
  state.events.widthInput = null
  state.events.showIndicatorsInput = null
  
  // 4. 渲染到 DOM（调用 UI 层）
  renderFrame(dom, frame, resources.normalSpaceWidth)
  
  // 5. 调度 CSS 覆盖层同步（延迟一帧）
  scheduleCssOverlaySync()
}
```

### CSS 覆盖层同步

```typescript
let cssOverlayRequestId = 0

function scheduleCssOverlaySync(): void {
  const requestId = ++cssOverlayRequestId
  requestAnimationFrame(function syncCssOverlayAfterLayout() {
    if (requestId !== cssOverlayRequestId) return  // 已被新请求取代
    syncCssRiverOverlay(dom, state.controls, resources.normalSpaceWidth)
  })
}
```

为什么需要延迟？
- CSS 栏使用浏览器原生文本布局
- 河流指示器需要测量实际渲染后的空格位置
- 必须在浏览器完成布局计算后执行（延迟一帧）

### 初始化流程

```typescript
// 等待字体加载
await document.fonts.ready

// 创建资源（预处理文本、测量空格/连字符宽度）
const resources = createDemoResources()

// 初始渲染
render()
```

## 关键代码路径与文件引用

### 依赖导入

```typescript
import { createDemoResources, buildDemoFrame, type DemoControls } from './justification-comparison.model.ts'
import { createDomCache, renderFrame, syncCssRiverOverlay } from './justification-comparison.ui.ts'
```

### 调用链

```
用户输入
  ↓
event listener → scheduleRender()
  ↓
requestAnimationFrame
  ↓
render()
  ├── buildDemoFrame(resources, controls)  [model.ts]
  │   ├── layoutParagraphsGreedy()         // CSS 模拟
  │   └── layoutParagraphsOptimal()        // Knuth-Plass
  │
  └── renderFrame(dom, frame, normalSpaceWidth)  [ui.ts]
      ├── paintCanvasColumn()              // Canvas 渲染
      └── updateMetricPanel()              // 指标更新
  ↓
scheduleCssOverlaySync()
  ↓
requestAnimationFrame
  ↓
syncCssRiverOverlay()  [ui.ts]
  └── 使用 Range API 测量 CSS 空格位置
```

## 依赖与外部交互

### 外部依赖

| 依赖 | 用途 |
|------|------|
| `justification-comparison.model.ts` | 布局算法、帧构建 |
| `justification-comparison.ui.ts` | DOM 操作、渲染 |
| `document.fonts.ready` | 等待字体加载 |
| `requestAnimationFrame` | 渲染调度 |

### 浏览器 API

```typescript
// 字体加载
await document.fonts.ready

// 动画帧调度
requestAnimationFrame(callback)

// 事件监听
addEventListener('input', handler)
addEventListener('resize', handler)
```

## 风险、边界与改进建议

### 已知风险

1. **字体加载超时**
   - `document.fonts.ready` 可能永远不会 resolve（如果字体加载失败）
   - 建议添加超时处理

2. **RAF 累积**
   - 如果 `render()` 执行时间过长，可能错过下一帧
   - 当前实现简单，但复杂场景可能需要节流

3. **CSS 覆盖层同步延迟**
   - 延迟一帧可能导致视觉闪烁
   - 快速调整宽度时，指示器可能滞后

### 边界情况处理

| 边界情况 | 处理方式 |
|---------|---------|
| 快速连续输入 | RAF 节流确保每帧最多渲染一次 |
| 组件卸载 | 当前无清理逻辑（页面级演示，无需卸载） |
| 字体加载失败 | 无特殊处理，使用回退字体 |

### 改进建议

1. **字体加载超时**
   ```typescript
   await Promise.race([
     document.fonts.ready,
     new Promise((_, reject) => 
       setTimeout(() => reject(new Error('Font timeout')), 5000)
     )
   ]).catch(() => console.warn('Fonts failed to load, using fallbacks'))
   ```

2. **防抖/节流优化**
   ```typescript
   // 对 resize 事件使用防抖
   import { debounce } from 'lodash-es'
   window.addEventListener('resize', debounce(scheduleRender, 100))
   ```

3. **错误边界**
   ```typescript
   function render(): void {
     try {
       // ... 渲染逻辑
     } catch (error) {
       console.error('Render failed:', error)
       showErrorMessage('渲染失败，请刷新页面重试')
     }
   }
   ```

4. **性能监控**
   ```typescript
   function render(): void {
     const start = performance.now()
     // ... 渲染逻辑
     const duration = performance.now() - start
     if (duration > 16) {
       console.warn(`Slow render: ${duration.toFixed(2)}ms`)
     }
   }
   ```

5. **状态持久化**
   ```typescript
   // 保存用户偏好到 localStorage
   const saved = localStorage.getItem('justification-demo')
   if (saved) {
     state.controls = JSON.parse(saved)
   }
   
   // 监听变化并保存
   window.addEventListener('beforeunload', () => {
     localStorage.setItem('justification-demo', JSON.stringify(state.controls))
   })
   ```

6. **URL 状态同步**
   ```typescript
   // 将控制状态同步到 URL hash，支持分享
   function updateUrl(controls: DemoControls) {
     const params = new URLSearchParams({
       width: String(controls.colWidth),
       indicators: String(controls.showIndicators)
     })
     history.replaceState(null, '', `#${params.toString()}`)
   }
   ```

### 代码组织建议

当前实现是简单的脚本文件，如果需要扩展，可以考虑：

1. **使用状态管理库**
   - 对于更复杂的状态，可引入 Zustand 或 Redux

2. **组件化**
   - 将三栏提取为可复用组件
   - 使用框架（如 React/Vue/Svelte）管理 UI

3. **测试**
   - 提取纯函数进行单元测试
   - 使用 Playwright 进行端到端测试

### 维护注意事项

- 添加新控制项时，需要在 `State` 类型和事件处理中同步更新
- 修改渲染逻辑时，注意保持 RAF 节流模式
- CSS 覆盖层同步延迟是设计决策，修改需谨慎
