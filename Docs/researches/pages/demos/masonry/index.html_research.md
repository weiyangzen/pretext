# 研究文档：pages/demos/masonry/index.html

## 场景与职责

`pages/demos/masonry/index.html` 是 Pretext 库 **Masonry（瀑布流）布局演示** 的入口 HTML 文件。它作为独立的浏览器页面存在，用于向用户展示：在完全不依赖 CSS 多列布局或 Grid/Flex 自动排布的前提下，仅通过 Pretext 的纯 JS 文本测量与布局 API，即可实现一个高性能、响应式的瀑布流卡片列表。该页面是项目 GitHub Pages 静态站点（`site/`）的组成部分，通过 `scripts/build-demo-site.ts` 参与构建与发布流程。

## 功能点目的

1. **提供演示容器**：为 `index.ts` 中的 TypeScript 逻辑提供挂载点（`<body>`）和模块脚本加载通道（`<script type="module" src="./index.ts"></script>`）。
2. **定义视觉基线**：通过内嵌 `<style>` 设定全局重置、页面背景、字体栈以及 `.card` 类的基础样式，确保卡片在绝对定位布局下具备一致的视觉呈现。
3. **保证移动端可用性**：通过 `viewport` meta 标签禁止用户缩放（`maximum-scale=1.0`），确保在移动设备上获得稳定的 `clientWidth` 读取和一致的布局计算。

## 具体技术实现

### 页面结构

```html
<!DOCTYPE html>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0" />
<body>
  <style>...</style>
  <script type="module" src="./index.ts"></script>
</body>
```

- 页面极其精简，没有 `<html>` 或 `<head>` 标签（HTML5 允许这种省略），以减少样板代码。
- 使用 `type="module"` 加载 `./index.ts`，依赖构建工具（Bun）在 dev server 或 `bun build` 阶段完成 TS→JS 的转换与打包。

### CSS 样式细节

| 选择器 | 关键属性 | 目的 |
|--------|----------|------|
| `*` | `margin: 0; padding: 0; box-sizing: border-box;` | 全局重置，消除浏览器默认差异 |
| `body` | `font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; background: #f0f0f0; overflow-y: scroll;` | 统一字体栈（与 TS 中 `prepare` 的 `font` 参数保持一致），强制始终显示垂直滚动条以避免布局跳动 |
| `.card` | `position: absolute; border-radius: 8px; padding: 16px; font-size: 15px; line-height: 22px; overflow: hidden;` | 卡片采用绝对定位，这是 masonry 布局的核心技术选择——TS 端计算好每个卡片的 `left`/`top`/`width`/`height` 后，直接通过内联样式写入 DOM |

### 构建与发布链路

- **开发阶段**：通过 `bun start`（或 `bun run start:watch`）启动的 dev server 直接服务该文件，Bun 会透明处理 `.ts` 模块导入。
- **发布阶段**：`scripts/build-demo-site.ts` 将该文件列入 `entrypoints` 数组，调用 `bun build` 将其与依赖打包输出到 `site/` 目录；随后通过 `moveBuiltHtml` 将产物重定位到 `site/masonry/index.html`，并重写相对资源路径。

## 关键代码路径与文件引用

- **自身路径**：`pages/demos/masonry/index.html`
- **直接加载的脚本**：`./index.ts`（同目录下的 `pages/demos/masonry/index.ts`）
- **构建脚本引用**：`scripts/build-demo-site.ts` 第 13 行（`entrypoints`）和第 38 行（`targets`）
- **入口页引用**：`pages/demos/index.html` 第 135 行通过 `<a class="card" href="/demos/masonry">` 链接到本演示

## 依赖与外部交互

| 依赖项 | 关系 | 说明 |
|--------|------|------|
| `pages/demos/masonry/index.ts` | 被加载方 | 页面唯一的逻辑入口，负责 masonry 计算、DOM 操作与事件监听 |
| `scripts/build-demo-site.ts` | 构建消费者 | 将本 HTML 作为构建入口点，产出静态站点文件 |
| `pages/demos/index.html` | 导航来源 | 演示列表页中的链接指向本页面 |
| Bun 运行时/构建器 | 外部工具 | 负责处理 `type="module"` 中的 `.ts` 导入，无需预编译 |

## 风险、边界与改进建议

### 风险

1. **CSS 与 JS 的字体/尺寸耦合**：`body` 的 `font-family`、`font-size`、`line-height` 必须与 `index.ts` 中传给 `prepare()` 的 `font` 字符串以及 `lineHeight` 常量严格保持一致。若一方修改而另一方未同步，会导致 Pretext 测量的文本高度与实际 DOM 渲染高度不一致，从而出现卡片重叠或底部留白。
2. **`overflow: hidden` 的裁剪风险**：`.card` 设置了 `overflow: hidden`，若某张卡片在特定宽度下实际渲染高度略高于 JS 计算值（例如浏览器亚像素舍入差异），底部文本可能被截断而不易察觉。
3. **viewport 缩放限制的可访问性问题**：`maximum-scale=1.0` 对视力障碍用户不友好，现代 Web 可访问性指南通常建议避免完全禁止缩放。

### 边界

- 本文件本身不包含任何动态逻辑，仅作为静态容器。所有动态行为（resize、scroll、虚拟列表裁剪）完全委托给 `index.ts`。
- 由于 `<script>` 使用 `type="module"`，在非常老旧的浏览器中无法运行；但 Pretext 本身的目标环境是现代浏览器，因此这是可接受的。

### 改进建议

1. **CSS 自定义属性同步**：将 `font-size`、`line-height`、`padding`、`gap`、`maxColWidth` 等常量提取为 CSS 自定义属性（`:root { --masonry-font-size: 15px; ... }`），并在 `index.ts` 中通过 `getComputedStyle(document.documentElement)` 读取，消除硬编码耦合。
2. **移除 `maximum-scale=1.0`**：将 viewport 改为 `width=device-width, initial-scale=1.0`，提升可访问性。
3. **增加 `title` 和语义化标签**：补充 `<title>Masonry Layout Demo | Pretext</title>` 以及 `<main>` 容器，改善 SEO 和可访问性树。
4. **考虑 `contain: layout` 优化**：在 `.card` 或容器上添加 `contain: layout paint`，提示浏览器减少重排范围，与已有的绝对定位策略配合可进一步提升滚动性能。
