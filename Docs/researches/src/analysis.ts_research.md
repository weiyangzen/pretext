# src/analysis.ts 研究文档

## 场景与职责

`analysis.ts` 是 Pretext 库的文本分析核心模块，负责文本的预处理、分段（segmentation）和规范化。它是两阶段布局架构的第一阶段（文本分析阶段），为后续的测量和布局阶段提供结构化数据。

### 核心职责
1. **空白字符规范化**：根据 CSS `white-space` 规则处理空白字符（`normal` 模式折叠空白，`pre-wrap` 模式保留）
2. **文本分段**：使用 `Intl.Segmenter` 进行语言感知的文本分段
3. **段合并策略**：将标点符号、特殊字符按语言规则合并到相邻词段
4. **多语言支持**：处理 CJK、阿拉伯语、缅甸语、泰语等复杂脚本的断行规则
5. **断点类型分类**：识别并标记 8 种不同的段断点类型

---

## 功能点目的

### 1. 空白字符处理

```typescript
export type WhiteSpaceMode = 'normal' | 'pre-wrap'
```

- **`normal` 模式**（默认）：折叠连续空白字符为单个空格，去除首尾空格
- **`pre-wrap` 模式**：保留普通空格、制表符和硬换行符，用于编辑器/输入场景

### 2. 段断点类型系统

```typescript
export type SegmentBreakKind =
  | 'text'           // 普通文本
  | 'space'          // 可折叠空格
  | 'preserved-space'// 保留空格（pre-wrap）
  | 'tab'            // 制表符
  | 'glue'           // 非断点粘合字符（NBSP, NNBSP, WJ, ZWNBSP）
  | 'zero-width-break'// 零宽断点（ZWSP）
  | 'soft-hyphen'    // 软连字符（ discretionary hyphen）
  | 'hard-break'     // 硬换行符
```

### 3. 多语言断行规则

#### CJK 处理（禁则处理/Kinsoku）
- **行首禁止字符**（`kinsokuStart`）：标点如 `，。、）」』` 等不能出现在行首
- **行尾禁止字符**（`kinsokuEnd`）：开括号如 `（「『` 等不能出现在行尾
- **前向粘附字符**：闭引号 `'’` 等需与前文保持连接
- **跨边界携带**：CJK 文本中，行尾开引号会被携带到下一行开头

#### 阿拉伯语处理
- **无空格标点合并**：冒号、逗号等标点与后续阿拉伯文字合并
- **组合标记处理**：带组合标记的标点后接文字时保持连接
- **前导空格+标记分离**：空格后接组合标记时，标记归属后续文字

#### 缅甸语处理
- **属格标记粘合**：`၏` (U+104F) 与后续文字合并

### 4. 特殊文本模式识别

#### URL 处理
- 识别 `https://` / `http://` / `www.` 开头的 URL
- 将路径和查询字符串合并为独立可断段
- 在 `?` 处分割为两个可断单元（路径+查询）

#### 数字/时间范围
- 合并时间格式如 `7:00-9:00`
- 合并连字符分隔的数字标识符如 `420-69-8008`
- 支持 Unicode 数字（如印地语 `२४×७`）

#### ASCII 标点链
- 合并无空格的标点链如 `foo;bar`、`foo:bar`
- 支持转义引号集群如 `\"word\"`

---

## 具体技术实现

### 核心数据结构

```typescript
// 分段片段（内部中间表示）
type SegmentationPiece = {
  text: string
  isWordLike: boolean
  kind: SegmentBreakKind
  start: number  // 在规范化文本中的起始偏移
}

// 合并后的分段结果
export type MergedSegmentation = {
  len: number
  texts: string[]
  isWordLike: boolean[]
  kinds: SegmentBreakKind[]
  starts: number[]
}

// 分析块（用于 pre-wrap 模式的硬换行分块）
export type AnalysisChunk = {
  startSegmentIndex: number
  endSegmentIndex: number
  consumedEndSegmentIndex: number
}

// 完整文本分析结果
export type TextAnalysis = { 
  normalized: string
  chunks: AnalysisChunk[] 
} & MergedSegmentation
```

