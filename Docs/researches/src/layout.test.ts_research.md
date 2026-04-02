# src/layout.test.ts 研究文档

## 场景与职责

`layout.test.ts` 是 Pretext 库的核心测试套件，使用 Bun 测试框架验证文本布局引擎的正确性。测试设计遵循"小而持久"的原则，专注于公共 API 的确定性行为验证，而非完整的浏览器级测试覆盖。

### 核心职责
1. **API 行为验证**：验证 `prepare`/`layout` 系列函数的公共契约
2. **不变量检查**：确保布局行为在各种边界条件下保持一致
3. **回归防护**：捕获已修复问题的回归
4. **快速反馈**：使用模拟 Canvas 后端，无需真实浏览器环境

### 测试哲学
- 保持测试套件小而持久
- 浏览器特定调查使用临时探针和检查器
- 不镜像完整实现细节

---

## 功能点目的

### 1. 测试分类

#### prepare 不变量测试（`describe('prepare invariants')`）
验证文本预处理阶段的正确性：
- 空白字符处理
- pre-wrap 模式行为
- 特殊字符（NBSP、ZWSP、软连字符）
- 多语言文本分段
- URL 和数字模式识别

#### layout 不变量测试（`describe('layout invariants')`）
验证布局计算阶段的正确性：
- 行数单调性
- 尾部空白处理
- 长词断行
- 混合方向文本
- 流式布局 API 对齐

### 2. 模拟 Canvas 实现

```typescript
class TestCanvasRenderingContext2D {
  font = ''
  measureText(text: string): { width: number }
}

class TestOffscreenCanvas {
  getContext(_kind: string): TestCanvasRenderingContext2D
}
```

模拟实现使用启发式宽度计算：
- 空格: `fontSize * 0.33`
- 制表符: `fontSize * 1.32`
- Emoji/宽字符: `fontSize`
- 标点: `fontSize * 0.4`
- 其他: `fontSize * 0.6`

### 3. 测试辅助函数

#### 宽度计算
```typescript
function measureWidth(text: string, font: string): number
function parseFontSize(font: string): number
function nextTabAdvance(lineWidth: number, spaceWidth: number, tabSize?: number): number
```

#### 文本切片
```typescript
function slicePreparedText(
  prepared: TestPreparedTextWithSegments,
  start: TestLayoutCursor,
  end: TestLayoutCursor
): string
```

#### 流式布局收集
```typescript
function collectStreamedLines(
  prepared: TestPreparedTextWithSegments,
  width: number,
  start?: TestLayoutCursor
): TestLayoutLine[]

function collectStreamedLinesWithWidths(
  prepared: TestPreparedTextWithSegments,
  widths: number[]
): TestLayoutLine[]
```

#### 文本重建验证
```typescript
function reconstructFromLineBoundaries(
  prepared: TestPreparedTextWithSegments,
  lines: TestLayoutLine[]
): string

function reconstructFromWalkedRanges(
  prepared: TestPreparedTextWithSegments,
  width: number
): string
```

---

## 具体技术实现

### 测试数据结构

```typescript
type TestLayoutCursor = {
  segmentIndex: number
  graphemeIndex: number
}

type TestPreparedTextWithSegments = {
  segments: string[]
}

type TestLayoutLine = {
  text: string
  width: number
  start: TestLayoutCursor
  end: TestLayoutCursor
}
```

### 关键测试用例分析

#### 1. 空白处理测试

```typescript
test('whitespace-only input stays empty', () => {
  const prepared = prepare('  \t\n  ', FONT)
  expect(layout(prepared, 200, LINE_HEIGHT)).toEqual({ lineCount: 0, height: 0 })
})
```
验证：纯空白输入在 normal 模式下产生空布局。

#### 2. pre-wrap 模式测试

```typescript
test('pre-wrap mode keeps ordinary spaces instead of collapsing them', () => {
  const prepared = prepareWithSegments('  Hello   World  ', FONT, { whiteSpace: 'pre-wrap' })
  expect(prepared.segments).toEqual(['  ', 'Hello', '   ', 'World', '  '])
  expect(prepared.kinds).toEqual(['preserved-space', 'text', 'preserved-space', 'text', 'preserved-space'])
})
```
验证：pre-wrap 模式正确保留空格并标记段类型。

#### 3. 软连字符测试

