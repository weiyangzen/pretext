# Scripts 目录深度研究报告

## 1. 场景与职责

`scripts/` 目录是 Pretext 项目的自动化测试与验证基础设施核心，承担以下关键职责：

### 1.1 核心使命

- **浏览器准确性验证**：通过自动化浏览器（Chrome/Safari/Firefox）对比 Pretext 库的布局计算结果与真实浏览器 DOM 渲染结果
- **性能基准测试**：测量 `prepare()` / `layout()` 等核心 API 的性能表现，与 DOM 操作进行对比
- **语料库回归测试**：使用多语言长文本语料库（日语、中文、阿拉伯语、泰语等）验证复杂文本场景的准确性
- **发布前验证**：确保 npm 包在 JS/TS 消费者环境中可用

### 1.2 使用场景

| 场景 | 主要脚本 | 触发方式 |
|------|----------|----------|
| 开发阶段快速验证 | `accuracy-check.ts`, `probe-check.ts` | `bun run accuracy-check` |
| CI/发布前检查 | `package-smoke-test.ts` | `bun run package-smoke-test` |
| 多语言语料库测试 | `corpus-check.ts`, `corpus-sweep.ts` | `bun run corpus-check --id=ja-rashomon` |
| 性能回归检测 | `benchmark-check.ts` | `bun run benchmark-check` |
| 状态仪表板生成 | `status-dashboard.ts`, `corpus-status.ts` | `bun run status-dashboard` |
| 静态站点构建 | `build-demo-site.ts` | `bun run site:build` |

---

## 2. 功能点目的

### 2.1 准确性验证脚本

#### `accuracy-check.ts`
- **目的**：在浏览器中运行 `/accuracy` 页面，对比 Pretext 预测高度与浏览器实际渲染高度
- **关键特性**：
  - 支持 Chrome/Safari/Firefox 三种浏览器
  - Firefox 使用代理服务器绕过 IPv6 连接问题
  - 支持 `--full` 标志输出完整行数据到 JSON
  - 支持 `--output=` 指定输出文件
  - 通过 URL hash 或 POST 方式接收报告

#### `probe-check.ts`
- **目的**：对任意文本进行单点探针式准确性验证
- **关键特性**：
  - 支持自定义文本、宽度、字体、行高、方向
  - 支持 `whiteSpace: 'pre-wrap'` 模式
  - 支持 `span`/`range` 两种浏览器行提取方法
  - 详细的断点不匹配诊断信息

#### `pre-wrap-check.ts`
- **目的**：专门验证 `whiteSpace: 'pre-wrap'` 模式的浏览器 oracle 测试
- **测试用例**：包含悬挂空格、硬换行、制表符、混合脚本缩进、RTL 缩进等 17 个边界场景

### 2.2 语料库测试脚本

#### `corpus-check.ts`
- **目的**：对单一语料库在指定宽度下进行详细诊断
- **关键特性**：
  - 支持 `--diagnose` 完整诊断模式
  - 支持 `--method=span/range` 切换浏览器行提取方法
  - 支持 `--font=` 和 `--lineHeight=` 覆盖默认字体设置
  - 支持 `--sliceStart=` / `--sliceEnd=` 文本切片
  - 输出首次断点不匹配详情、行宽漂移分析

#### `corpus-sweep.ts`
- **目的**：对语料库进行宽度范围扫描（默认 300-900px，步进 10px）
- **关键特性**：
  - 支持 `--all` 扫描所有语料库
  - 支持 `--samples=N` 采样模式（均匀分布 N 个宽度点）
  - 支持 `--diagnose` 对不匹配宽度自动调用 `corpus-check`
  - 输出 JSON 格式的扫描摘要

#### `corpus-font-matrix.ts`
- **目的**：测试同一语料库在不同字体变体下的表现
- **字体矩阵**：为日语（Hiragino Mincho/Sans）、中文（Songti/PingFang）、阿拉伯语（Geeza Pro/SF Arabic/Arial）等配置多字体变体

#### `corpus-representative.ts`
- **目的**：生成代表性语料库快照（`corpora/representative.json`）
- **覆盖范围**：10 个核心语料库 × 3 个宽度（300/600/800）× Chrome/Safari

#### `corpus-taxonomy.ts`
- **目的**：对语料库不匹配结果进行分类（Taxonomy）
- **分类类别**：
  - `edge-fit`：边缘适配问题（仅溢出）
  - `shaping-context`：字形塑形上下文漂移
  - `glue-policy`：标点/空格粘连策略问题
  - `boundary-discovery`：断点发现差异
  - `diagnostic-sensitivity`：诊断方法敏感性（span vs range）

