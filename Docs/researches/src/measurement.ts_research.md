# src/measurement.ts 研究文档

## 场景与职责

`measurement.ts` 是 Pretext 库的文本测量基础设施模块，负责提供跨浏览器一致的文本宽度测量能力。该模块抽象了 Canvas 测量上下文、段度量缓存、引擎特性检测和 emoji 校正等底层功能。

**核心职责：**
1. 管理 Canvas 测量上下文（`OffscreenCanvas` 或 DOM Canvas）
2. 提供段级别度量缓存，避免重复测量
3. 检测浏览器引擎特性（WebKit vs Chromium/Gecko）
4. 实现 emoji 宽度校正机制
5. 提供字素级别宽度计算（用于长单词断行）

**在架构中的位置：**
- 被 `layout.ts` 调用，用于测量文本段宽度
- 被 `line-break.ts` 调用，获取引擎配置
- 是测量层的唯一入口，向上层屏蔽底层差异

---

## 功能点目的

### 1. Canvas 测量上下文管理

| 函数 | 目的 |
|------|------|
| `getMeasureContext()` | 获取或创建 Canvas 2D 上下文，优先使用 `OffscreenCanvas` |

支持环境：
- 主线程：`OffscreenCanvas`（现代浏览器）
- 主线程降级：`document.createElement('canvas')`
- 不支持环境：抛出错误

### 2. 段度量缓存

| 函数 | 目的 |
|------|------|
| `getSegmentMetricCache(font)` | 获取指定字体的段缓存 Map |
| `getSegmentMetrics(seg, cache)` | 获取段的度量（带缓存） |

缓存结构：
```
Map<font, Map<segment, SegmentMetrics>>
```

### 3. 引擎特性检测

| 函数 | 目的 |
|------|------|
| `getEngineProfile()` | 返回当前浏览器的引擎特性配置 |

检测的 UA 特征：
- Safari（WebKit）：`vendor === 'Apple Computer, Inc.'` 且排除 Chrome/CriOS/FxiOS/EdgiOS
- Chromium：`Chrome/`, `Chromium/`, `CriOS/`, `Edg/`

### 4. Emoji 校正

| 函数 | 目的 |
|------|------|
| `getEmojiCorrection(font, fontSize)` | 计算 Canvas 与 DOM 的 emoji 宽度差 |
| `textMayContainEmoji(text)` | 快速检查文本是否可能包含 emoji |
| `getCorrectedSegmentWidth(...)` | 获取校正后的段宽度 |

问题背景：
- Chrome/Firefox 在 macOS 上 Canvas 测量的 emoji 宽度大于实际 DOM 渲染宽度（小字号时）
- 差值在每个字号下是常数，与字体无关
- Safari Canvas 和 DOM 一致（都较宽），无需校正

### 5. 字素宽度计算

| 函数 | 目的 |
|------|------|
| `getSegmentGraphemeWidths(...)` | 获取段的各字素宽度数组 |
| `getSegmentGraphemePrefixWidths(...)` | 获取累积前缀宽度数组 |

用于长单词的字素级断行。

### 6. 字体测量状态

| 函数 | 目的 |
|------|------|
| `getFontMeasurementState(font, needsEmojiCorrection)` | 一次性获取字体相关的所有测量状态 |

返回：
- 段缓存 Map
- 字体大小
- emoji 校正值

---

## 具体技术实现

### 关键数据结构

```typescript
// 段度量
type SegmentMetrics = {
  width: number                    // 段总宽度
  containsCJK: boolean             // 是否包含 CJK 字符
  emojiCount?: number              // emoji 数量（延迟计算）
  graphemeWidths?: number[] | null // 字素宽度（延迟计算）
  graphemePrefixWidths?: number[] | null // 前缀宽度（延迟计算）
}

// 引擎特性配置
type EngineProfile = {
  lineFitEpsilon: number           // 行宽容差
  carryCJKAfterClosingQuote: boolean // Chromium: 引号后携带 CJK
  preferPrefixWidthsForBreakableRuns: boolean // Safari: 使用前缀宽度
  preferEarlySoftHyphenBreak: boolean // Safari: 优先软连字符断行
}
```

