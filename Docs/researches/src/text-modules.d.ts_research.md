# src/text-modules.d.ts 研究文档

## 场景与职责

`text-modules.d.ts` 是 Pretext 库的 TypeScript 类型声明文件，为 `.txt` 文件导入提供模块类型定义。该文件使 TypeScript 编译器能够识别和理解项目中通过 `import` 语句加载的纯文本文件（如语料库文件）。

**核心职责：**
1. 声明 `*.txt` 文件的模块类型
2. 使 TypeScript 支持文本文件的原生导入
3. 确保类型安全地访问导入的文本内容

**在架构中的位置：**
- 位于 `src/` 目录，属于编译时类型系统的一部分
- 被 TypeScript 编译器自动识别（`.d.ts` 文件）
- 支持 `corpora/` 目录中文本语料的导入

---

## 功能点目的

### 1. 文本文件模块声明

TypeScript 默认无法识别非代码文件的导入。该声明文件告诉编译器：
- `*.txt` 文件可以作为模块导入
- 导入的默认导出是 `string` 类型

### 2. 语料库导入支持

项目中多处导入语料库文件：

```typescript
// pages/benchmark.ts
import jaKumoNoIto from '../corpora/ja-kumo-no-ito.txt' with { type: 'text' }
import zhZhufu from '../corpora/zh-zhufu.txt' with { type: 'text' }
```

没有该声明文件，TypeScript 会报错：
```
Cannot find module '../corpora/ja-kumo-no-ito.txt' or its corresponding type declarations.
```

---

## 具体技术实现

### 类型声明内容

```typescript
declare module '*.txt' {
  const text: string
  export default text
}
```

### 技术细节

| 元素 | 说明 |
|------|------|
| `declare module '*.txt'` | 声明匹配 `*.txt` 通配符的模块 |
| `const text: string` | 模块内默认导出的类型为字符串 |
| `export default text` | 导出默认的字符串常量 |

### 与 Import Attributes 的配合

项目使用 TypeScript 5.3+ 的 Import Attributes 语法：

```typescript
import text from './file.txt' with { type: 'text' }
```

该声明文件与 `with { type: 'text' }` 属性配合：
- TypeScript 类型系统：通过 `.d.ts` 文件识别模块类型
- 运行时加载：通过 `with { type: 'text' }` 指示构建工具以文本方式处理

### 构建工具支持

Bun 支持直接导入文本文件：
- `with { type: 'text' }` 属性告知 Bun 将文件内容作为字符串加载
- 无需额外的 loader 配置

---

## 依赖与外部交互

### 文件位置

```
src/
├── line-break.ts
├── measurement.ts
├── test-data.ts
├── text-modules.d.ts  <-- 本文件
├── analysis.ts
├── layout.ts
└── ...
```

### 受益文件

| 文件 | 导入的文本 |
|------|-----------|
| `pages/benchmark.ts` | 多个语料库 `.txt` 文件 |
| `pages/corpus.ts` | 语料库 `.txt` 文件 |
| `pages/gatsby.ts` | Gatsby 语料 `.txt` 文件 |
| `pages/probe.ts` | 可能导入的测试文本 |

### TypeScript 配置

`tsconfig.json` 中相关配置：

```json
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowArbitraryExtensions": true,
    "resolveJsonModule": true
  }
}
```

- `allowArbitraryExtensions`: 允许任意扩展名的模块导入
- `moduleResolution: "bundler"`: 使用 bundler 风格的模块解析

---

## 风险、边界与改进建议

### 已知风险

1. **通配符声明的局限性**
   - 所有 `.txt` 文件都被视为导出字符串
   - 无法区分不同用途的文本文件（语料 vs 配置）
   - 无法提供文件特定的类型信息

2. **构建工具依赖**
   - 依赖 Bun 的文本导入支持
   - 在其他运行时（Node.js、Deno）可能需要额外配置

3. **路径解析**
   - 相对路径 `../corpora/file.txt` 需要正确配置 baseUrl 或 paths
   - 移动文件位置可能导致类型错误

### 边界情况

| 场景 | 行为 |
|------|------|
| 导入不存在的 `.txt` 文件 | TypeScript 编译时无错误，运行时错误 |
| 空文本文件 | 导入为空字符串 `''` |
| 大文本文件 | 无特殊处理，完整加载到内存 |
| 非 UTF-8 编码 | 依赖 Bun 的编码处理 |

### 改进建议

1. **细化模块声明**
   - 为特定语料库路径提供更精确的类型：
   ```typescript
   declare module '*/corpora/*.txt' {
     const text: string
     export default text
   }
   ```

2. **添加元数据类型**
   - 如果需要，可以扩展为包含元数据的对象：
   ```typescript
   declare module '*.txt' {
     const content: string
     const path: string
     export default { content, path }
   }
   ```

3. **文档化使用方式**
   - 在 README 中说明语料库导入方式
   - 提供添加新语料的示例

4. **考虑替代方案**
   - 如果项目迁移到不支持原生文本导入的环境，考虑：
     - 使用构建插件转换
     - 运行时 `fetch` 加载
     - 将文本嵌入 JSON

### 与其他项目的兼容性

该模式基于 Bun 的文本导入特性：
- **Bun**: 原生支持 `with { type: 'text' }`
- **Vite**: 可通过插件支持
- **Webpack**: 需要 `raw-loader` 或类似配置
- **Node.js**: 实验性支持或需要 loader

如果项目需要支持多种构建工具，可能需要：
1. 条件类型声明
2. 构建时转换
3. 运行时加载抽象层
