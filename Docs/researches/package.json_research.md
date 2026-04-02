# package.json 研究文档

## 场景与职责

`package.json` 是 Node.js/Bun 生态系统的核心配置文件，定义了 Pretext 包的元数据、入口点、脚本命令和依赖关系。作为 `@chenglou/pretext` 包的发布清单，它控制着库的构建、测试、发布和运行时行为。

## 功能点目的

### 1. 包元数据
```json
{
  "name": "@chenglou/pretext",
  "version": "0.0.4",
  "license": "MIT",
  "type": "module"
}
```
- **name**：作用域包名，发布到 npm 的 `@chenglou` 组织下
- **version**：当前版本 `0.0.4`（语义化版本）
- **license**：MIT 开源许可
- **type**：`"module"` 强制使用 ES Modules（替代 CommonJS）

### 2. 发布配置
```json
{
  "publishConfig": { "access": "public" },
  "main": "./dist/layout.js",
  "types": "./dist/layout.d.ts"
}
```
- **publishConfig.access**：允许公开发布（作用域包默认私有）
- **main**：CommonJS 入口点（向后兼容）
- **types**：TypeScript 类型定义入口点

### 3. 导出映射（Exports Map）
```json
{
  "exports": {
    ".": {
      "types": "./dist/layout.d.ts",
      "import": "./dist/layout.js",
      "default": "./dist/layout.js"
    },
    "./demos/*": "./pages/demos/*",
    "./assets/*": "./pages/assets/*",
    "./package.json": "./package.json"
  }
}
```
- **`.`**：主入口，支持类型提示和 ESM 导入
- **`./demos/*`**：公开演示页面资源，允许用户 `import '@chenglou/pretext/demos/bubbles'`
- **`./assets/*`**：静态资源（字体、图片等）
- **`./package.json`**：允许工具读取包元数据

### 4. 发布文件白名单
```json
{
  "files": [
    "CHANGELOG.md",
    "dist",
    "src",
    "!src/layout.test.ts",
    "!src/test-data.ts",
    "pages/demos",
    "pages/assets"
  ]
}
```
- 明确指定哪些文件进入 npm 包
- **包含**：变更日志、构建产物、源码、演示和资源
- **排除**：测试文件和测试数据（以 `!` 开头）

### 5. 脚本命令（Scripts）

#### 开发服务器
```json
{
  "start": "HOST=${HOST:-127.0.0.1}; PORT=3000; ... bun pages/*.html ...",
  "start:lan": "HOST=0.0.0.0 bun run start",
  "start:watch": "... --watch --no-clear-screen ..."
}
```
- **start**：启动开发服务器（默认 127.0.0.1:3000）
  - 自动检测并释放被占用的端口（`lsof` + `kill`）
  - 使用 Bun 内置服务器托管静态文件
- **start:lan**：允许局域网访问（0.0.0.0）
- **start:watch**：启用热重载（HMR）

#### 构建与检查
```json
{
  "build:package": "rm -rf dist && tsc -p tsconfig.build.json",
  "check": "tsc && oxlint --type-aware src",
  "prepack": "rm -rf dist && tsc -p tsconfig.build.json"
}
```
- **build:package**：构建发布包到 `dist/`
- **check**：类型检查 + 类型感知 lint
- **prepack**：npm pack/publish 前自动构建

#### 准确性检查（Accuracy）
```json
{
  "accuracy-check": "bun run scripts/accuracy-check.ts",
  "accuracy-check:firefox": "ACCURACY_CHECK_BROWSER=firefox bun run scripts/accuracy-check.ts",
  "accuracy-check:safari": "ACCURACY_CHECK_BROWSER=safari bun run scripts/accuracy-check.ts",
  "accuracy-snapshot": "bun run scripts/accuracy-check.ts --full --output=accuracy/chrome.json"
}
```
- 多浏览器支持（Chrome/Firefox/Safari）
- 环境变量控制目标浏览器
- 生成 JSON 快照供回归测试

#### 基准测试（Benchmark）
```json
{
  "benchmark-check": "bun run scripts/benchmark-check.ts",
  "benchmark-check:safari": "BENCHMARK_CHECK_BROWSER=safari bun run scripts/benchmark-check.ts"
}
```
- 性能回归测试
- 同样支持多浏览器

#### 语料库检查（Corpus）
```json
{
  "corpus-check": "bun run scripts/corpus-check.ts",
  "corpus-sweep": "bun run scripts/corpus-sweep.ts",
  "corpus-font-matrix": "bun run scripts/corpus-font-matrix.ts",
  "corpus-status:refresh": "...多步骤刷新..."
}
```
- **corpus-check**：单宽度诊断
- **corpus-sweep**：多宽度扫描
- **corpus-font-matrix**：多字体矩阵测试
- **corpus-status:refresh**：刷新所有语料库状态快照

#### 兼容性别名
```json
{
  "gatsby-check": "bun run scripts/gatsby-check.ts",
  "gatsby-sweep": "bun run scripts/gatsby-sweep.ts"
}
```
- 历史兼容性保留
- 实际映射到 `corpus-check --id=en-gatsby-opening`

#### 其他工具脚本
```json
{
  "package-smoke-test": "bun run scripts/package-smoke-test.ts",
  "pre-wrap-check": "bun run scripts/pre-wrap-check.ts",
  "probe-check": "bun run scripts/probe-check.ts",
  "status-dashboard": "bun run scripts/status-dashboard.ts",
  "site:build": "rm -rf site && bun run scripts/build-demo-site.ts"
}
```

