# research-data 目录研究报告

## 场景与职责

`research-data/` 是 Pretext 仓库中用于存放**机器可读型研究产物**的目录，与 `accuracy/`、`benchmarks/`、`corpora/`、`status/` 等目录共同构成项目的数据/快照层。当前该目录仅包含一个检入文件：

- `research-data/system-ui-size-scan.json`

该目录的职责边界非常明确：**保存无法被常规测试断言直接覆盖、但对架构决策有长期约束力的研究发现**。具体到当前文件，它记录了 macOS 上 `system-ui` 字体在 Canvas `measureText()` 与 DOM 实际排版之间存在“光学变体（optical variants）”解析不一致的定量扫描结果。这一发现直接导致了：

1. 源码层面将 `system-ui` 标记为“不安全字体”；
2. 所有浏览器精度测试（accuracy check）和语料库测试（corpus check）均回避使用 `system-ui`，改用命名字体（如 `16px Helvetica` 等）；
3. 在 `TODO.md` 中留下一个长期悬而未决的开放问题：是否值得为 `system-ui` 实现窄域的 prepare-time DOM fallback。

## 功能点目的

`system-ui-size-scan.json` 的核心目的是**将一次人工/脚本化的字体矩阵探测结果固化为可版本控制的机器可读数据**，从而：

- **支撑文档可信度**：`RESEARCH.md` 中关于 "system-ui font resolution mismatch" 的论断不再停留在定性描述，而是有可回溯的 JSON 证据；
- **约束 API 边界**：`src/layout.ts` 头部注释直接引用该研究结论，向库的使用者说明 `system-ui` 在 macOS 上会导致 `layout()` 结果与真实 DOM 行高不一致；
- **防止回归误操作**：当未来有人提议“支持 system-ui”时，该文件提供了精确的失效尺寸表（10–12px、14px、26px 为失效点），避免重复做一遍扫描工作。

从数据形态上看，该文件属于**一次性研究快照（one-off research snapshot）**，而非由 CI 持续生成的动态报告。它不参与任何自动化流水线，也不被脚本在运行时读取。

## 具体技术实现

### JSON 数据结构

文件采用扁平结构，Schema 如下：

```typescript
type SystemUISizeScan = {
  font: string;        // 被测字体族名，固定为 "system-ui"
  platform: string;    // 测试平台，固定为 "macOS"
  results: Array<{
    startSizePx: number;   // 起始字号（px）
    endSizePx: number;     // 结束字号（px），与起始相同表示单点，不同表示连续区间
    match: boolean;        // Canvas 与 DOM 是否一致
    mismatchPct?: number;  // 当 match=false 时，记录不匹配样本的百分比
  }>;
};
```

当前检入的数据内容：

| startSizePx | endSizePx | match | mismatchPct |
|-------------|-----------|-------|-------------|
| 10          | 10        | false | 2.9         |
| 11          | 11        | false | 6.9         |
| 12          | 12        | false | 11.3        |
| 13          | 13        | true  | —           |
| 14          | 14        | false | 14.5        |
| 15          | 25        | true  | —           |
| 26          | 26        | false | 12.4        |
| 27          | 28        | true  | —           |

### 技术背景：为何会出现 mismatch

macOS 的 `system-ui` 会映射到 Apple 的 SF Pro 字体族。SF Pro 分为 **SF Pro Text**（小字号光学变体）和 **SF Pro Display**（大字号光学变体）。Canvas 2D 上下文与 DOM 渲染引擎在切换这两种光学变体的阈值上存在差异，导致某些字号下：

- Canvas `ctx.measureText(seg).width` 得到的宽度；
- 与 DOM 中实际 span 的 `getBoundingClientRect().width` 得到的宽度；

出现可观测的偏差（>0.5px 甚至数 px）。由于 Pretext 的核心架构是 **prepare 阶段一次性测宽、layout 阶段纯算术求和**，一旦 prepare 阶段的 Canvas 测宽与 DOM 真实宽度不一致，layout 算出的行数/行高就会系统性偏离浏览器真实排版。

