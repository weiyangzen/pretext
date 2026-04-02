# build-demo-site.ts 研究文档

## 场景与职责

`scripts/build-demo-site.ts` 是 Pretext 项目的**静态演示站点构建脚本**。它将 `pages/demos/` 下的多个 HTML 入口点通过 `bun build` 打包到 `site/` 目录，随后对产物进行路径重排与链接重写，使其适合作为纯静态站点（如 GitHub Pages）部署。

## 功能点目的

1. **多入口打包**：一次性将 9 个 demo HTML 页面及其依赖的 TypeScript/CSS/资源打包。
2. **目录结构规范化**：将 Bun 构建后的扁平/嵌套产物整理成以 slug 为子目录的静态站点结构（如 `bubbles.html` → `bubbles/index.html`）。
3. **资源链接重写**：修正相对 `src`/`href` 路径，确保子目录中的 HTML 能正确引用同级或上级资源。
4. **首页链接适配**：将 `index.html` 中对 `/demos/xxx` 的绝对链接改写成相对链接 `./xxx`，适配非根域部署。

## 具体技术实现

### 入口点与目标映射

```ts
const entrypoints = [
  'pages/demos/index.html',
  'pages/demos/accordion.html',
  'pages/demos/bubbles.html',
  'pages/demos/dynamic-layout.html',
  'pages/demos/editorial-engine.html',
  'pages/demos/justification-comparison.html',
  'pages/demos/masonry/index.html',
  'pages/demos/rich-note.html',
  'pages/demos/variable-typographic-ascii.html',
];

const targets = [
  { source: 'index.html', target: 'index.html' },
  { source: 'accordion.html', target: 'accordion/index.html' },
  { source: 'bubbles.html', target: 'bubbles/index.html' },
  // ... 其余类似
];
```

### 构建流程

1. **调用 Bun 构建**
   ```ts
   Bun.spawnSync(['bun', 'build', ...entrypoints, '--outdir', outdir], { cwd: root, stdout: 'inherit', stderr: 'inherit' })
   ```
   - `outdir` 为 `process.cwd() + '/site'`。
   - 构建失败时直接以相同 exit code 退出。

2. **移动并重写 HTML**
   对每个 `targets` 条目执行 `moveBuiltHtml(sourceRelativePath, targetRelativePath)`：
   - 通过 `resolveBuiltHtmlPath` 在 `site/` 或 `site/pages/demos/` 下定位构建产物。
   - 读取 HTML 文本。
   - `rebaseRelativeAssetUrls(html, sourcePath, targetPath)`：用正则 `\b(src|href)="([^"]+)"` 匹配相对路径（以 `.` 开头），将其从源文件视角解析为绝对路径，再计算到目标文件目录的相对路径，统一使用 `/` 分隔符。
   - `rewriteDemoLinksForStaticRoot(html, targetRelativePath)`：仅对 `index.html` 生效，将 `href="/demos/([^"]+)"` 替换为 `href="./$1"`。
   - 写入目标路径，按需创建父目录；若源路径与目标路径不同则删除源文件。

3. **清理中间目录**
   - `rm(path.join(outdir, 'pages'), { recursive: true, force: true })` 删除 Bun 构建残留的 `site/pages` 嵌套目录。

## 关键代码路径与文件引用

- 本文件：`scripts/build-demo-site.ts`
- 对应 npm script：`package.json` 中的 `"site:build": "rm -rf site && bun run scripts/build-demo-site.ts"`
- 被构建的页面源文件：`pages/demos/index.html`, `pages/demos/bubbles.html`, `pages/demos/dynamic-layout.html` 等
- 输出目录：`site/`

## 依赖与外部交互

- **Bun 运行时**：依赖 `Bun.spawnSync` 和 `Bun.file().exists()`。
- **Node 核心模块**：`node:fs/promises`, `node:path`。
- **无外部网络依赖**：纯本地构建脚本。

## 风险、边界与改进建议

1. **硬编码入口列表**：`entrypoints` 与 `targets` 需要手动同步。若新增 demo 页面但忘记加入列表，该页面不会进入站点。建议在 `pages/demos/` 目录使用 `Glob` 动态扫描 `*.html` 和 `*/index.html`。
2. **正则重写局限**：`rebaseRelativeAssetUrls` 仅处理 `src` 和 `href` 属性，且要求值以 `.` 开头。对于 CSS 中的 `url(...)`、`<link rel="icon">`、`<meta property="og:image">` 等不会重写。若 demo 页面引入复杂资源，可能出现 404。
3. **Bun 构建产物路径不稳定**：`resolveBuiltHtmlPath` 硬编码了两个候选路径（`site/relativePath` 和 `site/pages/demos/relativePath`），这是基于当前 Bun 构建器对 `pages/demos/index.html` 的处理行为。若 Bun 升级后输出结构变化，脚本会抛错。
4. **无缓存与增量构建**：每次运行都是全量 `rm -rf site` + `bun build`。对于 9 个入口点规模尚可，但若继续增长，构建时间会增加。可考虑引入 `--minify` 开关或增量构建策略。
5. **缺失 sitemap / 404 处理**：静态站点没有自动生成 sitemap 或 fallback `404.html`。GitHub Pages 部署时，深层路由刷新可能 404（虽然当前所有路由都是 `index.html` 子目录形式，刷新正常）。