### Canvas 上下文获取

```typescript
export function getMeasureContext(): CanvasRenderingContext2D | OffscreenCanvasRenderingContext2D {
  if (measureContext !== null) return measureContext

  if (typeof OffscreenCanvas !== 'undefined') {
    measureContext = new OffscreenCanvas(1, 1).getContext('2d')!
    return measureContext
  }

  if (typeof document !== 'undefined') {
    measureContext = document.createElement('canvas').getContext('2d')!
    return measureContext
  }

  throw new Error('Text measurement requires OffscreenCanvas or a DOM canvas context.')
}
```

关键点：
- 使用单例模式缓存上下文
- 优先 `OffscreenCanvas`（可在 Worker 中使用）
- 降级到 DOM Canvas

### 引擎检测实现

```typescript
export function getEngineProfile(): EngineProfile {
  if (typeof navigator === 'undefined') {
    // 非浏览器环境（如测试）使用默认配置
    return {
      lineFitEpsilon: 0.005,
      carryCJKAfterClosingQuote: false,
      preferPrefixWidthsForBreakableRuns: false,
      preferEarlySoftHyphenBreak: false,
    }
  }

  const ua = navigator.userAgent
  const vendor = navigator.vendor
  
  const isSafari =
    vendor === 'Apple Computer, Inc.' &&
    ua.includes('Safari/') &&
    !ua.includes('Chrome/') &&
    !ua.includes('Chromium/') &&
    !ua.includes('CriOS/') &&
    !ua.includes('FxiOS/') &&
    !ua.includes('EdgiOS/')
    
  const isChromium =
    ua.includes('Chrome/') ||
    ua.includes('Chromium/') ||
    ua.includes('CriOS/') ||
    ua.includes('Edg/')

  return {
    lineFitEpsilon: isSafari ? 1 / 64 : 0.005,
    carryCJKAfterClosingQuote: isChromium,
    preferPrefixWidthsForBreakableRuns: isSafari,
    preferEarlySoftHyphenBreak: isSafari,
  }
}
```

配置差异：

| 特性 | Safari | Chromium | 其他 |
|------|--------|----------|------|
| `lineFitEpsilon` | 1/64 (~0.016) | 0.005 | 0.005 |
| `carryCJKAfterClosingQuote` | false | true | false |
| `preferPrefixWidthsForBreakableRuns` | true | false | false |
| `preferEarlySoftHyphenBreak` | true | false | false |

### Emoji 校正机制

```typescript
function getEmojiCorrection(font: string, fontSize: number): number {
  let correction = emojiCorrectionCache.get(font)
  if (correction !== undefined) return correction

  const ctx = getMeasureContext()
  ctx.font = font
  const canvasW = ctx.measureText('\u{1F600}').width  // 😀
  correction = 0
  
  if (
    canvasW > fontSize + 0.5 &&  // Canvas 测量异常宽
    typeof document !== 'undefined' &&
    document.body !== null
  ) {
    // 创建临时 DOM 元素测量实际渲染宽度
    const span = document.createElement('span')
    span.style.font = font
    span.style.display = 'inline-block'
    span.style.visibility = 'hidden'
    span.style.position = 'absolute'
    span.textContent = '\u{1F600}'
    document.body.appendChild(span)
    const domW = span.getBoundingClientRect().width
    document.body.removeChild(span)
    
    if (canvasW - domW > 0.5) {
      correction = canvasW - domW
    }
  }
  
  emojiCorrectionCache.set(font, correction)
  return correction
}
```

校正计算：
```typescript
export function getCorrectedSegmentWidth(
  seg: string, 
  metrics: SegmentMetrics, 
  emojiCorrection: number
): number {
  if (emojiCorrection === 0) return metrics.width
  return metrics.width - getEmojiCount(seg, metrics) * emojiCorrection
}
```

关键点：
- 每个字体只校正一次（缓存）
- 需要 DOM 访问权限（主线程）
- 校正值为 Canvas 宽度 - DOM 宽度
- 乘以段内的 emoji 数量

### 字素宽度计算

