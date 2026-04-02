# src/bidi.ts 研究文档

## 场景与职责

`bidi.ts` 是 Pretext 库的双向文本（Bidirectional Text, Bidi）处理模块，专门用于为富文本渲染路径提供双向文本的元数据支持。该模块实现了 Unicode 双向算法（UBA, Unicode Bidirectional Algorithm）的简化版本，用于计算文本中每个字符的嵌入级别（embedding levels）。

### 核心职责
1. **字符双向类型分类**：将字符分类为 L（左到右）、R（右到左）、AL（阿拉伯字母）、AN（阿拉伯数字）等双向类型
2. **嵌入级别计算**：根据双向算法规则计算每个字符的嵌入级别
3. **段级别映射**：将字符级别的嵌入级别映射到分析段（segment）级别，供自定义渲染器使用

### 重要限制
- **非布局使用**：行断点引擎不消费这些级别，仅用于自定义渲染的元数据
- **简化实现**：基于 pdf.js 的实现 fork，专注于常见场景，非完整 UBA 实现

---

## 功能点目的

### 1. 双向字符类型系统

```typescript
type BidiType = 'L' | 'R' | 'AL' | 'AN' | 'EN' | 'ES' | 'ET' | 'CS' |
                'ON' | 'BN' | 'B' | 'S' | 'WS' | 'NSM'
```

类型说明：
- `L`: 强左到右字符（拉丁字母、数字等）
- `R`: 强右到左字符（希伯来字母）
- `AL`: 阿拉伯字母（Arabic Letter）
- `AN`: 阿拉伯数字（Arabic-Indic digits）
- `EN`: 欧洲数字（European digits）
- `ES`: 欧洲数字分隔符（+ -）
- `ET`: 欧洲数字终止符（货币符号等）
- `CS`: 通用数字分隔符（. , / :）
- `ON`: 其他中性字符（标点、符号）
- `BN`: 边界中性（控制字符、格式字符）
- `B`: 块分隔符（段落分隔符）
- `S`: 段分隔符（Tab）
- `WS`: 空白字符
- `NSM`: 非间距标记（组合字符）

### 2. 字符分类表

#### 基础 ASCII 表（0x00-0xFF）
- 硬编码 256 个字符的默认类型
- 覆盖 ASCII 字母、数字、标点、控制字符

#### 阿拉伯语表（0x600-0x6FF）
- 阿拉伯字母块的字符类型
- 区分 AL、AN、NSM、ON 等类型

### 3. 嵌入级别计算

#### 基础级别确定
```typescript
const startLevel = (len / numBidi) < 0.3 ? 0 : 1
```
- 如果双向字符占比 < 30%，基础级别为 0（LTR 上下文）
- 否则基础级别为 1（RTL 上下文）

#### 算法阶段

**W1-W7: 字符类型解析**
- W1: NSM 继承前驱类型
- W2: EN 在 AL 后变为 AN
- W3: AL 变为 R
- W4: ES/CS 在数字间变为 EN/AN
- W5: ET 序列邻接 EN 时变为 EN
- W6: 剩余 ES/ET/CS 变为 ON
- W7: EN 在 L 后变为 L

**N1-N2: 中性字符解析**
- N1: 中性序列在强类型间继承方向
- N2: 剩余中性字符继承嵌入方向

**I1-I2: 隐式级别解析**
- 根据最终字符类型调整嵌入级别

---

## 具体技术实现

### 核心数据结构

```typescript
// 双向类型定义
type BidiType = 'L' | 'R' | 'AL' | 'AN' | 'EN' | 'ES' | 'ET' | 'CS' |
                'ON' | 'BN' | 'B' | 'S' | 'WS' | 'NSM'

// 基础 ASCII 类型表（256 项）
const baseTypes: BidiType[] = [...]

// 阿拉伯语类型表（256 项，对应 U+0600-U+06FF）
const arabicTypes: BidiType[] = [...]
```

### 关键算法流程

#### 1. 字符分类 `classifyChar()`

```typescript
function classifyChar(charCode: number): BidiType {
  if (charCode <= 0x00ff) return baseTypes[charCode]!
  if (0x0590 <= charCode && charCode <= 0x05f4) return 'R'  // 希伯来语
  if (0x0600 <= charCode && charCode <= 0x06ff) return arabicTypes[charCode & 0xff]!
  if (0x0700 <= charCode && charCode <= 0x08AC) return 'AL' // 阿拉伯语扩展
  return 'L' // 默认左到右
}
```

#### 2. 级别计算 `computeBidiLevels()`

```
输入: str (规范化文本)
输出: Int8Array | null (每个字符的嵌入级别，null 表示无双向字符)

1. 初始化类型数组 types[]
2. 遍历字符分类，统计双向字符数 numBidi
3. 如果 numBidi === 0，返回 null（优化：无双向字符）
4. 确定基础级别 startLevel
5. 初始化级别数组 levels[]
6. 应用 W1-W7 规则解析类型
7. 应用 N1-N2 规则解析中性字符
8. 应用 I1-I2 规则计算最终级别
9. 返回级别数组
```

