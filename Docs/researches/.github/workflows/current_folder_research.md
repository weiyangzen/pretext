# .github/workflows 目录研究报告

## 概述

本目录包含 Pretext 项目的 GitHub Actions 工作流配置，目前仅包含一个工作流文件 `pages.yml`，用于自动化构建和部署演示站点到 GitHub Pages。

---

## 一、场景与职责

### 1.1 工作流定位

`pages.yml` 是 Pretext 项目唯一的 CI/CD 工作流，承担以下核心职责：

| 职责维度 | 具体描述 |
|---------|---------|
| **演示站点部署** | 将 `pages/demos/` 下的交互式演示构建为静态站点并部署到 GitHub Pages |
| **自动化触发** | 在 `main` 分支推送时自动触发，支持手动触发 (`workflow_dispatch`) |
| **构建产物管理** | 使用 Bun 构建工具链生成优化后的静态资源 |
| **并发控制** | 通过 `concurrency` 配置防止并行部署冲突 |

### 1.2 部署目标

- **生产环境**: GitHub Pages (`chenglou.me/pretext/`)
- **构建输出**: `site/` 目录（由 `scripts/build-demo-site.ts` 生成）
- **站点结构**: 扁平化 URL 结构（`/demos/accordion/` 而非 `/demos/accordion.html`）

---

## 二、功能点目的

### 2.1 触发策略

```yaml
on:
  push:
    branches:
      - main
  workflow_dispatch:
```

| 触发方式 | 目的 |
|---------|------|
| `push: main` | 确保主分支最新代码始终反映在演示站点 |
| `workflow_dispatch` | 允许手动重新部署（如修复构建问题后） |

### 2.2 权限模型

```yaml
permissions:
  contents: read      # 仅需读取仓库内容
  pages: write        # 写入 GitHub Pages 服务
  id-token: write     # 用于 OIDC 身份验证（actions/deploy-pages@v4 所需）
```

采用最小权限原则，遵循 GitHub 推荐的安全最佳实践。

### 2.3 并发控制

```yaml
concurrency:
  group: pages
  cancel-in-progress: true
```

- **分组策略**: 所有 Pages 部署共享同一并发组
- **取消策略**: 新部署自动取消进行中的旧部署，避免资源浪费和状态混乱

### 2.4 双阶段构建流程

工作流分为 `build` 和 `deploy` 两个 Job，实现构建与部署的解耦：

```
┌─────────┐    ┌──────────┐    ┌─────────┐
│  Build  │───→│ Artifact │───→│ Deploy  │
│  (构建)  │    │ (产物)    │    │ (部署)  │
└─────────┘    └──────────┘    └─────────┘
   ubuntu                        ubuntu
   latest                        latest
```

---

## 三、具体技术实现

### 3.1 Build Job 详细流程

| 步骤 | Action/命令 | 技术细节 |
|-----|-------------|---------|
| 1 | `actions/checkout@v5` | 检出仓库代码 |
| 2 | `oven-sh/setup-bun@v2` | 安装 Bun 运行时（指定 `latest` 版本） |
| 3 | `bun install --frozen-lockfile` | 依赖安装，锁定版本保证可复现性 |
| 4 | `bun run site:build` | 执行构建脚本（见下方详细分析） |
| 5 | `actions/configure-pages@v5` | 配置 GitHub Pages 环境（`enablement: true` 启用服务） |
| 6 | `actions/upload-pages-artifact@v4` | 上传 `site/` 目录作为部署产物 |

### 3.2 构建脚本深度分析 (`scripts/build-demo-site.ts`)

**入口点配置**（9 个演示页面）：

```typescript
const entrypoints = [
  'pages/demos/index.html',           // 演示首页
  'pages/demos/accordion.html',       // 手风琴组件演示
  'pages/demos/bubbles.html',         // 消息气泡演示
  'pages/demos/dynamic-layout.html',  // 动态布局演示
  'pages/demos/editorial-engine.html',// 编辑引擎演示
  'pages/demos/justification-comparison.html', // 文本对齐对比
  'pages/demos/masonry/index.html',   // 瀑布流布局
  'pages/demos/rich-note.html',       // 富文本演示
  'pages/demos/variable-typographic-ascii.html', // 可变字体 ASCII
]
```

**构建流程**:

