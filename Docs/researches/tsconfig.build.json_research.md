# tsconfig.build.json 研究文档

## 场景与职责

`tsconfig.build.json` 是 TypeScript 编译器的发布构建专用配置文件，负责将 `src/` 目录下的 TypeScript 源码编译为可发布的 JavaScript 和类型定义文件，输出到 `dist/` 目录。它与通用的 `tsconfig.json` 分离，专门用于包发布时的编译流程。

## 功能点目的

### 1. 配置继承
```json
{
  "extends": "./tsconfig.json"
}
```
- 继承基础配置（`tsconfig.json`）的所有编译器选项
- 避免重复定义，确保开发环境和构建环境的一致性
- 仅覆盖发布构建特定的选项

### 2. 启用文件输出
```json
{
  "compilerOptions": {
    "noEmit": false,
    "allowImportingTsExtensions": false
  }
}
```
- **`noEmit: false`**：允许 TypeScript 输出 `.js` 文件
  - 对比：`tsconfig.json` 中 `noEmit: true`（仅类型检查，不输出文件）
  - 这是构建配置与开发配置的核心区别
- **`allowImportingTsExtensions: false`**：禁止在导入语句中使用 `.ts` 扩展名
  - 确保输出的 JavaScript 代码符合标准 ESM 规范
  - 运行时环境不需要特殊配置来解析 `.ts` 导入

### 3. 类型定义生成
```json
{
  "compilerOptions": {
    "declaration": true
  }
}
```
- **`declaration: true`**：为每个 `.ts` 文件生成对应的 `.d.ts` 类型定义文件
- 这是发布 TypeScript 库的关键，允许用户获得类型提示和智能补全

### 4. 输入输出路径配置
```json
{
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist"
  },
  "include": ["src/**/*.ts"],
  "exclude": ["src/layout.test.ts", "src/test-data.ts"]
}
```
- **`rootDir: "./src"`**：指定源码根目录
  - 编译后的文件结构将相对于此目录保留
- **`outDir: "./dist"`**：指定输出目录
  - 对应 `package.json` 中的 `"main": "./dist/layout.js"`
- **`include`**：只编译 `src/` 下的 `.ts` 文件
- **`exclude`**：排除测试文件和测试数据
  - `src/layout.test.ts`：单元测试文件
  - `src/test-data.ts`：测试数据（语料库等）

## 具体技术实现

### 编译流程
```
tsc -p tsconfig.build.json
├── 读取 tsconfig.build.json
│   └── 继承 tsconfig.json 的基础配置
├── 扫描 src/**/*.ts（排除测试文件）
├── 类型检查
├── 生成 JavaScript (dist/*.js)
├── 生成类型定义 (dist/*.d.ts)
└── 输出到 dist/ 目录
```

### 路径映射示例
```
src/
├── layout.ts         → dist/layout.js + dist/layout.d.ts
├── analysis.ts       → dist/analysis.js + dist/analysis.d.ts
├── measurement.ts    → dist/measurement.js + dist/measurement.d.ts
├── line-break.ts     → dist/line-break.js + dist/line-break.d.ts
├── bidi.ts           → dist/bidi.js + dist/bidi.d.ts
├── layout.test.ts    → 被排除
└── test-data.ts      → 被排除
```

### 与 package.json 的集成
```json
// package.json
{
  "scripts": {
    "build:package": "rm -rf dist && tsc -p tsconfig.build.json",
    "prepack": "rm -rf dist && tsc -p tsconfig.build.json"
  }
}
```
- `build:package`：手动构建
- `prepack`：npm pack/publish 前自动构建

### 与 .gitignore 的关系
```gitignore
# .gitignore
dist
```
- `dist/` 被 Git 排除，不进入版本控制
- 但在发布时通过 `package.json` 的 `files` 字段包含：
  ```json
  "files": ["dist", ...]
  ```

## 关键代码路径与文件引用

### 相关文件
| 文件 | 关系 |
|------|------|
| `tsconfig.json` | 被继承的基础配置 |
| `package.json` | 引用 `dist/` 作为发布入口；定义构建脚本 |
| `src/*.ts` | 编译输入源文件 |
| `dist/` | 编译输出目录（构建时生成） |
| `.gitignore` | 排除 `dist/` 目录 |

### 配置继承链
```
tsconfig.build.json
└── extends: tsconfig.json
    ├── compilerOptions.lib: ["ESNext", "DOM"]
    ├── compilerOptions.target: "ESNext"
    ├── compilerOptions.module: "Preserve"
    └── ... (其他严格类型检查选项)
```

