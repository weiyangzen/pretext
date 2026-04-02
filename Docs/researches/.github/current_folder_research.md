# .github 目录研究文档

## 目录概述

`.github` 目录位于项目根目录 `/home/sansha/Github/pretext/`，仅包含一个子目录 `workflows/`，其中定义了单个 GitHub Actions 工作流文件 `pages.yml`。

该目录是 Pretext 项目与 GitHub 平台集成的唯一入口点，负责自动化构建和部署演示站点到 GitHub Pages。

---

## 场景与职责

### 核心场景

`.github/workflows/pages.yml` 工作流服务于以下核心场景：

1. **演示站点自动发布**
   - 每当代码推送到 `main` 分支时，自动触发构建流程
   - 将 TypeScript 演示页面构建为静态 HTML 站点
   - 部署到 GitHub Pages 供公众访问

2. **手动触发部署**
   - 支持通过 GitHub UI 手动触发 (`workflow_dispatch`)
   - 用于需要强制重新部署的场景（如配置变更、依赖更新）

3. **持续集成验证**
   - 构建过程隐式验证项目能否成功编译
   - 确保演示代码与库代码保持同步

### 职责边界

| 职责 | 归属 | 说明 |
|------|------|------|
| 站点构建 | `scripts/build-demo-site.ts` | 工作流调用此脚本完成实际构建 |
| 包发布 | 手动/npm | 工作流不负责 npm 包发布 |
| 测试执行 | 本地/CI 可选 | 工作流不运行测试套件 |
| 准确性检查 | 本地脚本 | 浏览器准确性验证在本地执行 |

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

**目的**：
- `push: main`：确保主分支的最新代码始终反映在线上演示站点
- `workflow_dispatch`：提供运维灵活性，允许在需要时手动重新部署

**设计考量**：
- 不对所有分支的 push 触发，避免开发中的代码污染公开站点
- 不对 PR 触发，因为 PR 可能包含未完成的工作

### 2. 权限控制 (`permissions`)

```yaml
permissions:
  contents: read
  pages: write
  id-token: write
```

**目的**：
- `contents: read`：仅读取仓库内容，遵循最小权限原则
- `pages: write`：写入 GitHub Pages 所需的权限
- `id-token: write`：用于 OIDC 身份验证（GitHub Pages 部署需要）

**安全考量**：
- 不使用 `write-all` 或宽泛权限，降低潜在安全风险
- 与 GitHub 推荐的 Pages 部署安全模型保持一致

### 3. 并发控制 (`concurrency`)

```yaml
concurrency:
  group: pages
  cancel-in-progress: true
```

**目的**：
- 防止多个部署任务同时运行导致的竞态条件
- 当新的 push 发生时，取消正在进行的旧部署，节省资源

**技术细节**：
- `group: pages`：所有 Pages 相关任务共享同一并发组
- `cancel-in-progress: true`：新任务启动时取消同组旧任务

### 4. 构建作业 (`jobs.build`)

**步骤分析**：

| 步骤 | 动作 | 目的 |
|------|------|------|
| 1 | `actions/checkout@v5` | 检出仓库代码 |
| 2 | `oven-sh/setup-bun@v2` | 安装 Bun 运行时 |
| 3 | `bun install --frozen-lockfile` | 安装依赖（锁定版本） |
| 4 | `bun run site:build` | 执行站点构建脚本 |
| 5 | `actions/configure-pages@v5` | 配置 Pages 环境 |
| 6 | `actions/upload-pages-artifact@v4` | 上传构建产物 |

**关键决策**：
- 使用 Bun 而非 Node.js：与项目开发环境保持一致
- `--frozen-lockfile`：确保 CI 使用与本地完全相同的依赖版本
- `bun-version: latest`：始终使用最新 Bun 版本，获取性能改进

### 5. 部署作业 (`jobs.deploy`)

**依赖关系**：
```yaml
needs: build
```

**环境配置**：
```yaml
environment:
  name: github-pages
  url: ${{ steps.deployment.outputs.page_url }}
```

