# wrap-geometry.ts 研究文档

## 场景与职责

`wrap-geometry.ts` 是 Pretext 演示项目中用于处理**文本环绕几何计算**的共享工具模块。它提供了从图像生成环绕轮廓（wrap hull）、多边形变换、点包含检测以及文本行槽位计算等核心几何功能。

**主要应用场景**：
1. **dynamic-layout.ts**：实现文本环绕 OpenAI 和 Claude logo 的效果
2. 任何需要将文本流环绕不规则形状（如图片、图标）的布局场景

## 功能点目的

### 1. 图像轮廓提取（Wrap Hull Generation）

**核心功能**：从 SVG/图片的 alpha 通道提取轮廓，生成可用于文本环绕的多边形。

**处理流程**：
1. 加载图像并解码
2. 缩放到最大 320px 维度（性能优化）
3. 提取每行的左右边界（基于 alpha 阈值 12）
4. 应用平滑算法（mean 或 envelope 模式）
5. 采样并生成归一化多边形点（0-1 范围）
6. 可选：应用凸包算法

### 2. 多边形变换

**功能**：将归一化的轮廓点变换到实际布局位置和旋转角度。

```typescript
transformWrapPoints(points: Point[], rect: Rect, angle: number): Point[]
```

支持：
- 缩放到目标矩形
- 平移到目标位置
- 绕中心点旋转

### 3. 点包含检测

**射线投射算法**（Ray Casting）：
```typescript
isPointInPolygon(points: Point[], x: number, y: boolean): boolean
```

用于检测鼠标是否悬停在 logo 上，实现交互效果。

### 4. 文本行槽位计算

**核心算法**：`carveTextLineSlots()`

**问题**：给定一个允许的区间（如列宽）和一组被阻挡的区间（如 logo 占据的空间），计算剩余可用的文本槽位。

**示例**：
```
基础区间:    80..............................420
阻挡区间:           200..............310
结果槽位:    80......200  310..............420
```

## 具体技术实现

### 类型定义

```typescript
export type Rect = { x: number; y: number; width: number; height: number }
export type Interval = { left: number; right: number }
export type Point = { x: number; y: number }
export type WrapHullMode = 'mean' | 'envelope'
export type WrapHullOptions = {
  smoothRadius: number   // 平滑半径（像素）
  mode: WrapHullMode     // 'mean' 平均或 'envelope' 包络
  convexify?: boolean    // 是否应用凸包
}
```

### 轮廓提取算法

**1. 图像预处理**：
```typescript
const maxDimension = 320
const aspect = image.naturalWidth / image.naturalHeight
const width = aspect >= 1 ? maxDimension : Math.max(64, Math.round(maxDimension * aspect))
const height = aspect >= 1 ? Math.max(64, Math.round(maxDimension / aspect)) : maxDimension
```

**2. Alpha 通道扫描**：
```typescript
const alphaThreshold = 12
for (let y = 0; y < height; y++) {
  let left = -1, right = -1
  for (let x = 0; x < width; x++) {
    const alpha = data[(y * width + x) * 4 + 3]!
    if (alpha < alphaThreshold) continue
    if (left === -1) left = x
    right = x
  }
  if (left !== -1 && right !== -1) {
    lefts[y] = left
    rights[y] = right + 1
  }
}
```

**3. 平滑处理**：
```typescript
for (let offset = -options.smoothRadius; offset <= options.smoothRadius; offset++) {
  const sampleIndex = y + offset
  if (sampleIndex < 0 || sampleIndex >= height) continue
  const left = lefts[sampleIndex]
  const right = rights[sampleIndex]
  if (left == null || right == null) continue
  leftSum += left
  rightSum += right
  leftEdge = Math.min(leftEdge, left)
  rightEdge = Math.max(rightEdge, right)
  count++
}

switch (options.mode) {
  case 'envelope':
    smoothedLefts[y] = leftEdge
    smoothedRights[y] = rightEdge
    break
  case 'mean':
    smoothedLefts[y] = leftSum / count
    smoothedRights[y] = rightSum / count
    break
}
```

**4. 采样与归一化**：
```typescript
const step = Math.max(1, Math.floor(validRows.length / 52))
for (let index = 0; index < validRows.length; index += step) {
  sampledRows.push(validRows[index]!)
}

// 生成归一化点
points.push({
  x: (smoothedLefts[y]! - boundLeft) / boundWidth,
  y: ((y + 0.5) - boundTop) / boundHeight,
})
```

### 凸包算法（Graham Scan 变种）

```typescript
function makeConvexHull(points: Point[]): Point[] {
  if (points.length <= 3) return points
  
  // 按 x 坐标排序
  const sorted = [...points].sort((a, b) => (a.x - b.x) || (a.y - b.y))
  
  // 构建下凸包
  const lower: Point[] = []
  for (const point of sorted) {
    while (lower.length >= 2 && cross(lower[lower.length - 2]!, lower[lower.length - 1]!, point) <= 0) {
      lower.pop()
    }
    lower.push(point)
  }
  
  // 构建上凸包
  const upper: Point[] = []
  for (let i = sorted.length - 1; i >= 0; i--) {
    const point = sorted[i]!
    while (upper.length >= 2 && cross(upper[upper.length - 2]!, upper[upper.length - 1]!, point) <= 0) {
      upper.pop()
    }
    upper.push(point)
  }
  
  return lower.slice(0, -1).concat(upper.slice(0, -1))
}
```

