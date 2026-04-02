# benchmark.html 研究文档

## 场景与职责

`pages/benchmark.html` 是 Pretext 库的性能基准测试页面容器。它提供了一个独立的浏览器环境，用于量化测量 Pretext 库相对于 DOM 原生布局的性能优势。该页面是项目性能验证和营销主张（"比 DOM 快 1500 倍"）的技术基础。

核心场景：
- **性能回归检测**：验证算法变更是否引入性能下降
- **竞品对比**：与 DOM batch/interleaved 模式进行 head-to-head 对比
- **多语言性能**：测试 CJK、阿拉伯语等复杂脚本的布局性能
- **富 API 基准**：评估 `layoutWithLines`、`walkLineRanges`、`layoutNextLine` 等高级 API 的性能特征

## 功能点目的

1. **性能测试容器**：加载并执行 `benchmark.ts` 中的多维度基准测试
2. **结果可视化**：通过表格展示各 API 的延迟（ms）和相对速度倍数
3. **状态反馈**：显示测试进度（"Benchmarking prepare()..." 等）

## 具体技术实现

### 页面结构

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Pretext — Benchmark</title>
  <!-- 深色主题样式 -->
</head>
<body>
  <h1>Pretext — Benchmark</h1>
  <p class="sub">Median ms per 500-text batch...</p>
  <div id="root"><p>Running benchmarks...</p></div>
  <script type="module" src="./benchmark.ts"></script>
</body>
</html>
```

### 样式系统

| 选择器 | 颜色 | 含义 |
|--------|------|------|
| `.fast td` | #81c784 (绿色) | 性能优秀（< 1.5× 最快） |
| `.mid td` | #ffcc80 (橙色) | 中等性能（1.5× - 10×） |
| `.slow td` | #ef9a9a (红色) | 性能较差（> 10×） |
| `.big` | #81c784 | 主指标高亮 |
| `.sep` | #444 | 分隔符 |

### 测试说明

页面副标题明确测试方法论：
> "Median ms per 500-text batch. Layout is normalized repeated throughput; DOM paths measure one real 400px → 300px resize batch."

这意味着：
- **Pretext layout()**: 重复测量 200 次取中位数，循环多个宽度以稳定亚毫秒计时
- **DOM 路径**: 测量一次真实的 400px→300px 调整大小批次（写入全部，然后读取全部）

## 关键代码路径与文件引用

### 依赖关系

```
benchmark.html
├── benchmark.ts (主逻辑)
│   ├── ../src/layout.ts (prepare, layout, layoutWithLines, walkLineRanges, layoutNextLine)
│   ├── ./report-utils.ts (导航报告)
│   └── ../corpora/*.txt (多语言语料)
└── 样式内联定义
```

### 相关文件

| 文件 | 关系 | 说明 |
|------|------|------|
| `benchmark.ts` | 直接依赖 | 基准测试逻辑实现 |
| `src/layout.ts` | 间接依赖 | 核心布局 API |
| `corpora/*.txt` | 间接依赖 | 多语言长文本语料 |
| `benchmarks/*.json` | 输出对比 | 检入的性能快照 |

## 依赖与外部交互

### 浏览器 API

- **DOM API**：创建可见容器进行 DOM 基准测试
- **Performance API**：`performance.now()` 用于高精度计时
- **Animation API**：`requestAnimationFrame` 用于状态更新

### 自动化集成

通过 URL 参数支持自动化：
- `?report=1`：启用报告模式
- `?requestId=<id>`：关联自动化测试请求

## 风险、边界与改进建议

### 已知限制

1. **前台执行要求**：基准测试必须在前台标签页执行，后台标签页会被节流
2. **系统负载敏感**：其他进程负载可能影响计时结果
3. **浏览器差异**：不同浏览器的计时精度和节流策略不同

### 测试方法论注意事项

| 问题 | 当前处理 |
|------|----------|
| 亚毫秒计时 | 重复执行 200 次取平均 |
| 垃圾回收干扰 | 10 次运行取中位数 |
| 编译器优化 | 使用 sink 变量防止优化消除 |
| 预热 | 2 次预热运行排除 JIT 编译时间 |

### 改进建议

1. **统计置信度**：当前仅使用中位数，可添加标准差、置信区间
2. **内存基准**：添加内存使用测量（`performance.memory`）
3. **帧率影响**：测量布局期间的帧率下降
4. **真实场景**：添加 React/Vue 组件集成测试
5. **可视化图表**：将结果渲染为趋势图而非纯表格
