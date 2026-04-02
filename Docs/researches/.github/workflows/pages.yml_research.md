# `.github/workflows/pages.yml` 研究文档

## 场景与职责

`.github/workflows/pages.yml` 是 Pretext 项目的 GitHub Actions 工作流配置文件，负责**自动构建并部署演示站点到 GitHub Pages**。这是项目对外展示的主要入口，将本地开发的多页面演示应用转换为可访问的静态网站。

### 核心职责

1. **持续部署 (CD)**：在 `main` 分支推送时自动触发构建和部署
2. **手动触发**：支持通过 `workflow_dispatch` 手动执行部署
3. **静态站点生成**：将 TypeScript/HTML 源码构建为纯静态文件
4. **GitHub Pages 托管**：利用 GitHub 提供的免费静态站点托管服务

### 业务价值

- **演示展示**：为潜在用户提供在线体验入口（如 https://chenglou.me/pretext/）
- **文档补充**：README 中引用的在线演示依赖此工作流
- **自动化运维**：无需人工干预即可完成从代码到线上站点的完整流程

---

## 功能点目的

### 1. 触发策略 (`on`)

```yaml
on:
  push:
    branches:
      - main
  workflow_dispatch:
```

| 触发方式 | 目的 |
|---------|------|
| `push: branches: [main]` | 主分支更新时自动部署，确保线上始终是最新稳定版本 |
| `workflow_dispatch` | 允许手动触发，用于紧急修复或测试部署流程 |

### 2. 权限控制 (`permissions`)

```yaml
permissions:
  contents: read      # 只读仓库内容
  pages: write        # 写入 GitHub Pages
  id-token: write     # 写入 OIDC token（用于安全部署）
```

采用最小权限原则：
- `contents: read`：仅需读取源码，无需写入
- `pages: write`：必须权限，用于发布站点
- `id-token: write`：支持 OIDC 身份验证，避免使用长期有效的 Personal Access Token

### 3. 并发控制 (`concurrency`)

```yaml
concurrency:
  group: pages
  cancel-in-progress: true
```

- **分组策略**：所有 `pages` 工作流共享同一并发组
- **取消进行中**：新触发时自动取消正在运行的旧实例，防止部署冲突和资源浪费

### 4. 构建作业 (`build` job)

| 步骤 | 动作 | 目的 |
|-----|------|------|
| 1 | `actions/checkout@v5` | 检出仓库代码 |
| 2 | `oven-sh/setup-bun@v2` | 安装 Bun 运行时（替代 Node.js，更快） |
| 3 | `bun install --frozen-lockfile` | 安装依赖（锁定版本确保可复现） |
| 4 | `bun run site:build` | 执行站点构建脚本 |
| 5 | `actions/configure-pages@v5` | 配置 GitHub Pages 环境 |
| 6 | `actions/upload-pages-artifact@v4` | 上传构建产物到 artifact |

### 5. 部署作业 (`deploy` job)

```yaml
deploy:
  needs: build          # 依赖 build 作业成功
  environment:
    name: github-pages
    url: ${{ steps.deployment.outputs.page_url }}
  steps:
    - uses: actions/deploy-pages@v4
```

- **环境隔离**：使用 `github-pages` 环境，支持部署保护规则
- **输出 URL**：部署完成后输出可访问的站点 URL

---

## 具体技术实现

### 构建流程详解

#### 阶段 1：环境准备

```yaml
- uses: actions/checkout@v5
- uses: oven-sh/setup-bun@v2
  with:
    bun-version: latest
```

- **Bun 选择**：项目完全基于 Bun 生态（`bun.lock` 锁文件）
- **版本策略**：使用 `latest` 获取最新稳定版，平衡稳定性和新特性

#### 阶段 2：依赖安装

```yaml
- run: bun install --frozen-lockfile
```

- `--frozen-lockfile`：严格按 `bun.lock` 安装，确保 CI 与本地环境一致

#### 阶段 3：站点构建

```yaml
- run: bun run site:build
```

对应 `package.json` 中的脚本：

```json
"site:build": "rm -rf site && bun run scripts/build-demo-site.ts"
```

**构建脚本 (`scripts/build-demo-site.ts`) 核心逻辑：**

1. **定义入口点**：9 个演示页面 HTML 文件
   ```typescript
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
   ]
   ```