```typescript
test('treats soft hyphens as discretionary break points', () => {
  const prepared = prepareWithSegments('trans\u00ADatlantic', FONT)
  // 宽布局：不触发断行
  const wide = layoutWithLines(prepared, 200, LINE_HEIGHT)
  expect(wide.lines.map(line => line.text)).toEqual(['transatlantic'])
  
  // 窄布局：触发软连字符断行
  const narrow = layoutWithLines(prefixed, softBreakWidth, LINE_HEIGHT)
  expect(narrow.lines.map(line => line.text)).toEqual(['foo trans-', 'atlantic'])
})
```
验证：软连字符在窄宽度下触发断行并显示连字符。

#### 4. 多语言分段测试

```typescript
test('keeps arabic punctuation attached to the preceding word', () => {
  const prepared = prepareWithSegments('مرحبا، عالم؟', FONT)
  expect(prepared.segments).toEqual(['مرحبا،', ' ', 'عالم؟'])
})
```
验证：阿拉伯语标点正确合并到前词。

#### 5. API 对齐测试

```typescript
test('layoutNextLine reproduces layoutWithLines exactly', () => {
  const prepared = prepareWithSegments('foo trans\u00ADatlantic said "hello" to 世界 and waved.', FONT)
  const expected = layoutWithLines(prepared, width, LINE_HEIGHT)
  
  const actual = []
  let cursor = { segmentIndex: 0, graphemeIndex: 0 }
  while (true) {
    const line = layoutNextLine(prepared, cursor, width)
    if (line === null) break
    actual.push(line)
    cursor = line.end
  }
  
  expect(actual).toEqual(expected.lines)
})
```
验证：流式 API `layoutNextLine` 与批处理 API `layoutWithLines` 结果一致。

#### 6. 边界光标重建测试

```typescript
test('rich line boundary cursors reconstruct normalized source text exactly', () => {
  const cases = ['a b c', '  Hello\t \n  World  ', ...]
  for (const text of cases) {
    const prepared = prepareWithSegments(text, FONT)
    const expected = prepared.segments.join('')
    for (const width of widths) {
      const batched = layoutWithLines(prepared, width, LINE_HEIGHT)
      const streamed = collectStreamedLines(prepared, width)
      expect(reconstructFromLineBoundaries(prepared, batched.lines)).toBe(expected)
      expect(reconstructFromLineBoundaries(prepared, streamed)).toBe(expected)
    }
  }
})
```
验证：通过行边界光标可以精确重建原始文本。

---

## 关键代码路径与文件引用

### 测试导入依赖

```typescript
import { beforeAll, beforeEach, describe, expect, test } from 'bun:test'
```

### 被测模块

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `layout.ts` | `prepare`, `prepareWithSegments`, `layout`, `layoutWithLines`, `layoutNextLine`, `walkLineRanges`, `clearCache`, `setLocale` | 主要 API 测试 |
| `line-break.ts` | `countPreparedLines`, `walkPreparedLines` | 行计数测试 |

### 测试生命周期

```typescript
beforeAll(async () => {
  // 注入模拟 Canvas
  Reflect.set(globalThis, 'OffscreenCanvas', TestOffscreenCanvas)
  // 动态导入被测模块（确保使用模拟环境）
  const mod = await import('./layout.ts')
  const lineBreakMod = await import('./line-break.ts')
  // 解构赋值到模块级变量
})

beforeEach(() => {
  // 每个测试前重置状态
  setLocale(undefined)
  clearCache()
})
```

### 测试组织结构