### 关键处理流程

#### 1. 主分析流程 `analyzeText()`

```
输入: text, profile, whiteSpace
输出: TextAnalysis

1. 获取空白配置 (getWhiteSpaceProfile)
2. 规范化空白字符
   - normal: normalizeWhitespaceNormal (折叠空白)
   - pre-wrap: normalizeWhitespacePreWrap (保留空白，统一换行符)
3. 如果规范化后为空，返回空结果
4. 构建合并分段 (buildMergedSegmentation)
5. 编译分析块 (compileAnalysisChunks)
6. 返回完整分析结果
```

#### 2. 分段构建流程 `buildMergedSegmentation()`

```
输入: normalized, profile, whiteSpaceProfile
输出: MergedSegmentation

1. 获取共享词分段器 (Intl.Segmenter)
2. 遍历每个词段，按字符类型拆分 (splitSegmentByBreakKind)
3. 应用合并规则（按优先级顺序）：
   a. CJK 闭引号后携带（profile.carryCJKAfterClosingQuote）
   b. CJK 行首禁止段合并到前文
   c. 缅甸语中介标记合并
   d. 阿拉伯语无空格标点后合并
   e. 重复单字符运行合并
   f. 左粘附标点合并到前文
4. 后向粘附处理（前向粘附集群后移到下文）
5. 紧凑化数组（移除空段）
6. 应用高级合并管道：
   - mergeGlueConnectedTextRuns (粘合字符连接)
   - mergeUrlLikeRuns (URL 合并)
   - mergeUrlQueryRuns (URL 查询合并)
   - mergeNumericRuns (数字范围合并)
   - splitHyphenatedNumericRuns (连字符数字拆分)
   - mergeAsciiPunctuationChains (ASCII 标点链)
   - carryTrailingForwardStickyAcrossCJKBoundary (CJK 边界携带)
7. 阿拉伯语特殊处理：空格+组合标记分离
```

#### 3. 字符分类 `classifySegmentBreakChar()`

```typescript
function classifySegmentBreakChar(ch: string, profile: WhiteSpaceProfile): SegmentBreakKind
```

分类逻辑：
- ` ` (空格): `preserved-space` (pre-wrap) / `space` (normal)
- `\t`: `tab`
- `\n`: `hard-break` (pre-wrap)
- `\u00A0` (NBSP), `\u202F` (NNBSP), `\u2060` (WJ), `\uFEFF` (ZWNBSP): `glue`
- `\u200B` (ZWSP): `zero-width-break`
- `\u00AD` (SHY): `soft-hyphen`
- 其他: `text`

### 关键辅助函数

#### CJK 检测 `isCJK()`

支持完整的 CJK Unicode 范围：
- CJK 统一表意文字：U+4E00-U+9FFF, U+3400-U+4DBF
- CJK 扩展 A-G：U+20000-U+3134F
- CJK 兼容字符：U+F900-U+FAFF, U+2F800-U+2FA1F
- 日文假名：U+3040-U+309F (平假名), U+30A0-U+30FF (片假名)
- 韩文音节：U+AC00-U+D7AF
- 全角/半角：U+FF00-U+FFEF

#### 粘附性检测

```typescript
// 左粘附标点检测（应合并到前文）
function isLeftStickyPunctuationSegment(segment: string): boolean

// 前向粘附集群检测（应合并到后文）  
function isForwardStickyClusterSegment(segment: string): boolean

// 转义引号集群检测
function isEscapedQuoteClusterSegment(segment: string): boolean

// 拆分尾部前向粘附集群
function splitTrailingForwardStickyCluster(text: string): { head: string, tail: string } | null
```

---

## 关键代码路径与文件引用

### 主要导出函数

