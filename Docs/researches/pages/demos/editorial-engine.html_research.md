# Research: pages/demos/editorial-engine.html

## 场景与职责

`editorial-engine.html` 是 Pretext “Editorial Engine” 演示的静态页面壳层。该页面呈现一个深色主题、全屏沉浸式的杂志风格布局，核心卖点是**实时动画障碍物（发光 orb）与多栏文本流的互动**：用户可拖拽 orb、点击暂停/继续动画，而正文会在每一帧重新计算并环绕这些移动的 orb 流动。与 `dynamic-layout.html` 类似，本页面仅提供最简 DOM 骨架与基础 CSS，所有布局、动画、交互逻辑均由 `editorial-engine.ts` 驱动。

## 功能点目的

1. **提供全屏沉浸式舞台**：`#stage` 占满 `100vw × 100vh`，无滚动条，所有内容通过绝对定位逐行渲染。  
2. **深色主题与氛围视觉**：径向渐变背景、金色/蓝色/粉色发光 orb、高对比度文本，营造高端 editorial 氛围。  
3. **为 JS 驱动的绝对定位文本预留样式契约**：定义 `.line`、`.headline-line`、`.drop-cap`、`.orb`、`.pullquote-box`、`.pullquote-line` 的基础样式，但不指定任何位置或尺寸。  
4. **最小化首屏依赖**：页面不内嵌任何初始化脚本，完全依赖外部模块 `editorial-engine.ts` 在加载后构建全部内容。

## 具体技术实现

### DOM 结构

```html
<div id="stage"></div>
<div class="hint">Drag the orbs &middot; Click to pause &middot; Zero DOM reads</div>
<a class="credit" href="https://x.com/somnai_dreams" target="_blank" rel="noreferrer">Made by @somnai_dreams</a>
<script type="module" src="./editorial-engine.ts"></script>
```

- `#stage`：唯一的内容挂载点。`editorial-engine.ts` 会动态创建并追加标题行、正文行、首字下沉（drop cap）、引用块（pull quote）、发光 orb 等元素。
- `.hint`：固定于顶部的操作提示，窄屏下隐藏。
- `.credit`：右下角固定署名链接，悬停变亮。

### CSS 关键规则

- **全局重置**：`* { box-sizing: border-box; margin: 0; }`
- **背景**：`radial-gradient(ellipse at 50% 40%, #0f0f14 0%, #0a0a0c 100%)`
- **文本行基础样式**：
  - `.line`：`position: absolute; white-space: pre; z-index: 1; color: #e8e4dc; user-select: text;`
  - `.headline-line`：加粗、白色、字母间距微调、层级更高（`z-index: 2`）。
  - `.pullquote-line`：斜体、暖金色（`#b8a070`）。
- **首字下沉**：`.drop-cap` 使用大字号、金色（`#c4a35a`）、`pointer-events: none`。
- **发光 orb**：`.orb` 使用 `border-radius: 50%`、径向渐变背景、多层 `box-shadow` 模拟发光效果，`will-change: transform` 提示浏览器优化。
- **响应式**：`@media (max-width: 760px)` 隐藏 `.hint` 与 `.credit`。

### 脚本引用

```html
<script type="module" src="./editorial-engine.ts"></script>
```

无 `modulepreload`，加载顺序完全由浏览器模块解析决定。

## 关键代码路径与文件引用

- **同目录脚本**：
  - `editorial-engine.ts` — 核心引擎，负责 orb 物理动画、 headline 自适应、多栏流、pull quote 定位、drop cap 渲染。
- **上游库**：
  - `../../src/layout.ts` — 被 `editorial-engine.ts` 引入，提供 `layoutNextLine`、`layoutWithLines`、`prepareWithSegments`、`walkLineRanges`。

## 依赖与外部交互

- **无第三方依赖**：纯原生 HTML/CSS，不引用任何外部库、字体 CDN 或图片资源。
- **与 TS 模块的契约**：
  - `#stage` 必须存在且为 `HTMLDivElement`，否则 `editorial-engine.ts` 顶层抛错。
  - `.hint` 与 `.credit` 的显隐逻辑由 CSS 媒体查询控制，`editorial-engine.ts` 不直接操作它们。

## 风险、边界与改进建议

1. **完全依赖外部模块**：页面无任何内联降级内容或骨架屏。若 `editorial-engine.ts` 加载失败或执行异常，用户将看到纯黑屏。建议：
   - 在 `#stage` 内增加一个极轻量的 `<noscript>` 或默认文本提示（如 `<p>Please enable JavaScript to view this demo.</p>`），在脚本成功执行后被覆盖或移除。  
2. **无预加载优化**：未对 `editorial-engine.ts` 使用 `<link rel="modulepreload">`，在慢网络下模块解析链（`editorial-engine.ts` → `../../src/layout.ts` → `src/analysis.js` / `src/measurement.js` 等）可能逐层延迟。建议增加：
   ```html
   <link rel="modulepreload" href="./editorial-engine.ts">
   <link rel="modulepreload" href="../../src/layout.ts">
   ```
   以缩短首屏可交互时间。  
3. **固定视口与滚动缺失**：`html, body { height: 100vh; overflow: hidden; }` 锁死了滚动。若文本长度超过设计容量，超出的行将被直接截断，用户无法通过滚动查看。作为艺术化演示这是有意为之，但在信息密度较高的场景中需改为允许滚动或分页。  
4. **字体回退链较长**：`font-family: "Iowan Old Style", "Palatino Linotype", "Book Antiqua", Palatino, serif;` 依赖系统预装字体。在 Windows 或 Linux 设备上若全部缺失，将回退到通用 `serif`，可能导致 Pretext 的 canvas 测量宽度与实际 DOM 渲染宽度出现细微偏差。建议：
   - 使用 `document.fonts.check()` 或在 `editorial-engine.ts` 中增加字体加载超时降级逻辑。
   - 或引入 Web Font（如 Google Fonts 的 `Crimson Text`）作为更可控的 fallback。  
5. **暗色对比度**：`.hint` 颜色为 `rgba(255,255,255,0.22)`，在部分显示器上可能难以辨识。建议提升至至少 `0.4` 以上以满足 WCAG 最低对比度要求。  
6. **改进建议**：
   - 增加 `aria-live` 区域用于朗读 orb 状态变化（如“Orb paused”），提升屏幕阅读器体验。
   - 为 `#stage` 增加 `role="img" aria-label="Interactive editorial layout with animated orbs"`，帮助辅助技术理解页面性质。
   - 考虑将 `.credit` 链接的 `target="_blank"` 增加 `rel="noopener noreferrer"`（当前已有 `noreferrer`，可补充 `noopener` 以符合安全最佳实践）。
