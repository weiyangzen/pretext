# gatsby-sweep.ts 研究文档

## 场景与职责

`scripts/gatsby-sweep.ts` 是 `scripts/corpus-sweep.ts` 的**兼容性别名脚本**。它将早期 `gatsby-sweep` 命令无缝桥接到新的 corpus 基础设施，固定对 `en-gatsby-opening` 语料执行宽度扫测，同时支持 `GATSBY_CHECK_*` 环境变量的向后兼容。

## 功能点目的

1. **保留旧命令入口**：`bun run gatsby-sweep` 继续可用，无需修改历史脚本和文档。
2. **固定语料与转发**：自动注入 `--id=en-gatsby-opening`，其余参数透传给 `corpus-sweep.ts`。
3. **环境变量桥接**：与 `gatsby-check.ts` 相同，将 `GATSBY_CHECK_BROWSER/PORT/TIMEOUT_MS` 映射为 `CORPUS_CHECK_*`。

## 具体技术实现

### 核心流程

1. **过滤参数**
   ```ts
   const forwardedArgs = process.argv.slice(2)
     .filter(arg => arg !== '--all' && !arg.startsWith('--id='));
   ```

2. **构造子进程命令**
   ```ts
   spawnSync('bun', [
     'run', 'scripts/corpus-sweep.ts',
     '--id=en-gatsby-opening',
     ...forwardedArgs
   ], { cwd: process.cwd(), env: { ...process.env, ...mappedEnv }, encoding: 'utf8' })
   ```
   注意：与 `gatsby-check.ts` 不同，这里**不注入 `--diagnose`**，因为 sweep 的默认行为是粗粒度扫测，诊断由用户显式通过 `forwardedArgs` 传入 `--diagnose` 决定。

3. **环境变量映射**
   | 源变量 | 目标变量 |
   |---|---|
   | `GATSBY_CHECK_BROWSER` | `CORPUS_CHECK_BROWSER` |
   | `GATSBY_CHECK_PORT` | `CORPUS_CHECK_PORT` |
   | `GATSBY_CHECK_TIMEOUT_MS` | `CORPUS_CHECK_TIMEOUT_MS` |

4. **IO 透传与退出码继承**
   - stdout/stderr 透传。
   - `result.error` 抛错。
   - `process.exit(result.status ?? 1)`。

## 关键代码路径与文件引用

- 本文件：`scripts/gatsby-sweep.ts`
- 实际执行文件：`scripts/corpus-sweep.ts`
- 被测语料：`corpora/en-gatsby-opening.txt`
- 被测页面：`pages/corpus.ts`
- 调用入口：`package.json` 中 `"gatsby-sweep"`、`"gatsby-sweep:safari"`

## 依赖与外部交互

- **Node/Bun 运行时**：`node:child_process`（`spawnSync`）。
- **本地浏览器**：间接依赖 `corpus-sweep.ts` 的浏览器自动化。
- **环境变量**：`GATSBY_CHECK_BROWSER`, `GATSBY_CHECK_PORT`, `GATSBY_CHECK_TIMEOUT_MS`。

## 风险、边界与改进建议

1. **与 `gatsby-check.ts` 保持对称**：`gatsby-check` 固定带 `--diagnose`，而 `gatsby-sweep` 不带。这是有意设计：sweep 用于快速扫面，check 用于深度诊断。文档中应明确区分两者用途，避免用户混淆。
2. **不支持 `--all` 的语义冲突**：过滤掉 `--all` 是因为固定 `--id` 与 `--all` 互斥。若用户传入 `--all`，会被静默忽略而非报错。虽然实际 `corpus-sweep.ts` 会收到无 `--id` 无 `--all` 的参数并抛错，但错误信息可能不够直观。建议在 `gatsby-sweep.ts` 层显式检测 `--all` 并给出更友好的错误提示。
3. **无独立帮助信息**：同 `gatsby-check.ts`，建议维护文档中的别名说明。
4. **未来废弃路径**：若 `en-gatsby-opening` 语料被移除或重命名，`gatsby-check.ts` 和 `gatsby-sweep.ts` 将同时失效。建议在 `corpora/sources.json` 加载时增加断言，确保 `en-gatsby-opening` 存在。
