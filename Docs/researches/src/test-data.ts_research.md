# src/test-data.ts 研究文档

## 场景与职责

`test-data.ts` 是 Pretext 库的共享测试数据模块，为浏览器精度测试页面（`pages/accuracy.ts`）和基准测试页面（`pages/benchmark.ts`）提供标准化的测试文本、字体大小和容器宽度配置。

**核心职责：**
1. 提供覆盖多语言、多脚本的测试文本集合
2. 定义标准化的字体大小和容器宽度参数
3. 确保精度测试和基准测试使用一致的数据集
4. 作为浏览器自动化测试的输入数据源

**在架构中的位置：**
- 被 `pages/accuracy.ts` 导入用于浏览器精度验证
- 被 `pages/benchmark.ts` 导入用于性能测试
- 是测试基础设施的一部分，非运行时库代码

---

## 功能点目的

### 1. 多语言测试文本（`TEXTS`）

提供 34 组测试文本，覆盖：

| 类别 | 数量 | 语言/脚本 | 测试重点 |
|------|------|-----------|----------|
| Latin | 6 | 英语 | 基础布局、标点、连字符 |
| Arabic | 2 | 阿拉伯语 | RTL 方向、连字 |
| Hebrew | 2 | 希伯来语 | RTL 方向 |
| Mixed LTR+RTL | 4 | 英语+阿拉伯语/希伯来语 | 双向文本 |
| CJK | 5 | 中文、日文、韩文 | 字符级断行、禁则 |
| Thai | 1 | 泰语 | 词边界分割 |
| Emoji | 2 | - | 宽度校正、渲染 |
| Multi-script | 2 | 多脚本混合 | 复杂场景 |
| Edge cases | 7 | - | 空文本、空格、换行、长单词 |

### 2. 字体大小参数（`SIZES`）

```typescript
export const SIZES = [12, 14, 15, 16, 18, 20, 24, 28] as const
```

覆盖常见正文字号：
- 小字：12-14px（注释、脚注）
- 正文：15-16px（标准正文）
- 大字：18-20px（标题、强调）
- 超大：24-28px（大标题）

### 3. 容器宽度参数（`WIDTHS`）

```typescript
export const WIDTHS = [150, 200, 250, 300, 350, 400, 500, 600] as const
```

覆盖常见布局宽度：
- 窄栏：150-200px（侧边栏、移动端）
- 中栏：250-350px（内容区、卡片）
- 宽栏：400-600px（文章主体、桌面端）

---

## 具体技术实现

### 数据结构

```typescript
// 测试文本项
{
  label: string    // 唯一标识，如 'Latin update', 'Arabic', 'Emoji mixed'
  text: string     // 测试文本内容
}

// 所有数组使用 const 断言，确保类型安全
export const TEXTS = [...] as const
export const SIZES = [...] as const
export const WIDTHS = [...] as const
```

### 测试文本详解

#### Latin 文本（6 条）

```typescript
{ label: 'Latin update', text: "Just tried the new update and it's so much better..." }
{ label: 'Latin compatibility', text: "Does anyone know if this works with the latest version?..." }
{ label: 'Latin short', text: "This is exactly what I was looking for..." }
{ label: 'Latin caching', text: "The key insight is that you can cache word measurements..." }
{ label: 'Latin punctuation', text: "Performance is critical for this kind of library..." }
{ label: 'Latin hyphenation', text: "One thing I noticed is that the line breaking algorithm..." }
```

特点：
- 不同长度（短句到长句）
- 包含撇号、问号、逗号等标点
- 包含技术术语和常见表达

#### RTL 文本（4 条）

```typescript
// Arabic
{ label: 'Arabic', text: "هذا النص باللغة العربية لاختبار دعم الاتجاه من اليمين إلى اليسار..." }
{ label: 'Arabic short', text: "مرحبا بالعالم، هذه تجربة لقياس النص العربي..." }

// Hebrew
{ label: 'Hebrew', text: "זהו טקסט בעברית כדי לבדוק תמיכה בכיוון מימין לשמאל..." }
{ label: 'Hebrew short', text: "שלום עולם, זוהי בדיקה למדידת טקסט עברי..." }
```

测试重点：
- RTL 方向处理
- 阿拉伯语连字
- 双向文本边界

#### 混合方向文本（4 条）

```typescript
{ label: 'Mixed en+ar', text: "The meeting is scheduled for يوم الثلاثاء at the main office..." }
{ label: 'Mixed report', text: "According to the report by محمد الأحمد, the results show..." }
{ label: 'Mixed en+he', text: "The project name is פרויקט חדש and it was started..." }
{ label: 'Mixed version', text: "Version 3.2.1 של התוכנה was released on January 15th..." }
```

测试重点：
- LTR 和 RTL 文本混合
- 数字在 RTL 文本中的处理
- 方向隔离边界

#### CJK 文本（5 条）

```typescript
// Chinese
{ label: 'Chinese', text: "这是一段中文文本，用于测试文本布局库对中日韩字符的支持..." }
{ label: 'Chinese short', text: "性能测试显示，新的文本测量方法比传统方法快了将近一千五百倍。" }

// Japanese
{ label: 'Japanese', text: "これはテキストレイアウトライブラリのテストです..." }
{ label: 'Japanese short', text: "パフォーマンスは非常に重要です..." }

// Korean
{ label: 'Korean', text: "이것은 텍스트 레이아웃 라이브러리의 테스트입니다..." }
```

