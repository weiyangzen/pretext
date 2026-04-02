# oxlintrc.json 研究文档

## 场景与职责

`oxlintrc.json` 是 Oxlint（高性能 JavaScript/TypeScript 静态分析工具）的配置文件，定义了项目代码质量和类型安全规则。Pretext 作为 TypeScript 文本布局库，通过此配置强制执行严格的类型检查和代码规范，确保核心算法的正确性和可维护性。

## 功能点目的

### 1. 插件配置
```json
"plugins": ["typescript", "oxc"]
```
- **typescript**：启用 TypeScript 特定规则
- **oxc**：启用 Oxlint 核心规则（基于 Oxc 编译器）

### 2. 类型安全规则

#### prefer-ts-expect-error
- **级别**：error
- **目的**：强制使用 `@ts-expect-error` 而非 `@ts-ignore`
- **原因**：`@ts-expect-error` 会在错误被修复后产生编译错误，防止注释残留

#### no-unnecessary-type-assertion
- **级别**：error
- **目的**：禁止不必要的类型断言（如 `as string` 当类型已经是 string 时）
- **收益**：减少代码噪音，避免误导性类型信息

#### no-unreachable
- **级别**：error
- **目的**：禁止不可达代码
- **场景**：防止 `return` 后的代码、永远为假的条件分支

### 3. 严格类型检查规则

#### @typescript-eslint/no-unsafe-assignment
- **级别**：error
- **目的**：禁止将 `any` 类型赋值给有明确类型的变量
- **重要性**：防止隐式的类型污染传播

#### @typescript-eslint/no-unsafe-return
- **级别**：error
- **目的**：禁止从函数返回 `any` 类型
- **重要性**：确保函数契约的完整性

#### @typescript-eslint/prefer-nullish-coalescing
```json
["error", { "ignorePrimitives": { "boolean": true } }]
```
- **目的**：优先使用 `??` 而非 `||` 进行空值合并
- **例外**：允许布尔值使用 `||`（因 `false || default` 是常见模式）
- **示例**：
  ```typescript
  // 推荐
  const value = maybeNull ?? defaultValue;
  // 布尔值例外允许
  const enabled = config.enabled || true;
  ```

#### @typescript-eslint/strict-boolean-expressions
```json
["error", {
  "allowNullableBoolean": true,
  "allowString": false
}]
```
- **目的**：强制显式布尔检查，避免隐式类型转换
- **配置**：
  - `allowNullableBoolean: true`：允许 `if (maybeBool)`（`boolean | null | undefined`）
  - `allowString: false`：禁止 `if (string)`（空字符串为 falsy 可能导致 bug）
- **示例**：
  ```typescript
  // 错误
  if (text) { ... }
  // 正确
  if (text !== '') { ... }
  ```

### 4. 穷尽性检查规则

#### @typescript-eslint/switch-exhaustiveness-check
```json
["error", {
  "allowDefaultCaseForExhaustiveSwitch": false,
  "considerDefaultExhaustiveForUnions": true,
  "requireDefaultForNonUnion": true
}]
```
- **目的**：确保 switch 语句处理所有可能的值
- **配置解析**：
  - `allowDefaultCaseForExhaustiveSwitch: false`：即使已穷尽所有 case，也不允许多余的 `default`
  - `considerDefaultExhaustiveForUnions: true`：对于联合类型，如果有 `default` 则视为穷尽
  - `requireDefaultForNonUnion`: true：非联合类型必须有 `default` case
- **示例**：
  ```typescript
  type BreakKind = 'text' | 'space' | 'tab';
  
  switch (kind) {
    case 'text': ...
    case 'space': ...
    case 'tab': ...
    // 不需要 default，因为联合类型已穷尽
  }
  ```

### 5. 条件优化规则

#### @typescript-eslint/no-unnecessary-condition
```json
["error", { "allowConstantLoopConditions": true }]
```
- **目的**：禁止不必要的条件检查（如永远为真/假的条件）
- **例外**：`allowConstantLoopConditions: true` 允许 `while (true)` 等循环模式
- **示例**：
  ```typescript
  // 错误：类型定义已确保非空
  if (array && array.length > 0) { ... }
  // 正确
  if (array.length > 0) { ... }
  ```

## 具体技术实现

