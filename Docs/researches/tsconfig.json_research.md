# tsconfig.json 研究文档

## 场景与职责

`tsconfig.json` 是 TypeScript 项目的基础配置文件，定义了 Pretext 库的编译器选项、模块解析策略和类型检查严格程度。作为 `tsconfig.build.json` 的父配置，它同时服务于开发时的类型检查和 IDE 支持，以及构建时的基础编译行为。

## 功能点目的

### 1. 运行时库与目标
```json
{
  "compilerOptions": {
    "lib": ["ESNext", "DOM"],
    "target": "ESNext",
    "module": "Preserve"
  }
}
```
- **`lib: ["ESNext", "DOM"]`**：包含最新的 ECMAScript API 和浏览器 DOM API
  - `ESNext`：支持最新 JavaScript 特性（如 `Array.prototype.toSorted`）
  - `DOM`：支持 `document`, `canvas`, `Intl` 等浏览器 API
- **`target: "ESNext"`**：输出最新 JavaScript 语法
  - 不转译现代语法（如箭头函数、async/await）
  - 依赖运行环境支持（现代浏览器和 Bun）
- **`module: "Preserve"`**：保留原始模块语法
  - 不转换 `import`/`export` 语句
  - 输出原生 ESM 代码

### 2. 模块解析策略
```json
{
  "compilerOptions": {
    "moduleDetection": "force",
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true
  }
}
```
- **`moduleDetection: "force"`**：强制将所有文件视为模块
  - 即使文件没有 `import`/`export` 也视为 ESM
  - 避免意外创建全局作用域
- **`moduleResolution: "bundler"`**：使用 bundler 风格的模块解析
  - 支持 `exports` 字段
  - 支持裸导入（bare imports）解析
  - 与 Vite、Webpack、Bun 等工具一致
- **`allowImportingTsExtensions: true`**：允许导入 `.ts` 文件
  - 源码中使用 `import './bidi.js'` 导入 `bidi.ts`
  - TypeScript 理解这种映射
- **`verbatimModuleSyntax: true`**：保留导入语句的语法
  - `import type` 和 `import` 严格区分
  - 类型导入在输出中被完全移除

### 3. 严格类型检查
```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noImplicitOverride": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "allowUnreachableCode": true,
    "noFallthroughCasesInSwitch": true,
    "noPropertyAccessFromIndexSignature": true
  }
}
```

#### 核心严格模式
- **`strict: true`**：启用所有严格类型检查选项
  - `noImplicitAny`：禁止隐式 `any`
  - `strictNullChecks`：严格空值检查
  - `strictFunctionTypes`：严格函数类型检查
  - `strictBindCallApply`：严格 `bind`/`call`/`apply` 检查
  - `strictPropertyInitialization`：严格属性初始化检查
  - `noImplicitThis`：禁止隐式 `any` 类型的 `this`
  - `alwaysStrict`：生成严格模式指令

#### 额外严格选项
- **`noUncheckedIndexedAccess: true`**：索引访问可能返回 `undefined`
  ```typescript
  const arr: number[] = [1, 2, 3];
  const x = arr[10]; // 类型为 number | undefined
  ```
- **`noImplicitReturns: true`**：所有代码路径必须返回（非 void 函数）
- **`noImplicitOverride: true`**：重写方法必须使用 `override` 关键字
- **`noUnusedLocals: true`**：报告未使用的局部变量
- **`noUnusedParameters: true`**：报告未使用的函数参数
- **`exactOptionalPropertyTypes: true`**：精确的可选属性类型
  - `undefined` 不能赋值给可选属性
  - 区分 `"prop"?: string` 和 `"prop": string | undefined`
- **`allowUnreachableCode: true`**：允许不可达代码（仅警告，不报错）
- **`noFallthroughCasesInSwitch: true`**：禁止 switch case 穿透（除非显式注释）
- **`noPropertyAccessFromIndexSignature: true`**：索引签名属性必须使用括号访问
  ```typescript
  interface Dict { [key: string]: string }
  const d: Dict = {};
  const x = d['key']; // 允许
  const y = d.key;    // 错误
  ```

### 4. 类型定义与兼容性
```json
{
  "compilerOptions": {
    "allowJs": true,
    "types": ["bun", "node"],
    "skipLibCheck": true,
    "noEmit": true
  }
}
```
- **`allowJs: true`**：允许 JavaScript 文件参与编译
  - 支持混合 TS/JS 项目
  - 可能对纯 TS 项目不必要，但提供灵活性
- **`types: ["bun", "node"]`**：显式包含的类型定义
  - `bun`：Bun 运行时 API（`Bun.file`, `Bun.write` 等）
  - `node`：Node.js API（文件系统、路径等）
- **`skipLibCheck: true`**：跳过声明文件（`.d.ts`）的类型检查
  - 加速编译
  - 避免第三方类型定义的问题
- **`noEmit: true`**：不输出文件（仅类型检查）
  - 开发配置的核心特征
  - 构建时由 `tsconfig.build.json` 覆盖为 `noEmit: false`

## 具体技术实现

### 配置继承机制
```
tsconfig.build.json
└── extends: tsconfig.json
    └── 继承所有 compilerOptions
    └── 覆盖 noEmit, allowImportingTsExtensions 等
```

### 类型检查流程
```
tsc (无参数，默认读取 tsconfig.json)
├── 解析配置
├── 扫描项目文件
├── 类型推断和检查
│   ├── 严格 null 检查
│   ├── 未使用变量分析
│   ├── 穷尽性检查
│   └── ...
└── 报告错误（不输出文件）
```