**目的**：
- 分离构建和部署为独立作业，提高可观测性和故障隔离
- 使用 GitHub Environments 功能，支持部署保护和审核
- 通过 `outputs.page_url` 在 PR 和提交状态中显示部署 URL

---

## 具体技术实现

### 构建流程详解

工作流调用的核心构建命令是 `bun run site:build`，其执行链路如下：

```
.github/workflows/pages.yml
  └─> bun run site:build
       └─> scripts/build-demo-site.ts
            └─> bun build [...entrypoints] --outdir site/
                 └─> 处理 HTML 文件 + TypeScript 打包
```

### 构建脚本关键逻辑 (`scripts/build-demo-site.ts`)

**入口点定义**：
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

**URL 重写策略**：

1. **资源路径重定向** (`rebaseRelativeAssetUrls`)
   - 将相对路径的 `src` 和 `href` 属性转换为相对于目标路径的正确路径
   - 处理跨目录引用的资源文件

2. **演示链接重写** (`rewriteDemoLinksForStaticRoot`)
   - 将 `/demos/slug` 格式的链接重写为 `./slug` 格式
   - 适配静态站点托管的 URL 结构

**目录结构转换**：

| 源路径 | 目标路径 | 说明 |
|--------|----------|------|
| `index.html` | `index.html` | 站点根页 |
| `accordion.html` | `accordion/index.html` | 子目录索引页 |
| `bubbles.html` | `bubbles/index.html` | 子目录索引页 |
| `masonry/index.html` | `masonry/index.html` | 保持目录结构 |

### GitHub Actions 版本选择

| 动作 | 版本 | 选择理由 |
|------|------|----------|
| checkout | v5 | 最新稳定版，支持稀疏检出 |
| setup-bun | v2 | 官方 Bun 动作，支持缓存 |
| configure-pages | v5 | 与 upload-pages-artifact v4 配套 |
| upload-pages-artifact | v4 | 最新版，支持大产物上传 |
| deploy-pages | v4 | 与 upload-pages-artifact v4 配套 |

---

## 关键代码路径与文件引用

### 直接依赖文件

```
.github/workflows/pages.yml
├── scripts/build-demo-site.ts (构建脚本)
├── package.json (依赖定义和脚本声明)
├── bun.lock (锁定文件)
└── pages/demos/*.html (演示页面入口)
```

### 间接依赖文件

```
scripts/build-demo-site.ts
├── pages/demos/*.ts (演示页面逻辑)
├── pages/demos/masonry/shower-thoughts.json (演示数据)
├── pages/assets/*.svg (静态资源)
└── src/*.ts (库源码，被演示页面引用)
```

### 构建产物流向

```
[Source]                    [Build]                    [Deploy]
--------                    -------                    --------
pages/demos/*.html    ──►   site/                 ──►  GitHub Pages
pages/demos/*.ts           (Bun 构建输出)              (chenglou.me/pretext/)
pages/assets/*             
```

---

## 依赖与外部交互

### 外部服务依赖

| 服务 | 用途 | 可靠性 |
|------|------|--------|
| GitHub Actions | CI/CD 执行环境 | 高 |
| GitHub Pages | 静态站点托管 | 高 |
| Bun 注册表 | 运行时和依赖下载 | 中-高 |
| npm 注册表 | 间接依赖（通过 Bun） | 高 |

### 内部依赖关系

```
.github/workflows/pages.yml
    │
    ├─depends on─> package.json (scripts.site:build)
    │
    ├─depends on─> scripts/build-demo-site.ts
    │       ├─depends on─> Bun.build API
    │       └─depends on─> node:fs/promises
    │
    └─depends on─> pages/demos/* (构建源文件)
```

### 环境变量和 Secrets

工作流**不**使用以下常见模式：
- 无 `env` 部分定义的环境变量
- 无 `secrets.GITHUB_TOKEN` 显式引用（由 `actions/deploy-pages` 内部使用）
- 无外部服务 API keys

这种设计降低了配置复杂度和泄露风险。

---

## 风险、边界与改进建议

### 当前风险

#### 1. Bun 版本浮动风险

**现状**：
```yaml
with:
  bun-version: latest
```

**风险**：
- Bun 新版本可能引入破坏性变更
- 构建可能因 Bun 更新而意外失败

