# variable-typographic-ascii.ts 研究文档

## 场景与职责

`variable-typographic-ascii.ts` 是 Variable Typographic ASCII 艺术演示的核心逻辑实现。它结合粒子物理模拟、亮度场计算和 Pretext 的精确文本测量能力，创建了一个动态的艺术可视化效果。

**核心创新点**：
- 使用 Pretext 的 `prepareWithSegments()` 精确测量每个字符在特定字体（字重、样式）下的宽度
- 根据亮度目标和目标单元格宽度，智能选择最佳字符
- 对比展示比例字体（proportional）和等宽字体（monospace）的 ASCII 艺术效果

## 功能点目的

### 1. 粒子系统与亮度场

**物理模拟**：
- 120 个粒子在两个吸引子之间运动
- 粒子在 Canvas 上绘制为径向渐变精灵
- 亮度场使用 `Float32Array` 进行高效数值计算

**亮度场计算**：
```typescript
const brightnessField = new Float32Array(FIELD_COLS * FIELD_ROWS)
```

粒子位置被 splat 到亮度场中，形成动态变化的亮度分布。

### 2. 字符调色板（Palette）系统

**字体变体矩阵**：
- 3 种字重：300 (Light)、500 (Medium)、800 (Extra Bold)
- 2 种样式：normal、italic
- 总计：6 种字体变体

**字符集**：
```typescript
const CHARSET = ' .,:;!+-=*#@%&abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'
```

**调色板条目**：
```typescript
type PaletteEntry = {
  char: string
  weight: number
  style: FontStyleVariant
  font: string
  width: number        // 通过 Pretext 测量
  brightness: number   // 通过 Canvas 渲染测量
}
```

### 3. 智能字符选择算法

**目标**：找到同时满足亮度匹配和宽度匹配的字符。

**算法步骤**：
1. 二分查找定位亮度相近的字符范围
2. 在邻域（±15 个字符）中评估综合得分
3. 选择得分最低的字符

**评分函数**：
```typescript
const brightnessError = Math.abs(entry.brightness - targetBrightness) * 2.5
const widthError = Math.abs(entry.width - targetCellW) / targetCellW
const score = brightnessError + widthError
```

亮度误差权重更高（2.5x），确保视觉形状优先。

### 4. 双模式渲染

**比例字体模式**：
- 使用 Georgia 字体家族
- 每个字符独立包装在 `<span>` 中
- 应用对应的字重、样式和透明度类

**等宽字体模式**：
- 使用 Courier New
- 直接设置文本内容
- 作为视觉对比基准

## 具体技术实现

### 关键常量

```typescript
const COLS = 50              // ASCII 艺术列数
const ROWS = 28              // ASCII 艺术行数
const FONT_SIZE = 14         // 字体大小（px）
const LINE_HEIGHT = 16       // 行高（px）
const TARGET_ROW_W = 440     // 目标行宽度（px）
const FIELD_OVERSAMPLE = 2   // 亮度场超采样率
const PARTICLE_N = 120       // 粒子数量
```

### 亮度测量

```typescript
function estimateBrightness(ch: string, font: string): number {
  const size = 28
  bCtx.clearRect(0, 0, size, size)
  bCtx.font = font
  bCtx.fillStyle = '#fff'
  bCtx.textBaseline = 'middle'
  bCtx.fillText(ch, 1, size / 2)
  
  const data = bCtx.getImageData(0, 0, size, size).data
  let sum = 0
  for (let index = 3; index < data.length; index += 4) {
    sum += data[index]!  // 累加 alpha 通道
  }
  return sum / (255 * size * size)  // 归一化到 0-1
}
```

通过 Canvas 渲染字符并分析像素 alpha 值来估计视觉亮度。

### 宽度测量（Ptext 集成）

```typescript
function measureWidth(ch: string, font: string): number {
  const prepared = prepareWithSegments(ch, font)
  return prepared.widths.length > 0 ? prepared.widths[0]! : 0
}
```

使用 Pretext 的 `prepareWithSegments()` 获取精确字符宽度。

### 调色板构建流程

```typescript
const palette: PaletteEntry[] = []
for (const style of STYLES) {
  for (const weight of WEIGHTS) {
    const font = `${style === 'italic' ? 'italic ' : ''}${weight} ${FONT_SIZE}px ${PROP_FAMILY}`
    for (const ch of CHARSET) {
      if (ch === ' ') continue
      const width = measureWidth(ch, font)           // Pretext 测量
      const brightness = estimateBrightness(ch, font) // Canvas 测量
      palette.push({ char: ch, weight, style, font, width, brightness })
    }
  }
}

// 归一化亮度
const maxBrightness = Math.max(...palette.map(entry => entry.brightness))
for (let index = 0; index < palette.length; index++) {
  palette[index]!.brightness /= maxBrightness
}

// 按亮度排序
palette.sort((a, b) => a.brightness - b.brightness)
```