### 槽位切割算法

```typescript
export function carveTextLineSlots(base: Interval, blocked: Interval[]): Interval[] {
  let slots: Interval[] = [base]
  
  for (const interval of blocked) {
    const next: Interval[] = []
    for (const slot of slots) {
      // 无重叠，保留原槽位
      if (interval.right <= slot.left || interval.left >= slot.right) {
        next.push(slot)
        continue
      }
      // 左剩余
      if (interval.left > slot.left) {
        next.push({ left: slot.left, right: interval.left })
      }
      // 右剩余
      if (interval.right < slot.right) {
        next.push({ left: interval.right, right: slot.right })
      }
    }
    slots = next
  }
  
  // 过滤过窄的槽位（< 24px）
  return slots.filter(slot => slot.right - slot.left >= 24)
}
```

### 多边形 X 坐标计算（用于区间提取）

```typescript
function getPolygonXsAtY(points: Point[], y: number): number[] {
  const xs: number[] = []
  let a = points[points.length - 1]
  
  for (const b of points) {
    // 检查边是否与水平线相交
    if ((a.y <= y && y < b.y) || (b.y <= y && y < a.y)) {
      // 线性插值计算 x 坐标
      xs.push(a.x + ((y - a.y) * (b.x - a.x)) / (b.y - a.y))
    }
    a = b
  }
  
  xs.sort((a, b) => a - b)
  return xs
}
```

## 依赖与外部交互

### 模块依赖

| 模块 | 关系 | 说明 |
|------|------|------|
| `dynamic-layout.ts` | 主要消费者 | 使用所有导出功能 |
| `bubbles.ts` | 无直接依赖 | 使用基础布局 API |

### Web API 使用

- **`Image`**：加载和解码图像
- **`OffscreenCanvas`**：高性能图像处理（如果可用）
- **`createImageBitmap`**（可选）：更高效的图像解码

### 缓存机制

```typescript
const wrapHullByKey = new Map<string, Promise<Point[]>>()

export function getWrapHull(src: string, options: WrapHullOptions): Promise<Point[]> {
  const key = `${src}::${options.mode}::${options.smoothRadius}::${options.convexify ? 'convex' : 'raw'}`
  const cached = wrapHullByKey.get(key)
  if (cached !== undefined) return cached
  
  const promise = makeWrapHull(src, options)
  wrapHullByKey.set(key, promise)
  return promise
}
```

基于参数的缓存避免重复计算相同轮廓。

## 风险、边界与改进建议

### 当前风险

1. **内存泄漏**：`wrapHullByKey` 是全局 Map，没有清理机制，长期运行的应用可能累积缓存
2. **图像加载失败**：`image.decode()` 可能失败，但没有错误恢复机制
3. **CORS 限制**：跨域图像可能导致 Canvas 污染，无法读取像素数据

### 边界情况

1. **完全透明图像**：`validRows.length === 0` 时抛出错误
2. **单像素图像**：凸包算法可能产生退化结果
3. **极大旋转角度**：`transformWrapPoints` 使用标准三角函数，支持任意角度
4. **负宽度区间**：`carveTextLineSlots` 假设输入有效，不处理负宽度

### 改进建议

1. **缓存清理**：
   ```typescript
   // 添加 LRU 缓存或大小限制
   const MAX_CACHE_SIZE = 100
   if (wrapHullByKey.size > MAX_CACHE_SIZE) {
     const firstKey = wrapHullByKey.keys().next().value
     wrapHullByKey.delete(firstKey)
   }
   ```

2. **错误处理**：
   ```typescript
   async function makeWrapHull(src: string, options: WrapHullOptions): Promise<Point[]> {
     try {
       await image.decode()
     } catch (error) {
       // 返回简单的矩形轮廓作为回退
       return [
         { x: 0, y: 0 }, { x: 1, y: 0 },
         { x: 1, y: 1 }, { x: 0, y: 1 }
       ]
     }
     // ...
   }
   ```

3. **CORS 支持**：
   ```typescript
   const image = new Image()
   image.crossOrigin = 'anonymous'
   image.src = src
   ```

4. **性能优化**：
   - 使用 Web Worker 进行轮廓计算
   - 使用 `OffscreenCanvas` 替代主线程 Canvas
   - 对 `getPolygonXsAtY` 使用空间索引（如 R-tree）优化大量查询

5. **更多平滑模式**：
   ```typescript
   export type WrapHullMode = 'mean' | 'envelope' | 'median' | 'gaussian'
   ```

6. **单元测试**：
   ```typescript
   // 测试 carveTextLineSlots
   test('carveTextLineSlots basic', () => {
     const base = { left: 0, right: 100 }
     const blocked = [{ left: 30, right: 70 }]
     const slots = carveTextLineSlots(base, blocked)
     expect(slots).toEqual([
       { left: 0, right: 30 },
       { left: 70, right: 100 }
     ])
   })
   ```
