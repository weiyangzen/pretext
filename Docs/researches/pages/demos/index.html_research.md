# index.html 研究文档

## 场景与职责

`index.html` 是 Pretext 演示站点的入口页面（Landing Page），作为 GitHub Pages 站点根页面使用。它提供了一个简洁、美观的导航界面，展示所有可用的 Pretext 演示，让用户能够快速了解库的功能范围并跳转到感兴趣的演示。

该页面是一个纯静态 HTML 文件，不依赖 JavaScript，体现了渐进增强的设计理念。

## 功能点目的

### 1. 演示导航
提供所有 Pretext 演示的入口链接，每个演示以卡片形式展示，包含：
- 演示名称（标题）
- 简短描述（功能说明）

### 2. 品牌展示
- 展示 Pretext 项目名称
- 提供简洁的项目介绍
- 使用一致的视觉设计系统

### 3. 响应式布局
- 桌面端：双列网格布局
- 移动端：单列堆叠布局
- 自适应内容宽度

## 具体技术实现

### 视觉设计系统

页面使用 CSS 自定义属性（变量）定义设计令牌：

```css
:root {
  color-scheme: light;
  --page: #f5f1ea;        /* 页面背景色 */
  --panel: #fffdf8;       /* 卡片背景色 */
  --ink: #201b18;         /* 主文本色 */
  --muted: #6d645d;       /* 次要文本色 */
  --rule: #d8cec3;        /* 边框色 */
  --accent: #955f3b;      /* 强调色 */
}
```

### 布局结构

```
body
└── main.page
    ├── .eyebrow          /* 项目标签 */
    ├── h1                /* 页面标题 */
    ├── .intro            /* 介绍文本 */
    └── section.grid      /* 演示卡片网格 */
        └── a.card × 8    /* 演示链接卡片 */
```

### 响应式断点

```css
@media (max-width: 760px) {
  .page {
    width: min(100vw - 20px, 940px);
    padding-top: 22px;
  }
  .grid {
    grid-template-columns: 1fr;  /* 单列 */
  }
}
```

### 卡片交互效果

悬停状态使用 CSS 过渡：
```css
.card {
  transition: transform 160ms ease, 
              box-shadow 160ms ease, 
              border-color 160ms ease;
}
.card:hover {
  transform: translateY(-1px);
  border-color: color-mix(in srgb, var(--accent) 35%, var(--rule) 65%);
  box-shadow: 0 22px 44px rgb(54 40 23 / 0.12);
}
```

## 关键代码路径与文件引用

### 演示链接映射

| 链接路径 | 演示名称 | 描述 |
|---------|---------|------|
| `/demos/accordion` | Accordion | 手风琴组件，展示文本高度计算 |
| `/demos/bubbles` | Bubbles | 消息气泡，展示紧凑多行布局 |
| `/demos/dynamic-layout` | Dynamic Layout | 固定高度编辑排版，障碍物避让 |
| `/demos/variable-typographic-ascii` | Variable Typographic ASCII | 可变字体 ASCII 艺术 |
| `/demos/editorial-engine` | Editorial Engine | 动画球体、实时重排、引言块 |
| `/demos/justification-comparison` | Justification Comparison | 对齐算法对比（CSS/Greedy/Knuth-Plass） |
| `/demos/rich-note` | Rich Text | 富文本、代码片段、标签芯片 |
| `/demos/masonry` | Masonry | 瀑布流布局，高度预测 |

### 对应实现文件

每个演示链接对应同目录下的 TypeScript 文件：
- `accordion.ts`
- `bubbles.ts`
- `dynamic-layout.ts`
- `variable-typographic-ascii.ts`
- `editorial-engine.ts`
- `justification-comparison.ts` (+ `.model.ts`, `.ui.ts`, `.data.ts`)
- `rich-note.ts`
- `masonry.ts`

## 依赖与外部交互

### 外部依赖

该页面是纯 HTML/CSS，无 JavaScript 依赖。

### 字体依赖

使用系统字体栈：
- 主字体：Helvetica Neue, Helvetica, Arial, sans-serif
- 标题：Georgia, Times New Roman, serif
- 标签：SF Mono, ui-monospace, monospace

### 资源加载

- 无外部 CSS/JS 文件
- 无图片资源
- 依赖系统字体（无需网络加载）

## 风险、边界与改进建议

### 已知风险

1. **链接失效风险**
   - 硬编码的演示路径如果与实际文件不匹配，会导致 404
   - 建议：添加构建时链接验证

2. **浏览器兼容性**
   - 使用 `color-mix()` CSS 函数，在旧版浏览器（2023年前）可能不支持
   - 回退策略：浏览器会忽略不支持的属性，使用默认颜色

### 边界情况处理

| 边界情况 | 处理方式 |
|---------|---------|
| 极窄视口 | `min(940px, calc(100vw - 32px))` 确保最小边距 |
| 卡片文本溢出 | 描述文本限制行高，无固定高度 |
| 无 JavaScript | 页面完全可用，纯 HTML/CSS |

### 改进建议

1. **可访问性增强**
   - 添加 `aria-label` 描述每个演示的具体功能
   - 添加跳转到主内容的链接（Skip Link）
   - 确保颜色对比度符合 WCAG AA 标准

2. **SEO 优化**
   - 添加 meta description
   - 添加 Open Graph 标签
   - 添加结构化数据（JSON-LD）

3. **视觉增强**
   - 为每个卡片添加图标或缩略图预览
   - 添加演示标签（如 "性能"、"排版"、"交互"）
   - 添加难度指示器

4. **功能扩展**
   - 添加搜索/过滤功能
   - 添加排序选项（按名称、按复杂度）
   - 添加"新"或"推荐"标记

5. **深色模式支持**
   ```css
   @media (prefers-color-scheme: dark) {
     :root {
       --page: #1a1714;
       --panel: #252018;
       --ink: #f5f1ea;
       --muted: #a09888;
       --rule: #3d352c;
       --accent: #c47a4a;
     }
   }
   ```

### 维护注意事项

- 添加新演示时，需要同步更新此文件
- 建议将演示列表数据提取为 JSON，通过脚本生成 HTML
- 保持描述文本简洁（不超过两行）