### 亮度查找表构建

```typescript
const brightnessLookup: BrightnessEntry[] = []
for (let brightnessByte = 0; brightnessByte < 256; brightnessByte++) {
  const brightness = brightnessByte / 255
  const monoChar = MONO_RAMP[Math.min(MONO_RAMP.length - 1, (brightness * MONO_RAMP.length) | 0)]!
  
  if (brightness < 0.03) {
    brightnessLookup.push({ monoChar, propHtml: ' ' })
    continue
  }
  
  const match = findBest(brightness)
  const alphaIndex = Math.max(1, Math.min(10, Math.round(brightness * 10)))
  brightnessLookup.push({
    monoChar,
    propHtml: `<span class="${wCls(match.weight, match.style)} a${alphaIndex}">${esc(match.char)}</span>`,
  })
}
```

预计算 256 级亮度对应的字符，实现 O(1) 查询。

### 渲染循环

```typescript
function render(now: number): void {
  // 1. 更新吸引子位置（基于时间）
  const attractor1X = Math.cos(now * 0.0007) * CANVAS_W * 0.25 + CANVAS_W / 2
  // ...
  
  // 2. 更新粒子物理
  for (const particle of particles) {
    // 应用吸引力和随机扰动
  }
  
  // 3. 绘制粒子到 Canvas
  sCtx.drawImage(particleSprite, particle.x - SPRITE_R, particle.y - SPRITE_R)
  
  // 4. 更新亮度场
  for (let index = 0; index < brightnessField.length; index++) {
    brightnessField[index] *= FIELD_DECAY  // 衰减
  }
  splatFieldStamp(particle.x, particle.y, particleFieldStamp)
  
  // 5. 生成 ASCII 行
  for (let row = 0; row < ROWS; row++) {
    let propHtml = ''
    let monoText = ''
    for (let col = 0; col < COLS; col++) {
      const brightnessByte = computeBrightness(row, col)
      const entry = brightnessLookup[brightnessByte]!
      propHtml += entry.propHtml
      monoText += entry.monoChar
    }
    rowNodes[row]!.propNode.innerHTML = propHtml
    rowNodes[row]!.monoNode.textContent = monoText
  }
  
  requestAnimationFrame(render)
}
```

## 依赖与外部交互

### 核心依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `../../src/layout.ts` | `prepareWithSegments` | 字符宽度测量 |

### DOM 交互

```typescript
const sourceBox = getRequiredDiv('source-box')  // Canvas 容器
const propBox = getRequiredDiv('prop-box')      // 比例字体容器
const monoBox = getRequiredDiv('mono-box')      // 等宽字体容器
```

### 与 HTML 的协作

- HTML 提供容器结构和基础样式
- TypeScript 动态生成行元素并填充内容
- CSS 类（`.w3`, `.w5`, `.w8`, `.it`, `.a1`-`.a10`）在 HTML 中定义

## 风险、边界与改进建议

### 当前风险

1. **内存使用**：`brightnessField` 大小为 `FIELD_COLS * FIELD_ROWS = 100 * 56 = 5600` 个 float，相对较小
2. **DOM 更新频率**：每帧更新 28 行 × 2 个面板 = 56 个 DOM 节点，使用 `innerHTML` 可能导致垃圾回收压力
3. **字体加载**：如果 Georgia 字体未加载完成，测量可能不准确

### 边界情况

1. **亮度为 0**：映射到空格 `' '`
2. **亮度接近 1**：选择最大字重、最高不透明度的字符
3. **字符宽度为 0**：过滤掉（`if (width <= 0) continue`）

### 改进建议

1. **字体加载等待**：
   ```typescript
   await document.fonts.ready
   // 然后构建调色板
   ```

2. **DOM 更新优化**：
   ```typescript
   // 使用 textContent 替代 innerHTML 对于等宽模式
   // 使用 DocumentFragment 批量更新
   ```

3. **Worker 线程**：
   ```typescript
   // 将亮度场计算移到 Worker
   const worker = new Worker('./ascii-worker.ts')
   worker.postMessage({ brightnessField }, [brightnessField.buffer])
   ```

4. **调色板持久化**：
   ```typescript
   // 使用 localStorage 缓存调色板数据
   const cached = localStorage.getItem('ascii-palette')
   if (cached) {
     palette = JSON.parse(cached)
   } else {
     buildPalette()
     localStorage.setItem('ascii-palette', JSON.stringify(palette))
   }
   ```

5. **更多字体变体**：
   ```typescript
   const WEIGHTS = [300, 400, 500, 600, 700, 800] as const
   // 增加更多字重选择
   ```

6. **响应式字符密度**：
   ```typescript
   // 根据视口大小动态调整 COLS/ROWS
   const COLS = window.innerWidth < 600 ? 30 : 50
   ```
