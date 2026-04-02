# sources.json 研究文档

## 场景与职责

`corpora/sources.json` 是 Pretext 项目的**语料库元数据源文件**，是长文本语料测试系统的**唯一事实来源 (Single Source of Truth)**。它定义了所有可用于测试的语料的基本信息，包括标识、语言、字体配置、尺寸参数等，是 `corpus-check`、`corpus-sweep`、`corpus-representative` 等工具的输入数据源。

### 核心职责

1. **语料注册中心**：集中管理所有长文本语料的元数据
2. **测试配置源**：为测试工具提供字体、行高、宽度范围等配置
3. **多语言支持**：支持 17 种语料，涵盖 13 种语言的多种文字系统
4. **数据源追踪**：记录语料的来源 URL 和获取方式

### 语料分类

| 类别 | 语料数量 | 代表语料 |
|------|----------|----------|
| 产品形态文本 | 1 | `mixed-app-text` |
| 欧洲语言 | 1 | `en-gatsby-opening` |
| CJK 文字 | 5 | `ja-rashomon`, `zh-zhufu`, `ko-unsu-joh-eun-nal` 等 |
| 东南亚文字 | 5 | `th-nithan-vetal-story-1`, `my-cunning-heron-teacher`, `km-prachum-reuang-preng-khmer` |
| 中东文字 | 5 | `ar-risalat-al-ghufran`, `he-masaot-binyamin-metudela`, `ur-chughd` 等 |

---

## 功能点目的

### 1. 语料元数据定义

```typescript
interface CorpusSource {
  id: string;                    // 唯一标识符（kebab-case）
  language: string;              // ISO 639-1 语言代码
  direction: 'ltr' | 'rtl';      // 文本方向
  title: string;                 // 语料标题（本地语言）
  source_url: string;            // 数据来源 URL
  output: string;                // 本地文本文件路径
  font_family: string;           // CSS font-family
  font_size_px: number;          // 字体大小（像素）
  line_height_px: number;        // 行高（像素）
  default_width: number;         // 默认测试宽度
  min_width: number;             // 最小测试宽度
  max_width: number;             // 最大测试宽度
  characters: number;            // 字符数
  lines: number;                 // 原始文本行数
  acquired_via: string;          // 获取方式
  notes: string;                 // 备注说明
}
```

### 2. 字体配置策略

每个语料配置适合其文字系统的字体栈：

| 语料 | 字体栈 | 设计考虑 |
|------|--------|----------|
| `mixed-app-text` | `"Helvetica Neue", Arial, "Apple SD Gothic Neo", ...` | 多脚本回退 |
| `en-gatsby-opening` | `Georgia, "Times New Roman", serif` | 英文衬线 |
| `ja-rashomon` | `"Hiragino Mincho ProN", "Yu Mincho", ...` | 日文明朝体 |
| `zh-zhufu` | `"Songti SC", "PingFang SC", ...` | 中文宋体 |
| `ur-chughd` | `"Noto Nastaliq Urdu", "DecoType Nastaleeq", ...` | 乌尔都 Nastaliq |

### 3. 尺寸标准化

```typescript
// 默认配置模式
const DEFAULTS = {
  font_size_px: 18,      // 或 20
  line_height_px: 30,    // 或 32、34、38（根据文字调整）
  default_width: 600,
  min_width: 300,
  max_width: 900,
}

// 行高调整规则：
// - 拉丁/CJK：1.6x 字体大小
// - 阿拉伯/希伯来：1.7x 字体大小
// - 乌尔都 Nastaliq：1.9x 字体大小（38px for 20px font）
```

### 4. 数据获取方式

| 方式 | 说明 | 示例 |
|------|------|------|
| `synthetic` | 人工构造的测试文本 | `mixed-app-text` |
| `copy` | 直接复制 | `en-gatsby-opening` |
| `parse` | HTML 解析提取 | `ja-rashomon`, `th-nithan-vetal-story-1` |
| `extract` | 结构化提取 | `ar-risalat-al-ghufran-part-1` |
| `raw-trim` | 原始文本修剪 | `zh-zhufu`, `my-bad-deeds-return-to-you-teacher` |
| `parse-trim` | 解析后修剪 | `zh-guxiang` |

---

## 具体技术实现

### 数据结构详解

**完整条目示例**（`ja-rashomon`）：

```json
{
  "id": "ja-rashomon",
  "language": "ja",
  "direction": "ltr",
  "title": "羅生門",
  "source_url": "https://ja.wikisource.org/wiki/%E7%BE%85%E7%94%9F%E9%96%80",
  "output": "corpora/ja-rashomon.txt",
  "font_family": "\"Hiragino Mincho ProN\", \"Yu Mincho\", \"Noto Serif CJK JP\", serif",
  "font_size_px": 20,
  "line_height_px": 32,
  "default_width": 600,
  "min_width": 300,
  "max_width": 900,
  "characters": 5701,
  "lines": 69,
  "acquired_via": "parse",
  "notes": "Japanese prose stress corpus from Wikisource, trimmed to the story body with ruby readings and page/license scaffolding removed."
}
```

