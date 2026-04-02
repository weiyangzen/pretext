# pages/probe.html 研究文档

## 场景与职责

`probe.html` 是 Pretext 项目的**文本探针（Text Probe）**诊断工具的页面容器。它提供一个简洁的深色主题界面，用于单片段（single-snippet）浏览器断行测试。与 `gatsby.ts` 的完整语料测试不同，probe 专注于**快速、针对性的文本片段验证**。

该页面是浏览器自动化测试和人工调试的入口点，支持：
1. **参数化测试**：通过 URL 参数指定任意文本、宽度、字体
2. **双模式浏览器行提取**：Range 方法和 Span 方法
3. **RTL/多语言支持**：支持阿拉伯语等从右到左文本

## 功能点目的

### 1. 单片段快速测试
- 无需准备完整语料文件
- 通过 URL 参数直接传入测试文本
- 即时显示 Pretext 预测 vs 浏览器实际结果

### 2. 可视化反馈
- 深色主题界面，适合长时间调试
- 实时统计信息显示（宽度、行数、差异）
- 文本渲染区域清晰展示

### 3. 浏览器行提取方法对比
- 支持 `method=range`（默认）和 `method=span` 两种提取方式
- 自动检测提取器敏感性（extractor sensitivity）
- 帮助识别浏览器特定的行提取问题

## 具体技术实现

### 页面结构
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Pretext — Text Probe</title>
  <!-- 内联 CSS：深色主题 -->
</head>
<body>
  <h1>Text Probe</h1>
  <p class="sub">Single-snippet browser line-break probe.</p>
  <div class="stats" id="stats">Loading...</div>
  <div id="book"></div>
  <script type="module" src="./probe.ts"></script>
</body>
</html>
```

### 样式设计
```css
/* 深色主题配色 */
body {
  background: #0a0a0a;
  color: #e0e0e0;
}

#book {
  background: #1a1a1a;
  color: #d4d4d4;
  padding: 40px;
  border-radius: 8px;
}

/* 文本渲染样式 */
white-space: normal;
word-wrap: break-word;
overflow-wrap: break-word;
```

### 关键 DOM 元素
| 元素 ID | 用途 |
|---------|------|
| `stats` | 显示测试统计信息（宽度、行数、差异） |
| `book` | 文本渲染和目标区域 |

## 关键代码路径与文件引用

### 文件关系
```
pages/
├── probe.html          # 本文件（页面容器）
├── probe.ts            # 核心逻辑脚本（模块导入）
└── diagnostic-utils.ts # 共享诊断工具函数
```

### 脚本加载
```html
<script type="module" src="./probe.ts"></script>
```

- 使用 ES Module 加载
- 相对路径 `./probe.ts` 指向同目录下的 TypeScript 文件
- 构建/开发服务器负责 TypeScript 转译

### 依赖的 probe.ts 接口
`probe.html` 作为容器，依赖 `probe.ts` 提供的功能：
- URL 参数解析（`text`, `width`, `font`, `lineHeight`, `dir`, `lang`, `method`）
- Pretext 布局计算
- 浏览器行提取（Range/Span 双方法）
- 报告生成与发布

## 依赖与外部交互

### 无直接外部依赖
该 HTML 文件本身：
- 不直接导入 JavaScript（由 probe.ts 处理）
- 不加载外部 CSS（内联样式）
- 不依赖外部字体（使用系统字体）

### 系统字体栈
```css
font-family: system-ui, -apple-system, sans-serif;  /* 界面 */
font-family: "SF Mono", monospace;                  /* 统计信息 */
```

### 浏览器兼容性
| 特性 | 兼容性 |
|------|--------|
| CSS Flexbox | 所有现代浏览器 |
| CSS `box-sizing: border-box` | 所有现代浏览器 |
| ES Modules | 所有现代浏览器 |

## 风险、边界与改进建议

### 当前风险

1. **无降级处理**
   - 如果 `probe.ts` 加载失败，页面仅显示 "Loading..."
   - 没有错误状态显示
   - **建议**：添加加载超时和错误提示

2. **固定内边距**
   - 硬编码 `padding: 40px` 与 `probe.ts` 中的 `PADDING` 常量耦合
   - 两边不一致会导致测量偏差
   - **建议**：通过 CSS 自定义属性或数据属性共享配置

3. **无 viewport 适配**
   - 固定宽度布局，在移动设备上可能溢出
   - **建议**：添加响应式断点或缩放控制

### 边界情况

1. **空文本**：`probe.ts` 处理空文本情况
2. **超长文本**：容器宽度固定，超长文本会换行
3. **特殊字符**：依赖 `probe.ts` 的编码处理

### 改进建议

1. **添加加载状态**
   ```html
   <script>
     // 加载超时处理
     setTimeout(() => {
       const stats = document.getElementById('stats');
       if (stats.textContent === 'Loading...') {
         stats.textContent = 'Error: Failed to load probe module';
       }
     }, 10000);
   </script>
   ```

2. **响应式布局**
   ```css
   @media (max-width: 600px) {
     #book {
       padding: 20px;
       border-radius: 0;
     }
   }
   ```

3. **主题切换**
   - 添加亮色主题选项
   - 使用 `prefers-color-scheme` 媒体查询

4. **配置同步**
   ```html
   <!-- 使用 CSS 自定义属性 -->
   <style>
     :root {
       --probe-padding: 40px;
     }
     #book {
       padding: var(--probe-padding);
     }
   </style>
   ```

5. **SEO 和元数据**
   ```html
   <meta name="description" content="Pretext text layout diagnostic probe">
   <meta name="viewport" content="width=device-width, initial-scale=1.0">
   ```
