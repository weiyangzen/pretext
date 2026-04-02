# Research: pages/demos/dynamic-layout-text.ts

## 场景与职责

`dynamic-layout-text.ts` 是 `dynamic-layout` 演示的**内容数据源**。它仅导出一个字符串常量 `BODY_COPY`，供 `dynamic-layout.ts` 导入并作为两栏（或窄屏单栏）正文流排版的原始文本。该文本摘录自 Leopold Aschenbrenner 的《Situational Awareness: The Decade Ahead》（2024 年 6 月），用于为演示页面提供一段足够长、具有叙事性的英文长文本。

## 功能点目的

1. **将内容从布局逻辑中剥离**：避免 `dynamic-layout.ts` 因嵌入大段文本而膨胀，使布局算法与内容解耦。  
2. **提供连续长文本流**：正文长度约 1500+ 词，足以填满桌面端两栏或移动端单栏，展示 Pretext 的跨栏光标传递（cursor handoff）能力。  
3. **作为可替换内容模块**：由于仅导出单一字符串，未来若需替换为其他文章、多语言版本或 CMS 数据，只需修改此文件即可，无需触碰复杂的布局代码。

## 具体技术实现

### 导出接口

```ts
export const BODY_COPY = `\
You can see the future first in San Francisco...
`
```

- 使用模板字符串（template literal）并以反斜杠 `\` 结尾开启，确保字符串首行紧贴 `` ` ``，避免产生前导换行。
- 文本内部包含标准英文段落，使用普通空格与换行符分隔段落。

### 内容结构

文本为一段连续的散文，包含多个自然段，主题围绕：
- 旧金山 AI 产业的算力扩张（$10B → $100B → $1T clusters）
- AGI 竞赛的时间线预测（2025-2026 超越大学毕业生，2030 年前达到 superintelligence）
- 地缘政治与国家安全影响

由于文件内容被截断显示，实际文本在源码中是一个完整的、足够长的段落集合。

## 关键代码路径与文件引用

- **唯一消费者**：
  - `dynamic-layout.ts` — 通过 `import { BODY_COPY } from './dynamic-layout-text.ts'` 引入，随后调用 `prepareWithSegments(BODY_COPY, BODY_FONT)` 进行预处理。
- **间接依赖**：
  - `../../src/layout.ts` — `dynamic-layout.ts` 使用其 `prepareWithSegments` 将 `BODY_COPY` 转换为可布局的 `PreparedTextWithSegments`。

## 依赖与外部交互

- **零运行时依赖**：本文件不导入任何模块，也不使用任何浏览器 API。
- **零副作用**：仅导出常量，模块加载即完成。

## 风险、边界与改进建议

1. **版权问题**：文本摘录自外部出版物。虽然这是内部演示仓库，但若项目未来开源或公开展示，需确认是否获得授权或改用原创/公有领域文本。  
2. **硬编码内容不可配置**：当前仅支持单一英文长文本，无法通过 URL query 或 localStorage 动态切换。对于演示页面而言足够，但若需 A/B 测试不同文本长度对布局性能的影响，则需额外封装。  
3. **文件体积**：长文本直接内嵌在 TS 源码中，会增加 bundle 体积（约 ~10KB）。作为演示页面，这属于可接受范围；若用于生产，可考虑懒加载（`fetch` 本地 `.txt`）以减小编译产物。  
4. **改进建议**：
   - 将文本拆分为段落数组 `export const BODY_PARAGRAPHS: string[]`，使 `dynamic-layout.ts` 能在段落边界处更智能地处理硬断行（hard breaks），而非依赖原始换行符。
   - 增加一个简短的 `HEADLINE_COPY` 或 `CREDIT_COPY` 导出，进一步将 `dynamic-layout.ts` 中剩余的硬编码文本（如 `HEADLINE_TEXT`、`CREDIT_TEXT`）也迁移到内容文件，实现完全的内容-布局分离。