### 消费接口

**1. 测试脚本加载** (`scripts/corpus-sweep.ts` 第 121-123 行)

```typescript
async function loadSources(): Promise<CorpusMeta[]> {
  return await Bun.file('corpora/sources.json').json()
}
```

**2. 浏览器页面加载** (`pages/corpus.ts` 第 14-16 行)

```typescript
import sourcesData from '../corpora/sources.json' with { type: 'json' }

async function loadSources(): Promise<CorpusMeta[]> {
  return sourcesData as CorpusMeta[]
}
```

**3. 类型定义** (`scripts/corpus-sweep.ts` 第 13-19 行)

```typescript
type CorpusMeta = {
  id: string
  language: string
  title: string
  min_width?: number
  max_width?: number
}
```

### 字体回退机制

```
┌─────────────────────────────────────────────────────────────────┐
│                     字体回退层级                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  主字体（系统特定）                                               │
│  ├── macOS: "Hiragino Mincho ProN", "Songti SC", "Thonburi"     │
│  └── 其他: "Noto Serif CJK JP", "Noto Sans Thai"                │
│                              │                                  │
│                              ▼                                  │
│  次字体（跨平台）                                                 │
│  ├── "Yu Mincho", "PingFang SC", "Noto Sans CJK KR"            │
│  └── "Geeza Pro", "Kohinoor Devanagari"                        │
│                              │                                  │
│                              ▼                                  │
│  通用回退                                                         │
│  ├── serif, sans-serif                                         │
│  └── system-ui (不推荐，macOS 上 canvas 和 DOM 可能解析不同)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 文本文件关联

```
corpora/sources.json
│
├─ 引用 ──> corpora/mixed-app-text.txt
├─ 引用 ──> corpora/en-gatsby-opening.txt
├─ 引用 ──> corpora/ja-rashomon.txt
├─ 引用 ──> corpora/ja-kumo-no-ito.txt
├─ 引用 ──> corpora/ko-unsu-joh-eun-nal.txt
├─ 引用 ──> corpora/zh-zhufu.txt
├─ 引用 ──> corpora/zh-guxiang.txt
├─ 引用 ──> corpora/th-nithan-vetal-story-1.txt
├─ 引用 ──> corpora/th-nithan-vetal-story-7.txt
├─ 引用 ──> corpora/my-cunning-heron-teacher.txt
├─ 引用 ──> corpora/my-bad-deeds-return-to-you-teacher.txt
├─ 引用 ──> corpora/km-prachum-reuang-preng-khmer-volume-7-stories-1-10.txt
├─ 引用 ──> corpora/ar-risalat-al-ghufran-part-1.txt
├─ 引用 ──> corpora/ar-al-bukhala.txt
├─ 引用 ──> corpora/hi-eidgah.txt
├─ 引用 ──> corpora/he-masaot-binyamin-metudela.txt
└─ 引用 ──> corpora/ur-chughd.txt
```

---

## 关键代码路径与文件引用

### 读取路径

| 文件 | 职责 | 读取方式 |
|------|------|----------|
| `scripts/corpus-sweep.ts` | 宽度扫描测试 | `Bun.file('corpora/sources.json').json()` |
| `scripts/corpus-check.ts` | 单语料诊断 | `Bun.file('corpora/sources.json').json()` |
| `scripts/corpus-representative.ts` | 锚点测试 | 通过 `pages/corpus.ts` 间接使用 |
| `pages/corpus.ts` | 浏览器测试页面 | `import sourcesData from '../corpora/sources.json'` |

### 关联文件

| 文件 | 关系 | 说明 |
|------|------|------|
| `corpora/*.txt` | 被引用 | 17 个语料文本文件 |
| `corpora/TAXONOMY.md` | 相关 | 语料分类法文档 |
| `corpora/dashboard.json` | 派生 | 聚合状态报告 |

### 命令行调用

```bash
# 使用 sources.json 的测试命令
bun run corpus-check --id=ja-rashomon          # 单语料检查
bun run corpus-sweep --all                     # 全量扫描
bun run corpus-sweep --id=zh-zhufu --step=10   # 单语料密集扫描
bun run corpus-representative                  # 锚点测试
```

---

## 依赖与外部交互

### 外部数据源

| 语料 | 来源 | URL 模式 |
|------|------|----------|
| 日文 | Wikisource | `https://ja.wikisource.org/wiki/...` |
| 中文 | Wikisource | `https://zh.wikisource.org/wiki/...` |
| 韩文 | Wikisource | `https://ko.wikisource.org/wiki/...` |
| 泰文 | Wikisource | `https://th.wikisource.org/wiki/...` |
| 缅甸文 | Wikisource | `https://my.wikisource.org/wiki/...` |
| 高棉文 | Wikisource | `https://wikisource.org/wiki/...` |
| 阿拉伯文 | Wikisource | `https://ar.wikisource.org/wiki/...` |
| 印地文 | Wikisource | `https://hi.wikisource.org/wiki/...` |
| 希伯来文 | Wikisource | `https://he.wikisource.org/wiki/...` |
| 乌尔都文 | Wikisource | `https://wikisource.org/wiki/...` |
| 英文 | Gutenberg | `https://www.gutenberg.org/ebooks/...` |

### 系统依赖

| 依赖 | 用途 |
|------|------|
| 系统字体 | 测试时实际渲染使用的字体 |
| Chrome/Safari | 浏览器测试 |
| Bun | JSON 解析和脚本执行 |

### 数据一致性

```
┌─────────────────────────────────────────────────────────────────┐
│                    sources.json 一致性要求                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. ID 唯一性                                                    │
│     └── 17 个语料的 id 必须唯一                                  │
│                                                                 │
│  2. 文件存在性                                                   │
│     └── output 字段引用的 .txt 文件必须存在                      │
│                                                                 │
│  3. 语言代码有效性                                               │
│     └── language 字段应为有效 ISO 639-1 代码                     │
│                                                                 │
│  4. 宽度范围有效性                                               │
│     └── min_width < default_width < max_width                    │
│                                                                 │
│  5. 字体栈有效性                                                 │
│     └── font_family 应为有效 CSS font-family 值                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 风险、边界与改进建议

### 已知风险

#### 1. 字体可用性差异

**问题**：配置的字体在不同操作系统上可用性不同。

**示例**：
- `"Hiragino Mincho ProN"`：仅 macOS
- `"Songti SC"`：macOS 有，Windows 可能需安装
- `"Noto Nastaliq Urdu"`：需手动安装

**影响**：测试结果可能因系统字体差异而不同。

**缓解**：字体栈设计包含多层回退。

#### 2. 源链接失效

**问题**：Wikisource 页面可能变更或删除。

**风险语料**：
- 依赖特定 Wikisource 页面结构的语料
- 使用 `parse` 或 `extract` 获取方式的语料

**缓解**：
- 已下载的 `.txt` 文件作为本地备份
- `notes` 字段记录获取方式便于重新获取

#### 3. 字符数统计准确性

**问题**：`characters` 字段为人工统计，可能与实际有偏差。

**影响**：
- 不准确的数据分析
- 性能测试基准偏差

### 边界条件

| 边界 | 当前值 | 说明 |
|------|--------|------|
| 语料数量 | 17 | 固定集合 |
| 宽度范围 | 300-900px | 所有语料统一 |
| 字体大小 | 18px 或 20px | 二选一 |
| 文本来源 | 公有领域 | 均为古腾堡或维基文库 |
| 更新频率 | 手动 | 无自动同步机制 |

### 改进建议

#### 1. 字体可用性检测

```typescript
// 建议：添加字体可用性检查脚本
async function checkFontAvailability(): Promise<Record<string, string[]>> {
  const sources = await loadJson<CorpusSource[]>('corpora/sources.json')
  const results: Record<string, string[]> = {}
  
  for (const source of sources) {
    const fonts = source.font_family.split(',').map(f => f.trim().replace(/["']/g, ''))
    const available = fonts.filter(font => document.fonts.check(`12px ${font}`))
    results[source.id] = available
  }
  
  return results
}
```

#### 2. 自动化源同步

```bash
# 建议：添加同步脚本
bun run corpus-sync --id=ja-rashomon
# 1. 读取 source_url
# 2. 下载/解析页面
# 3. 更新 .txt 文件
# 4. 更新 characters/lines 统计
```

#### 3. 动态字符统计

```typescript
// 建议：自动计算字符数
function updateCorpusStats(source: CorpusSource): CorpusSource {
  const text = readFileSync(source.output, 'utf8')
  return {
    ...source,
    characters: [...text].length,  // 正确处理 Unicode
    lines: text.split('\n').length,
  }
}
```

#### 4. 扩展宽度范围

```json
// 建议：支持响应式宽度测试
{
  "id": "mixed-app-text",
  "width_ranges": [
    { "name": "mobile", "min": 320, "max": 428 },
    { "name": "tablet", "min": 768, "max": 834 },
    { "name": "desktop", "min": 1024, "max": 1920 }
  ]
}
```

#### 5. 字体矩阵扩展

```json
// 建议：定义多字体测试矩阵
{
  "id": "zh-zhufu",
  "font_matrix": [
    "\"Songti SC\", serif",
    "\"PingFang SC\", sans-serif",
    "\"Noto Serif CJK SC\", serif"
  ]
}
```

#### 6. 版本控制增强

```json
// 建议：添加版本信息
{
  "version": "2024.03.31",
  "last_updated": "2026-03-31T10:07:49.046Z",
  "corpora": [...]
}
```

### 维护 checklist

- [ ] 添加新语料时确保 `id` 唯一且符合命名规范
- [ ] 验证 `output` 字段的 `.txt` 文件存在
- [ ] 检查 `source_url` 可访问
- [ ] 更新 `characters` 和 `lines` 统计
- [ ] 在目标系统上验证字体可用性
- [ ] 更新 `corpora/TAXONOMY.md` 分类文档
- [ ] 重新生成依赖的仪表盘文件
- [ ] 更新 `AGENTS.md` 和 `DEVELOPMENT.md` 中的相关引用
