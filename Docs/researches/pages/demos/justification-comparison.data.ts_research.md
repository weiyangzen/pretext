# justification-comparison.data.ts 研究文档

## 场景与职责

`justification-comparison.data.ts` 是 Justification Comparison 演示的数据层模块，负责提供：

1. **演示文本内容**：关于排版对齐算法的四段教育性文本
2. **排版常量**：字体、字号、行高等设计参数
3. **连字符词典**：手工维护的音节划分映射，用于软连字符插入
4. **词缀规则**：常见前缀和后缀列表，用于启发式分词

该模块是纯数据文件，无业务逻辑，可被模型层（`.model.ts`）安全导入使用。

## 功能点目的

### 1. 教育性文本内容

四段文本系统地介绍了排版对齐问题：

| 段落 | 主题 |
|-----|------|
| 1 | 排版颜色与阅读舒适度的关系，"河流"（rivers）现象 |
| 2 | 传统排版系统解决河流问题的技术 |
| 3 | Knuth-Plass 算法介绍（TeX 系统） |
| 4 | 现代 CSS 对齐的局限性 |

文本选择 intentionally 包含大量长单词，以突出不同对齐算法的差异。

### 2. 连字符化支持

提供两种分词机制：

1. **精确词典** (`HYPHEN_EXCEPTIONS`)：手工维护的 100+ 单词音节划分
2. **启发式规则** (`PREFIXES` + `SUFFIXES`)：基于词缀的简易分词

### 3. 设计系统常量

统一的排版参数，确保演示的视觉一致性：
- 字体：Georgia, Times New Roman, serif
- 字号：15px
- 行高：24px（1.6 倍行距）
- 内边距：12px
- 段落间距：14.4px（行高的 0.6 倍）

## 具体技术实现

### 文本数据结构

```typescript
export const PARAGRAPHS = [
  `The relationship between typographic colour...`,
  `Traditional typesetting systems addressed...`,
  `The Knuth-Plass algorithm, developed by...`,
  `Modern CSS justification operates on...`,
] as const
```

使用 `as const` 确保类型为字面量元组，而非普通字符串数组。

### 连字符词典结构

```typescript
export const HYPHEN_EXCEPTIONS: Record<string, readonly string[]> = {
  'extensively': ['ex', 'ten', 'sive', 'ly'],
  'relationship': ['re', 'la', 'tion', 'ship'],
  // ... 100+ 条目
}
```

设计决策：
- 键为小写、去标点的标准化形式
- 值为音节数组，保留原始大小写信息
- 使用 `readonly` 防止意外修改

### 词缀列表

```typescript
export const PREFIXES = [
  'anti', 'auto', 'be', 'bi', 'co', 'com', 'con', /* ... */
] as const

export const SUFFIXES = [
  'able', 'ible', 'tion', 'sion', 'ment', 'ness', /* ... */
] as const
```

词缀匹配策略（在 `.model.ts` 中实现）：
1. 优先检查精确词典匹配
2. 其次检查前缀匹配（单词开头）
3. 最后检查后缀匹配（单词结尾）
4. 无匹配时返回原词（不分割）

### 设计常量

```typescript
export const FONT_FAMILY = 'Georgia, "Times New Roman", serif'
export const FONT_SIZE = 15
export const LINE_HEIGHT = 24
export const FONT = `${FONT_SIZE}px ${FONT_FAMILY}`  // "15px Georgia, ..."
export const PAD = 12
export const PARA_GAP = LINE_HEIGHT * 0.6  // 14.4px
```

## 关键代码路径与文件引用

### 导出符号

```typescript
// 文本内容
export const PARAGRAPHS

// 排版参数
export const FONT_FAMILY
export const FONT_SIZE
export const LINE_HEIGHT
export const FONT
export const PAD
export const PARA_GAP

// 分词数据
export const HYPHEN_EXCEPTIONS
export const PREFIXES
export const SUFFIXES
```

### 消费者

