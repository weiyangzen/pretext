# 研究报告：scripts/package-smoke-test.ts

## 场景与职责

`scripts/package-smoke-test.ts` 是 Pretext 项目在发布前执行的关键“制品置信度”检查脚本。它的核心职责是：在本地模拟一次 `npm pack`，生成真实的 `.tgz` 发布包，并在临时隔离环境中验证该包能被纯 JavaScript ESM 消费者和 TypeScript 消费者正确安装、导入与类型检查。该脚本对应 `package.json` 中的 `package-smoke-test` 命令，是 DEVELOPMENT.md 明确推荐的“发布或变更打包配置前 quickest confidence check”。

## 功能点目的

1. **打包一致性验证**：确保 `npm pack` 生成的 tarball 结构与 `package.json` 的 `files`、`exports`、`main`、`types` 字段一致，没有遗漏 `dist/` 产物或声明文件。
2. **运行时可用性验证**：验证消费者可以按 ESM 方式 `import * as pretext from '@chenglou/pretext'`，且 `prepare`、`layout` 等核心 API 真实存在。
3. **类型声明可用性验证**：验证 TypeScript 消费者能正确获得类型提示，并且类型系统会对错误用法（如把 `string` 传给期望 `number` 的 `width` 参数）报编译错误。
4. **环境隔离与清理**：全程在系统临时目录下创建隔离项目，避免污染仓库；支持 `--keep-temp` 调试标志保留现场。

## 具体技术实现

### 关键流程

脚本采用自顶向下的顺序执行结构（Top-level await）：

1. **创建临时根目录**：使用 `mkdtemp(path.join(tmpdir(), 'pretext-package-smoke-'))` 在系统 tmp 下创建唯一目录。
2. **打包阶段** (`packPackage`)：
   - 在临时目录下创建 `pack/` 子目录。
   - 调用 `npm pack --pack-destination <packDir>` 生成 tarball。
   - 断言目录下恰好只有 1 个 `.tgz` 文件。
3. **JS ESM 消费验证** (`smokeJavaScriptEsm`)：
   - 创建临时项目目录 `js-esm/`，写入 `type: 'module'` 的 `package.json`。
   - 用 `npm install --ignore-scripts --no-package-lock <tarball>` 安装包。
   - 写入 `index.js`，执行命名空间导入并断言 `prepare` 和 `layout` 为函数。
   - 用 `node index.js` 运行，成功则打印 `js-esm ok`。
4. **TypeScript 消费验证** (`smokeTypeScript`)：
   - 创建临时项目目录 `ts/`，同样安装 tarball。
   - 写入 `tsconfig.json`：启用 `strict`、`noEmit`、`nodenext` 模块解析，与库源码风格对齐。
   - 第一轮：写入合法的 `index.ts`，调用 `tsc` 进行类型检查，期望通过。
   - 第二轮：故意写入错误用法（把字符串 `'100'` 传给 `layout` 的 `width`），再次调用 `tsc`。
   - 断言第二轮编译失败，且错误输出包含 `Argument of type 'string' is not assignable to parameter of type 'number'` 或类似文本。
   - 成功则打印 `ts ok`。
5. **清理**：若全部成功且未传入 `--keep-temp`，则递归删除临时根目录。

### 数据结构

- `tempRoot: string` — 临时根目录路径。
- `keepTemp: boolean` — 由 `process.argv.includes('--keep-temp')` 决定。
- `run()` 返回 `{ exitCode, stdout, stderr }`，支持 `allowFailure` 模式用于捕获预期失败的编译输出。

### 命令与协议

- `npm pack --pack-destination <dir>`：生成发布包。
- `npm install --ignore-scripts --no-package-lock <tarball>`：在隔离项目中安装包，避免执行生命周期脚本和生成 lockfile。
- `node index.js`：验证运行时。
- `<repo>/node_modules/.bin/tsc`（Windows 上为 `tsc.cmd`）：验证类型系统。
- 所有命令通过 `Bun.spawnSync()` 同步执行，便于立即失败或捕获输出。

## 关键代码路径与文件引用

- **脚本自身**：`scripts/package-smoke-test.ts`（191 行）。
- **打包配置**：`package.json`（`files`、`exports`、`main`、`types`、`scripts.package-smoke-test`）。
- **编译配置**：`tsconfig.build.json`（决定 `dist/` 产物内容，脚本间接验证其输出）。
- **被测库入口**：`src/layout.ts`（`prepare`、`layout` 的实现源文件，经 `tsconfig.build.json` 编译为 `dist/layout.js` + `dist/layout.d.ts`）。
- **调用方**：`package.json` 中的 `scripts` 定义；DEVELOPMENT.md 中的人工工作流说明。

## 依赖与外部交互

### 内部依赖

- `node:fs/promises`（`mkdir`、`mkdtemp`、`readdir`、`rm`、`writeFile`）
- `node:os`（`tmpdir`）
- `node:path`
- `Bun.spawnSync`（Bun 运行时内置）

### 外部依赖

- **Node.js / npm**：必须安装 npm 且 `npm` 命令在 PATH 中。
- **Bun**：脚本使用 `Bun.spawnSync`，只能在 Bun 环境下直接运行（`bun run scripts/package-smoke-test.ts`）。
- **TypeScript 编译器**：依赖仓库已安装的 `typescript` devDependency，通过 `node_modules/.bin/tsc` 调用。
- **目标包 `@chenglou/pretext`**：脚本硬编码了包名，与 `package.json` 的 `name` 字段一致。

## 风险、边界与改进建议

### 风险与边界

1. **平台差异**：`tscBinaryName()` 对 Windows 做了 `tsc.cmd` 兼容，但 `npm pack`/`npm install` 的路径拼接和 `Bun.spawnSync` 行为在 Windows 上仍需验证。
2. **Bun 锁定**：脚本深度依赖 `Bun.spawnSync`，无法直接在 Node.js 下运行，限制了 CI 环境的灵活性。
3. **npm 版本差异**：不同 npm 版本的 `pack` 输出文件名或元数据可能略有差异，但 tarball 数量断言（恰好 1 个）通常稳定。
4. **类型错误文本脆弱性**：对 `tsc` 错误消息的字符串匹配（`Argument of type 'string'...` / `Type 'string' is not assignable...`）在 TypeScript 大版本升级后可能因措辞变化而失效。
5. **无网络隔离**：`npm install` 会尝试解析 tarball 的 peerDependencies（若存在），可能触发网络请求；当前包无 peerDeps，因此风险低。
6. **清理竞态**：`finally` 块在同步抛异常时可能无法执行，导致临时目录残留；`--keep-temp` 可用于调试，但默认应可靠清理。

### 改进建议

1. **增强类型错误断言的鲁棒性**：与其匹配具体错误文案，不如解析 `tsc` 的 JSON 输出（`--pretty false` 或未来使用 `--errorFormat json`）并检查错误码 `2345`（参数类型不兼容）。
2. **支持 Node.js 原生运行**：将 `Bun.spawnSync` 替换为 `node:child_process` 的 `spawnSync`，使脚本在 Node.js 和 Bun 双环境下可运行。
3. **验证更多导出表面**：当前仅检查 `prepare` 和 `layout`。可扩展为遍历 `package.json` 的 `exports` 字段，自动生成对应 import 断言。
4. **添加 `types` 字段缺失检测**：显式检查 tarball 中是否包含 `./dist/layout.d.ts`，防止 `declaration: true` 被意外关闭。
5. **CI 集成**：建议在 `.github/workflows` 中加入 `bun run package-smoke-test` 作为发布前 gate，目前工作流中似乎未覆盖（需进一步确认 workflows 内容）。
