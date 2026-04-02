# corpus-status.ts 研究文档

## 场景与职责

`scripts/corpus-status.ts` 是 Pretext 项目的**语料仪表盘聚合生成脚本**。它读取已检入的多个 JSON 快照（representative、sweep、accuracy），结合脚本内硬编码的语料元数据与人工 notes，生成 `corpora/dashboard.json`——为团队和自动化工具提供机器可读的长文本语料健康概览。

## 功能点目的

1. **多源数据聚合**：将 `corpora/representative.json`、`corpora/chrome-sampled.json`、`corpora/chrome-step10.json`、`accuracy/*.json` 合并为统一视图。
2. **分类健康展示**：将语料分为 `productShaped`（混合应用文本）和 `longForm`（各语言长文本），每类展示 chrome/safari 锚点差异、chrome 采样/步进扫测统计。
3. **回归门控摘要**：在 `browserRegressionGate` 中汇总三大浏览器的 accuracy 总匹配数/不匹配数，作为引擎改动的快速红绿灯指标。
4. **人工 notes 结构化**：将 `FINE_SWEEP_NOTES` 和 `FONT_MATRIX_NOTES` 等人工观察写入 dashboard，便于追踪已知问题类别。

## 具体技术实现

### 关键数据结构

```ts
type AccuracySnapshot = { total?: number; matchCount?: number; mismatchCount?: number };
type RepresentativeRow = { corpusId: string; width: number; diffPx: number };
type RepresentativeSnapshot = { browsers: Partial<Record<BrowserKind, { rows: RepresentativeRow[] }>> };
type SweepSummary = { corpusId: string; language: string; title: string; widthCount: number; exactCount: number };
type AnchorSummary = { exactWidths: number[]; mismatches: Array<{ width: number; diffPx: number }> };
type CorpusDashboardMeta = { id: string; notes: string };
type FineSweepNote = { corpusId: string; result: {...}; notes: string };
type FontMatrixNote = { corpusId: string; status: string; notes: string };
```

### 硬编码元数据

脚本内维护三份常量数组：

- `PRODUCT_SHAPED`：仅包含 `mixed-app-text`，附 notes 说明剩余 Chrome-only `710px` miss 的已知原因。
- `LONG_FORM`：包含 15 条长文本语料（en-gatsby-opening、ja-rashomon、zh-zhufu、th-nithan-vetal-story-1 等），每条附 notes 描述当前健康状态与已知问题类别。
- `FINE_SWEEP_NOTES`：记录 `ar-risalat-al-ghufran-part-1`、`my-cunning-heron-teacher`、`ur-chughd` 的细粒度扫测观察。
- `FONT_MATRIX_NOTES`：记录各语料在字体矩阵测试中的表现摘要。

### 核心流程

1. **参数解析**
   - `--output=`：默认 `corpora/dashboard.json`。

2. **数据加载**
   ```ts
   const representative = await loadJson<RepresentativeSnapshot>('corpora/representative.json');
   const chromeSampled = await loadJson<SweepSummary[]>('corpora/chrome-sampled.json');
   const chromeStep10 = await loadJson<SweepSummary[]>('corpora/chrome-step10.json');
   const chromeAccuracy = await loadJson<AccuracySnapshot>('accuracy/chrome.json');
   const safariAccuracy = await loadJson<AccuracySnapshot>('accuracy/safari.json');
   const firefoxAccuracy = await loadJson<AccuracySnapshot>('accuracy/firefox.json');
   ```

3. **索引构建**
   - `indexRepresentativeRows`：按 `corpusId` 将 representative 行分桶为 `Map<string, RepresentativeRow[]>`。
   - `indexSweepSummaries`：将 sweep 数组转为 `Map<string, SweepSummary>`。

4. **摘要计算**
   - `summarizeAnchors(rows)`：对某一语料的 representative rows 排序，分离 `exactWidths`（diffPx 四舍五入为 0）和 `mismatches`（非 0）。
   - `summarizeAccuracy(snapshot)`：提取 total / matchCount / mismatchCount。

5. **Dashboard 组装**
   输出结构包含：
   - `generatedAt`：ISO 时间戳。
   - `sources`：记录各输入文件路径，便于溯源。
   - `browserRegressionGate`：三大浏览器的 accuracy 汇总。
   - `productShaped` / `longForm`：遍历硬编码 meta 数组，合并 representative anchors、sweep 统计、notes。
   - `fineSweepNotes` / `fontMatrixNotes`：原样写入。

6. **落盘**
   - `mkdirSync(dirname(output), { recursive: true })`
   - `writeFileSync(output, JSON.stringify(dashboard, null, 2) + '\n', 'utf8')`

## 关键代码路径与文件引用

- 本文件：`scripts/corpus-status.ts`
- 输入数据：
  - `corpora/representative.json`（`scripts/corpus-representative.ts` 生成）
  - `corpora/chrome-sampled.json`（`scripts/corpus-sweep.ts --all --samples=9` 生成）
  - `corpora/chrome-step10.json`（`scripts/corpus-sweep.ts --all --start=300 --end=900 --step=10` 生成）
  - `accuracy/chrome.json`, `accuracy/safari.json`, `accuracy/firefox.json`（`scripts/accuracy-check.ts --full --output=...` 生成）
- 输出：`corpora/dashboard.json`
- 调用链：`package.json` 中 `"corpus-status"` 和 `"corpus-status:refresh"`

## 依赖与外部交互

- **Node/Bun 运行时**：`node:fs`, `node:path`。
- **无浏览器交互**：纯本地 JSON 聚合脚本，运行极快。
- **无外部网络依赖**。

## 风险、边界与改进建议

1. **硬编码 meta 与输入数据不同步**：若 `corpora/sources.json` 新增语料，必须同步在 `LONG_FORM` 中增加条目并撰写 notes，否则 dashboard 不会展示该语料。建议增加启动断言：校验 `LONG_FORM` 的 ID 集合与 `sources.json` 的交集/差集，并给出警告。
2. **输入文件缺失即崩溃**：`loadJson` 使用 `Bun.file(path).json()`，若任一输入文件不存在会抛异常。对于可选输入（如 `firefoxAccuracy` 在部分环境下可能未生成），应增加容错降级（设为 null 并跳过）。
3. **无数据新鲜度校验**：脚本不检查输入文件的 mtime，可能用 stale 数据生成 dashboard。建议在 `sources` 中增加每个输入文件的 `generatedAt`（若输入 JSON 包含该字段），或在 dashboard 中标注“数据可能未同步刷新”。
4. **输出格式无 schema 校验**：`corpora/dashboard.json` 被外部工具消费，但目前没有 TypeScript 类型导出或 JSON Schema。建议增加 `scripts/corpus-status.ts` 同级的一个 `dashboard-schema.json`，防止结构漂移。
5. **人工 notes 维护成本高**：`FINE_SWEEP_NOTES` 和 `FONT_MATRIX_NOTES` 需要手动更新。可考虑在 `corpus-sweep.ts` 和 `corpus-font-matrix.ts` 的输出中增加自动分类标签，减少人工 notes 的重复劳动。