### 6. 开发依赖
```json
{
  "devDependencies": {
    "@types/bun": "latest",
    "oxlint": "^1.51.0",
    "oxlint-tsgolint": "^0.15.0",
    "tsgolint": "^0.0.1",
    "typescript": "6.0.2"
  }
}
```
- **@types/bun**：Bun 运行时类型定义
- **oxlint + 插件**：高性能 TypeScript linter
- **typescript**：TypeScript 6.0（最新主要版本）

## 具体技术实现

### 端口管理逻辑
start 脚本中的端口释放逻辑：
```bash
pids=$(lsof -tiTCP:$PORT -sTCP:LISTEN 2>/dev/null)
if [ -n "$pids" ]; then
  echo "Freeing port $PORT: terminating $pids"
  kill $pids 2>/dev/null || true
  sleep 1
  # 如果仍在占用，强制 kill -9
fi
```
- 使用 `lsof` 查找监听进程
- 优雅终止（SIGTERM）后等待 1 秒
- 必要时强制终止（SIGKILL）

### 构建流程
```
npm publish
└── prepack hook
    └── rm -rf dist && tsc -p tsconfig.build.json
        └── 读取 tsconfig.build.json
            └── 输出到 dist/
```

### 环境变量传递
```bash
ACCURACY_CHECK_BROWSER=firefox bun run scripts/accuracy-check.ts
```
- 脚本通过 `process.env.ACCURACY_CHECK_BROWSER` 读取
- 默认值为 `'chrome'`

## 关键代码路径与文件引用

### 核心文件关系
| 文件 | 关系 |
|------|------|
| `tsconfig.build.json` | `build:package` 和 `prepack` 使用的配置 |
| `tsconfig.json` | `check` 脚本使用的配置 |
| `oxlintrc.json` | `check` 脚本中 `oxlint` 的配置来源 |
| `scripts/*.ts` | 各脚本命令的实现 |
| `pages/` | `start` 脚本服务的静态文件 |
| `dist/` | 构建输出目录（gitignore 排除，但发布包含） |

### 发布流程
```
1. 开发者运行: npm publish
2. prepack hook 触发: rm -rf dist && tsc -p tsconfig.build.json
3. npm 根据 files 字段打包
4. 包包含: CHANGELOG.md, dist/, src/, pages/demos/, pages/assets/
5. 包排除: src/layout.test.ts, src/test-data.ts (通过 ! 语法)
```

### 入口点解析
```javascript
// 用户代码
import { prepare, layout } from '@chenglou/pretext';

// Node.js 解析过程
// 1. 读取 package.json exports["."]
// 2. TypeScript: 使用 ./dist/layout.d.ts 进行类型检查
// 3. Runtime: 导入 ./dist/layout.js
```

## 依赖与外部交互

### 与 Bun 的交互
- 使用 Bun 作为包管理器和运行时
- `bun.lock` 替代 `package-lock.json`
- Bun 原生支持 TypeScript，无需 ts-node

### 与 npm 的交互
- 发布到 npm registry（`@chenglou` 作用域）
- 遵循 npm 包规范（main, types, files, exports）

### 与 GitHub Actions 的交互
`.github/workflows/pages.yml`:
```yaml
- run: bun install --frozen-lockfile
- run: bun run site:build  # 使用 package.json 中的脚本
```

### 与浏览器的交互
- 准确性检查和基准测试通过浏览器自动化（Playwright/Puppeteer）
- 环境变量控制目标浏览器

## 风险、边界与改进建议

### 风险点

1. **版本号管理**
   - 当前 `0.0.4` 表明仍在早期开发
   - 需要建立版本发布流程

2. **@types/bun 使用 latest**
   - 可能导致不同开发者环境类型定义不一致
   - 建议：锁定到具体版本

3. **端口硬编码**
   - `PORT=3000` 硬编码在 start 脚本中
   - 如果 3000 被系统占用，自动释放逻辑可能干扰其他服务

4. **多浏览器测试依赖**
   - Safari 自动化需要 macOS 环境
   - CI（Ubuntu）可能无法运行 Safari 测试

### 边界情况

1. **Windows 兼容性**
   - start 脚本使用 `lsof` 和 shell 语法
   - Windows 开发者可能需要 WSL 或修改脚本

2. **src 包含在发布中**
   - 允许用户调试时查看源码
   - 增加包体积（权衡：调试便利 vs 下载大小）

3. **演示页面作为导出**
   - `./demos/*` 导出允许用户直接引用
   - 但演示代码可能包含未公开 API 的使用

### 改进建议

1. **版本锁定**
   ```json
   "@types/bun": "1.3.10"
   ```

2. **添加引擎要求**
   ```json
   "engines": {
     "bun": ">=1.0.0"
   }
   ```

3. **CI 集成 lint 检查**
   在 `.github/workflows/pages.yml` 添加：
   ```yaml
   - name: Check
     run: bun run check
   ```

4. **添加 prepare 脚本**
   ```json
   "prepare": "bun run build:package"
   ```
   - 确保从 git 安装时自动构建

5. **文档化脚本用途**
   在 `DEVELOPMENT.md` 添加脚本分类说明：
   ```markdown
   ### 脚本分类
   - 开发: start, start:lan, start:watch
   - 构建: build:package, site:build
   - 检查: check, package-smoke-test
   - 测试: accuracy-check, benchmark-check, corpus-check
   ```

6. **考虑添加 repository 字段**
   ```json
   "repository": {
     "type": "git",
     "url": "https://github.com/chenglou/pretext.git"
   }
   ```

7. **添加 keywords 字段**
   ```json
   "keywords": ["text-layout", "typography", "canvas", "i18n", "cjk", "bidi"]
   ```