### 与 Emoji 校正机制的对比

在 `src/measurement.ts` 中，项目对 Emoji 宽度偏差实现了自动校正（`getEmojiCorrection`）：

```typescript
// src/measurement.ts:123-151
function getEmojiCorrection(font: string, fontSize: number): number {
  // ... 创建隐藏 span，比较 canvasW 与 domW ...
}
```

该机制在检测到 Canvas 与 DOM 不一致时，会计算一个常量修正值 `correction`，并在后续 `getCorrectedSegmentWidth` 中扣除。然而，**`system-ui` 的 mismatch 目前没有任何类似的运行时校正**。`research-data/system-ui-size-scan.json` 的存在，正是用来说明“这个问题被发现、被量化，但尚未被自动修复”。

## 关键代码路径与文件引用

### 直接引用该 JSON 或其结论的文件

1. **`RESEARCH.md`**（第 55–70 行）
   - 章节标题："Discovery: system-ui font resolution mismatch"
   - 直接给出该 JSON 的文件链接，并解读了 mismatch 的聚类尺寸（10–12px、14px、26px）。

2. **`AGENTS.md`**（第 64 行）
   - 明确指令：`system-ui` is unsafe for accuracy; canvas and DOM can resolve different fonts on macOS.

3. **`src/layout.ts`**（第 28–31 行）
   - 源码头部 Limitations 注释块：
     ```
     //   - system-ui font: canvas resolves to different optical variants than DOM on macOS.
     //     Use named fonts (Helvetica, Inter, etc.) for guaranteed accuracy.
     //     See RESEARCH.md "Discovery: system-ui font resolution mismatch".
     ```

4. **`README.md`**（第 138 行）
   - 用户可见的 API 文档：`- system-ui is unsafe for layout() accuracy on macOS. Use a named font.`

5. **`TODO.md`**（第 41 行）
   - 开放问题："Whether strong real-world demand for system-ui would justify a narrow prepare-time DOM fallback."

6. **`corpora/TAXONOMY.md`**（第 135、143 行）
   - 在“字体解析不一致（font-resolution mismatch）”分类下，将 historical `system-ui` mismatch 作为典型示例。

### 该 JSON 未出现在以下路径中

- `scripts/*`：没有任何脚本读取或生成 `research-data/` 下的文件；
- `pages/*.ts`：浏览器测试页面不加载该 JSON；
- `package.json`：没有针对 `research-data` 的 npm script；
- `src/measurement.ts`：虽然这里是最适合放置 DOM fallback 的地方，但目前代码中完全没有引用该文件的数据。

### Git 历史

该文件于 commit `d178a84`（2026-03-30，作者 Cheng Lou）随 dashboard JSON 化改造一起被引入。提交信息为 "Replace markdown status tables with JSON dashboards"。这表明该文件的创建动机与项目整体“将研究结论和状态快照从 Markdown 表格迁移到机器可读 JSON”的治理策略一致。

## 依赖与外部交互

### 上游依赖（生成该数据所需的外部条件）

- **硬件/平台**：macOS（文件显式标注 platform 为 macOS）。
- **浏览器引擎**：需要同时比较 Canvas 2D `measureText` 与 DOM `getBoundingClientRect` 的行为。
- **字体栈**：系统必须安装 Apple SF Pro 字体族（macOS 系统自带）。
- **探测方法**：虽然生成脚本未保留在仓库中，但从 `mismatchPct` 字段可推断，原始探测 likely 使用了与 accuracy/corpus check 类似的浏览器自动化框架（Playwright/Puppeteer），在多个字号下对大量文本样本进行 Canvas-vs-DOM 对比，统计不匹配比例。

### 下游消费方

