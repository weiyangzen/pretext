# rich-note.ts 研究文档

## 场景与职责

`rich-note.ts` 是 Pretext 库的一个高级演示页面，展示如何在富文本场景中混合使用多种文本样式（普通文本、链接、代码片段）和原子级 UI 组件（chips）。该演示模拟了一个类似项目管理工具中的任务卡片界面，其中包含：

- 普通正文文本（body）
- 链接文本（link）
- 代码片段（code）
- 各种功能性标签 chips（mention、status、priority、time、count）

该页面的核心职责是展示 Pretext 的 `layoutNextLine()` API 如何处理**异构内联内容**的流式布局——即文本和不可拆分的原子组件（chips）混合时的换行策略。

## 功能点目的

### 1. 异构内联项目（Inline Items）系统

页面定义了两种内联项目类型：

```typescript
type TextInlineItem = {
  kind: 'text'
  className: string
  chromeWidth: number      // 代码片段的额外宽度（padding/border）
  endCursor: LayoutCursor  // 文本结束位置
  fullText: string
  fullWidth: number
  leadingGap: number       // 前导空格宽度
  prepared: PreparedTextWithSegments
}

type ChipInlineItem = {
  kind: 'chip'
  className: string
  leadingGap: number
  text: string
  width: number            // 包含 CHIP_CHROME_WIDTH（22px）
}
```

**设计目的**：
- Chips 是原子级的，不允许在行内拆分
- 文本段可以在任意边界换行（通过 `layoutNextLine`）
- 代码片段有额外的 chrome 宽度需要计入布局计算

### 2. 交互式宽度调节

通过滑块控件实时调整文本区域宽度（260px - 760px），观察：
- Chips 保持完整，不会被截断
- 文本自然换行
- 代码片段和链接样式保持

### 3. 流式布局算法

核心布局逻辑 `layoutInlineItems()` 实现了以下策略：
- 优先尝试将整个项目放入当前行
- 文本项目可以部分换行（通过 `layoutNextLine`）
- Chips 必须整体放入，否则换行
- 维护 `textCursor` 状态实现跨行文本延续

## 具体技术实现

### 关键数据结构

```typescript
// 样式配置
const TEXT_STYLES = {
  body: { className: 'frag frag--body', chromeWidth: 0, font: BODY_FONT },
  code: { className: 'frag frag--code', chromeWidth: 14, font: CODE_FONT },
  link: { className: 'frag frag--link', chromeWidth: 0, font: LINK_FONT },
} satisfies Record<TextStyleName, TextStyleModel>

const CHIP_CLASS_NAMES = {
  count: 'frag chip chip--count',
  mention: 'frag chip chip--mention',
  priority: 'frag chip chip--priority',
  status: 'frag chip chip--status',
  time: 'frag chip chip--time',
} satisfies Record<ChipTone, string>
```

### 核心流程

**1. 准备阶段（prepareInlineItems）**

```typescript
function prepareInlineItems(specs: RichInlineSpec[]): InlineItem[]
```

- 遍历 `INLINE_SPECS` 数组
- 对文本段：使用 `prepareWithSegments()` 预处理，通过 `layoutNextLine()` 测量整行宽度
- 对 chips：使用 `prepareWithSegments()` 测量标签文本，加上 `CHIP_CHROME_WIDTH`
- 处理前导/后导空格，设置 `leadingGap`

**2. 布局阶段（layoutInlineItems）**

```typescript
function layoutInlineItems(items: InlineItem[], maxWidth: number): RichLine[]
```

算法逻辑：
```
对于每个项目：
  如果是 chip：
    检查是否能在当前行容纳（leadingGap + width）
    如果不能，结束当前行，开始新行
    chip 必须整体放入
  
  如果是 text：
    检查是否已完全消费（cursorsMatch）
    计算可用宽度（减去 leadingGap 和 chromeWidth）
    尝试放入整段文本
    如果不能，使用 layoutNextLine() 获取部分行
    更新 textCursor 以支持跨行延续
```

