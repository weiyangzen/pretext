# emoji-test.html 研究文档

## 场景与职责

`pages/emoji-test.html` 是 Pretext 库的 emoji 宽度兼容性测试页面。它专门用于验证 Canvas `measureText()` 与 DOM 实际渲染在 emoji 字符上的宽度差异，这是解决 Pretext 核心精度问题的关键诊断工具。

**核心问题**：
- Chrome/Firefox 在 macOS 上，Canvas 测量 emoji 比 DOM 实际渲染更宽（小字号 <24px）
- 这种差异在 Apple Color Emoji 字体上尤为明显
- 差异值在给定字号下是恒定的（font-independent），可用于修正

该页面服务于：
- **emoji 修正验证**：确认修正值是否跨字体一致
- **新字号探测**：测试未覆盖字号的差异模式
- **浏览器兼容性**：对比不同浏览器的 emoji 渲染行为

## 功能点目的

### 1. 多字体/多字号扫描

测试 7 种字体族 × 11 种字号 × 52 个 emoji 的宽度差异：

```javascript
const fonts = [
  '"Helvetica Neue", Helvetica, Arial, sans-serif',
  'Georgia, "Times New Roman", serif',
  // ... 共 7 种
]

const sizes = [10, 12, 14, 15, 16, 18, 20, 22, 24, 28, 32]

const emoji = [
  '\u{1F600}', '\u{1F602}',  // 基础表情
  '\u{1F44D}', '\u{1F44E}',  // 手势
  '\u{1F1FA}\u{1F1F8}',      // 国旗（区域指示符序列）
  '\u{1F44D}\u{1F3FB}',      // 肤色变体
  // ... 共 52 个
]
```

### 2. 字体独立性验证

核心假设验证：**给定字号下，emoji 的 Canvas-DOM 差异在所有字体上相同**

验证逻辑：
```javascript
// 对每个字号，收集每个 emoji 在所有字体上的差异
const emojiDiffsByFont = new Map()  // emoji → Map<font, diff>

// 检查每个 emoji 的差异是否跨字体一致
for (const [e, fontDiffs] of emojiDiffsByFont) {
  const vals = [...fontDiffs.values()]
  const variance = Math.max(...vals) - Math.min(...vals)
  if (variance > 0.01) {
    allSame = false  // 发现字体相关差异
  }
}
```

### 3. 修正表生成

自动生成可用于 `measurement.ts` 的修正表：

```javascript
// 输出示例：
// 10px: +0.67px (52/52 emoji)
// 12px: +0.33px (48/52 emoji)
// 16px: no correction needed
// 24px+: no correction needed
```

## 具体技术实现

### 测试架构

```javascript
// OffscreenCanvas 用于 Canvas 测量
const canvas = new OffscreenCanvas(1, 1)
const ctx = canvas.getContext('2d')

// 隐藏容器用于 DOM 测量
const container = document.createElement('div')
container.style.cssText = 'position:absolute;top:-9999px;visibility:hidden;white-space:nowrap'
document.body.appendChild(container)
```

### 测量流程

```javascript
for (const fontFamily of fonts) {
  for (const sz of sizes) {
    ctx.font = `${sz}px ${fontFamily}`
    
    // 批量创建 span
    const spans = []
    for (const e of emoji) {
      const span = document.createElement('span')
      span.style.font = `${sz}px ${fontFamily}`
      span.style.display = 'inline-block'
      span.textContent = e
      container.appendChild(span)
      spans.push(span)
    }
    
    // 批量测量
    for (let i = 0; i < emoji.length; i++) {
      const canvasW = ctx.measureText(emoji[i]).width
      const domW = spans[i].getBoundingClientRect().width
      const diff = canvasW - domW
      // 记录差异
    }
    
    container.innerHTML = ''  // 清理
  }
}
```

### 输出格式

页面生成三类输出：

1. **逐字体逐字号报告**：
   ```
   "Helvetica Neue", Helvetica, Arial, sans-serif
     10px: 0/52 match  diffs: +0.67
     12px: 4/52 match  diffs: +0.33
     16px: 52/52 match
   ```

2. **字体独立性分析**：
   ```
   ========== ANALYSIS: Font independence ==========
   10px: CONSTANT across fonts
   12px: CONSTANT across fonts
   ```

3. **修正表**：
   ```
   ========== CORRECTION TABLE ==========
   10px: +0.67px (52/52 emoji)
   12px: +0.33px (48/52 emoji)
   16px: no correction needed
   ```

4. **16px 详细分析**：
   ```
   Mismatched (12): 👍(+0.5) 👎(+0.5) ...
   Matched (40): 😀 😂 🤩 ...
   ```

## 关键代码路径与文件引用

### 依赖关系

```
emoji-test.html
├── 内联 JavaScript (无外部依赖)
├── OffscreenCanvas API
└── DOM API
```

### 相关文件

| 文件 | 关系 | 说明 |
|------|------|------|
| `src/measurement.ts` | 消费方 | 使用本页生成的修正值 |
| `src/layout.ts` | 消费方 | 通过 measurement.ts 应用修正 |

## 依赖与外部交互

### 浏览器 API

| API | 用途 |
|-----|------|
| `OffscreenCanvas` | 离屏 Canvas 测量 |
| `CanvasRenderingContext2D.measureText()` | Canvas 宽度测量 |
| `document.createElement('span')` | DOM 测量容器 |
| `span.getBoundingClientRect()` | DOM 宽度测量 |

### 无外部依赖

该页面是自包含的，不依赖 Pretext 库的其他部分，可独立运行用于快速验证。

## 风险、边界与改进建议

### 已知发现

根据页面输出和 `AGENTS.md`：

1. **macOS Apple Color Emoji**：
   - 小字号（<24px）Canvas 测量比 DOM 宽
   - 差异值在给定字号下是恒定的
   - 24px 及以上通常无需修正

2. **Safari**：
   - Canvas 和 DOM 都更宽（相对于 fontSize）
   - 但两者一致，所以修正值为 0

3. **字体独立性**：
   - 差异值在给定字号下跨字体一致
   - 验证了修正表的可行性

### 边界情况

| 场景 | 处理 |
|------|------|
| 区域指示符序列（国旗） | 作为单个 grapheme 测试 |
| 肤色变体 | 包含在测试集中 |
| ZWJ 序列（家庭、职业） | 当前未全面覆盖 |
| 新 emoji | 需要定期更新测试集 |

### 改进建议

1. **扩展测试集**：
   - 添加更多 ZWJ 序列（👨‍👩‍👧‍👦, 👩‍💻 等）
   - 添加最新 Unicode 版本的 emoji
   - 添加文本呈现选择器变体（☀️ vs ☀）

2. **自动化集成**：
   - 将结果自动写入 `src/measurement.ts`
   - CI 中检测新字号/新浏览器的差异

3. **跨平台测试**：
   - Windows（Segoe UI Emoji）
   - Android（Noto Color Emoji）
   - Linux（ varies ）

4. **可视化**：
   - 添加差异热力图
   - 显示实际渲染对比

### 当前修正表（推测）

基于页面逻辑和 `AGENTS.md` 描述：

| 字号 | 修正值 | 说明 |
|------|--------|------|
| 10px | +0.67px | 最大修正 |
| 12px | +0.33px | 中等修正 |
| 14px | ~0.17px | 小修正 |
| 16px+ | 0 | 无需修正 |

这些值在 `measurement.ts` 中通过 `getEngineProfile()` 和 `getCorrectedSegmentWidth()` 应用。