```
describe('prepare invariants')
├── whitespace-only input stays empty
├── collapses ordinary whitespace runs and trims the edges
├── pre-wrap mode keeps ordinary spaces
├── pre-wrap mode keeps hard breaks as explicit segments
├── pre-wrap mode normalizes CRLF into a single hard break
├── pre-wrap mode keeps tabs as explicit segments
├── keeps non-breaking spaces as glue
├── keeps standalone non-breaking spaces as visible glue content
├── keeps narrow no-break spaces as glue content
├── keeps word joiners as glue content
├── treats zero-width spaces as explicit break opportunities
├── treats soft hyphens as discretionary break points
├── keeps closing punctuation attached to the preceding word
├── keeps arabic punctuation attached to the preceding word
├── keeps arabic punctuation-plus-mark clusters attached
├── keeps arabic no-space punctuation clusters together
├── keeps leading arabic combining marks with the following word
├── keeps devanagari danda punctuation attached
├── keeps myanmar punctuation attached
├── keeps myanmar possessive marker attached
├── keeps opening quotes attached to the following word
├── keeps apostrophe-led elisions attached
├── keeps stacked opening quotes attached
├── treats ascii quotes as opening and closing glue by context
├── treats escaped ascii quote clusters as opening and closing glue
├── keeps URL-like runs together as one breakable segment
├── keeps no-space ascii punctuation chains together
├── keeps numeric time ranges together
├── splits hyphenated numeric identifiers at preferred boundaries
├── keeps unicode-digit numeric expressions together
├── does not attach opening punctuation to following whitespace
├── keeps japanese iteration marks attached to the preceding kana
├── carries trailing cjk opening punctuation forward across segment boundaries
├── keeps em dashes breakable
├── coalesces repeated punctuation runs into a single segment
├── applies CJK and Hangul punctuation attachment rules
├── adjacent CJK text units stay breakable after visible text
├── treats astral CJK ideographs as CJK break units
├── prepare and prepareWithSegments agree on layout behavior
└── locale can be reset without disturbing later prepares

describe('layout invariants')
├── line count grows monotonically as width shrinks
├── trailing whitespace hangs past the line edge
├── breaks long words at grapheme boundaries
├── mixed-direction text is a stable smoke test
├── layoutNextLine reproduces layoutWithLines exactly
├── mixed-script canary keeps layoutWithLines and layoutNextLine aligned
├── layout and layoutWithLines stay aligned when ZWSP triggers narrow grapheme breaking
├── layoutWithLines strips leading collapsible space after a ZWSP break
├── layoutNextLine can resume from any fixed-width line start
├── rich line boundary cursors reconstruct normalized source text exactly
├── soft-hyphen round-trip uses source slices instead of rendered line text
├── layoutNextLine variable-width streaming stays contiguous
├── layoutNextLine variable-width streaming stays contiguous in pre-wrap mode
├── pre-wrap mode keeps hanging spaces visible at line end
├── pre-wrap mode treats hard breaks as forced line boundaries
├── pre-wrap mode treats tabs as hanging whitespace aligned to tab stops
├── pre-wrap mode treats consecutive tabs as distinct tab stops
├── pre-wrap mode keeps whitespace-only middle lines visible
├── pre-wrap mode keeps trailing spaces before a hard break
├── pre-wrap mode keeps trailing tabs before a hard break
├── pre-wrap mode restarts tab stops after a hard break
├── layoutNextLine stays aligned with layoutWithLines in pre-wrap mode
├── pre-wrap mode keeps empty lines from consecutive hard breaks
├── pre-wrap mode does not invent an extra trailing empty line
├── overlong breakable segments wrap onto a fresh line
├── walkLineRanges reproduces layoutWithLines geometry
└── countPreparedLines stays aligned with the walked line counter
```

---

## 依赖与外部交互

### 外部依赖

| 依赖 | 用途 |
|------|------|
| `bun:test` | 测试框架（describe, test, expect, beforeAll, beforeEach） |
| `Intl.Segmenter` | 字素分段（用于测试辅助函数） |

### 内部辅助常量

```typescript
const FONT = '16px Test Sans'
const LINE_HEIGHT = 19
const emojiPresentationRe = /\p{Emoji_Presentation}/u
const punctuationRe = /[.,!?;:%)\]}'""'»›…—-]/u
const graphemeSegmenter = new Intl.Segmenter(undefined, { granularity: 'grapheme' })
```

---

## 风险、边界与改进建议

### 已知风险

1. **模拟 Canvas 精度限制**
   - 启发式宽度计算与真实浏览器 Canvas 可能有差异
   - 某些边界情况可能在模拟环境中通过但在真实浏览器中失败

2. **测试覆盖局限**
   - 仅覆盖主要 API 路径
   - 浏览器特定行为（如 emoji 校正）无法在单元测试中验证

3. **动态导入依赖**
   - 使用 `beforeAll` 动态导入被测模块
   - 如果模块初始化有副作用，可能影响测试结果

### 边界条件

1. **空输入处理**
   - 空白-only 输入返回空布局
   - 空段数组处理

2. **极端宽度**
   - 宽度为 0 时的行为
   - 极大宽度（单行布局）

3. **长文本性能**
   - 测试使用短文本，未覆盖长文本性能

### 改进建议

1. **测试覆盖扩展**
   - 添加更多边界条件测试（空宽度、极大宽度）
   - 添加并发测试（验证缓存线程安全）
   - 添加性能基准测试

2. **模拟精度提升**
   - 考虑使用更精确的字体度量模拟
   - 添加与真实浏览器结果的对比测试

3. **测试组织优化**
   - 将大型测试文件拆分为多个主题文件
   - 添加测试数据生成器（property-based testing）

4. **CI 集成**
   - 添加覆盖率报告
   - 添加多浏览器自动化测试

### 相关文档

- `AGENTS.md`: "Keep `src/layout.test.ts` small and durable. For browser-specific or narrow hypothesis work, prefer throwaway probes/scripts and promote only the stable invariants into permanent tests."
- `DEVELOPMENT.md`: 测试命令和开发工作流
