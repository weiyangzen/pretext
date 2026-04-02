# accuracy.html 研究文档

## 场景与职责

`pages/accuracy.html` 是 Pretext 库的浏览器精度测试页面（Accuracy Sweep）的 HTML 容器。它提供了一个独立的浏览器环境，用于对比 Pretext 库的高度预测与浏览器 DOM 实际渲染结果之间的差异。这是项目质量保证体系的核心组成部分，用于验证文本布局算法在不同字体、字号和容器宽度下的准确性。

该页面主要服务于：
- **自动化精度测试**：通过浏览器自动化脚本加载此页面，执行跨字体/尺寸/宽度的全面扫描
- **开发调试**：开发者可直接在浏览器中打开查看可视化测试结果
- **回归测试**：与 `accuracy/chrome.json`、`accuracy/safari.json`、`accuracy/firefox.json` 等检入快照对比，检测算法变更引入的精度回归

## 功能点目的

1. **测试环境容器**：提供独立的 HTML 页面环境，加载并执行 `accuracy.ts` 中的测试逻辑
2. **视觉反馈**：通过深色主题的表格展示测试结果，高亮显示不匹配项（over/under）
3. **状态指示**：显示测试加载状态，直至 TypeScript 模块完成初始化

## 具体技术实现

### 页面结构

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Pretext — Accuracy Sweep</title>
  <!-- 深色主题样式定义 -->
</head>
<body>
  <h1>Pretext — Accuracy Sweep</h1>
  <p class="sub">Compares library height predictions against DOM ground truth...</p>
  <div id="root"><p>Loading...</p></div>
  <script type="module" src="./accuracy.ts"></script>
</body>
</html>
```

### 样式系统

| 选择器 | 用途 |
|--------|------|
| `.over td` | 预测高度 > 实际高度（红色 #ef9a9a）|
| `.under td` | 预测高度 < 实际高度（橙色 #ffcc80）|
| `.perfect` | 全部通过时的成功提示（绿色 #81c784）|
| `.text` | 文本片段展示，支持 `word-break: break-all` |
| `col.num` / `col.text` | 表格列宽控制 |

### URL 参数支持（由 accuracy.ts 处理）

- `?requestId=<id>`：关联自动化测试请求
- `?full=1`：包含完整的行数据（rows）在报告中
- `?reportEndpoint=<url>`：大型报告的 POST 端点

## 关键代码路径与文件引用

### 依赖关系

```
accuracy.html
├── accuracy.ts (主逻辑)
│   ├── ../src/layout.ts (prepareWithSegments, layout, layoutWithLines)
│   ├── ./diagnostic-utils.ts (getDiagnosticUnits)
│   ├── ./report-utils.ts (publishNavigationPhase, publishNavigationReport)
│   └── ../src/test-data.ts (TEXTS, SIZES, WIDTHS)
└── 样式内联定义
```

### 相关文件

| 文件 | 关系 | 说明 |
|------|------|------|
| `accuracy.ts` | 直接依赖 | 测试逻辑实现 |
| `src/layout.ts` | 间接依赖 | 核心布局 API |
| `src/test-data.ts` | 间接依赖 | 测试文本数据集 |
| `accuracy/*.json` | 输出对比 | 检入的浏览器快照 |

## 依赖与外部交互

### 浏览器 API

- **DOM API**：`document.getElementById`, `document.createElement`, `getBoundingClientRect`
- **Range API**：`document.createRange`, `range.setStart/setEnd`, `range.getClientRects`（在 accuracy.ts 中使用）
- **Canvas API**：`canvas.getContext('2d')`, `ctx.measureText`（在 measurement.ts 中）

### 自动化集成

页面通过 URL hash 与自动化框架通信：
- 加载阶段：`#phase=loading&requestId=xxx`
- 测量阶段：`#phase=measuring&requestId=xxx`
- 报告阶段：`#phase=posting&requestId=xxx` 或 `#report=...`

## 风险、边界与改进建议

### 已知限制

1. **字体依赖性**：测试使用系统字体回退链，不同操作系统可能解析到不同字体，导致结果不一致
2. **Emoji 修正**：macOS 上小字号（<24px）的 Apple Color Emoji 需要特殊修正
3. **system-ui 字体**：canvas 和 DOM 可能解析到不同光学变体（macOS 已知问题）

### 边界情况

- 空文本、纯空白、超长无空格单词
- 混合 LTR/RTL 文本
- CJK 字符的禁则处理（kinsoku）
- 软连字符（soft hyphen）的显示/隐藏

### 改进建议

1. **字体加载等待**：当前依赖 `document.fonts.ready`，但某些浏览器可能不完全支持
2. **响应式布局**：当前为固定布局，可考虑添加移动端适配
3. **实时过滤**：添加按字体、尺寸、宽度过滤不匹配项的交互控件
4. **历史趋势**：集成历史快照对比，显示精度随时间的变化趋势