- **人类读者**：开发者通过 `RESEARCH.md`、`README.md`、`AGENTS.md` 阅读并理解该限制；
- **代码注释**：`src/layout.ts` 的注释约束了 API 设计，防止在 hot path 中为 `system-ui` 引入 DOM 回退；
- **未来可能的脚本**：如果 TODO 中的 "narrow prepare-time DOM fallback" 被实现，该 JSON 将成为实现时的校准依据（知道哪些字号区间需要 fallback）。

### 与相邻数据目录的关系

| 目录 | 内容性质 | 与 research-data 的区别 |
|------|----------|------------------------|
| `accuracy/` | 浏览器精度测试快照（chrome.json 等） | 动态、定期刷新，被 CI/脚本消费 |
| `benchmarks/` | 性能基准快照 | 动态、定期刷新 |
| `corpora/` | 长文本语料库精度快照 | 动态、定期刷新 |
| `status/` | 聚合 dashboard（由 `scripts/status-dashboard.ts` 生成） | 完全自动化生成 |
| `research-data/` | 一次性研究发现 | 静态、手工检入、无自动化消费 |

## 风险、边界与改进建议

### 风险

1. **平台单一性风险**：文件仅标注 `macOS`，未覆盖 iOS、Windows、Linux 等平台的 `system-ui` 行为。在 Windows 上 `system-ui` 可能映射到 Segoe UI，其 Canvas/DOM 一致性状态未知。
2. **数据陈旧风险**：macOS 系统更新或浏览器版本升级可能改变 SF Pro 的阈值。该 JSON 没有 `generatedAt` 或 `browserVersion` 字段，未来难以判断数据是否过期。
3. **静态孤岛风险**：该文件目前没有任何脚本消费，意味着它只能被“读到并记住”的人使用。新贡献者如果不读 `RESEARCH.md`，可能不会注意到这个限制。
4. **与页面样式的割裂**：讽刺的是，仓库内多个 HTML 页面（`pages/accuracy.html`、`pages/benchmark.html`、`pages/corpus.html`、`pages/probe.html` 等）的 body 样式都声明了 `font-family: system-ui, -apple-system, sans-serif`。虽然这些页面只是 UI  chrome 而非被测文本的字体，但这种用法在视觉上与“system-ui 不安全”的结论并存，容易对新人造成困惑。

### 边界

- **非运行时配置**：该 JSON 不是 `layout()` 的运行时参数，也不被 `src/measurement.ts` 读取；
- **非测试断言**：它不参与任何 pass/fail 判断；
- **非生成器脚本**：仓库中不包含重新生成该文件的脚本，因此它属于“只读研究遗产”。

### 改进建议

1. **增加元数据字段**：建议在 JSON 中补充 `generatedAt`、`browserVersion`、`engineVersion`（Pretext 版本号）和 `sampleCount`（用于计算 mismatchPct 的样本量），以提升可追溯性。
2. **扩展平台矩阵**：如果资源允许，补充 Windows / Linux 的扫描结果，或至少将文件名改为 `system-ui-macos-size-scan.json`，避免让人误以为这是跨平台结论。
3. **自动化校验（轻量级）**：在 `scripts/` 中增加一个可选的 `system-ui-check.ts` 脚本，定期（如每月手动运行一次）复测该 JSON 中的关键字号点，验证 mismatch 是否仍然存在。这可以将静态快照转化为可复现的研究工具。
4. **运行时防御（若 TODO 被采纳）**：如果未来决定支持 `system-ui`，应在 `src/measurement.ts` 的 `getFontMeasurementState` 或新增函数中实现 prepare-time DOM fallback：当检测到字体包含 `system-ui` 且平台为 macOS 且字号落在已知 bad tuples（10–12, 14, 26）时，用一次 DOM 测量替代 Canvas 测量。该 JSON 的 `results` 数组可直接作为白名单/黑名单输入。
5. **文档一致性**：考虑在 `pages/*.html` 的样式注释中增加一句说明，解释为何页面 UI 使用 `system-ui` 而测试字体不使用，以减少认知摩擦。
