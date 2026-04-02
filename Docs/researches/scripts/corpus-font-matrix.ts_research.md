# corpus-font-matrix.ts 研究文档

## 场景与职责

`scripts/corpus-font-matrix.ts` 是 Pretext 项目的**语料字体矩阵回归脚本**。它对 `corpora/sources.json` 中的特定语料，在多个备选字体变体（如日文 Hiragino Mincho vs Hiragino Sans）下执行宽度扫测，评估不同字体对布局预测精度的影响。该脚本主要用于捕获字体相关的 shaping/度量漂移问题。

## 功能点目的

1. **字体变体矩阵测试**：为每个语料配置一组 `FontVariant`（id、label、font CSS、lineHeight），在相同宽度集合下逐一测试。
2. **批量宽度扫测**：支持 `start/end/step` 线性扫测或 `--samples` 均匀采样，生成每个变体在每个宽度下的高度差异。
3. **差异汇总与分桶**：将 nonzero diff 按像素值分桶（如 `+32px: 450, 600`），便于快速识别系统性偏移模式。
4. **快照落盘**：支持 `--output=` 将完整矩阵摘要写入 JSON。

## 具体技术实现

### 关键数据结构

```ts
type FontVariant = { id: string; label: string; font: string; lineHeight: number };

type VariantResult = {
  id: string; label: string; font: string; lineHeight: number;
  widthCount: number; exactCount: number;
  mismatches: Array<{ width: number; diffPx: number; predictedHeight: number; actualHeight: number }>;
};

type MatrixSummary = {
  corpusId: string; title: string; browser: BrowserKind;
  widths: number[]; variants: VariantResult[];
};

type MatrixOptions = {
  id: string; browser: BrowserKind; port: number; timeoutMs: number;
  output: string | null; samples: number | null;
  start: number; end: number; step: number;
};
```

### 字体矩阵配置

脚本内硬编码 `FONT_MATRIX: Record<string, FontVariant[]>`，覆盖以下语料：

- `ja-kumo-no-ito`, `ja-rashomon`：Hiragino Mincho ProN / Hiragino Sans
- `ko-unsu-joh-eun-nal`：Apple SD Gothic Neo / AppleMyungjo
- `zh-zhufu`, `zh-guxiang`：Songti SC / PingFang SC
- `th-nithan-vetal-story-1`, `th-nithan-vetal-story-7`：Thonburi / Ayuthaya
- `my-cunning-heron-teacher`, `my-bad-deeds-return-to-you-teacher`：Myanmar MN / Myanmar Sangam MN / Noto Sans Myanmar
- `km-prachum-reuang-preng-khmer-volume-7-stories-1-10`：Khmer Sangam MN / Khmer MN
- `hi-eidgah`：Kohinoor Devanagari / Devanagari Sangam MN / ITF Devanagari
- `ar-risalat-al-ghufran-part-1`：Geeza Pro / SF Arabic / Arial
- `he-masaot-binyamin-metudela`：Times New Roman / SF Hebrew
- `ur-chughd`：Noto Nastaliq Urdu / Geeza Pro

### 核心流程

1. **参数解析**
   - `--id=`：指定语料 ID（必需）。
   - `--browser=` / `CORPUS_CHECK_BROWSER`：`chrome`（默认）或 `safari`。
   - `--start=`（默认 300）、`--end=`（默认 900）、`--step=`（默认 10）。
   - `--samples=`：若提供，则按均匀采样替代 step 扫测。
   - `--port=` / `CORPUS_CHECK_PORT`、 `--timeout=` / `CORPUS_CHECK_TIMEOUT_MS`。
   - `--output=`：JSON 落盘路径。

2. **宽度集合生成**
   - `getSweepWidths(meta, options)`：
     - 若 `samples` 非空，在 `[max(start, min_width), min(end, max_width)]` 区间内均匀采样并去重排序。
     - 否则按 `step` 生成等差数列。

3. **浏览器与会话准备**
   - 与 `corpus-sweep.ts` 类似，使用 `ensurePageServer`、`acquireBrowserAutomationLock`、`createBrowserSession`。

4. **逐变体扫测**
   对每个 `FontVariant`：
   - 生成 `requestId`。
   - 启动 `startPostedReportServer<CorpusSweepReport>(requestId)` 接收 POST 报告（因扫测宽度多，数据量大，避免 hash 长度限制）。
   - 构造 URL：
     ```
     /corpus?id=<corpusId>&widths=<w1,w2,...>&report=1&requestId=...&font=<variant.font>&lineHeight=<variant.lineHeight>&reportEndpoint=<endpoint>
     ```
   - `loadPostedReport` 等待报告。
   - 解析 `report.rows`，过滤 `Math.round(diffPx) !== 0` 的项为 mismatches，计算 `exactCount`。
   - `bucketMismatches` 将 mismatches 按 diffPx 值分桶，生成人类可读的汇总字符串。

5. **结果输出**
   - `printSummary` 打印语料名、宽度列表、每个变体的 exact/mismatch 统计及分桶结果。
   - 若指定 `--output`，将 `MatrixSummary` 写入 JSON。

## 关键代码路径与文件引用

- 本文件：`scripts/corpus-font-matrix.ts`
- 浏览器自动化基座：`scripts/browser-automation.ts`
- POST 报告服务器：`scripts/report-server.ts`
- 被测页面：`pages/corpus.ts`（处理 `widths` 批量参数与 `font` / `lineHeight` 覆盖）
- 语料元数据：`corpora/sources.json`
- 输出/调用：`package.json` 中 `"corpus-font-matrix"`、`"corpus-font-matrix:safari"`；结果通常用于手动评估或更新 `corpora/dashboard.json` 中的 `fontMatrixNotes`。

## 依赖与外部交互

- **Node/Bun 运行时**：`node:fs`（`writeFileSync`）、`node:child_process`。
- **本地浏览器**：Chrome / Safari（AppleScript）。
- **环境变量**：`CORPUS_CHECK_BROWSER`, `CORPUS_CHECK_PORT`, `CORPUS_CHECK_TIMEOUT_MS`。
- **上游页面**：`pages/corpus.ts` 的 `runSweep` 批量测量逻辑。

## 风险、边界与改进建议

1. **FONT_MATRIX 硬编码且与 sources.json 不同步**：若新增语料到 `sources.json` 但未在 `FONT_MATRIX` 中配置，脚本会直接抛错。建议将字体变体配置抽离到独立 JSON 文件（如 `corpora/font-matrix.json`），并在启动时做键集合校验。
2. **不支持 Firefox**：与 benchmark-check 类似，仅支持 Chrome/Safari。若字体矩阵需要在 Gecko 上验证，需扩展浏览器支持。
3. **POST server 单线程**：`startPostedReportServer` 在同一端口上只处理一个报告；若浏览器因某种原因 POST 两次，第二次会被 204 静默丢弃。当前逻辑通过 `requestId` 匹配第一次有效数据，行为可接受。
4. **采样与 step 的互斥逻辑清晰但无混合模式**：无法做到“在 300-900 范围内 step=10，但只测其中 9 个样本”。若需要更灵活的矩阵策略（如只测锚点宽度），需手动传宽度列表，但本脚本不支持直接传宽度列表。
5. **输出格式未纳入 dashboard 自动生成**：`corpus-status.ts` 读取的是 `corpora/representative.json` 和 sweep JSON，不读取字体矩阵输出。字体矩阵结果目前以人工 notes 形式维护在 `corpus-status.ts` 的 `FONT_MATRIX_NOTES` 中。建议将矩阵输出也纳入 `corpora/dashboard.json` 的自动生成流程。
