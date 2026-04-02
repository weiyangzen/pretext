# corpus-taxonomy.ts 研究文档

## 场景与职责

`scripts/corpus-taxonomy.ts` 是 Pretext 项目的**语料不匹配分类脚本**。它在粗粒度扫测的基础上，对每个不匹配宽度提取 `pages/corpus.ts` 返回的断点诊断信息，依据规则将不匹配归类到 `edge-fit`、`shaping-context`、`glue-policy`、`boundary-discovery`、`diagnostic-sensitivity` 或 `unknown`，帮助团队理解误差分布并指导修复优先级。

## 功能点目的

1. **误差分类 Taxonomy**：将大量的 width-level mismatches 归纳为少数几类可行动的根因桶。
2. **Extractor 敏感性问题识别**：自动检测 span-probe 与 range 提取结果不一致导致的伪阴性（`diagnostic-sensitivity`）。
3. **快速定位系统性问题**：例如大量 `+32px` 差异若集中在 `edge-fit`，说明是临界容纳策略差异；若集中在 `shaping-context`，说明是 canvas 分段累加与整行测量漂移。
4. **支持覆盖参数**：与 `corpus-check.ts` 一致，支持 `--font=`、`--lineHeight=` 覆盖。

## 具体技术实现

### 关键数据结构

```ts
type CorpusBreakMismatch = {
  line: number; deltaText: string; reasonGuess: string;
  oursContext: string; browserContext: string;
};

type CorpusSweepRow = {
  width: number; diffPx: number;
  browserLineMethod?: 'span-probe' | 'range';
  maxLineWidthDrift?: number;
  firstBreakMismatch?: CorpusBreakMismatch | null;
};

type TaxonomyCategory =
  | 'edge-fit' | 'shaping-context' | 'glue-policy'
  | 'boundary-discovery' | 'diagnostic-sensitivity' | 'unknown';

type TaxonomyEntry = {
  width: number; diffPx: number; category: TaxonomyCategory;
  reason: string; deltaText: string;
};
```

### 分类规则（`classifyTaxonomy`）

```ts
const quoteOrPunctuationRe = /["'“”‘’«»‹›「」『』（）()［］【】。，、！？!?,.;:—-]/;

function classifyTaxonomy(row: CorpusSweepRow): TaxonomyCategory {
  const mismatch = row.firstBreakMismatch;
  const reason = mismatch?.reasonGuess ?? '';

  if (reason.includes('only') && reason.includes('overflow')) {
    return 'edge-fit';
  }
  if (reason.includes('segment sum drifts')) {
    return 'shaping-context';
  }
  if (row.browserLineMethod === 'span-probe' && (row.maxLineWidthDrift ?? 0) === 0 && mismatch === null) {
    return 'diagnostic-sensitivity';
  }
  if (mismatch != null && quoteOrPunctuationRe.test(mismatch.deltaText)) {
    return 'glue-policy';
  }
  if (mismatch != null) {
    return 'boundary-discovery';
  }
  return 'unknown';
}
```

规则解释：
- **edge-fit**：`reasonGuess` 包含 "only" 和 "overflow"，对应 `classifyBreakMismatch` 中“仅溢出 x px 但仍保留文本”的临界情况。
- **shaping-context**：`reasonGuess` 包含 "segment sum drifts"，说明 canvas 分段宽度累加与整行测量不一致。
- **diagnostic-sensitivity**：使用 span-probe 提取、且 maxLineWidthDrift 为 0、且没有 break mismatch——意味着 Pretext 与浏览器实际行高一致，但 extractor 方法导致误判。
- **glue-policy**：存在 break mismatch 且差异文本涉及标点/引号，说明是标点粘连策略差异。
- **boundary-discovery**：存在 break mismatch 但不涉及标点，说明是通用断点边界差异。
- **unknown**：其余无法归类的情况。

### 核心流程