#### `corpus-status.ts`
- **目的**：基于已检入的 JSON 快照生成语料库仪表板（`corpora/dashboard.json`）
- **输入源**：`corpora/representative.json`, `corpora/chrome-sampled.json`, `corpora/chrome-step10.json`

### 2.3 性能测试脚本

#### `benchmark-check.ts`
- **目的**：运行 `/benchmark` 页面的性能测试套件
- **测试内容**：
  - 顶层批处理基准：`prepare()`, `layout()`
  - 富行 API：`layoutWithLines()`, `walkLineRanges()`, `layoutNextLine()`
  - Pre-wrap 块压力测试
  - 阿拉伯长文压力测试
  - 长文语料库压力测试（分析/测量/准备/布局阶段拆分）
- **关键特性**：`foreground: true` 确保浏览器在前台运行（避免节流）

### 2.4 兼容性别名脚本

#### `gatsby-check.ts` / `gatsby-sweep.ts`
- **目的**：向后兼容的别名，指向 `en-gatsby-opening` 语料库检查
- **实现**：通过 `spawnSync` 调用 `corpus-check.ts` / `corpus-sweep.ts` 并传递环境变量

### 2.5 发布验证脚本

#### `package-smoke-test.ts`
- **目的**：验证 npm 包在真实消费者环境中的可用性
- **测试流程**：
  1. `npm pack` 生成 tarball
  2. 创建临时 JS ESM 项目，验证 `import * as pretext` 和 API 存在性
  3. 创建临时 TS 项目，验证类型定义和编译时类型检查
  4. 验证错误用法会被 TS 编译器捕获

### 2.6 站点构建脚本

#### `build-demo-site.ts`
- **目的**：构建 GitHub Pages 静态演示站点
- **流程**：
  1. 使用 `bun build` 构建所有 demo 入口点
  2. 重新组织输出目录结构（将扁平结构转为子目录结构）
  3. 重写相对资源 URL 和 demo 链接

### 2.7 仪表板生成脚本

#### `status-dashboard.ts`
- **目的**：生成主状态仪表板（`status/dashboard.json`）
- **输入源**：`accuracy/*.json`, `benchmarks/*.json`
- **输出**：浏览器准确性汇总、性能基准索引

---

## 3. 具体技术实现

### 3.1 浏览器自动化架构

#### 核心模块：`browser-automation.ts`

**BrowserSession 抽象**：
```typescript
type BrowserSession = {
  navigate: (url: string) => MaybePromise<void>
  readLocationUrl: () => MaybePromise<string>
  close: () => void
}
```

**三种浏览器实现**：

| 浏览器 | 实现技术 | 特点 |
|--------|----------|------|
| Chrome | AppleScript | 通过 `osascript` 控制 Google Chrome，支持前台/后台模式 |
| Safari | AppleScript | 创建新文档/窗口，设置 URL，读取位置 |
| Firefox | WebSocket BiDi | 使用 Firefox 的远程调试协议（WebSocket），headless 模式 |

**Firefox BiDi 实现细节**：
- 启动参数：`--headless --new-instance --profile <dir> --remote-debugging-port <port>`
- WebSocket 连接到 `ws://127.0.0.1:${port}/session`
- 使用 `browsingContext.navigate` 和 `script.evaluate` 进行导航和脚本执行

#### 浏览器自动化锁

**文件锁机制**：
- 锁目录：`$TMPDIR/pretext-browser-automation-locks/`
- 锁文件：`${browser}.lock`
- 元数据：JSON 格式存储 `{ pid, startedAt }`
- 自修复：检测到锁持有者进程已死亡时自动清理

**关键函数**：
```typescript
acquireBrowserAutomationLock(browser: BrowserKind, timeoutMs = 120_000): Promise<BrowserAutomationLock>
```

#### 页面服务器管理

**端口解析策略**：
```typescript
const LOOPBACK_BASES = [
  'http://127.0.0.1',
  'http://localhost', 
  'http://[::1]',
]
```

**服务器启动**：
- 命令：`bun --port=${port} --no-hmr pages/*.html`
- 重试逻辑：20 秒超时，每 100ms 检查一次端口可达性

### 3.2 报告传输协议

#### Hash-based 报告（小数据）

**流程**：
1. 页面将报告序列化为 JSON
2. 设置 `location.hash = '#report=' + encodeURIComponent(json)`
3. 自动化脚本轮询 `session.readLocationUrl()`
4. 解析 hash 提取报告