2. **Bun 构建**：使用 `Bun.build()` 打包所有入口点
   ```typescript
   const result = Bun.spawnSync(
     ['bun', 'build', ...entrypoints, '--outdir', outdir],
     { cwd: root, stdout: 'inherit', stderr: 'inherit' }
   )
   ```

3. **URL 重写**：将扁平结构转换为友好的子路径结构
   - `accordion.html` → `accordion/index.html`
   - 支持相对路径导航（`./accordion`）

4. **资源路径修复**：
   - `rebaseRelativeAssetUrls()`：调整相对资源路径（`src`/`href`）
   - `rewriteDemoLinksForStaticRoot()`：重写首页演示链接为相对路径

#### 阶段 4：Pages 配置

```yaml
- uses: actions/configure-pages@v5
  with:
    enablement: true
```

- 启用 GitHub Pages 功能（如果尚未启用）

#### 阶段 5：产物上传

```yaml
- uses: actions/upload-pages-artifact@v4
  with:
    path: site
```

- 上传 `site/` 目录内容作为部署 artifact

#### 阶段 6：部署执行

```yaml
- uses: actions/deploy-pages@v4
```

- 将 artifact 部署到 GitHub Pages 服务

---

## 关键代码路径与文件引用

### 工作流文件

```
.github/workflows/pages.yml          # 本文件，定义 CI/CD 流程
```

### 构建脚本

```
scripts/build-demo-site.ts           # 站点构建核心逻辑
```

### 源码入口

```
pages/demos/index.html               # 演示首页
pages/demos/accordion.html           # 手风琴演示
pages/demos/bubbles.html             # 气泡消息演示
pages/demos/dynamic-layout.html      # 动态布局演示
pages/demos/editorial-engine.html    # 编辑器引擎演示
pages/demos/justification-comparison.html  # 对齐比较演示
pages/demos/masonry/index.html       # 瀑布流演示
pages/demos/rich-note.html           # 富文本演示
pages/demos/variable-typographic-ascii.html  # ASCII 艺术演示
```

### 核心库源码（被演示页面引用）

```
src/layout.ts                        # 主布局 API（prepare/layout）
src/analysis.ts                      # 文本分析和分段
src/measurement.ts                   # Canvas 测量和缓存
src/line-break.ts                    # 换行算法
src/bidi.ts                          # 双向文本支持
```

### 共享模块

```
shared/navigation-state.ts           # 导航状态管理（用于浏览器自动化测试）
```

### 配置依赖

```
package.json                         # 定义 site:build 脚本和 Bun 依赖
tsconfig.json                        # TypeScript 配置
bun.lock                             # 依赖锁定文件
```

---

## 依赖与外部交互

### 外部 Actions 依赖

| Action | 版本 | 用途 |
|--------|------|------|
| `actions/checkout` | v5 | 代码检出 |
| `oven-sh/setup-bun` | v2 | Bun 运行时安装 |
| `actions/configure-pages` | v5 | GitHub Pages 配置 |
| `actions/upload-pages-artifact` | v4 | 构建产物上传 |
| `actions/deploy-pages` | v4 | 站点部署 |

### 运行时依赖

| 依赖 | 说明 |
|------|------|
| Bun | JavaScript/TypeScript 运行时和构建工具 |
| GitHub Pages | 静态站点托管服务 |

### 项目内部依赖

```
pages.yml
    └── bun run site:build
            └── scripts/build-demo-site.ts
                    └── Bun.build() 编译 pages/demos/*.html
                            └── 引用 pages/demos/*.ts
                                    └── 引用 src/layout.ts 等核心库
```

### 数据流向

```
Git Push (main)
    │
    ▼
GitHub Actions Runner
    │
    ├── Checkout Code
    ├── Setup Bun
    ├── Install Dependencies
    ├── Build Site (site/)
    │       └── TypeScript/HTML → Bundled JS + HTML
    ├── Upload Artifact
    │
    ▼
GitHub Pages
    └── https://<user>.github.io/pretext/
```

---

## 风险、边界与改进建议

### 当前风险

#### 1. Bun 版本浮动风险

**问题**：使用 `bun-version: latest` 可能导致构建行为不一致

```yaml
- uses: oven-sh/setup-bun@v2
  with:
    bun-version: latest  # 可能引入破坏性变更
```

**影响**：Bun 新版本可能改变构建输出或引入 bug