### 配置继承机制
- Oxlint 支持配置文件继承（类似 ESLint 的 `extends`）
- 当前配置为基础配置，未使用继承

### 与 TypeScript 的集成
- Oxlint 使用 Oxc 编译器解析 TypeScript，速度比 ESLint + TypeScript 解析器快 50-100 倍
- 支持类型感知（type-aware）的 lint 规则

### 命令行使用
```bash
# 基础检查
oxlint src

# 类型感知检查（使用 tsconfig.json）
oxlint --type-aware src

# package.json 中的 check 脚本
"check": "tsc && oxlint --type-aware src"
```

## 关键代码路径与文件引用

### 相关文件
| 文件 | 关系 |
|------|------|
| `package.json` | 定义 `check` 脚本：`"check": "tsc && oxlint --type-aware src"` |
| `tsconfig.json` | 提供类型信息给 `--type-aware` 模式 |
| `src/**/*.ts` | 被 lint 的目标文件 |
| `bun.lock` | 锁定 `oxlint@1.51.0` 和 `oxlint-tsgolint@0.15.0` 版本 |

### 脚本调用链
```
bun run check
├── tsc (类型检查)
└── oxlint --type-aware src (基于类型的 lint)
    └── 读取 oxlintrc.json 配置
```

### 与 CI 的集成
`.github/workflows/pages.yml`:
```yaml
- run: bun install --frozen-lockfile
# 注意：CI 当前未显式运行 oxlint，依赖开发者本地检查
```

## 依赖与外部交互

### Oxlint 工具链
| 包 | 作用 |
|----|------|
| `oxlint` | 核心 linter，基于 Rust/Oxc |
| `oxlint-tsgolint` | TypeScript 特定规则的插件包 |
| `tsgolint` | 额外的 TypeScript 规则 |

### 与 TypeScript 编译器的关系
- Oxlint 不替代 `tsc`，而是互补
- `tsc` 负责类型检查和 `.d.ts` 生成
- Oxlint 负责代码质量和风格规则

### 与编辑器集成
- 支持 LSP 协议，可在 VS Code 等编辑器中实时显示错误
- 需要安装 Oxlint 扩展

## 风险、边界与改进建议

### 风险点

1. **规则过于严格**
   - `strict-boolean-expressions` 禁止字符串隐式转换，可能增加样板代码
   - 新贡献者可能需要时间适应

2. **类型感知性能**
   - `--type-aware` 模式需要读取 `tsconfig.json` 和类型定义
   - 大型项目可能变慢（但 Oxlint 仍比 ESLint 快）

3. **规则冲突**
   - 某些规则可能与 Prettier 或其他格式化工具冲突
   - 当前项目未使用 Prettier，避免了冲突

### 边界情况

1. **测试文件处理**
   - 当前配置未区分源文件和测试文件
   - `src/layout.test.ts` 可能包含故意违反规则的代码（如测试边缘情况）

2. **第三方类型定义**
   - `@types/bun` 和 `@types/node` 的类型可能不够精确
   - 可能导致误报或需要 `@ts-expect-error`

### 改进建议

1. **添加 CI lint 检查**
   在 `.github/workflows/pages.yml` 中添加：
   ```yaml
   - name: Lint
     run: bun run check
   ```

2. **测试文件特殊配置**
   添加 `overrides` 配置（如果 Oxlint 支持）：
   ```json
   {
     "overrides": [
       {
         "files": ["*.test.ts"],
         "rules": {
           "@typescript-eslint/no-unsafe-assignment": "warn"
         }
       }
     ]
   }
   ```

3. **添加更多实用规则**
   考虑启用：
   ```json
   {
     "rules": {
       "no-console": "warn",
       "eqeqeq": ["error", "always"],
       "prefer-const": "error"
     }
   }
   ```

4. **文档化规则决策**
   在 `DEVELOPMENT.md` 中添加规则说明：
   ```markdown
   ### 代码规范
   - 使用 `??` 而非 `||` 进行空值合并（布尔值除外）
   - 显式比较字符串和数字，避免隐式布尔转换
   - switch 语句必须穷尽所有联合类型成员
   ```

5. **考虑添加 import 排序规则**
   ```json
   {
     "rules": {
       "sort-imports": ["error", { "ignoreDeclarationSort": true }]
     }
   }
   ```