**缓解**：
- 项目使用 `bun.lock` 锁定依赖，降低运行时影响
- 构建失败时易于定位是 Bun 版本问题

#### 2. 单点故障（Ubuntu 最新版）

**现状**：
```yaml
runs-on: ubuntu-latest
```

**风险**：
- `ubuntu-latest` 标签会随 GitHub 更新而变更
- 可能因镜像更新导致构建环境变化

**缓解**：
- 使用标准 Bun 和 Node 工具链，环境兼容性好
- 问题出现时易于回退到特定版本

#### 3. 无构建缓存

**现状**：
- 每次构建都重新安装依赖
- 没有启用 Bun 的依赖缓存

**影响**：
- 构建时间略长（对于此项目规模影响有限）

### 边界情况

#### 1. 构建产物大小限制

- GitHub Pages 有 1GB 站点大小限制
- 当前演示站点远小于此限制
- 若未来添加大量语料或字体文件，需监控

#### 2. 并发部署限制

- `concurrency` 配置确保单部署流
- 快速连续推送可能导致部分提交未被部署（被后续取消）
- 这是可接受的行为，最终一致性保证

#### 3. 分支保护依赖

- 工作流假设 `main` 分支是受保护的
- 若 `main` 可直接推送，存在部署不稳定代码的风险
- 这是仓库设置层面的问题，非工作流问题

### 改进建议

#### 建议 1：固定 Bun 版本

```yaml
# 当前
bun-version: latest

# 建议
bun-version: '1.x'  # 或具体版本如 '1.1.0'
```

**理由**：提高构建可重现性，避免意外失败。

#### 建议 2：添加构建缓存

```yaml
- uses: oven-sh/setup-bun@v2
  with:
    bun-version: latest
- uses: actions/cache@v4
  with:
    path: ~/.bun/install/cache
    key: ${{ runner.os }}-bun-${{ hashFiles('bun.lock') }}
```

**理由**：减少构建时间，降低对 Bun 注册表的依赖。

#### 建议 3：添加构建状态通知

```yaml
- name: Notify on failure
  if: failure()
  uses: actions/github-script@v7
  with:
    script: |
      github.rest.issues.create({
        owner: context.repo.owner,
        repo: context.repo.repo,
        title: 'Pages deployment failed',
        body: `Failed run: ${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}`
      })
```

**理由**：及时感知部署失败，尤其是当团队不常查看 Actions 标签时。

#### 建议 4：添加 PR 预览部署

**现状**：PR 不会触发部署，无法预览变更效果。

**建议**：添加 PR 预览工作流（使用第三方服务如 Cloudflare Pages 或 Netlify），或至少在 PR 中运行构建验证（不实际部署）。

```yaml
# 新增 .github/workflows/pr-check.yml
name: PR Build Check
on:
  pull_request:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: oven-sh/setup-bun@v2
      - run: bun install --frozen-lockfile
      - run: bun run site:build
      # 不实际部署，仅验证构建成功
```

#### 建议 5：监控部署频率

**观察**：当前配置每次 push 到 main 都部署。

**建议**：若部署过于频繁，可考虑：
- 添加路径过滤，仅当 `pages/`、`src/`、`scripts/` 变更时触发
- 添加合并队列，批量处理多个 PR

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'pages/**'
      - 'src/**'
      - 'scripts/build-demo-site.ts'
      - 'package.json'
      - 'bun.lock'
```

---

## 总结

`.github/workflows/pages.yml` 是一个精简、专注的 GitHub Actions 工作流，其设计遵循以下原则：

1. **单一职责**：仅负责演示站点的构建和部署
2. **最小权限**：仅请求必要的权限
3. **简单可靠**：使用标准动作和成熟模式
4. **与开发流程解耦**：不介入测试、发布等其他流程

该工作流与项目的整体架构保持一致：Pretext 是一个浏览器端文本布局库，其 CI/CD 需求相对简单，重点在于确保演示站点与代码同步更新。

主要改进机会集中在**构建缓存**、**版本固定**和**PR 验证**三个方面，但当前配置已满足项目的基本需求。