**导航状态追踪**：
```typescript
type NavigationPhase = 'loading' | 'measuring' | 'posting'
// URL hash: #phase=measuring&requestId=xxx
```

#### POST-based 报告（大数据）

**适用场景**：语料库扫描产生的大量行数据

**流程**：
1. 脚本启动 `startPostedReportServer<T>(expectedRequestId)`
2. 服务器监听 `127.0.0.1:随机端口`
3. 页面通过 `fetch(reportEndpoint, { method: 'POST', body: json })` 发送
4. 脚本通过 `waitForReport(timeoutMs)` 接收

**服务器实现**（`report-server.ts`）：
- CORS 支持：`access-control-allow-origin: *`
- 请求 ID 验证：只接受匹配 `expectedRequestId` 的报告
- 超时处理：默认 120 秒

### 3.3 语料库诊断技术

#### 浏览器行提取方法

**Span Probe 方法**：
```typescript
// 为每个诊断单元创建 <span>
for (const unit of units) {
  const span = document.createElement('span')
  span.textContent = unit.text
  div.appendChild(span)
}
// 通过 getBoundingClientRect() 检测行边界
```

**Range 方法**：
```typescript
// 使用 DOM Range API
range.setStart(textNode, unit.start)
range.setEnd(textNode, unit.end)
const rects = range.getClientRects()
```

**方法选择策略**：
- 泰语/老挝语/高棉语/缅甸语：强制使用 Range（span 会干扰布局）
- RTL 文本：默认使用 Range
- 其他：根据 span/range 结果一致性自动选择

#### 断点不匹配分类

**分类逻辑**（`corpus-taxonomy.ts`）：
```typescript
function classifyTaxonomy(row: CorpusSweepRow): TaxonomyCategory {
  if (reason.includes('only') && reason.includes('overflow')) return 'edge-fit'
  if (reason.includes('segment sum drifts')) return 'shaping-context'
  if (spanProbeReliable && mismatch === null) return 'diagnostic-sensitivity'
  if (quoteOrPunctuationRe.test(mismatch.deltaText)) return 'glue-policy'
  if (mismatch != null) return 'boundary-discovery'
  return 'unknown'
}
```

### 3.4 性能测试方法论

**基准测试结构**：
```typescript
function bench(fn: (repeatIndex: number) => void, sampleRepeats = 1, warmup = 2, runs = 10): number {
  // Warmup 阶段
  for (let i = 0; i < warmup; i++) runRepeated()
  // 测量阶段
  const times: number[] = []
  for (let i = 0; i < runs; i++) {
    const t0 = performance.now()
    runRepeated()
    times.push((performance.now() - t0) / sampleRepeats)
  }
  return median(times)
}
```

**防优化技术**：
- 所有基准测试使用 `sink` 变量累积结果，防止 V8 死代码消除
- 结果写入 `document.body.dataset` 确保存活

### 3.5 共享模块

#### `shared/navigation-state.ts`

**职责**：URL hash 状态编解码

**关键函数**：
```typescript
buildNavigationPhaseHash(phase: NavigationPhase, requestId?: string): string
buildNavigationReportHash(report: unknown): string
readNavigationReportText(urlOrHash: string): string
readNavigationPhaseState(urlOrHash: string): NavigationPhaseState | null
```

---

## 4. 关键代码路径与文件引用

### 4.1 准确性检查流程

```
accuracy-check.ts
  └── browser-automation.ts
        ├── createBrowserSession(browser)
        │     ├── createChromeSession() → AppleScript
        │     ├── createSafariSession() → AppleScript  
        │     └── createFirefoxSession() → WebSocket BiDi
        ├── ensurePageServer() → spawn bun server
        └── loadHashReport() / loadPostedReport()
              └── shared/navigation-state.ts
  └── report-server.ts (for --full mode)
  └── pages/accuracy.ts (browser-side)
        └── src/layout.ts (prepare, layout)
```

### 4.2 语料库检查流程

```
corpus-check.ts
  └── browser-automation.ts
  └── pages/corpus.ts
        ├── prepareWithSegments() → src/layout.ts
        ├── layout() → src/layout.ts
        ├── layoutWithLines() → src/layout.ts
        └── pages/diagnostic-utils.ts
              ├── getDiagnosticUnits()
              ├── measureCanvasTextWidth()
              └── measureDomTextWidth()
```

### 4.3 性能基准流程

```
benchmark-check.ts
  └── browser-automation.ts
  └── pages/benchmark.ts
        ├── prepare(), layout() → src/layout.ts
        ├── layoutWithLines(), walkLineRanges(), layoutNextLine() → src/layout.ts
        └── profilePrepare() → src/layout.ts
```