### 与 Bun 的集成
- Bun 内置 TypeScript 支持，读取 `tsconfig.json`
- Bun 运行时类型由 `@types/bun` 提供
- `types: ["bun", "node"]` 确保两者都可用

### 与 Oxlint 的集成
```bash
oxlint --type-aware src
```
- `--type-aware` 模式读取 `tsconfig.json` 获取类型信息
- 结合类型信息的 lint 规则（如 `no-unsafe-assignment`）

## 关键代码路径与文件引用

### 相关文件
| 文件 | 关系 |
|------|------|
| `tsconfig.build.json` | 继承此配置，用于发布构建 |
| `package.json` | 定义 `"check": "tsc && oxlint --type-aware src"` |
| `oxlintrc.json` | Oxlint 的 `--type-aware` 模式读取此配置 |
| `src/**/*.ts` | 受此配置约束的源文件 |

### 脚本调用链
```
bun run check
├── tsc
│   └── 读取 tsconfig.json
│   └── 类型检查（noEmit: true）
└── oxlint --type-aware src
    └── 读取 tsconfig.json 获取类型信息
```

### 源码中的类型应用示例
```typescript
// src/layout.ts
export type PreparedText = {
  readonly [preparedTextBrand]: true
  readonly width: number
  readonly height: number
}

// strict: true 确保：
// - 所有属性类型必须显式声明
// - null/undefined 必须显式处理
// - 索引访问返回 number | undefined（noUncheckedIndexedAccess）
```

## 依赖与外部交互

### 与 TypeScript 编译器的交互
- 版本：6.0.2（由 `package.json` 指定）
- 命令行：`tsc`（无参数，自动查找 `tsconfig.json`）

### 与编辑器的交互
- VS Code 等编辑器自动读取 `tsconfig.json`
- 提供：
  - 智能提示（IntelliSense）
  - 实时错误检查
  - 重构支持
  - 跳转到定义

### 与 Bun 运行时的交互
- Bun 读取 `tsconfig.json` 的 `compilerOptions.paths`（虽然当前未使用）
- Bun 使用 `types` 字段确定包含哪些类型定义

### 与测试的交互
```bash
bun test
```
- Bun 的测试运行器使用 `tsconfig.json` 进行类型检查
- 测试文件 `src/layout.test.ts` 受此配置约束

## 风险、边界与改进建议

### 风险点

1. **过于严格的配置**
   - `noUncheckedIndexedAccess` 可能导致大量 `| undefined` 类型
   - 需要频繁使用非空断言（`!`）或类型守卫
   - 新贡献者学习曲线陡峭

2. **`allowUnreachableCode: true`**
   - 允许不可达代码可能隐藏逻辑错误
   - 建议改为 `false` 或仅用于特定文件

3. **类型定义冲突**
   - `types: ["bun", "node"]` 可能产生冲突
   - 两者都定义了 `Buffer`、`process` 等全局变量

### 边界情况

1. **JavaScript 文件**
   - `allowJs: true` 但项目实际全为 TypeScript
   - 如果引入 JS 依赖，类型检查可能不完整

2. **第三方库类型**
   - `skipLibCheck: true` 跳过 `.d.ts` 检查
   - 如果第三方类型有误，可能在运行时暴露

3. **模块解析边缘情况**
   - `moduleResolution: "bundler"` 是相对较新的选项
   - 某些旧工具可能不完全支持

### 改进建议

1. **调整不可达代码策略**
   ```json
   {
     "compilerOptions": {
       "allowUnreachableCode": false
     }
   }
   ```
   或添加注释禁用：
   ```typescript
   // @ts-expect-error 故意保留的调试代码
   console.log('unreachable');
   ```

2. **考虑使用 paths 别名**
   ```json
   {
     "compilerOptions": {
       "paths": {
         "@/*": ["./src/*"]
       }
     }
   }
   ```
   简化深层导入：
   ```typescript
   // 替代 import '../../../measurement.js'
   import { getSegmentMetrics } from '@/measurement.js'
   ```

3. **添加 isolatedModules**
   ```json
   {
     "compilerOptions": {
       "isolatedModules": true
     }
   }
   ```
   - 确保每个文件可以独立编译
   - 对 Babel、esbuild、Bun 等工具更友好

4. **考虑 esModuleInterop**
   虽然 `verbatimModuleSyntax: true` 已处理大部分情况，但如果需要与 CommonJS 库互操作：
   ```json
   {
     "compilerOptions": {
       "esModuleInterop": true,
       "allowSyntheticDefaultImports": true
     }
   }
   ```

5. **文档化严格规则处理模式**
   在 `DEVELOPMENT.md` 添加：
   ```markdown
   ### 处理严格类型检查
   - 数组索引访问返回 `T | undefined`，使用类型守卫处理
   - 可选属性与 `undefined` 是不同的类型
   - 未使用的变量/参数会被报错，使用 `_` 前缀忽略
   ```

6. **添加注释说明配置意图**
   在 `tsconfig.json` 顶部添加：
   ```json
   {
     "_comment": "Base TypeScript config. See tsconfig.build.json for build overrides."
   }
   ```

7. **考虑严格性分级**
   如果严格配置阻碍开发效率，可考虑：
   - 保持 `strict: true`
   - 但将 `noUncheckedIndexedAccess` 设为 `false`
   - 或在特定文件使用 `// @ts-nocheck`（不推荐常规使用）
