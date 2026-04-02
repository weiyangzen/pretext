# variable-typographic-ascii.html 研究文档

## 场景与职责

`variable-typographic-ascii.html` 是 Variable Typographic ASCII 艺术演示的 HTML 外壳文件。该演示展示了如何使用 Pretext 库进行精确的字符宽度测量，以在比例字体（proportional font）中创建保持形状的 ASCII 艺术效果。

**核心场景**：
- 对比三种渲染方式：
  1. **Source Field**：粒子系统的原始亮度场（Canvas 渲染）
  2. **Proportional × 3 Weights × Italic**：使用 Georgia 字体的三种字重（300/500/800）和两种样式（normal/italic）
  3. **Monospace × Single Weight**：使用等宽字体的传统 ASCII 艺术

## 功能点目的

### 1. 三面板对比布局

```html
<div class="comparison">
  <div class="panel">
    <div class="panel-label">Source Field</div>
    <div class="art-box" id="source-box"></div>
  </div>
  <div class="panel">
    <div class="panel-label">Proportional × 3 Weights × Italic</div>
    <div class="art-box" id="prop-box"></div>
  </div>
  <div class="panel">
    <div class="panel-label">Monospace × Single Weight</div>
    <div class="art-box" id="mono-box"></div>
  </div>
</div>
```

**设计目的**：直观展示比例字体在精确宽度测量下的 ASCII 艺术潜力。

### 2. 视觉设计系统

**颜色方案**：
- 背景：深色渐变 `#0a0a12` → `#06060a`
- 文本：暖灰色 `#ddd8d0`
- 强调色：金色 `#c4a35a`（用于 ASCII 字符）
- 等宽面板：蓝灰色 `rgba(130,155,210,0.7)`

**字体配置**：
```css
/* 比例字体面板 */
.art-row {
  font-family: Georgia, Palatino, "Times New Roman", serif;
  font-size: 14px;
}

/* 等宽字体面板 */
#mono-box .art-row {
  font: 400 14px/16px "Courier New", Courier, monospace;
}
```

### 3. 透明度层级系统

```css
.a1 { color: rgba(196,163,90,0.10); }
.a2 { color: rgba(196,163,90,0.20); }
/* ... */
.a10 { color: rgba(196,163,90,1.0); }
```

10 级透明度用于表示亮度场的不同强度。

### 4. 字重和样式类

```css
.w3 { font-weight: 300; }  /* Light */
.w5 { font-weight: 500; }  /* Medium */
.w8 { font-weight: 800; }  /* Extra Bold */
.it { font-style: italic; }
```

## 具体技术实现

### CSS 架构

**容器布局**：
```css
body {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 32px 20px 60px;
}

.comparison {
  display: flex;
  gap: 28px;
  flex-wrap: wrap;
  justify-content: center;
}

.panel {
  width: min(468px, calc(100vw - 40px));
}
```

使用 flexbox 实现响应式三列布局，在小屏幕上自动换行。

**艺术盒子样式**：
```css
.art-box {
  width: 100%;
  height: calc(var(--art-height) + 28px);
  border: 1px solid rgba(255,255,255,0.05);
  border-radius: 10px;
  padding: 14px;
  background: rgba(0,0,0,0.4);
  overflow: hidden;
  box-shadow: inset 0 0 30px rgba(0,0,0,0.5), 0 4px 20px rgba(0,0,0,0.3);
}
```

**行样式**：
```css
.art-row {
  display: block;
  width: fit-content;
  margin-inline: auto;
  white-space: pre;  /* 关键：保留空格和换行 */
}
```

`white-space: pre` 确保 ASCII 艺术中的空格被正确渲染。

### 关键 CSS 变量

```css
:root {
  --art-height: 448px;
}
```

控制 Canvas 和艺术盒子的高度。

### 模块加载

```html
<script type="module" src="./variable-typographic-ascii.ts"></script>
```

使用 ES Module 加载 TypeScript 逻辑（由开发服务器处理转换）。

## 依赖与外部交互

### 文件依赖

| 文件 | 关系 | 说明 |
|------|------|------|
| `variable-typographic-ascii.ts` | 逻辑实现 | 粒子系统、亮度场计算、字符选择 |
| `svg.d.ts` | 类型支持 | 如需要导入 SVG 资源 |

### 外部资源

- **Google Fonts**（可选）：Georgia 和 Courier New 是系统字体，无需网络加载
- **Canvas API**：用于粒子渲染和亮度计算

### 浏览器要求

- 支持 ES Module
- 支持 Canvas 2D Context
- 支持 `OffscreenCanvas`（在 worker 中可选）

## 风险、边界与改进建议

### 当前风险

1. **固定尺寸**：`--art-height: 448px` 是硬编码的，与 TypeScript 中的 `ROWS = 28` 和 `LINE_HEIGHT = 16` 耦合
2. **字体可用性**：依赖系统字体 Georgia 和 Courier New，在某些系统上可能回退到不同字体
3. **性能**：频繁的 DOM 更新（每帧更新所有行）可能在低性能设备上卡顿

### 边界情况

1. **视口过小**：当视口小于约 520px 时，面板会垂直堆叠
2. **字体加载失败**：如果 Georgia 不可用，会回退到 Palatino 或 Times New Roman，可能导致宽度测量不匹配

### 改进建议

1. **CSS 变量同步**：
   ```css
   :root {
     --art-height: 448px;  /* 与 TS 中的 ROWS * LINE_HEIGHT 同步 */
     --art-cols: 50;
     --art-rows: 28;
   }
   ```

2. **字体加载检测**：
   ```javascript
   // 在 TS 中
   document.fonts.ready.then(() => {
     // 开始渲染
   })
   ```

3. **响应式改进**：
   ```css
   @media (max-width: 480px) {
     :root {
       --art-height: 320px;  /* 减小高度 */
     }
     .art-row {
       font-size: 10px;  /* 减小字号 */
     }
   }
   ```

4. **无障碍性**：
   ```html
   <div class="art-box" id="prop-box" role="img" aria-label="Proportional ASCII art visualization"></div>
   ```

5. **性能优化**：
   - 使用 `content-visibility: auto` 优化离屏渲染
   - 考虑使用 `requestIdleCallback` 进行非关键更新
