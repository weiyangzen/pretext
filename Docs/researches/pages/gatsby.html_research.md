# gatsby.html 研究文档

## 场景与职责

`pages/gatsby.html` 是 Pretext 库的《了不起的盖茨比》长文本演示页面容器。它提供了一个交互式界面，用于可视化验证 Pretext 在英语长文本（27.9 万字符）上的布局准确性。这是项目的主要演示页面之一，也是英语语料测试的入口点。

核心场景：
- **英语长文本验证**：使用《了不起的盖茨比》开篇作为标准英语语料
- **交互式演示**：通过滑块实时调整宽度，观察 Pretext 与 DOM 的同步
- **兼容性别名**：`gatsby-check` 和 `gatsby-sweep` 脚本的路由目标
- **性能感知**：显示 Pretext vs DOM 的耗时对比

## 功能点目的

1. **经典文学演示**：使用《了不起的盖茨比》作为广为人知的英语文本样本
2. **实时交互**：滑块控制容器宽度（300-900px），即时更新布局
3. **性能对比**：显示 Pretext 计算耗时 vs DOM 重排耗时
4. **统计面板**：高度差异、行数对比等关键指标

## 具体技术实现

### 页面结构

```html
<body>
  <h1>The Great Gatsby — Resize Demo</h1>
  <p class="sub">Pretext vs DOM reflow on a full novel. Drag slider to resize.</p>
  
  <div class="controls">
    <span>Width:</span>
    <input type="range" id="slider" min="300" max="900" value="600">
    <span class="val" id="val">600px</span>
  </div>
  
  <div class="stats" id="stats"></div>
  <div id="book"></div>
  
  <script type="module" src="./gatsby.ts"></script>
</body>
```

### 样式系统

| 选择器 | 用途 |
|--------|------|
| `#book` | 文本渲染区，Georgia 字体，40px 内边距 |
| `.controls` | 控制面板，flex 布局 |
| `#slider` | 宽度滑块，flex 自适应 |
| `.stats` | 统计信息，等宽字体，灰色 |
| `body` | 深色主题，居中布局 |

### 固定配置

```css
#book {
  font: 16px/26px Georgia, "Times New Roman", serif;
  color: #d4d4d4;
  background: #1a1a1a;
  padding: 40px;
  border-radius: 8px;
  white-space: normal;
  word-wrap: break-word;
  overflow-wrap: break-word;
}
```

### URL 参数支持（由 gatsby.ts 处理）

- `?report=1`：启用详细报告模式
- `?diagnostic=full|light`：诊断详细程度
- `?requestId=<id>`：自动化测试标识
- `?reportEndpoint=<url>`：报告提交端点
- `?widths=a,b,c`：批量扫描模式
- `?width=<number>`：指定初始宽度

## 关键代码路径与文件引用

### 依赖关系

```
gatsby.html
├── gatsby.ts (主逻辑)
│   ├── ../src/layout.ts
│   ├── ./diagnostic-utils.ts
│   ├── ./report-utils.ts
│   └── ../corpora/en-gatsby-opening.txt
└── 样式内联定义
```

### 相关文件

| 文件 | 关系 | 说明 |
|------|------|------|
| `gatsby.ts` | 直接依赖 | 测试逻辑实现 |
| `corpora/en-gatsby-opening.txt` | 间接依赖 | 盖茨比文本内容（27.9万字符） |
| `corpus.ts` | 功能相似 | 多语言语料通用实现 |

## 依赖与外部交互

### 浏览器 API

- **DOM API**：滑块事件监听、样式更新
- **CSS**：`white-space: normal`, `word-wrap: break-word`

### 自动化集成

作为兼容性别名入口：
- `gatsby-check` → 单宽度检查
- `gatsby-sweep` → 批量宽度扫描

这些脚本现在路由到英语语料入口，与 `corpus.html?id=en-gatsby-opening` 等价。

## 风险、边界与改进建议

### 已知限制

1. **固定字体**：使用 Georgia 字体，不支持通过 URL 参数覆盖
2. **固定文本**：仅支持《了不起的盖茨比》开篇
3. **无语料切换**：不像 `corpus.html` 支持多语料选择

### 改进建议

1. **功能合并**：考虑将功能合并到 `corpus.html`，减少维护负担
2. **字体覆盖**：添加 `?font=` 参数支持
3. **文本选择**：支持选择其他英语语料
4. **历史对比**：显示与历史快照的对比

### 与 corpus.html 的关系

| 特性 | gatsby.html | corpus.html |
|------|-------------|-------------|
| 语料数量 | 1（固定） | 17（可选） |
| 字体配置 | 固定 | 可覆盖 |
| 诊断深度 | 完整 | 可配置 |
| 代码复用 | 独立实现 | 通用实现 |
| 维护状态 | 兼容性保留 | 主要发展 |

`gatsby.html` 目前作为兼容性别名保留，新功能主要在 `corpus.html` 中实现。