1. **Bun 构建**: 使用 `bun build` 处理所有入口点，输出到 `site/` 目录
2. **路径重写**: 通过 `rebaseRelativeAssetUrls()` 调整相对资源路径
3. **URL 扁平化**: 通过 `rewriteDemoLinksForStaticRoot()` 将 `/demos/xxx.html` 转换为 `/demos/xxx/index.html` 结构
4. **目录清理**: 删除中间构建目录 `site/pages/`

**路径映射表**:

| 源文件 | 目标路径 | 说明 |
|-------|---------|------|
| `index.html` | `index.html` | 站点根首页 |
| `accordion.html` | `accordion/index.html` | 手风琴演示（目录式URL） |
| `bubbles.html` | `bubbles/index.html` | 气泡演示 |
| `dynamic-layout.html` | `dynamic-layout/index.html` | 动态布局 |
| `editorial-engine.html` | `editorial-engine/index.html` | 编辑引擎 |
| `justification-comparison.html` | `justification-comparison/index.html` | 对齐对比 |
| `masonry/index.html` | `masonry/index.html` | 瀑布流（已是目录结构） |
| `rich-note.html` | `rich-note/index.html` | 富文本 |
| `variable-typographic-ascii.html` | `variable-typographic-ascii/index.html` | ASCII艺术 |

### 3.3 Deploy Job 详细流程

| 配置项 | 值 | 说明 |
|-------|---|------|
| `needs` | `build` | 依赖 build job 完成 |
| `environment` | `github-pages` | GitHub Pages 部署环境 |
| `url` | `${{ steps.deployment.outputs.page_url }}` | 部署后输出的站点URL |
| 部署 Action | `actions/deploy-pages@v4` | 官方 Pages 部署 Action |

---

## 四、关键代码路径与文件引用

### 4.1 工作流文件

```
.github/workflows/pages.yml          # 本工作流定义
```

### 4.2 构建相关文件

```
scripts/build-demo-site.ts           # 站点构建脚本（被 bun run site:build 调用）
package.json                         # 定义 site:build 脚本命令
```

### 4.3 演示页面源文件

```
pages/demos/index.html               # 演示首页
pages/demos/accordion.html           # 手风琴演示
pages/demos/accordion.ts             # 手风琴逻辑
pages/demos/bubbles.html             # 气泡演示
pages/demos/bubbles.ts               # 气泡逻辑
pages/demos/dynamic-layout.html      # 动态布局演示
pages/demos/dynamic-layout.ts        # 动态布局逻辑
pages/demos/editorial-engine.html    # 编辑引擎演示
pages/demos/editorial-engine.ts      # 编辑引擎逻辑
pages/demos/justification-comparison.html  # 对齐对比
pages/demos/justification-comparison.ts    # 对齐对比逻辑
pages/demos/masonry/index.html       # 瀑布流演示
pages/demos/rich-note.html           # 富文本演示
pages/demos/rich-note.ts             # 富文本逻辑
pages/demos/variable-typographic-ascii.html  # ASCII艺术
pages/demos/variable-typographic-ascii.ts    # ASCII艺术逻辑
```

### 4.4 共享资源

```
pages/demos/bubbles-shared.ts        # 气泡演示共享代码
pages/demos/dynamic-layout-text.ts   # 动态布局文本数据
pages/demos/justification-comparison.data.ts   # 对齐对比数据
pages/demos/justification-comparison.model.ts  # 对齐对比模型
pages/demos/justification-comparison.ui.ts     # 对齐对比UI
pages/demos/wrap-geometry.ts         # 换行几何计算
pages/demos/svg.d.ts                 # SVG类型定义
pages/assets/                        # 静态资源目录
```

### 4.5 核心库文件（被演示页面引用）

```
src/layout.ts                        # 核心布局API
src/analysis.ts                      # 文本分析与归一化
src/measurement.ts                   # Canvas 测量运行时
src/line-break.ts                    # 换行核心逻辑
src/bidi.ts                          # 双向文本处理
```

---

## 五、依赖与外部交互

### 5.1 GitHub Actions 生态依赖

| Action | 版本 | 用途 |
|--------|------|------|
| `actions/checkout` | v5 | 代码检出 |
| `oven-sh/setup-bun` | v2 | Bun 运行时安装 |
| `actions/configure-pages` | v5 | Pages 服务配置 |
| `actions/upload-pages-artifact` | v4 | 构建产物上传 |
| `actions/deploy-pages` | v4 | 站点部署 |

### 5.2 运行时依赖