### 构建调用链
```
npm publish / bun run build:package
└── rm -rf dist && tsc -p tsconfig.build.json
    ├── 读取配置
    ├── 类型检查（继承的 strict: true）
    ├── 生成 JS + .d.ts
    └── 输出到 dist/
```

## 依赖与外部交互

### 与 TypeScript 编译器的交互
- 使用 TypeScript 6.0.2（由 `package.json` 的 `devDependencies` 指定）
- 命令行调用：`tsc -p tsconfig.build.json`
  - `-p` 或 `--project` 指定配置文件路径

### 与 Bun 的交互
- Bun 内置 TypeScript 支持，但发布构建仍使用官方 `tsc`
- 原因：
  1. `tsc` 生成 `.d.ts` 类型定义文件
  2. Bun 的转译不输出类型定义
  3. 确保与标准 TypeScript 生态兼容

### 与 npm 发布的交互
```
npm publish
├── prepack hook 触发
│   └── tsc -p tsconfig.build.json
│       └── 生成 dist/
├── npm 打包 files 字段指定的文件
│   └── 包含 dist/
└── 发布到 registry
```

### 与源码导入语法的交互
项目源码使用 `.js` 扩展名导入（符合 TypeScript ESM 要求）：
```typescript
// src/layout.ts
import { computeSegmentLevels } from './bidi.js'
import { analyzeText } from './analysis.js'
```

`tsconfig.json` 中的配置支持这种语法：
```json
{
  "compilerOptions": {
    "allowImportingTsExtensions": true
  }
}
```

但在构建配置中关闭此选项，确保输出代码的兼容性。

## 风险、边界与改进建议

### 风险点

1. **配置漂移风险**
   - 两个配置文件（`tsconfig.json` 和 `tsconfig.build.json`）
   - 如果基础配置修改，可能影响构建行为
   - 建议：修改 `tsconfig.json` 后必须测试构建

2. **输出目录未清理风险**
   - `rm -rf dist` 在脚本中执行，但配置本身不强制清理
   - 如果直接运行 `tsc -p tsconfig.build.json`，可能残留旧文件

3. **测试文件排除遗漏**
   - 当前手动列出 `src/layout.test.ts` 和 `src/test-data.ts`
   - 新增测试文件可能被意外包含

### 边界情况

1. **声明文件生成失败**
   - 如果源码有类型错误，`declaration: true` 可能生成不完整的 `.d.ts`
   - 但 `strict: true` 和类型检查会先捕获错误

2. **循环依赖**
   - TypeScript 可以处理循环依赖，但可能影响 tree-shaking
   - 项目使用 ESM，`import` 语句的静态分析有助于优化

3. **路径别名**
   - 当前配置不使用路径别名（如 `@/`）
   - 所有导入使用相对路径，简化构建配置

### 改进建议

1. **使用通配符排除测试文件**
   ```json
   {
     "exclude": ["src/**/*.test.ts", "src/test-data.ts"]
   }
   ```
   或更严格：
   ```json
   {
     "exclude": ["src/**/*.test.ts", "src/**/test-*.ts"]
   }
   ```

2. **添加声明映射（Declaration Map）**
   ```json
   {
     "compilerOptions": {
       "declarationMap": true
     }
   }
   ```
   - 生成 `.d.ts.map` 文件
   - 支持"跳转到定义"跳转到 TypeScript 源码而非 `.d.ts`

3. **添加源码映射（Source Map）**
   ```json
   {
     "compilerOptions": {
       "sourceMap": true
     }
   }
   ```
   - 便于用户调试时映射回 TypeScript 源码

4. **验证构建输出脚本**
   添加 `scripts/verify-build.ts`：
   ```typescript
   import { existsSync } from 'fs';
   
   const required = ['dist/layout.js', 'dist/layout.d.ts'];
   for (const file of required) {
     if (!existsSync(file)) {
       console.error(`Missing: ${file}`);
       process.exit(1);
     }
   }
   console.log('Build verified');
   ```

5. **考虑使用 project references**
   如果项目规模扩大：
   ```json
   // tsconfig.json
   {
     "references": [{ "path": "./tsconfig.build.json" }]
   }
   ```
   - 支持增量构建
   - 更好的大型项目管理

6. **文档化配置差异**
   在 `tsconfig.build.json` 顶部添加注释：
   ```json
   {
     "_comment": "Build-specific overrides to tsconfig.json. See Docs/researches/tsconfig.build.json_research.md"
   }
   ```