### 4.4 仪表板生成流程

```
status-dashboard.ts
  ├── accuracy/chrome.json
  ├── accuracy/safari.json
  ├── accuracy/firefox.json
  ├── benchmarks/chrome.json
  └── benchmarks/safari.json
  └── → status/dashboard.json

corpus-status.ts
  ├── corpora/representative.json
  ├── corpora/chrome-sampled.json
  ├── corpora/chrome-step10.json
  └── accuracy/*.json
  └── → corpora/dashboard.json
```

---

## 5. 依赖与外部交互

### 5.1 运行时依赖

| 依赖 | 用途 | 来源 |
|------|------|------|
| Bun | 脚本执行、TypeScript 直接运行、spawn 子进程 | 系统安装 |
| osascript | macOS 浏览器自动化（Chrome/Safari） | 系统自带 |
| Firefox | 浏览器测试 | `/Applications/Firefox.app` |
| Chrome | 浏览器测试 | `/Applications/Google Chrome.app` |
| Safari | 浏览器测试 | 系统自带 |

### 5.2 项目内部依赖

| 被依赖模块 | 用途 |
|------------|------|
| `src/layout.ts` | 核心布局 API（prepare, layout, layoutWithLines 等） |
| `src/test-data.ts` | 准确性测试用的共享文本语料 |
| `pages/*.ts` | 浏览器端测试页面逻辑 |
| `pages/diagnostic-utils.ts` | 诊断工具函数（canvas/DOM 测量） |
| `pages/report-utils.ts` | 浏览器端报告发布 |
| `corpora/sources.json` | 语料库元数据 |
| `corpora/*.txt` | 多语言长文本语料 |

### 5.3 输出产物

| 输出文件 | 生成脚本 | 用途 |
|----------|----------|------|
| `accuracy/*.json` | `accuracy-check.ts --full` | 浏览器准确性快照 |
| `benchmarks/*.json` | `benchmark-check.ts` | 性能基准快照 |
| `corpora/representative.json` | `corpus-representative.ts` | 代表性语料库状态 |
| `corpora/chrome-sampled.json` | `corpus-sweep.ts --samples=9` | 采样扫描结果 |
| `corpora/chrome-step10.json` | `corpus-sweep.ts --step=10` | 步进扫描结果 |
| `corpora/dashboard.json` | `corpus-status.ts` | 语料库仪表板 |
| `status/dashboard.json` | `status-dashboard.ts` | 主状态仪表板 |
| `site/` | `build-demo-site.ts` | GitHub Pages 站点 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 浏览器自动化锁竞争
- **风险**：多进程同时请求同一浏览器会导致锁等待超时（默认 120s）
- **缓解**：锁文件包含 PID 和启动时间，支持死锁自修复
- **建议**：CI 环境中确保串行执行，或分配不同浏览器给不同任务

#### Firefox 代理复杂性
- **风险**：`accuracy-check.ts` 为 Firefox 启动代理服务器处理 IPv6 回环问题
- **代码位置**：`startProxyServer()` 函数（第 104-130 行）
- **建议**：评估是否可通过 Bun 服务器配置直接解决

#### 字体加载时序
- **风险**：`probe.ts` 和 `corpus.ts` 使用 `document.fonts.ready` 等待字体，但 Safari/旧浏览器可能不支持
- **代码**：`pages/probe.ts:493-496`
- **缓解**：有 fallback 直接调用 `init()`

#### 提取器敏感性
- **风险**：Span probe 和 Range 方法在某些脚本（泰语、阿拉伯语）下给出不同结果
- **影响**：诊断结果可能因方法选择而不同
- **缓解**：`corpus.ts` 自动检测并标记 `extractorSensitivity`

### 6.2 边界限制

#### 平台限制
- **macOS 独占**：Chrome/Safari 自动化依赖 AppleScript，仅能在 macOS 运行
- **Firefox 支持**：虽然使用 BiDi 协议理论上跨平台，但脚本路径硬编码 `/Applications/Firefox.app`

#### 浏览器限制
- **Safari 限制**：不支持 headless 模式，必须前台运行（`foreground: true`）
- **Chrome 限制**：后台标签页可能被节流，影响基准测试准确性

#### 语料库限制
- **字体依赖**：语料库测试依赖系统安装的特定字体（如 Hiragino Mincho ProN）
- **DPR 敏感**：高 DPR 屏幕可能影响 canvas 测量精度