测试重点：
- 字符级断行（无空格语言）
- CJK 禁则处理（标点不能出现在行首/尾）
- 谚文音节组合

#### Thai（1 条）

```typescript
{ label: 'Thai', text: "นี่คือข้อความทดสอบสำหรับไลบรารีจัดวางข้อความ ทดสอบการตัดคำภาษาไทย" }
```

测试重点：
- 泰语词边界分割（无显式空格）
- `Intl.Segmenter` 的泰语支持

#### Emoji 文本（2 条）

```typescript
{ label: 'Emoji mixed', text: "The quick 🦊 jumped over the lazy 🐕 and then went home 🏠..." }
{ label: 'Emoji dense', text: "Great work! 👏👏👏 This is exactly what we needed 🎯..." }
```

测试重点：
- Emoji 宽度校正
- ZWJ 序列（如 👏👏👏 可能是单个簇）
- Emoji 与文本混合布局

#### 多脚本混合（2 条）

```typescript
{ label: 'Multi-script', text: "Hello مرحبا שלום 你好 こんにちは 안녕하세요 สวัสดี — a greeting in seven scripts!" }
{ label: 'Numbers+RTL', text: "The price is $42.99 (approximately ٤٢٫٩٩ ريال or ₪158.50) including tax." }
```

测试重点：
- 七脚本混合（Latin, Arabic, Hebrew, CJK, Korean, Thai）
- 阿拉伯数字和印度数字
- 货币符号

#### 边界情况（7 条）

```typescript
{ label: 'Empty', text: "" }                                    // 空文本
{ label: 'Single char', text: "A" }                             // 单字符
{ label: 'Whitespace', text: "   " }                            // 纯空格
{ label: 'Newlines', text: "Hello\nWorld\nMultiple\nLines" }     // 硬换行
{ label: 'Long word', text: "Superlongwordwithoutanyspaces..." } // 长单词
{ label: 'Long mixed', text: "In the heart of القاهرة القديمة..." } // 长混合文本
```

测试重点：
- 空输入处理
- 空白折叠
- pre-wrap 模式硬换行
- 超长单词的字素断行
- 混合脚本长文本

---

## 依赖与外部交互

### 导出

```typescript
export const TEXTS  // 测试文本数组
export const SIZES  // 字体大小数组
export const WIDTHS // 容器宽度数组
```

### 被导入方

| 导入方 | 用途 |
|--------|------|
| `pages/accuracy.ts` | 浏览器精度测试的输入数据 |
| `pages/benchmark.ts` | 性能基准测试的输入数据 |

### 使用示例

```typescript
import { TEXTS, SIZES, WIDTHS } from '../src/test-data.ts'

// 精度测试：遍历所有组合
for (const { label, text } of TEXTS) {
  for (const fontSize of SIZES) {
    for (const width of WIDTHS) {
      // 测试该组合
    }
  }
}
```

总组合数：34 × 8 × 8 = 2,176 种测试配置

---

## 风险、边界与改进建议

### 已知风险

1. **数据集规模**
   - 2,176 种组合在完整精度测试中耗时较长
   - 实际运行中可能需要采样或并行化

2. **语料代表性**
   - 泰语只有 1 条测试，覆盖可能不足
   - 缺少南亚其他脚本（印地语、孟加拉语等）
   - 缺少蒙古语、藏语等垂直布局脚本

3. **字体依赖**
   - 测试数据未指定字体，依赖系统字体
   - 不同系统上相同文本可能表现不同

4. **静态数据**
   - 文本内容固定，可能无法覆盖新发现的边界情况
   - 需要定期更新以反映实际使用场景

### 边界情况

| 场景 | 当前覆盖 |
|------|----------|
| 空文本 | ✅ Empty |
| 单字符 | ✅ Single char |
| 纯空格 | ✅ Whitespace |
| 硬换行 | ✅ Newlines |
| 超长单词 | ✅ Long word |
| 零宽字符 | 部分（通过其他文本） |
| 软连字符 | 部分（Latin hyphenation） |
| 组合字符 | 部分（Arabic, Hebrew） |

### 改进建议

1. **扩展语料覆盖**
   - 增加更多泰语测试
   - 添加印地语（Devanagari）、孟加拉语、泰米尔语
   - 考虑 emoji ZWJ 序列专项测试

2. **动态生成测试**
   - 基于语料库随机采样生成长文本
   - 组合不同特征生成边界情况

3. **字体标准化**
   - 为每个脚本指定推荐测试字体
   - 提供字体加载检查机制

4. **分层测试策略**
   - 快速模式：采样关键组合
   - 完整模式：所有组合
   - 诊断模式：针对失败组合的扩展测试

5. **与实际语料对齐**
   - 从 `corpora/` 目录的实际语料中提取代表性片段
   - 保持测试数据与语料库的一致性

### 与 corpora/ 的关系

`test-data.ts` 与 `corpora/` 目录的关系：

| | test-data.ts | corpora/ |
|--|--------------|----------|
| 目的 | 快速浏览器测试 | 深度语料验证 |
| 规模 | 34 条短文本 | 多本书籍级长文本 |
| 覆盖 | 多脚本采样 | 单语言深度覆盖 |
| 使用场景 | 开发时快速验证 | 发布前全面检查 |
| 自动化 | 是（accuracy-check） | 是（corpus-sweep） |

建议保持两者互补：`test-data.ts` 用于快速反馈，`corpora/` 用于深度验证。
