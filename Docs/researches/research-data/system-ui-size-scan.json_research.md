# Research: `research-data/system-ui-size-scan.json`

## 场景与职责

`research-data/system-ui-size-scan.json` 是 Pretext 项目检入仓库的**机器可读研究快照**，用于记录并固化一个特定平台（macOS）上的字体解析不一致问题：当使用 CSS 泛字体关键字 `system-ui` 时，浏览器 Canvas 2D 上下文与 DOM 在特定字号下会解析出不同的光学变体（optical variants），导致 Pretext 基于 Canvas 的文本测量与真实 DOM 排版产生系统性偏差。

该文件的核心职责包括：
- **证据固化**：将一次手动/探针式的字号扫描结果（10px–28px）以结构化 JSON 保存，供文档、代码注释和决策引用。
- **风险标定**：为 `README.md`、`AGENTS.md`、`src/layout.ts` 等处的“`system-ui` 不安全”声明提供数据支撑。
- **决策锚点**：在 `TODO.md` 和 `RESEARCH.md` 中作为是否值得投入“prepare-time DOM fallback”或“lookup table”等修复方案的事实依据。

## 功能点目的

### 1. 记录 Canvas-DOM 字体解析差异的“坏区间”

在 macOS 上，`system-ui` 通常会回落到 Apple 的 SF Pro 字体族。Apple 为不同字号提供了两个光学变体：
- **SF Pro Text**：用于较小字号（通常 ≤ 特定阈值）
- **SF Pro Display**：用于较大字号

Canvas `2d` 的 `measureText()` 与 DOM CSS 引擎在决定“当前字号该用哪个光学变体”时，**阈值并不完全一致**。这导致 Pretext 的 `prepare()` 阶段测得的字宽与 DOM 实际渲染字宽不同，最终 `layout()` 预测的行高/换行位置出现错误。

JSON 中的 `match: false` 即表示该字号（或字号区间）存在可观测的准确性失配；`mismatchPct` 则量化了失配测试用例的百分比。

### 2. 支撑工程决策：拒绝修复 vs. 条件修复

`RESEARCH.md` 明确列出了基于该扫描数据的结论：
- **可信路径**：若未来要支持 `system-ui`，唯一可信的方案是“narrow prepare-time DOM fallback for detected bad tuples”（在检测到特定坏组合时，回退到 DOM 测量）。
- **不可信路径**：查表（lookup tables）、朴素缩放（naive scaling）、猜测已解析字体并替换（guessed resolved-font substitution）都被认为不够可靠。

该 JSON 因此成为评估“修复 ROI”的基准：只有强真实世界需求才值得为这几个离散字号引入 DOM fallback。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 数据结构

```json
{
  "font": "system-ui",
  "platform": "macOS",
  "results": [
    { "startSizePx": 10, "endSizePx": 10, "match": false, "mismatchPct": 2.9 },
    { "startSizePx": 11, "endSizePx": 11, "match": false, "mismatchPct": 6.9 },
    { "startSizePx": 12, "endSizePx": 12, "match": false, "mismatchPct": 11.3 },
    { "startSizePx": 13, "endSizePx": 13, "match": true },
    { "startSizePx": 14, "endSizePx": 14, "match": false, "mismatchPct": 14.5 },
    { "startSizePx": 15, "endSizePx": 25, "match": true },
    { "startSizePx": 26, "endSizePx": 26, "match": false, "mismatchPct": 12.4 },
    { "startSizePx": 27, "endSizePx": 28, "match": true }
  ]
}
```

字段语义：
- `font` / `platform`：问题发生的字体关键字与操作系统。
- `results`：按字号升序排列的扫描区间。
  - `startSizePx` / `endSizePx`：闭区间字号范围（单位 px）。
  - `match`: `true` 表示在该区间内 Canvas 与 DOM 测量一致（或差异低于可观测阈值）；`false` 表示不一致。
  - `mismatchPct`：仅在 `match: false` 时出现，表示在该字号下 accuracy sweep 中失配用例占总测试用例的百分比。

### 生成方式（推断）

仓库中**没有常驻脚本**直接生成此 JSON。根据 `git log`（commit `d178a844`），它是在“Replace markdown status tables with JSON dashboards”这一变更中首次被检入的。结合项目工作流推断，其生成路径大致如下：

1. **探针/手动扫描**：作者 likely 运行了一次基于 `pages/accuracy.ts` + `scripts/accuracy-check.ts` 的自定义 sweep，但将 `FONTS` 临时替换为 `system-ui`，并将 `SIZES` 扩展/细化为 10px–28px。
2. **聚合结果**：对每个字号，统计 `mismatchCount / total * 100` 得到 `mismatchPct`；若差异为 0（或低于 1px 高度容差），则标记 `match: true`。
3. **区间合并**：将连续且状态相同的字号合并为区间（如 15–25 均为 `match: true`，27–28 均为 `match: true`）。

### 与主测量管道的关联

