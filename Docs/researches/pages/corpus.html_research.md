# corpus.html 研究文档

## 场景与职责

`pages/corpus.html` 是 Pretext 库的多语言语料库压力测试页面容器。它提供了一个交互式界面，用于可视化验证 Pretext 在长文本、多语言场景下的布局准确性。与 `accuracy.html` 的批量扫描不同，此页面专注于深度分析单个语料在特定宽度下的行为。

核心场景：
- **语料可视化**：交互式查看 17 种语言的语料库渲染效果
- **宽度扫描**：通过滑块或 URL 参数批量测试多个宽度
- **诊断分析**：深度分析首行不匹配的原因（断点位置、宽度漂移等）
- **提取器验证**：对比 span-probe 和 range 两种浏览器行提取方法

## 功能点目的

1. **语料选择器**：下拉菜单选择 17 种语言的语料库
2. **宽度控制**：滑块控制容器宽度（300-900px）
3. **统计面板**：显示 Pretext vs DOM 的耗时、高度、行数对比
4. **文本渲染区**：`#book` 元素展示实际渲染的文本内容

## 具体技术实现

### 页面结构

```html
<body>
  <h1>Corpus Stress Demo</h1>
  <p class="sub">Reusable long-form canary for multilingual browser-layout sweeps.</p>
  
  <div class="controls">
    <label>
      <span>Corpus:</span>
      <select id="corpus"></select>
    </label>
    <span>Width:</span>
    <input type="range" id="slider" min="300" max="900" value="600">
    <span class="val" id="val">600px</span>
  </div>
  
  <div class="stats" id="stats"></div>
  <div id="book"></div>
  
  <script type="module" src="./corpus.ts"></script>
</body>
```

### 样式系统

| 选择器 | 用途 |
|--------|------|
| `#book` | 文本渲染区，40px 内边距，圆角背景 |
| `.controls` | 控制面板，flex 布局 |
| `#corpus` | 语料选择下拉，深色主题 |
| `#slider` | 宽度滑块，flex 自适应 |
| `.stats` | 统计信息，等宽字体 |

### URL 参数支持（由 corpus.ts 处理）

- `?id=<corpusId>`：指定初始语料
- `?width=<number>`：指定初始宽度
- `?widths=a,b,c`：批量扫描模式
- `?diagnostic=full|light`：诊断详细程度
- `?method=span|range`：强制指定提取方法
- `?font=<font>`：覆盖默认字体
- `?lineHeight=<number>`：覆盖默认行高
- `?sliceStart=<n>&sliceEnd=<n>`：文本切片
- `?reportEndpoint=<url>`：报告提交端点

## 关键代码路径与文件引用

### 依赖关系

```
corpus.html
├── corpus.ts (主逻辑)
│   ├── ../src/layout.ts
│   ├── ./diagnostic-utils.ts
│   ├── ./report-utils.ts
│   └── ../corpora/sources.json
└── 样式内联定义
```

### 相关文件

| 文件 | 关系 | 说明 |
|------|------|------|
| `corpus.ts` | 直接依赖 | 语料测试逻辑 |
| `corpora/sources.json` | 间接依赖 | 语料元数据 |
| `corpora/*.txt` | 间接依赖 | 语料文本内容 |

## 依赖与外部交互

### 浏览器 API

- **DOM API**：动态创建选择器选项、更新滑块值
- **CSS**：`white-space: normal`, `word-wrap: break-word`

### 自动化集成

支持通过 URL 参数进行自动化测试：
- 批量宽度扫描：`?widths=300,400,500,600`
- 特定语料：`?id=ja-rashomon`
- 报告提交：`?reportEndpoint=http://localhost:3000/report`

## 风险、边界与改进建议

### 已知限制

1. **字体加载**：依赖 `document.fonts.ready`，某些浏览器支持不完整
2. **大语料性能**：超大文本（如 25 万字符的阿拉伯语）可能影响交互响应
3. **RTL 布局**：需要正确设置 `dir="rtl"` 属性

### 改进建议

1. **虚拟滚动**：超大语料时启用虚拟滚动
2. **差异高亮**：在行级显示 Pretext 与浏览器的差异位置
3. **导出功能**：导出当前视图为图片或 PDF
4. **并排对比**：左右分栏显示 Pretext 预测 vs DOM 实际
