# bun.lock 研究文档

## 场景与职责

`bun.lock` 是 Bun 包管理器的锁定文件（lockfile），用于精确记录项目依赖的确切版本和解析结果。它确保在不同环境（开发机、CI/CD、其他开发者机器）中安装的依赖完全一致，消除"在我机器上能运行"的问题。

## 功能点目的

### 1. 依赖版本锁定
- 记录每个依赖包的确切版本号（如 `oxlint@1.51.0`）
- 记录子依赖的解析结果
- 防止因语义化版本（semver）范围导致的意外更新

### 2. 可重现安装
- 通过 `bun install --frozen-lockfile` 确保 CI/CD 使用完全相同的依赖树
- 加速安装过程（Bun 可直接读取锁定文件而无需解析依赖）

### 3. 安全验证
- 包含每个包的完整性哈希（integrity hash）
- 检测依赖是否被篡改

### 4. 平台特定依赖处理
- 记录可选的平台特定二进制包（如 `darwin-arm64`, `linux-x64`）
- 支持多平台开发团队的协作

## 具体技术实现

### 文件结构
```json
{
  "lockfileVersion": 1,
  "configVersion": 1,
  "workspaces": { ... },
  "packages": { ... }
}
```

### 关键字段解析

#### lockfileVersion / configVersion
- 锁定文件格式版本，用于 Bun 升级时的兼容性处理

#### workspaces
```json
"": {
  "name": "pretext",
  "devDependencies": {
    "@types/bun": "latest",
    "oxlint": "^1.51.0",
    ...
  }
}
```
- `""` 表示根工作区
- `devDependencies`：开发依赖（不随包发布）
- 版本前缀：`^` 允许次版本和补丁版本更新

#### packages
每个包条目格式：
```json
"package-name": ["resolved-version", "", { "os": "...", "cpu": "..." }, "integrity-hash"]
```

例如：
```json
"@oxlint/binding-darwin-arm64": [
  "@oxlint/binding-darwin-arm64@1.51.0",  // 解析后的完整包名
  "",                                      // 占位符
  { "os": "darwin", "cpu": "arm64" },      // 平台约束
  "sha512-3QJbeYaMHn6Bh2XeBXuITSsbnIctyTjvHf5nRjKYrT9pPeErNIpp5VDEeAXC0CZSwSVTsc8WOSDwgrAI24JolQ=="  // 完整性校验
]
```

### 平台特定包分析

项目依赖两个主要工具链，均有大量平台特定二进制包：

#### Oxlint (JavaScript/TypeScript Linter)
- 主包：`oxlint@1.51.0`
- 平台绑定包：`@oxlint/binding-*`（17 个平台变体）
- 覆盖平台：Android、Darwin、FreeBSD、Linux、OpenHarmony、Windows
- 架构：arm、arm64、x64、ia32、ppc64、riscv64、s390x

#### oxlint-tsgolint (TypeScript 特定规则)
- 主包：`oxlint-tsgolint@0.15.0`
- 平台包：`@oxlint-tsgolint/*`（6 个平台变体）
- 覆盖主要桌面平台

### 依赖关系图
```
pretext (root)
├── @types/bun@1.3.10
│   └── bun-types@1.3.10
│       └── @types/node@25.3.3
│           └── undici-types@7.18.2
├── oxlint@1.51.0
│   └── [17 个平台特定绑定包]
│   └── peer: oxlint-tsgolint@>=0.15.0
├── oxlint-tsgolint@0.15.0
│   └── [6 个平台特定包]
├── tsgolint@0.0.1
└── typescript@6.0.2
```

## 关键代码路径与文件引用

### 相关文件
| 文件 | 关系 |
|------|------|
| `package.json` | 定义依赖范围和版本约束，`bun.lock` 是其解析结果 |
| `.github/workflows/pages.yml` | 使用 `bun install --frozen-lockfile` 确保可重现构建 |
| `DEVELOPMENT.md` | 文档说明使用 `bun install` 初始化项目 |

### 脚本使用
```bash
# 安装依赖（使用锁定文件）
bun install

# CI 环境强制使用锁定文件
bun install --frozen-lockfile

# 更新依赖并重新生成锁定文件
bun update
```

## 依赖与外部交互

### 与 Bun 运行时的交互
- Bun 读取 `bun.lock` 直接获取依赖解析结果，无需像 npm/yarn 那样重新计算依赖树
- 支持 Bun 的文本锁定格式（此文件为 JSON 格式，Bun 也支持二进制格式 `.lockb`）

### 与 CI/CD 的交互
`.github/workflows/pages.yml`:
```yaml
- run: bun install --frozen-lockfile
```
- `--frozen-lockfile` 确保如果 `package.json` 和 `bun.lock` 不一致时安装失败
- 防止 CI 使用意外更新的依赖版本

### 与包发布的交互
- `package.json` 的 `devDependencies` 中的包不会随 Pretext 发布到 npm
- 最终用户安装 `@chenglou/pretext` 时不会下载这些开发依赖

## 风险、边界与改进建议

### 风险点

1. **版本漂移风险**
   - `@types/bun` 使用 `latest` 标签，每次安装可能获取不同版本
   - 建议：锁定到具体版本以确保完全可重现

2. **平台包体积**
   - 17 个 Oxlint 平台包 + 6 个 oxlint-tsgolint 平台包
   - 虽然 Bun 只下载当前平台需要的包，但锁定文件体积较大

3. **锁定文件格式变更**
   - Bun 的锁定文件格式仍在演进
   - `lockfileVersion: 1` 未来可能需要升级

### 边界情况

1. **跨平台开发**
   - macOS (ARM64) 开发者提交的锁定文件包含 `darwin-arm64` 包
   - Linux CI 会忽略这些包，只安装 `linux-x64` 变体
   - 锁定文件设计支持这种场景

2. **可选依赖**
   - `oxlint-tsgolint` 标记为 `optionalPeers`
   - 即使某些平台包缺失，安装也能继续

### 改进建议

1. **锁定 @types/bun 版本**
   ```json
   "@types/bun": "1.3.10"
   // 替代 "latest"
   ```

2. **定期更新依赖**
   ```bash
   # 每月运行一次
   bun update
   bun run check  # 确保更新后测试通过
   ```

3. **添加锁定文件验证 CI 步骤**
   ```yaml
   - name: Verify lockfile
     run: |
       bun install --frozen-lockfile
       git diff --exit-code bun.lock
   ```

4. **考虑使用二进制锁定格式**
   - Bun 支持 `.lockb` 二进制格式，体积更小、解析更快
   - 权衡：可读性 vs 性能

5. **文档化依赖更新流程**
   在 `DEVELOPMENT.md` 中添加：
   ```markdown
   ### 更新依赖
   1. 运行 `bun update` 更新锁定文件
   2. 运行 `bun run check` 验证类型和 lint
   3. 运行 `bun test` 确保测试通过
   4. 提交 `bun.lock` 变更
   ```