Pretext 的核心测量链路如下，其中 `system-ui` 问题发生在第 2 步：

1. **`src/layout.ts` → `prepare()` / `prepareWithSegments()`**
   - 调用 `src/analysis.ts` 对文本进行分段（word / grapheme / punctuation / space / tab / break 等）。
   - 调用 `src/measurement.ts` → `getSegmentMetrics()` 对每个 segment 进行字宽测量。

2. **`src/measurement.ts` → `getMeasureContext()`**
   - 使用 `OffscreenCanvas(1,1).getContext('2d')` 或 DOM `<canvas>` 获取 2D 上下文。
   - 设置 `ctx.font = font`（如 `"12px system-ui"`），然后 `ctx.measureText(seg).width`。
   - **问题点**：此时 Canvas 解析出的实际字体可能与 DOM 不同。

3. **`src/layout.ts` → `layout()` / `layoutWithLines()`**
   - 基于缓存的 segment 宽度做纯算术换行。
   - 若第 2 步测得的宽度已偏离 DOM，则预测出的 `height` 和 `lines` 都会出错。

4. **`pages/accuracy.ts` → 校验逻辑**
   - 创建隐藏 DOM `<div>`，设置同样的 `font`、`width`、`wordWrap: 'break-word'`。
   - 比较 `div.getBoundingClientRect().height`（actual）与 `layout(...).height`（predicted）。
   - 当 `system-ui` 在特定字号下解析不一致时，就会出现 `|actual - predicted| >= 1px`，被记录为 mismatch。

## 关键代码路径与文件引用

### 直接引用该 JSON 的文件

- **`RESEARCH.md`**（第 55–76 行）
  - 章节 “Discovery: system-ui font resolution mismatch” 直接链接到该 JSON，并解释其数据含义（SF Pro Text vs SF Pro Display 阈值差异）。

### 受该 JSON/结论影响的代码与文档

- **`src/layout.ts`**（第 28–31 行注释）
  - 在模块头注释的 “Limitations” 中明确警告：
    > `- system-ui font: canvas resolves to different optical variants than DOM on macOS. Use named fonts (Helvetica, Inter, etc.) for guaranteed accuracy. See RESEARCH.md "Discovery: system-ui font resolution mismatch".`

- **`README.md`**（第 138 行）
  - Caveats 章节：
    > `- system-ui is unsafe for layout() accuracy on macOS. Use a named font.`

- **`AGENTS.md`**（第 64 行）
  - 内部注意事项：
    > `- system-ui is unsafe for accuracy; canvas and DOM can resolve different fonts on macOS.`

- **`corpora/TAXONOMY.md`**（第 129–145 行）
  - 在 `font-mismatch` 分类下，将该问题列为典型示例：
    > `Examples: - historical system-ui mismatch`

- **`TODO.md`**（第 41 行）
  - 待决策问题：
    > `- Whether strong real-world demand for system-ui would justify a narrow prepare-time DOM fallback.`

### 相关基础设施（用于理解 mismatch 的检测机制）

- **`pages/accuracy.ts`**
  - 浏览器端 accuracy sweep 页面。定义了 `FONTS`、`SIZES`、`WIDTHS`、`TEXTS`，并执行 `runSweep()` 比较 DOM 高度与 `layout()` 预测高度。
  - 注意：该页面的**默认测试字体栈**并不包含 `system-ui`（使用的是 Helvetica Neue、Georgia、Verdana、Courier New），因此该 JSON 并非由常驻 CI 直接生成，而是来自一次特设的扫描。

- **`scripts/accuracy-check.ts`**
  - Node 端自动化脚本，负责启动临时 page server、打开浏览器、读取 `pages/accuracy.ts` 产出的报告，并将结果写入 `accuracy/<browser>.json`。
  - 该脚本具备完整的 mismatch 统计能力（`total`、`matchCount`、`mismatchCount`、`mismatchPct` 的计算逻辑可在此复现）。

- **`src/measurement.ts`**
  - 核心测量模块。`getMeasureContext()` 和 `getSegmentMetrics()` 是 Canvas 测量的入口。`getEmojiCorrection()` 展示了项目如何处理“Canvas 与 DOM 不一致”的同类问题（通过一次 DOM 校准读取计算修正值），但 `system-ui` 的泛字体不一致目前**没有对应的运行时修正**。

- **`src/test-data.ts`**
  - 定义了 accuracy sweep 使用的标准语料（拉丁、阿拉伯、希伯来、CJK、泰语、表情、混合方向等），`system-ui-size-scan.json` 中的 `mismatchPct` 即基于这些语料在特定字号下的失配比例。

## 依赖与外部交互

### 内部依赖

- **Pretext 测量核心**：`src/measurement.ts` → `getMeasureContext()` 提供的 Canvas 2D 上下文。
- **Pretext 布局核心**：`src/layout.ts` → `prepare()` / `layout()` 的预测能力。
- **Accuracy 校验页面**：`pages/accuracy.ts` 提供的 DOM-vs-predicted 比较框架。
- **自动化脚本层**：`scripts/accuracy-check.ts` / `scripts/browser-automation.ts` 提供的浏览器驱动与报告收集能力。

