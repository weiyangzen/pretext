# gatsby-check.ts 研究文档

## 场景与职责

`scripts/gatsby-check.ts` 是 `scripts/corpus-check.ts` 的**兼容性别名脚本**。它将 Pretext 早期基于 Gatsby 英文语料的独立检查流程，平滑迁移到新的共享 corpus 基础设施上。对用户而言，运行 `bun run gatsby-check` 等价于对 `en-gatsby-opening` 语料执行带诊断的单宽度/窄幅检查。

## 功能点目的

1. **向后兼容 CLI**：保留 `gatsby-check` 命令，使旧文档、 muscle memory 和 CI 脚本无需修改即可继续工作。
2. **环境变量桥接**：将 `GATSBY_CHECK_BROWSER`、`GATSBY_CHECK_PORT`、`GATSBY_CHECK_TIMEOUT_MS` 映射为 `corpus-check.ts` 所需的 `CORPUS_CHECK_*` 环境变量。
3. **参数透传**：除过滤掉 `--all` 和 `--id=` 外，其余所有命令行参数原样转发给 `corpus-check.ts`。

## 具体技术实现

### 核心流程

1. **过滤参数**
   ```ts
   const forwardedArgs = process.argv.slice(2)
     .filter(arg => arg !== '--all' && !arg.startsWith('--id='));
   ```
   移除用户可能误传的 `--id=` 和 `--all`，确保目标语料固定为 `en-gatsby-opening`。

2. **构造子进程命令**
   ```ts
   spawnSync('bun', [
     'run', 'scripts/corpus-check.ts',
     '--id=en-gatsby-opening',
     '--diagnose',
     ...forwardedArgs
   ], { cwd: process.cwd(), env: { ...process.env, ...mappedEnv }, encoding: 'utf8' })
   ```

3. **环境变量映射**
   | 源变量 | 目标变量 |
   |---|---|
   | `GATSBY_CHECK_BROWSER` | `CORPUS_CHECK_BROWSER` |
   | `GATSBY_CHECK_PORT` | `CORPUS_CHECK_PORT` |
   | `GATSBY_CHECK_TIMEOUT_MS` | `CORPUS_CHECK_TIMEOUT_MS` |
   
   映射仅在源变量已定义时生效，避免覆盖用户显式设置的 `CORPUS_CHECK_*`。

4. **IO 透传与退出码继承**
   - 子进程 stdout/stderr 写入当前进程。
   - 若 `result.error` 存在则抛错。
   - 最终以 `result.status ?? 1` 退出。

## 关键代码路径与文件引用

- 本文件：`scripts/gatsby-check.ts`
- 实际执行文件：`scripts/corpus-check.ts`
- 被测语料：`corpora/en-gatsby-opening.txt`
- 被测页面：`pages/corpus.ts`
- 调用入口：`package.json` 中 `"gatsby-check"`、`"gatsby-check:safari"`

## 依赖与外部交互

- **Node/Bun 运行时**：`node:child_process`（`spawnSync`）。
- **本地浏览器**：间接依赖 `corpus-check.ts` 的浏览器自动化。
- **环境变量**：`GATSBY_CHECK_BROWSER`, `GATSBY_CHECK_PORT`, `GATSBY_CHECK_TIMEOUT_MS`。

## 风险、边界与改进建议

1. **功能冻结，不应新增逻辑**：该脚本的唯一职责是转发。任何新的 gatsby 专属逻辑都应下沉到 `corpus-check.ts` 或 `pages/corpus.ts`，避免别名脚本膨胀。
2. **参数过滤可能误伤**：若用户传入类似 `--idempotent` 的参数（以 `--id` 开头但不是 `--id=`），会被错误过滤。当前过滤条件为 `!arg.startsWith('--id=')`，已避免误伤 `--idle` 等，但 `--id=foo` 仍会被过滤。这是预期行为。
3. **无独立文档与帮助信息**：运行 `bun run gatsby-check --help` 会实际转发给 `corpus-check.ts --help`，但 `corpus-check.ts` 的 help 中不会提到 `GATSBY_CHECK_*` 环境变量。建议在 `DEVELOPMENT.md` 或 `package.json` 描述中保留别名说明。
4. **退出码行为**：`process.exit(result.status ?? 1)` 在 `result.status` 为 `0` 时正常退出，为 `null` 时（如信号终止）退 `1`。这与 `corpus-check.ts` 直接运行的行为一致，合理。
