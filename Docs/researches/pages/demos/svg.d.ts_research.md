# svg.d.ts 研究文档

## 场景与职责

`svg.d.ts` 是一个 TypeScript 声明文件（Type Declaration File），用于为项目中的 SVG 资源提供类型支持。在 Pretext 项目中，这个文件使得 TypeScript 编译器能够正确识别和处理通过 ES Module 导入的 `.svg` 文件。

## 功能点目的

### 1. 模块声明（Module Declaration）

```typescript
declare module '*.svg' {
  const src: string
  export default src
}
```

**核心功能**：
- 为所有以 `.svg` 结尾的文件提供类型声明
- 声明 SVG 导入的默认导出是一个 `string` 类型
- 该字符串通常是 SVG 文件在构建后的 URL 路径（由构建工具如 Vite、Webpack 处理）

### 2. 与构建工具的协作

在现代前端构建工具中：
- **Vite**：`import logo from './logo.svg'` 会返回处理后的文件 URL
- **Webpack (with url-loader/file-loader)**：同样返回文件 URL 或 base64 Data URL
- **TypeScript 编译器**：需要这个声明来理解非标准 JS/TS 文件的导入

## 具体技术实现

### TypeScript 声明模块语法

```typescript
declare module '*.svg' {
  const src: string
  export default src
}
```

**语法解析**：
- `declare module '*.svg'`：为匹配 `*.svg` 模式的所有模块创建类型声明
- `const src: string`：声明模块内有一个名为 `src` 的常量，类型为字符串
- `export default src`：将该常量作为默认导出

### 使用示例

在 `dynamic-layout.ts` 中：
```typescript
import openaiLogoUrl from '../assets/openai-symbol.svg'
import claudeLogoUrl from '../assets/claude-symbol.svg'

// TypeScript 知道 openaiLogoUrl 是 string 类型
const OPENAI_LOGO_SRC = resolveImportedAssetUrl(openaiLogoUrl)
```

## 依赖与外部交互

### 文件位置

- 路径：`pages/demos/svg.d.ts`
- 作用域：该目录及其子目录下的 TypeScript 文件

### 与其他文件的交互

| 文件 | 交互方式 | 说明 |
|------|----------|------|
| `dynamic-layout.ts` | `import openaiLogoUrl from '../assets/openai-symbol.svg'` | 使用 SVG 类型声明 |
| `tsconfig.json` | 包含 `.d.ts` 文件 | TypeScript 自动加载声明文件 |

### 构建工具配置

该声明文件假设构建工具会：
1. 处理 `.svg` 导入
2. 返回文件路径字符串（或 Data URL）

## 风险、边界与改进建议

### 当前限制

1. **单一导出模式**：只支持默认导出（`export default`），不支持命名导出
2. **仅字符串类型**：假设 SVG 总是作为字符串 URL 导入，不支持：
   - React 组件形式（如 `@svgr/webpack` 提供的）
   - SVG 内容字符串（内联 SVG）
   - SVG 作为对象（包含 viewBox、width、height 等元数据）

### 改进建议

1. **支持多种导入模式**：
   ```typescript
   // 默认导出（URL）
   declare module '*.svg' {
     const src: string
     export default src
   }
   
   // 如果未来使用 @svgr 等工具
   declare module '*.svg?component' {
     import type { ComponentType, SVGProps } from 'react'
     const component: ComponentType<SVGProps<SVGSVGElement>>
     export default component
   }
   
   // 内联 SVG 字符串
   declare module '*.svg?raw' {
     const content: string
     export default content
   }
   ```

2. **更精确的类型**：
   ```typescript
   // 使用模板字面量类型表示 URL
   type ImageUrl = `/${string}.svg` | `data:image/svg+xml,${string}`
   declare module '*.svg' {
     const src: ImageUrl
     export default src
   }
   ```

3. **集中管理声明**：
   - 当前位于 `pages/demos/` 目录
   - 建议移到项目根目录或 `src/types/` 以便全局使用
   - 或创建 `global.d.ts` 统一管理所有资源类型

### 相关文件扩展建议

如果项目需要支持其他资源类型，可以扩展此模式：

```typescript
// 在 svg.d.ts 或新的 assets.d.ts 中

declare module '*.png' {
  const src: string
  export default src
}

declare module '*.jpg' {
  const src: string
  export default src
}

declare module '*.woff2' {
  const src: string
  export default src
}
```