#### 3. 段级别映射 `computeSegmentLevels()`

```typescript
export function computeSegmentLevels(
  normalized: string, 
  segStarts: number[]
): Int8Array | null
```

- 计算完整文本的双向级别
- 提取每个段起始位置对应的级别
- 返回段级别数组（长度等于段数）

### 算法复杂度

- **时间复杂度**: O(n)，n 为文本长度
- **空间复杂度**: O(n)，类型数组和级别数组
- **提前退出**: 无双向字符时立即返回 null

---

## 关键代码路径与文件引用

### 主要导出函数

| 函数 | 用途 | 调用方 |
|------|------|--------|
| `computeSegmentLevels()` | 计算段级别 | `layout.ts` -> `measureAnalysis()` |

### 内部调用关系

```
computeSegmentLevels
└── computeBidiLevels
    ├── classifyChar (每个字符)
    ├── W1-W7 类型解析循环
    ├── N1-N2 中性解析循环
    └── I1-I2 级别计算循环
```

### 代码位置

| 功能 | 行号范围 |
|------|----------|
| 类型定义 | 7-8 |
| 基础类型表 | 10-30 |
| 阿拉伯类型表 | 32-55 |
| 字符分类 | 57-63 |
| 级别计算 | 65-162 |
| 段级别映射 | 164-173 |

---

## 依赖与外部交互

### 外部依赖

无外部依赖，纯 TypeScript 实现。

### 被依赖方

| 文件 | 导入内容 |
|------|----------|
| `layout.ts` | `computeSegmentLevels` |

### 数据流

```
layout.ts prepareInternal()
├── analyzeText() -> TextAnalysis (normalized, starts)
├── measureAnalysis()
│   ├── ... 测量逻辑 ...
│   └── computeSegmentLevels(analysis.normalized, segStarts)
│       └── 返回 Int8Array | null
└── 存储到 prepared.segLevels
```

---

## 风险、边界与改进建议

### 已知风险

1. **简化实现限制**
   - 非完整 UBA 实现，可能无法处理极端复杂的双向场景
   - 不支持显式方向格式化字符（LRE, RLE, LRO, RLO, PDF, LRI, RLI, FSI, PDI）

2. **字符范围限制**
   - 仅覆盖 ASCII、希伯来语 (U+0590-U+05F4)、阿拉伯语 (U+0600-U+08AC)
   - 其他 RTL 脚本（如叙利亚语、塔安娜语）被简单归类为 AL，可能不准确

3. **性能考虑**
   - 每次调用 `computeSegmentLevels` 都重新计算完整文本的级别
   - 对于长文本，O(n) 遍历可能成为性能瓶颈

### 边界条件

1. **空文本**
   - 空字符串返回 null

2. **无双向字符**
   - 纯 LTR 文本（无双向字符）返回 null，作为优化标志

3. **单段文本**
   - 返回长度为 1 的 Int8Array

4. **混合脚本**
   - LTR 文本中嵌入 RTL 片段正确处理
   - RTL 文本中嵌入 LTR 片段依赖基础级别判断

### 改进建议

1. **功能完整性**
   - 实现完整的 UBA 规则（X1-X10 显式方向格式化）
   - 支持更多的 RTL 脚本（叙利亚语、塔安娜语、尼科语等）

2. **性能优化**
   - 缓存已计算的双向级别结果
   - 对于不变文本，避免重复计算
   - 考虑使用 Web Worker 进行异步计算

3. **API 改进**
   - 暴露更多元数据（字符类型、运行边界等）
   - 支持自定义基础级别（强制 RTL/LTR 上下文）

4. **测试覆盖**
   - 增加 UBA 官方测试用例验证
   - 增加复杂嵌套双向文本测试
   - 增加性能基准测试

### 相关参考

- [Unicode Standard Annex #9: Unicode Bidirectional Algorithm](https://unicode.org/reports/tr9/)
- [pdf.js bidi.js](https://github.com/mozilla/pdf.js/blob/master/src/core/bidi.js) - 原始实现来源
- [Sebastian Markbage's text-layout](https://github.com/chenglou/text-layout) - 研究基础

### 注意事项

根据 `AGENTS.md` 的说明：
> Bidi levels now stay on the rich `prepareWithSegments()` path as custom-rendering metadata only. The opaque fast `prepare()` handle should not pay for bidi metadata that `layout()` does not consume, and line breaking itself does not read those levels.

这意味着：
- 双向级别仅在 `prepareWithSegments()` 路径计算
- `prepare()` 快速路径不计算双向级别
- 行断点逻辑完全不依赖双向级别