### 外部依赖

- **macOS 系统字体解析行为**：
  - 该 JSON 的数据完全依赖于 macOS 上 `system-ui` → SF Pro 的解析规则。若 Apple 未来调整 SF Pro Text/Display 的切换阈值，或改变 Canvas 与 DOM 的字体解析一致性，该快照会过时。
- **浏览器实现**：
  - 问题在 Chrome、Safari、Firefox 上均被观察到（因为根源是操作系统级别的字体解析，而非特定浏览器的布局引擎差异）。

### 无运行时加载

该 JSON **不被任何源码在运行时 `import` 或 `fetch`**。它是一个纯粹的文档/研究产物，仅通过 `RESEARCH.md` 的 Markdown 链接被人阅读，或通过仓库文件系统被开发者查阅。

## 风险、边界与改进建议

### 风险

1. **静态快照过时风险**
   - 该 JSON 记录的是某一时点（commit `d178a844`，约 2026-03-30）的 macOS 行为。若 Apple 更新系统字体或浏览器改变 `system-ui` 解析逻辑，JSON 中的 `match: true/false` 区间可能不再准确，从而误导后续开发者。

2. **无运行时防护**
   - 目前 Pretext 对 `system-ui` 的处理是“文档警告 + 代码注释”，而非运行时拦截或自动降级。用户若忽略文档，仍会在生产环境遭遇不可预测的换行/高度预测错误。

3. **误用为修复依据的风险**
   - `mismatchPct` 仅基于 `src/test-data.ts` 的标准语料。若用户的实际文本以 CJK、阿拉伯文或特殊标点为主，失配比例和表现可能与 JSON 中的数值不同。不应将该 JSON 当作通用保证。

4. **与 Emoji 修正的不对称性**
   - Pretext 对 Emoji（Apple Color Emoji）的 Canvas-DOM 差异有**自动运行时修正**（`getEmojiCorrection()`），但对 `system-ui` 没有。这种不对称性可能让用户困惑：为什么一个底层测量问题被修正而另一个没有。

### 边界

- **平台边界**：问题仅限于 macOS。Windows 的 `system-ui`（通常回落到 Segoe UI）和 Linux 的 `system-ui` 不在该 JSON 的覆盖范围内。
- **字号边界**：扫描仅覆盖 10px–28px。更小（<10px）或更大（>28px）的字号行为未知。
- **语料边界**：`mismatchPct` 基于固定的 `TEXTS` 语料，不代表所有真实世界文本。
- **修复范围边界**：`RESEARCH.md` 已明确排除了 lookup table、naive scaling、guessed substitution 等“半吊子”修复方案，将可信修复路径限制在非常窄的 DOM fallback。

### 改进建议

1. **增加生成脚本与可复现性**
   - 建议在 `scripts/` 下增加一个一次性的 `system-ui-scan.ts` 脚本（或将其能力合并到 `accuracy-check.ts` 的某个 flag 下），使得任何人都可以在 macOS 上复现该扫描，并重新生成 JSON。当前“无脚本、仅检入结果”的模式不利于后续验证。

2. **运行时显式检测与警告（开发模式）**
   - 在 `prepare()` 中增加开发环境（`process.env.NODE_ENV !== 'production'` 或 `typeof __DEV__ !== 'undefined'`）的检测：若传入的 `font` 字符串包含 `system-ui` 且平台为 macOS，可在控制台输出一次性的 `console.warn`，引用 `RESEARCH.md` 和该 JSON，降低用户误用概率。

3. **探索窄域 DOM fallback（若需求强烈）**
   - `TODO.md` 已提出此方向。若未来决定支持，可参考 `getEmojiCorrection()` 的实现模式：
     - 在 `prepare()` 首次遇到 `system-ui` 时，用隐藏 DOM span 测量一个代表性字符（如拉丁字母 “M” 或一个完整短句）的宽度。
     - 与 Canvas 测量值比较；若差异超过阈值，则对该字体**全程回退到 DOM 测量**（牺牲性能换取正确性），或至少标记该字体为“不可靠”并建议替换。
   - 注意：由于 `system-ui` 的不一致性随字号变化，简单的“每字体一次”校准可能不够，需要按 `(fontFamily, fontSize)` 对进行校准，这会增加缓存复杂度。

4. **扩展扫描覆盖**
   - 若维持 JSON 快照模式，建议至少补充：
     - 更大字号（如 8px, 9px, 32px, 48px, 64px）的扫描结果。
     - 非 macOS 平台（如 Windows 11 + Segoe UI）的对照数据，以明确问题的平台特异性。

5. **版本化或增加生成元数据**
   - 在 JSON 中增加 `generatedAt`、`macOSVersion`、`browserVersion`、`testDataVersion` 等字段，提高快照的可追溯性和可信度。
