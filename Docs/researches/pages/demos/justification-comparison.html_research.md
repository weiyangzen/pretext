# justification-comparison.html 研究文档

## 场景与职责

`justification-comparison.html` 是 Justification Comparison 演示的页面容器，提供三栏并排对比布局，展示三种不同的文本对齐算法：

1. **CSS / Greedy**：浏览器原生两端对齐（贪婪算法）
2. **Pretext (Hyphenation)**：带连字符化的贪婪算法
3. **Pretext (Knuth-Plass)**：全局最优断行算法

该页面通过视觉对比，直观展示不同算法在排版质量上的差异，特别是"河流"（rivers）现象的分布。

## 功能点目的

### 1. 算法对比可视化
三栏并排布局，相同文本内容、相同列宽，直接对比：
- 行数差异
- 词间距均匀度
- 河流（过宽词间距）的出现频率

### 2. 交互式探索
- **宽度滑块**：实时调整列宽（200px - 600px），观察算法在不同宽度下的表现
- **河流指示器开关**：高亮显示过宽的词间距，直观展示河流分布

### 3. 质量指标展示
每栏底部显示量化指标：
- 行数
- 平均词间距偏差
- 最大词间距偏差
- 河流空间数量

### 4. 视觉区分
- CSS 栏使用真实 DOM 文本渲染
- Pretext 栏使用 Canvas 渲染
- 通过颜色编码（绿/黄/红）直观展示质量等级

## 具体技术实现

### 页面结构

```html
body
├── .topbar           /* 顶部信息栏 */
├── .page             /* 主内容区 */
│   ├── h1            /* 页面标题 */
│   ├── .subtitle     /* 副标题 */
│   ├── .controls     /* 控制栏（滑块、开关） */
│   └── .columns      /* 三栏对比区 */
│       ├── #col0     /* CSS / Greedy */
│       ├── #col2     /* Pretext + Hyphenation */
│       └── #col3     /* Pretext + Knuth-Plass */
└── footer.footer     /* 页脚 */
```

### 三栏布局差异

| 特性 | CSS 栏 | Pretext 栏 (×2) |
|-----|--------|----------------|
| 渲染方式 | DOM (`<div class="css-text">`) | Canvas (`<canvas>`) |
| 河流指示器 | 绝对定位覆盖层 (`position: absolute`) | Canvas 绘制 |
| 文本更新 | `textContent` | 重绘 Canvas |
| 度量来源 | `Range.getClientRects()` | 布局计算 |

### CSS 栏特殊结构

```html
<div class="css-col" id="cssCol">
  <div class="css-text" id="cssText"></div>      <!-- 文本内容 -->
  <div class="css-river-overlay" id="cssRiverOverlay"></div>  <!-- 河流高亮 -->
</div>
```

河流指示器实现：
- 在文本渲染后，使用 `Range` API 测量每个空格的实际宽度
- 在覆盖层上创建对应位置的彩色标记
- 颜色强度与空格宽度成正比

### 响应式设计

```css
.columns {
  display: flex;
  gap: 24px;
  justify-content: center;
  overflow-x: auto;  /* 允许水平滚动 */
}
.column {
  flex: 0 0 auto;    /* 不伸缩，保持固定宽度 */
  min-width: 200px;
}
```

当视口不足以容纳三栏时，允许水平滚动而非压缩列宽。

### 控制组件

**宽度滑块**：
```html
<input type="range" id="widthSlider" min="200" max="600" value="364">
```

**河流指示器开关**：
```html
<label class="toggle" for="showIndicators">
  <input type="checkbox" id="showIndicators" checked>
  <span>Toggle red visualizers</span>
</label>
```

### 度量面板结构

```html
<div class="metrics" id="metrics0">
  <div class="metric-row">
    <span class="metric-label">Lines</span>
    <span class="metric-value">12</span>
  </div>
  <!-- ... 其他指标 -->
</div>
```

质量等级颜色：
- `.good` (绿色): 偏差 < 15%
- `.ok` (黄色): 偏差 15% - 35%
- `.bad` (红色): 偏差 > 35%

## 关键代码路径与文件引用

### 脚本加载

```html
<script type="module" src="./justification-comparison.ts"></script>
```

入口模块链：
1. `justification-comparison.ts` - 事件处理和渲染调度
2. `justification-comparison.model.ts` - 布局算法和状态计算
3. `justification-comparison.ui.ts` - DOM 操作和渲染
4. `justification-comparison.data.ts` - 数据和常量