| 导入者 | 用途 |
|-------|------|
| `justification-comparison.model.ts` | 核心布局算法使用所有导出 |
| `justification-comparison.ui.ts` | 渲染层使用 `PARAGRAPHS`, `LINE_HEIGHT`, `PAD`, `PARA_GAP` |

### 使用示例（来自 model.ts）

```typescript
import {
  FONT,
  HYPHEN_EXCEPTIONS,
  LINE_HEIGHT,
  PAD,
  PARA_GAP,
  PARAGRAPHS,
  PREFIXES,
  SUFFIXES,
} from './justification-comparison.data.ts'

// 预处理文本
const basePreparedParagraphs = PARAGRAPHS.map(paragraph => 
  prepareWithSegments(paragraph, FONT)
)

// 分词
function hyphenateWord(word: string): string[] {
  const lower = word.toLowerCase().replace(/[.,;:!?"'—–-]/g, '')
  const exactMatch = HYPHEN_EXCEPTIONS[lower]
  if (exactMatch !== undefined) { /* ... */ }
  // 检查前缀/后缀...
}
```

## 依赖与外部交互

### 外部依赖

该模块无外部导入，是纯数据定义。

### 数据规模

| 数据项 | 规模 |
|-------|------|
| PARAGRAPHS | 4 段文本，约 1200 字符 |
| HYPHEN_EXCEPTIONS | 约 130 个词条 |
| PREFIXES | 58 个前缀 |
| SUFFIXES | 27 个后缀 |

### 数据维护

连字符词典基于以下来源构建：
- 美式英语音节划分规则
- 排版惯例（如 `-tion`、`-ing` 等常见后缀）
- 演示文本中实际出现的高频词汇

**注意**：这不是完整的连字符词典，仅覆盖演示文本中的词汇。

## 风险、边界与改进建议

### 已知风险

1. **连字符词典不完整**
   - 仅覆盖演示文本中的词汇，无法处理通用文本
   - 如果扩展演示文本，需要同步扩展词典

2. **词缀规则过于简单**
   - 前缀/后缀匹配可能产生错误的音节边界
   - 例如 "recreation" 可能被错误分割为 "re-creation" 而非 "rec-re-ation"

3. **硬编码英文假设**
   - 词典仅支持英文
   - 无法处理其他语言的连字符规则

### 边界情况处理

边界情况在消费者（`.model.ts`）中处理：

| 边界情况 | 处理方式 |
|---------|---------|
| 单词不在词典中 | 使用词缀启发式或返回原词 |
| 单词长度 < 5 | 跳过连字符化（`lower.length < 5`） |
| 标点符号 | 分词前去除标点（`replace(/[.,;:!?"'—–-]/g, '')`） |
| 大小写混合 | 使用小写进行匹配，保留原始大小写输出 |

### 改进建议

1. **使用标准连字符库**
   - 集成 `hyphen` 或 `hypher` 等成熟库
   - 支持多语言（通过 pattern 文件）
   - 减少维护负担

2. **动态词典扩展**
   - 运行时缓存未覆盖单词的划分结果
   - 提供反馈机制收集用户修正

3. **更智能的分词算法**
   - 实现 Knuth-Liang 连字符算法
   - 使用 TeX 的 pattern 文件

4. **数据验证**
   ```typescript
   // 添加运行时验证
   function validateHyphenation(word: string, parts: string[]): boolean {
     const reconstructed = parts.join('')
     const normalized = word.toLowerCase().replace(/[^a-z]/g, '')
     return reconstructed === normalized
   }
   ```

5. **国际化支持**
   - 添加德语、法语等语言的连字符支持
   - 考虑不同语言的连字符规则差异

6. **代码生成**
   - 从外部词典文件（如 TeX 的 `hyphen.tex`）自动生成 `HYPHEN_EXCEPTIONS`
   - 构建时验证词典完整性

### 维护注意事项

- 修改 `PARAGRAPHS` 时，需要检查新词汇是否在 `HYPHEN_EXCEPTIONS` 中
- 添加新词条时，确保键为小写、去标点形式
- 测试分词结果：`parts.join('')` 应等于原词的去标点形式