**3. 渲染阶段（renderBody）**

- 使用绝对定位的 `line-row` div
- 每行内的 fragments 使用 inline-block span
- 通过 `marginLeft` 应用 `leadingGap`

### 关键代码路径

**空格宽度测量**：
```typescript
function measureCollapsedSpaceWidth(font: string): number {
  const joinedWidth = measureSingleLineWidth(prepareWithSegments('A A', font))
  const compactWidth = measureSingleLineWidth(prepareWithSegments('AA', font))
  return Math.max(0, joinedWidth - compactWidth)
}
```

通过比较 "A A" 和 "AA" 的宽度差来精确测量空格宽度。

**文本行测量**：
```typescript
function measureSingleLineWidth(prepared: PreparedTextWithSegments): number {
  let maxWidth = 0
  walkLineRanges(prepared, UNBOUNDED_WIDTH, line => {
    if (line.width > maxWidth) maxWidth = line.width
  })
  return maxWidth
}
```

使用 `walkLineRanges` 配合极大宽度（100,000px）来获取单行宽度。

## 依赖与外部交互

### 核心依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `../../src/layout.ts` | `layoutNextLine`, `prepareWithSegments`, `walkLineRanges` | 文本布局和测量 |
| `rich-note.html` | DOM 结构、CSS 样式 | 页面渲染 |

### DOM 交互

```typescript
const domCache = {
  root: document.documentElement,
  noteBody: getRequiredDiv('note-body'),
  widthSlider: getRequiredInput('width-slider'),
  widthValue: getRequiredSpan('width-value'),
}
```

- 监听 `input` 事件调整宽度
- 监听 `resize` 事件响应视口变化
- 使用 `requestAnimationFrame` 批量渲染

### 与 Pretext API 的交互

1. **`prepareWithSegments(text, font)`**：预处理文本，获取带段落的准备对象
2. **`layoutNextLine(prepared, startCursor, maxWidth)`**：获取下一行布局结果
3. **`walkLineRanges(prepared, maxWidth, callback)`**：遍历所有行范围（用于测量）

## 风险、边界与改进建议

### 当前风险

1. **硬编码常量**：`CHIP_CHROME_WIDTH = 22` 和 `TEXT_STYLES.code.chromeWidth = 14` 需要与 CSS 保持同步
2. **字体回退**：如果指定字体不可用，测量可能不准确
3. **无限循环风险**：`layoutInlineItems` 中的 `break lineLoop` 逻辑需要确保进度推进

### 边界情况

1. **空文本段**：`trimmedText.length === 0` 时跳过，但 `pendingGap` 状态仍需维护
2. **极窄宽度**：当 `maxWidth` 小于单个字符宽度时，算法仍能工作但会产生大量短行
3. **跨语言文本**：混合英文、中文（北京）、阿拉伯文（جاهز）需要 Pretext 的国际化支持

### 改进建议

1. **动态 chrome 宽度测量**：
   ```typescript
   // 建议：从实际 DOM 元素测量而非硬编码
   const codeElement = document.createElement('span')
   codeElement.className = 'frag frag--code'
   codeElement.textContent = ' '
   document.body.appendChild(codeElement)
   const chromeWidth = codeElement.getBoundingClientRect().width
   document.body.removeChild(codeElement)
   ```

2. **支持更多 chip 类型**：当前 chip 类型是硬编码的，可以通过配置对象扩展

3. **性能优化**：
   - 对 `measureCollapsedSpaceWidth` 的结果进行缓存（已实现）
   - 使用 `ResizeObserver` 替代 `window.resize` 事件

4. **可访问性**：
   - 为 chips 添加 `role="button"` 或 `role="status"`
   - 为滑块添加 `aria-valuemin`, `aria-valuemax`, `aria-valuenow`

5. **测试覆盖**：
   - 添加单元测试验证 `layoutInlineItems` 在各种宽度下的行为
   - 测试极端情况（空内容、超长单词、零宽度）