### 元素 ID 映射

| ID | 用途 | 消费者 |
|----|------|--------|
| `widthSlider` | 宽度滑块 | `justification-comparison.ts` |
| `widthVal` | 宽度显示 | `justification-comparison.ui.ts` |
| `showIndicators` | 河流开关 | `justification-comparison.ts` |
| `col0` / `col2` / `col3` | 列容器 | `justification-comparison.ui.ts` |
| `cssCol` | CSS 栏容器 | `justification-comparison.ui.ts` |
| `cssText` | CSS 文本区 | `justification-comparison.ui.ts` |
| `cssRiverOverlay` | 河流覆盖层 | `justification-comparison.ui.ts` |
| `c2` / `c3` | Canvas 元素 | `justification-comparison.ui.ts` |
| `metrics0` / `metrics2` / `metrics3` | 度量面板 | `justification-comparison.ui.ts` |

### CSS 类名约定

| 类名 | 用途 |
|-----|------|
| `.css-col` | CSS 栏容器样式 |
| `.css-text` | CSS 文本样式（`text-align: justify`） |
| `.css-river-overlay` | 河流指示器定位参考 |
| `.col-canvas-wrap` | Canvas 容器（边框、圆角） |
| `.metrics` | 度量面板样式 |
| `.metric-value.good/.ok/.bad` | 质量等级颜色 |

## 依赖与外部交互

### 外部依赖

| 依赖 | 用途 |
|------|------|
| `justification-comparison.ts` | 主逻辑模块 |
| 系统字体 | Georgia, Helvetica Neue 等 |

### DOM API 使用

CSS 栏河流检测使用以下 API：
```javascript
const range = document.createRange()
range.setStart(textNode, charIndex)
range.setEnd(textNode, charIndex + 1)
const rects = range.getClientRects()  // 获取空格实际渲染位置
```

### Canvas API

Pretext 栏使用 2D Canvas 渲染：
```javascript
const ctx = canvas.getContext('2d')
ctx.font = '15px Georgia, "Times New Roman", serif'
ctx.fillText(text, x, y)
```

## 风险、边界与改进建议

### 已知风险

1. **字体加载时序**
   - Canvas 和 DOM 可能使用不同的字体回退策略
   - 如果 Georgia 未加载，度量可能不一致
   - 当前依赖 `document.fonts.ready`（在入口模块中）

2. **浏览器差异**
   - CSS `text-align: justify` 行为在不同浏览器可能有细微差异
   - `Range.getClientRects()` 对空格的返回可能不一致

3. **性能考虑**
   - 每次调整宽度时，CSS 栏需要重新计算河流指示器（涉及 DOM 读取）
   - 频繁调整可能导致布局抖动

### 边界情况处理

| 边界情况 | 处理方式 |
|---------|---------|
| 极窄宽度 (< 200px) | 滑块最小值限制 |
| 极宽宽度 (> 600px) | 滑块最大值限制 |
| 浏览器不支持 Canvas | 页面将无法显示 Pretext 栏（需添加降级） |
| 字体加载失败 | 使用系统默认字体，可能影响对比准确性 |

### 改进建议

1. **字体加载保障**
   ```html
   <link rel="preload" href="..." as="font" type="font/ttf" crossorigin>
   ```

2. **加载状态指示**
   - 添加字体加载进度指示
   - 在字体就绪前禁用交互

3. **可访问性增强**
   - 为滑块添加 `aria-valuemin`, `aria-valuemax`, `aria-valuenow`
   - 为开关添加 `aria-checked`
   - 添加键盘快捷键（如 +/- 调整宽度）

4. **响应式改进**
   - 在小屏幕上自动切换到垂直堆叠布局
   - 或提供"单栏模式"切换

5. **导出功能**
   - 添加"导出对比图"按钮
   - 使用 `canvas.toDataURL()` 生成图片

6. **动画过渡**
   - 宽度调整时添加平滑过渡动画
   - 使用 CSS `transition` 或 Canvas 插值

7. **详细分析模式**
   - 添加"显示断行点"选项
   - 显示每个断行处的 badness 值

### 维护注意事项

- 修改列宽范围时，需同步更新滑块的 `min`/`max` 和显示值
- 添加新算法对比栏时，需遵循现有的 ID 命名约定（`col{n}`, `metrics{n}`）
- 保持三栏的 HTML 结构一致，便于 CSS 样式复用