```typescript
export function getSegmentGraphemeWidths(
  seg: string,
  metrics: SegmentMetrics,
  cache: Map<string, SegmentMetrics>,
  emojiCorrection: number,
): number[] | null {
  if (metrics.graphemeWidths !== undefined) return metrics.graphemeWidths

  const widths: number[] = []
  const graphemeSegmenter = getSharedGraphemeSegmenter()
  for (const gs of graphemeSegmenter.segment(seg)) {
    const graphemeMetrics = getSegmentMetrics(gs.segment, cache)
    widths.push(getCorrectedSegmentWidth(gs.segment, graphemeMetrics, emojiCorrection))
  }

  metrics.graphemeWidths = widths.length > 1 ? widths : null
  return metrics.graphemeWidths
}
```

前缀宽度（Safari 策略）：
```typescript
export function getSegmentGraphemePrefixWidths(...): number[] | null {
  // 累积计算："abc" -> [w("a"), w("ab"), w("abc")]
  let prefix = ''
  for (const gs of graphemeSegmenter.segment(seg)) {
    prefix += gs.segment
    const prefixMetrics = getSegmentMetrics(prefix, cache)
    prefixWidths.push(getCorrectedSegmentWidth(prefix, prefixMetrics, emojiCorrection))
  }
}
```

---

## 依赖与外部交互

### 导入依赖

```typescript
import { isCJK } from './analysis.js'
```

### 被调用方

| 调用方 | 调用函数 |
|--------|----------|
| `layout.ts` | `getFontMeasurementState`, `getSegmentMetrics`, `getCorrectedSegmentWidth`, `getSegmentGraphemeWidths`, `getSegmentGraphemePrefixWidths`, `textMayContainEmoji` |
| `line-break.ts` | `getEngineProfile` |

### 外部 API 依赖

- `Intl.Segmenter`（字素分割）
- `CanvasRenderingContext2D.measureText()`
- `OffscreenCanvas` 或 `document.createElement('canvas')`
- `document.body`（emoji 校正时需要）

---

## 风险、边界与改进建议

### 已知风险

1. **Emoji 校正的 DOM 依赖**
   - 需要 `document.body` 存在
   - 在 Worker 中或 SSR 环境下无法执行
   - 当前在非浏览器环境返回 0 校正

2. **UA 检测的脆弱性**
   - 依赖 `navigator.userAgent` 和 `navigator.vendor`
   - 未来浏览器可能改变 UA 格式
   - 新出现的浏览器引擎可能无法正确识别

3. **缓存内存占用**
   - `segmentMetricCaches` 按字体缓存所有段
   - 大量不同字体或长文本可能导致内存增长
   - 提供 `clearMeasurementCaches()` 但需手动调用

4. **单例 Canvas 上下文**
   - 所有测量共享一个上下文
   - 字体设置频繁切换可能影响性能

### 边界情况

| 场景 | 处理 |
|------|------|
| 非浏览器环境 | 返回默认引擎配置，emoji 校正为 0 |
| 无 `document.body` | emoji 校正返回 0 |
| 字体大小解析失败 | `parseFontSize` 返回 16（默认值） |
| 空段 | 缓存并返回 width = 0 |
| 单字素段 | `graphemeWidths` 返回 null（优化） |

### 改进建议

1. **运行时容差校准**
   - 类似 emoji 校正，可以运行时测量确定 `lineFitEpsilon`
   - 消除硬编码的浏览器差异

2. **Worker 环境支持**
   - 提供不依赖 DOM 的纯 Canvas 模式
   - 或者通过 postMessage 委托主线程进行 emoji 校正

3. **缓存策略优化**
   - 考虑 LRU 淘汰策略限制缓存大小
   - 或提供内存压力回调

4. **字体回退处理**
   - 当前假设字体已加载
   - 可考虑监听字体加载事件

5. **SSR 安全**
   - 确保所有 DOM 访问都有环境检查
   - 提供 SSR 友好的 mock 实现

### 测试覆盖

测试主要位于 `src/layout.test.ts`，使用 `TestOffscreenCanvas` mock：
- 基础测量功能
- 缓存行为
- emoji 校正（mock 环境可能有限）

建议增加：
- 真实浏览器环境下的 emoji 校正验证
- 多字体缓存行为
- 内存使用测试