**缓解**：项目使用 `bun.lock` 锁定依赖，但 Bun 运行时本身的变更无法控制

#### 2. 单点故障（Ubuntu Only）

```yaml
runs-on: ubuntu-latest
```

**问题**：仅在 Ubuntu 上测试构建，未验证其他平台兼容性

**影响**：如果项目有平台相关代码（如路径处理），可能在其他环境失败

#### 3. 无构建缓存

**问题**：每次构建都重新安装依赖，无 `actions/cache` 配置

**影响**：构建时间随依赖量增加而增长

#### 4. 无测试门禁

**问题**：工作流不包含测试步骤（如 `bun test` 或 `bun run check`）

**影响**：可能部署带有类型错误或单元测试失败的代码

### 边界条件

#### 1. 构建产物大小限制

- GitHub Pages 有 1GB 存储限制和 100GB 月流量限制
- 当前演示站点较小，但未来增加大量资源可能触及限制

#### 2. 并发部署

- `concurrency` 配置确保同一时间只有一个部署在进行
- 快速连续推送可能导致部分构建被取消

#### 3. 演示页面入口维护

`build-demo-site.ts` 中的入口点列表需要手动维护：

```typescript
const entrypoints = [
  'pages/demos/index.html',
  // ... 需要手动添加新演示
]
```

新增演示页面时容易遗漏，导致新页面未部署

### 改进建议

#### 1. 锁定 Bun 版本

```yaml
- uses: oven-sh/setup-bun@v2
  with:
    bun-version: '1.2.0'  # 锁定到具体版本
```

或从 `.bun-version` 文件读取：

```yaml
- uses: oven-sh/setup-bun@v2
  with:
    bun-version-file: '.bun-version'
```

#### 2. 添加依赖缓存

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.bun/install/cache
    key: ${{ runner.os }}-bun-${{ hashFiles('bun.lock') }}
```

#### 3. 添加质量门禁

```yaml
- run: bun run check      # 类型检查 + Lint
- run: bun test           # 单元测试
- run: bun run package-smoke-test  # 包构建测试
```

#### 4. 动态发现入口点

修改 `build-demo-site.ts` 自动发现 `pages/demos/` 下的所有 HTML：

```typescript
import { glob } from 'node:fs/promises'

const entrypoints = await Array.fromAsync(glob('pages/demos/**/*.html'))
```

#### 5. 添加部署预览（PR 预览）

为 Pull Request 创建预览部署：

```yaml
on:
  pull_request:
    paths:
      - 'pages/**'
      - 'src/**'
```

使用 `actions/deploy-pages` 的预览功能或第三方服务（如 Cloudflare Pages）

#### 6. 添加健康检查

部署后验证站点可访问：

```yaml
- run: curl --fail ${{ steps.deployment.outputs.page_url }}
```

#### 7. 多平台构建验证

如果项目需要跨平台支持：

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]
```

---

## 附录：相关配置完整引用

### package.json 相关脚本

```json
{
  "scripts": {
    "site:build": "rm -rf site && bun run scripts/build-demo-site.ts",
    "start": "HOST=${HOST:-127.0.0.1}; PORT=3000; ... bun pages/*.html ...",
    "check": "tsc && oxlint --type-aware src",
    "build:package": "rm -rf dist && tsc -p tsconfig.build.json"
  }
}
```

### 构建脚本核心逻辑摘要

```typescript
// scripts/build-demo-site.ts
const entrypoints = [/* 9 HTML files */]
const outdir = path.join(root, 'site')

// 1. Bun 构建
Bun.spawnSync(['bun', 'build', ...entrypoints, '--outdir', outdir])

// 2. 重写为子路径结构
const targets = [
  { source: 'index.html', target: 'index.html' },
  { source: 'accordion.html', target: 'accordion/index.html' },
  // ...
]

// 3. 修复资源路径
function rebaseRelativeAssetUrls(html, sourcePath, targetPath) {
  return html.replace(/\b(src|href)="([^"]+)"/g, /* ... */)
}
```

### 工作流完整代码

```yaml
name: Publish Demo Site

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: oven-sh/setup-bun@v2
        with:
          bun-version: latest
      - run: bun install --frozen-lockfile
      - run: bun run site:build
      - uses: actions/configure-pages@v5
        with:
          enablement: true
      - uses: actions/upload-pages-artifact@v4
        with:
          path: site

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```