| 依赖 | 来源 | 用途 |
|------|------|------|
| Bun | `oven-sh/setup-bun@v2` | JavaScript/TypeScript 运行时和构建工具 |
| Node.js API | Bun 内置 | `node:fs/promises`, `node:path` |

### 5.3 项目内部依赖

```
build-demo-site.ts
    ├── Bun.spawnSync()          # Bun 进程管理
    ├── node:fs/promises         # 文件系统操作
    ├── node:path                # 路径处理
    └── pages/demos/*.html       # 构建入口

site:build (package.json script)
    └── scripts/build-demo-site.ts
```

### 5.4 外部服务交互

| 服务 | 交互方式 | 目的 |
|------|---------|------|
| GitHub Pages | `actions/deploy-pages@v4` | 静态站点托管 |
| GitHub OIDC | `id-token: write` 权限 | 安全身份验证（无需长期凭证） |

---

## 六、风险、边界与改进建议

### 6.1 当前风险

| 风险类别 | 具体描述 | 严重程度 |
|---------|---------|---------|
| **Bun 版本漂移** | 使用 `bun-version: latest` 可能导致构建行为不一致 | 中 |
| **无构建缓存** | 未配置 `actions/cache` 加速依赖安装 | 低 |
| **单点故障** | 仅有一个工作流，无备用部署路径 | 低 |
| **无测试前置** | 部署前未运行测试套件 (`bun test`) | 中 |
| **Node 版本锁定** | 未显式锁定 Node 版本（依赖 Bun 内置） | 低 |

### 6.2 边界条件

| 边界场景 | 当前行为 | 注意事项 |
|---------|---------|---------|
| 并发部署 | 取消进行中的部署 | 确保 `cancel-in-progress: true` 符合预期 |
| 构建失败 | 工作流终止，不部署 | 需监控 Actions 通知 |
| 非 main 分支推送 | 不触发部署 | feature 分支需手动触发 |
| 大文件构建 | 无特殊处理 | 监控 `site/` 目录大小 |

### 6.3 改进建议

#### 建议 1: 锁定 Bun 版本

```yaml
# 当前
- uses: oven-sh/setup-bun@v2
  with:
    bun-version: latest

# 建议
- uses: oven-sh/setup-bun@v2
  with:
    bun-version: 1.2.0  # 或从 .bun-version 文件读取
```

**理由**: 避免 Bun 升级导致的意外构建失败。

#### 建议 2: 添加构建缓存

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.bun/install/cache
    key: ${{ runner.os }}-bun-${{ hashFiles('bun.lock') }}
    restore-keys: |
      ${{ runner.os }}-bun-
```

**理由**: 减少依赖安装时间，提升工作流执行速度。

#### 建议 3: 添加测试前置步骤

```yaml
- run: bun test
- run: bun run check  # 类型检查 + lint
```

**理由**: 防止有问题的代码进入演示站点。

#### 建议 4: 添加构建产物验证

```yaml
- run: |
    if [ ! -f site/index.html ]; then
      echo "Build artifact missing!"
      exit 1
    fi
```

**理由**: 确保关键文件存在后再上传产物。

#### 建议 5: 支持预览部署（PR 预览）

```yaml
on:
  pull_request:
    paths:
      - 'pages/**'
      - 'src/**'
```

**理由**: 让贡献者在合并前预览变更效果。

### 6.4 监控与可观测性

| 监控项 | 建议方案 |
|-------|---------|
| 部署状态 | GitHub Actions 通知 + 仓库徽章 |
| 站点可用性 | 外部监控（如 UptimeRobot） |
| 构建时长 | Actions 内置统计 |
| 产物大小 | 自定义步骤记录 `site/` 目录大小 |

---

## 七、总结

`.github/workflows/pages.yml` 是一个精简但功能完整的 GitHub Pages 部署工作流，其核心设计特点包括：

1. **双阶段架构**: 构建与部署分离，符合 CI/CD 最佳实践
2. **最小权限**: 仅申请必要的 `pages:write` 和 `id-token:write` 权限
3. **并发安全**: 通过 `concurrency` 配置防止部署冲突
4. **Bun 原生**: 充分利用 Bun 的快速构建能力

该工作流与 Pretext 项目的技术栈（Bun + TypeScript）高度契合，通过 `scripts/build-demo-site.ts` 实现了复杂的路径重写和 URL 扁平化逻辑，确保演示站点具有优雅的 URL 结构。