1. **参数解析**
   - `--id=`：指定语料 ID（必需）。
   - `--browser=` / `CORPUS_CHECK_BROWSER`：`chrome`（默认）或 `safari`。
   - `--port=` / `CORPUS_CHECK_PORT`、 `--timeout=` / `CORPUS_CHECK_TIMEOUT_MS`（默认 180s）。
   - `--start=`（默认 300）、`--end=`（默认 900）、`--step=`（默认 10）。
   - `--samples=`：均匀采样替代 step。
   - `--font=`、`--lineHeight=`：覆盖。

2. **宽度集合生成**
   - 与 `corpus-sweep.ts` 相同，支持位置参数传入宽度列表（优先级高于 start/end/step）。

3. **服务与浏览器启动**
   - `ensurePageServer`、`acquireBrowserAutomationLock`、`createBrowserSession`。

4. **批量诊断与分类**
   - 构造 URL：
     ```
     /corpus?id=<corpusId>&widths=<...>&report=1&diagnostic=full&requestId=...&reportEndpoint=...
     ```
   - 通过 `loadPostedReport` 获取带 `diagnostic=full` 的 `CorpusSweepReport`。
   - 遍历 `report.rows`：
     - `diffPx === 0` 跳过。
     - 否则调用 `classifyTaxonomy` 得到 category，组装 `TaxonomyEntry`。

5. **结果输出**
   - `printEntries` 按 category 分桶，打印每类数量及前 8 条示例（宽度、diffPx、reason、deltaText）。

## 关键代码路径与文件引用

- 本文件：`scripts/corpus-taxonomy.ts`
- 浏览器自动化基座：`scripts/browser-automation.ts`
- POST 报告服务器：`scripts/report-server.ts`
- 被测页面：`pages/corpus.ts`（`addDiagnostics` 生成 `firstBreakMismatch`、`maxLineWidthDrift`、`browserLineMethod`）
- 诊断工具：`pages/diagnostic-utils.ts`（`formatBreakContext`、`classifyBreakMismatch` 等被页面调用）
- 语料元数据：`corpora/sources.json`

## 依赖与外部交互

- **Node/Bun 运行时**：`node:child_process`。
- **本地浏览器**：Chrome / Safari（AppleScript）。
- **环境变量**：`CORPUS_CHECK_BROWSER`, `CORPUS_CHECK_PORT`, `CORPUS_CHECK_TIMEOUT_MS`。
- **上游页面**：`pages/corpus.ts` 的 `diagnostic=full` 报告格式。

## 风险、边界与改进建议

1. **分类规则基于字符串匹配，脆弱性较高**：`reason.includes('only') && reason.includes('overflow')` 和 `reason.includes('segment sum drifts')` 强依赖 `pages/corpus.ts` 中 `classifyBreakMismatch` 的文案。若页面侧修改文案，分类会大量落入 `unknown`。建议将 reason 改为机器可读枚举值（如 `'edge-fit' | 'segment-sum-drift' | ...`）。
2. **diagnostic-sensitivity 判定过于严格**：要求 `maxLineWidthDrift === 0` 且 `mismatch === null`。由于浮点精度，drift 可能为 `0.001` 级别，导致本可识别的 extractor 敏感性问题被漏分。建议引入 epsilon 容差（如 `<= 0.005`）。
3. **glue-policy 的标点正则范围有限**：当前正则未覆盖所有 Unicode 标点（如阿拉伯标点、天城文 danda）。对于非 CJK/Latin 语料，可能误判为 `boundary-discovery`。建议引用 Unicode 标点属性或扩展正则范围。
4. **无落盘能力**： unlike `corpus-sweep.ts`，本脚本不支持 `--output=`。若需要将 taxonomy 结果用于自动化报告或 dashboard，必须手动复制终端输出。建议增加 `--output` 支持，输出结构化 JSON。
5. **仅支持单语料**：没有 `--all` 模式。若需批量生成所有语料的 taxonomy，需外部脚本循环调用。建议增加 `--all` 或 `--batch` 支持。