| 函数 | 用途 | 调用方 |
|------|------|--------|
| `analyzeText()` | 主分析入口 | `layout.ts` -> `prepareInternal()` |
| `normalizeWhitespaceNormal()` | 空白规范化 | 内部 |
| `isCJK()` | CJK 字符检测 | `measurement.ts`, 内部 |
| `endsWithClosingQuote()` | 闭引号检测 | `layout.ts`, 内部 |
| `setAnalysisLocale()` | 设置分段语言 | `layout.ts` -> `setLocale()` |
| `clearAnalysisCaches()` | 清除缓存 | `layout.ts` -> `clearCache()` |

### 内部调用关系

```
analyzeText
├── getWhiteSpaceProfile
├── normalizeWhitespaceNormal / normalizeWhitespacePreWrap
├── buildMergedSegmentation
│   ├── getSharedWordSegmenter (Intl.Segmenter)
│   ├── splitSegmentByBreakKind
│   │   └── classifySegmentBreakChar
│   ├── 合并规则循环（多种条件判断）
│   ├── mergeGlueConnectedTextRuns
│   ├── mergeUrlLikeRuns
│   ├── mergeUrlQueryRuns
│   ├── mergeNumericRuns
│   ├── splitHyphenatedNumericRuns
│   ├── mergeAsciiPunctuationChains
│   └── carryTrailingForwardStickyAcrossCJKBoundary
└── compileAnalysisChunks
```

---

## 依赖与外部交互

### 外部依赖

| 依赖 | 用途 |
|------|------|
| `Intl.Segmenter` | 语言感知的词分段（浏览器原生 API） |
| Unicode 属性转义 (`\p{Script=Arabic}`, `\p{M}`, `\p{Nd}`) | 脚本和字符类别检测 |

### 被依赖方

| 文件 | 导入内容 |
|------|----------|
| `layout.ts` | `analyzeText`, `setAnalysisLocale`, `clearAnalysisCaches`, `isCJK`, `endsWithClosingQuote`, 类型定义 |
| `measurement.ts` | `isCJK` |

### 共享状态

```typescript
// 模块级共享状态（需通过 clearAnalysisCaches 重置）
let sharedWordSegmenter: Intl.Segmenter | null = null
let segmenterLocale: string | undefined
```

---

## 风险、边界与改进建议

### 已知风险

1. **共享分段器状态**
   - `sharedWordSegmenter` 是模块级单例，多线程/多实例场景可能竞争
   - 语言切换需调用 `setAnalysisLocale()`，否则可能使用错误语言分段

2. **Unicode 范围维护**
   - CJK 检测硬编码 Unicode 范围，需随 Unicode 版本更新
   - 新增 CJK 扩展块需手动添加

3. **正则表达式性能**
   - `combiningMarkRe`, `arabicScriptRe` 等正则每次调用重新编译
   - 高吞吐量场景可能成为瓶颈

### 边界条件

1. **空文本处理**
   - 空白-only 输入返回空分段数组
   - 调用方需处理 `len === 0` 情况

2. **pre-wrap 模式空行**
   - 连续 `\n\n` 产生空分析块
   - `compileAnalysisChunks` 正确处理空块

3. **超长 URL**
   - URL 合并可能产生极长段，影响后续字素拆分性能

### 改进建议

1. **性能优化**
   - 预编译常用正则表达式为模块级常量
   - 考虑使用 `Intl.Segmenter` 的流式接口（如果浏览器支持）

2. **可维护性**
   - 将 Unicode 范围定义提取到独立数据文件
   - 添加自动化脚本检查 Unicode 版本更新

3. **功能扩展**
   - 支持 `break-all` / `keep-all` CSS 配置
   - 支持 `strict` / `loose` 行尾规则
   - 支持垂直文本布局（vertical-rl）

4. **测试覆盖**
   - 增加东南亚语言（泰语、老挝语、高棉语）的边界测试
   - 增加 emoji ZWJ 序列的断行测试

### 相关文档

- `AGENTS.md`: 项目架构和实现笔记
- `README.md`: 公共 API 文档
- `RESEARCH.md`: 浏览器精度和性能研究
