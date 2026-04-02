# Research: pages/demos/accordion.ts

## 场景与职责

`accordion.ts` 是 Accordion 演示的业务逻辑层。它在浏览器端驱动一个四栏折叠面板，核心卖点是：**所有面板高度均通过 Pretext 的纯 JS 文本测量 API 计算，而非读取 DOM（`offsetHeight` / `getBoundingClientRect`）**。这避免了布局抖动（layout thrashing），在 resize 或切换展开状态时无需触发浏览器重排即可立即得到精确高度。

## 功能点目的

1. **零 DOM 测量的高度计算**：利用 `prepare()` 缓存文本宽度 + `layout()` 算术折行，实时算出每个 `.accordion-copy` 对应的行数与总高。  
2. **响应式重算**：窗口 resize、字体加载完成（`document.fonts.ready`）后自动通过 `requestAnimationFrame` 重新计算并更新高度。  
3. **状态管理**：维护当前展开项 `openItemId` 与点击事件队列 `clickedItemId`，在渲染帧中统一消纳。  
4. **元信息展示**：在 `.accordion-meta` 中显示“Measurement: N lines · H px”，直观证明计算来源。

## 具体技术实现

### 数据模型

```ts
type AccordionItem = { id: string; title: string; text: string }
type State = { openItemId: string | null; events: { clickedItemId: string | null } }
type DomCache = { list: HTMLElement; items: AccordionItemDom[] }
```

- `items` 数组硬编码 4 条内容，其中第 4 条包含混合文本（中文、阿拉伯文、emoji、URL、窄不间断空格 `\u202F`、软连字符 `\u00AD`），用于展示 Pretext 的国际化与特殊字符处理能力。

### 核心流程

1. **初始化（`boot`）**
   - 查询 `#list` 及其子节点，构建 `DomCache`。
   - `initializeStaticContent()` 将 `items` 的 `title` 和 `text` 写入对应 DOM 节点。
   - 绑定点击委托事件：点击 `.accordion-toggle` 时记录 `clickedItemId` 并 `scheduleRender()`。
   - 监听 `document.fonts.ready` 与 `window.resize`，均触发重渲染。

2. **字体缓存（`refreshPrepared`）**
   - 读取第一个 `.accordion-copy` 的 `getComputedStyle`，拼接成 CSS `font` 字符串。
   - 若字体变化，对每个 `item.text` 调用 `prepare(text, font)`，结果存入 `preparedCache.items`。

3. **渲染帧（`render`）**
   - 读取 `.accordion-copy` 的 `width`（**唯一一次 DOM 几何读取**，用于得到可用内容宽度）和 `.accordion-inner` 的上下 padding。
   - 处理点击状态：若 `clickedItemId` 与当前 `openItemId` 相同则收起，否则展开。
   - 遍历 4 个 prepared 文本，调用 `layout(prepared, contentWidth, lineHeight)` 得到 `lineCount` 与 `height`。
   - 计算 `panelHeights = Math.ceil(height + paddingY)`，并更新 DOM：
     - `.accordion-body.style.height`
     - `.accordion-glyph.style.transform`（旋转箭头）
     - `.accordion-toggle.setAttribute('aria-expanded')`
     - `.accordion-meta.textContent`

### 关键工具函数

- `getFontFromStyles(styles)`：兼容 `styles.font` 为空时，手动拼接 `fontStyle fontVariant fontWeight fontSize / lineHeight fontFamily`。
- `parsePx(value)`：安全解析 `getComputedStyle` 返回的像素值。

## 关键代码路径与文件引用

- **上游 API**：
  - `../../src/layout.ts`：导入 `layout`、`prepare`、`PreparedText`。
    - `prepare()` 内部调用 `src/analysis.ts`（分词、归一化）和 `src/measurement.ts`（canvas 测宽）。
    - `layout()` 内部调用 `src/line-break.ts` 的 `countPreparedLines()`。
- **同目录**：
  - `accordion.html` — 提供 DOM 骨架与 CSS 过渡。

## 依赖与外部交互

- **浏览器 API**：`document.fonts.ready`、`requestAnimationFrame`、`getComputedStyle`、`getBoundingClientRect`。
- **Pretext 库**：依赖 `layout()` 的折行算法与 `prepare()` 的 canvas 测量缓存。若库内部 emoji 修正或 CJK 分词逻辑变更，本演示的高度结果会同步变化。

## 风险、边界与改进建议

1. **DOM 读取点**：虽然宣传“无 DOM 测量”，但代码中仍在 `render()` 里读取了 `firstCopy.getBoundingClientRect().width` 与 `getComputedStyle`。这是**为了得到容器可用宽度**，而非测量文本高度本身，语义上合理，但文档表述需精确。  
2. **字体缓存失效**：`refreshPrepared` 仅在 `font` 字符串变化时重新 `prepare()`。若 CSS 中 `@font-face` 在 `document.fonts.ready` 之后动态加载新字重，缓存不会自动刷新。建议监听更细粒度的字体加载事件，但演示场景下风险极低。  
3. **硬编码数据**：`items` 数组内嵌在 TS 中，不利于国际化或 CMS 化。作为演示代码无可厚非。  
4. **性能边界**：4 个面板的 `prepare()` 在字体就绪后一次性执行，时间复杂度 O(N)（N 为总字符数）。对于生产环境中数百个面板，应考虑虚拟滚动或按需 `prepare()`。  
5. **改进建议**：可将 `preparedCache` 的键从纯 `font` 字符串扩展为 `font + contentWidth`，因为某些高级排版引擎可能随容器宽度改变字距或连字，但 Pretext 当前实现基于固定字宽模型，暂不需要。