### 6.3 改进建议

#### 架构改进

1. **跨平台浏览器自动化**
   - 评估 Playwright 或 Puppeteer 替代 AppleScript
   - 保持轻量级：仅使用 DevTools Protocol 子集
   - 优先级：低（当前 macOS 优先策略有效）

2. **报告传输优化**
   - 当前：Hash-based（小数据）+ POST-based（大数据）双模式
   - 建议：统一使用 BroadcastChannel 或 SharedWorker（同域场景）
   - 优先级：低（当前方案稳定）

3. **并发执行支持**
   - 当前：单浏览器锁确保串行
   - 建议：支持多浏览器实例并行（需独立 profile/port）
   - 优先级：中（可缩短 CI 时间）

#### 代码质量改进

1. **类型安全**
   - 部分 JSON 解析使用 `as T` 强制转换
   - 建议：使用 Zod 或 valibot 进行运行时验证
   - 文件：`corpus-status.ts`, `status-dashboard.ts`

2. **错误处理**
   - 部分 AppleScript 错误仅做 best-effort 处理
   - 建议：增加更详细的错误分类和重试策略
   - 文件：`browser-automation.ts:52-58`

3. **配置集中化**
   - 当前：超时、端口等配置分散在各脚本
   - 建议：引入 `scripts/config.ts` 集中管理
   - 优先级：低

#### 功能扩展

1. **移动端浏览器支持**
   - 当前：仅桌面 Chrome/Safari/Firefox
   - 建议：通过 remote debugging 支持 iOS Safari 和 Android Chrome
   - 优先级：中（对移动端准确性验证有价值）

2. **可视化报告**
   - 当前：JSON + 控制台输出
   - 建议：HTML 报告生成，包含差异可视化
   - 优先级：低

3. **增量测试**
   - 当前：全量扫描
   - 建议：基于 git diff 识别受影响的语料库/宽度组合
   - 优先级：中（可加速 PR 验证）

### 6.4 维护要点

1. **定期刷新快照**：当修改 `src/analysis.ts`, `src/measurement.ts`, `src/line-break.ts`, `src/layout.ts` 时，需刷新所有 JSON 快照

2. **字体矩阵维护**：新增语料库时需同步更新 `corpus-font-matrix.ts` 的 `FONT_MATRIX`

3. **语料库元数据**：`corpora/sources.json` 中的 `min_width`, `max_width` 影响扫描范围

4. **Node.js 兼容性**：脚本使用 Bun 特性（如 `Bun.file()`, `Bun.spawnSync()`），迁移到 Node 需 polyfill

---

## 附录：脚本索引表

| 脚本 | 行数 | 核心功能 | 依赖脚本 | 输出 |
|------|------|----------|----------|------|
| `accuracy-check.ts` | 259 | 浏览器准确性验证 | browser-automation.ts, report-server.ts | 控制台/JSON |
| `benchmark-check.ts` | 144 | 性能基准测试 | browser-automation.ts | 控制台/JSON |
| `browser-automation.ts` | 715 | 浏览器自动化核心 | shared/navigation-state.ts | - |
| `build-demo-site.ts` | 89 | 静态站点构建 | - | site/ |
| `corpus-check.ts` | 338 | 语料库单点诊断 | browser-automation.ts | 控制台 |
| `corpus-font-matrix.ts` | 515 | 字体矩阵测试 | browser-automation.ts, report-server.ts | 控制台/JSON |
| `corpus-representative.ts` | 239 | 代表性快照生成 | browser-automation.ts, report-server.ts | JSON |
| `corpus-status.ts` | 340 | 语料库仪表板 | - | JSON |
| `corpus-sweep.ts` | 362 | 语料库宽度扫描 | browser-automation.ts, report-server.ts | JSON |
| `corpus-taxonomy.ts` | 258 | 不匹配分类 | browser-automation.ts, report-server.ts | 控制台 |
| `gatsby-check.ts` | 33 | 兼容性别名 | corpus-check.ts | - |
| `gatsby-sweep.ts` | 33 | 兼容性别名 | corpus-sweep.ts | - |
| `package-smoke-test.ts` | 191 | 包发布验证 | - | 控制台 |
| `pre-wrap-check.ts` | 267 | pre-wrap 模式验证 | browser-automation.ts | 控制台 |
| `probe-check.ts` | 154 | 单点探针测试 | browser-automation.ts | 控制台 |
| `report-server.ts` | 85 | POST 报告服务器 | browser-automation.ts | - |
| `status-dashboard.ts` | 115 | 主仪表板生成 | - | JSON |
