# rich-note.html 研究文档

## 场景与职责

`rich-note.html` 是 Rich Text 演示的页面容器，展示 Pretext 在富文本布局场景下的应用能力：

1. **混合内容布局**：普通文本、链接、代码片段和原子芯片（chips）的混合排版
2. **芯片不换行**：标签芯片作为原子单元，在文本流中保持完整
3. **动态宽度调整**：通过滑块实时调整文本宽度，观察布局自适应
4. **响应式设计**：适配不同视口大小

该演示模拟了现代应用中常见的富文本消息或笔记卡片布局。

## 功能点目的

### 1. 富文本元素混合

展示四种内联元素的自然混排：
- **普通文本**：主体内容
- **链接文本**：可点击的蓝色下划线文本
- **代码片段**：等宽字体、圆角背景的内联代码
- **原子芯片**：标签式 UI 元素（提及、状态、优先级、时间、计数）

### 2. 芯片原子性

芯片（chips）具有以下特性：
- **不换行**：芯片作为一个整体在文本流中移动
- **最小宽度**：即使空间紧张也保持完整显示
- **视觉区分**：不同语义类型使用不同颜色主题

### 3. 实时宽度调整

通过滑块控件（260px - 760px）实时调整文本区域宽度，观察：
- 文本如何重新换行
- 芯片如何"跳跃"到新行以保持完整
- 不同宽度下的阅读体验

### 4. 视觉设计

- 笔记卡片式布局（圆角、阴影、边框）
- 温暖色调（米色、棕色系）
- 清晰的视觉层次

## 具体技术实现

### 页面结构

```html
body
└── main.page
    ├── section.intro       /* 标题和介绍 */
    ├── .toolbar            /* 宽度滑块控制 */
    └── section.preview     /* 预览区域 */
        └── article.note-shell   /* 笔记卡片外壳 */
            └── .note-body       /* 笔记内容区 */
                └── .line-row × n   /* 行容器 */
                    └── .frag / .chip  /* 文本片段和芯片 */
```

### CSS 自定义属性

```css
:root {
  color-scheme: light;
  --page: #f5f1ea;           /* 页面背景 */
  --panel: #fffdf8;          /* 卡片背景 */
  --ink: #201b18;            /* 主文本色 */
  --muted: #6d645d;          /* 次要文本色 */
  --rule: #d8cec3;           /* 边框色 */
  --accent: #955f3b;         /* 强调色/链接色 */
  --note-width: 556px;       /* 卡片宽度（动态更新） */
  --note-content-width: 516px;  /* 内容区宽度（动态更新） */
}
```

### 行布局

```css
.line-row {
  position: absolute;        /* 精确定位每行 */
  left: 0;
  display: flex;
  align-items: baseline;
  flex-wrap: nowrap;         /* 行内不换行 */
  gap: 0;
  width: max-content;        /* 根据内容自动扩展 */
}
```

使用绝对定位而非流式布局，因为：
- Pretext 计算的是精确的行位置
- 需要精确控制每个片段的位置
- 芯片作为原子单元需要特殊处理

### 文本片段样式

```css
.frag {
  display: inline-block;
  white-space: pre;          /* 保留空白 */
  font: 500 17px/1 "Helvetica Neue", Helvetica, Arial, sans-serif;
  color: var(--ink);
  vertical-align: baseline;
}

.frag--link {
  color: var(--accent);
  text-decoration: underline;
  text-decoration-thickness: 1px;
  text-underline-offset: 0.14em;
}

.frag--code {
  padding: 2px 7px 3px;
  border-radius: 9px;
  background: rgba(17, 31, 43, 0.08);
  font: 600 14px/1 "SF Mono", ui-monospace, Menlo, monospace;
}
```

### 芯片样式

```css
.chip {
  display: inline-flex;
  align-items: center;
  padding: 0 10px;
  min-height: 24px;
  border-radius: 999px;      /* 完全圆角 */
  font: 700 12px/1 "Helvetica Neue", Helvetica, Arial, sans-serif;
  white-space: nowrap;       /* 芯片内不换行 */
  border: 1px solid transparent;
  vertical-align: baseline;
  transform: translateY(-1px);  /* 视觉对齐微调 */
}

.chip--mention { background: rgba(21, 90, 136, 0.12); color: #155a88; }
.chip--status { background: rgba(196, 129, 20, 0.12); color: #916207; }
.chip--priority { background: rgba(176, 44, 44, 0.1); color: #8e2323; }
.chip--time { background: rgba(70, 118, 77, 0.11); color: #355f38; }
.chip--count { background: rgba(67, 57, 122, 0.1); color: #483e83; }
```

### 响应式断点

```css
@media (max-width: 900px) {
  .preview {
    justify-content: center;
  }
}

@media (max-width: 640px) {
  .page {
    width: min(100vw - 20px, 940px);
    padding-top: 22px;
  }
  h1 { font-size: 30px; }
  .toolbar {
    grid-template-columns: 1fr;  /* 垂直堆叠 */
    justify-items: stretch;
    border-radius: 22px;
  }
  .note-shell {
    border-radius: 24px;
    padding-inline: 14px;
  }
}
```

### 控制栏布局

```css
.toolbar {
  display: grid;
  grid-template-columns: auto minmax(0, 1fr) auto;  /* 标签 | 滑块 | 值 */
  gap: 14px;
  align-items: center;
}
```

## 关键代码路径与文件引用

### 脚本加载

```html
<script type="module" src="./rich-note.ts"></script>
```

实现文件：`rich-note.ts`

### 元素 ID 映射

| ID | 用途 | 消费者 |
|----|------|--------|
| `width-slider` | 宽度滑块 | `rich-note.ts` |
| `width-value` | 宽度显示 | `rich-note.ts` |
| `note-shell` | 笔记卡片外壳 | `rich-note.ts`（设置 CSS 变量） |
| `note-body` | 笔记内容容器 | `rich-note.ts`（渲染行） |

### CSS 类名约定

| 类名 | 用途 |
|-----|------|
| `.note-shell` | 卡片外壳（边框、阴影、圆角） |
| `.note-body` | 内容区（相对定位，作为行定位参考） |
| `.line-row` | 单行容器（绝对定位） |
| `.frag` | 普通文本片段 |
| `.frag--link` | 链接文本 |
| `.frag--code` | 代码片段 |
| `.chip` | 芯片基础样式 |
| `.chip--mention` | 提及芯片（@用户） |
| `.chip--status` | 状态芯片 |
| `.chip--priority` | 优先级芯片 |
| `.chip--time` | 时间芯片 |
| `.chip--count` | 计数芯片 |

## 依赖与外部交互

### 外部依赖

| 依赖 | 用途 |
|------|------|
| `rich-note.ts` | 主逻辑模块（布局计算、事件处理、渲染） |
| Pretext 库 | 文本测量和布局 |
| 系统字体 | Helvetica Neue, SF Mono 等 |

### 动态 CSS 变量更新

```javascript
// 来自 rich-note.ts
domCache.root.style.setProperty('--note-width', `${noteWidth}px`)
domCache.root.style.setProperty('--note-content-width', `${bodyWidth}px`)
```

### 行渲染

```javascript
// 来自 rich-note.ts
const row = document.createElement('div')
row.className = 'line-row'
row.style.top = `${lineIndex * LINE_HEIGHT}px`

for (const part of line.fragments) {
  const element = document.createElement('span')
  element.className = part.className
  element.textContent = part.text
  if (part.leadingGap > 0) {
    element.style.marginLeft = `${part.leadingGap}px`
  }
  row.appendChild(element)
}
```

## 风险、边界与改进建议

### 已知风险

1. **绝对定位依赖**
   - 行使用 `position: absolute`，需要精确计算每行位置
   - 如果 Pretext 计算与实际渲染有偏差，可能出现重叠或间隙

2. **字体可用性**
   - 依赖 Helvetica Neue 和 SF Mono，在某些系统可能不可用
   - 虽然有回退字体，但度量可能略有差异

3. **芯片溢出**
   - 如果单个芯片宽度超过容器宽度，布局可能出现问题
   - 当前实现假设芯片总能放入某行

4. **CJK 文本**
   - 演示文本包含中文（"北京"、"中文"）和阿拉伯文（"جاهز"、"عربي"）
   - 需要确保字体对这些字符有良好的支持

### 边界情况处理

边界情况在 `rich-note.ts` 中处理：

| 边界情况 | 处理方式 |
|---------|---------|
| 芯片宽度 > 可用宽度 | 强制换行到下一行 |
| 文本片段宽度 > 可用宽度 | 使用 `layoutNextLine` 在字素边界断行 |
| 视口极窄 | 滑块最小值限制为 260px |
| 字体未加载 | 等待 `document.fonts.ready` |

### 改进建议

1. **字体加载保障**
   ```html
   <link rel="preconnect" href="https://fonts.googleapis.com">
   <link rel="preload" href="..." as="font" crossorigin>
   ```

2. **芯片溢出处理**
   ```css
   .chip {
     max-width: 100%;
     overflow: hidden;
     text-overflow: ellipsis;
   }
   ```

3. **动画过渡**
   ```css
   .line-row {
     transition: top 150ms ease;
   }
   .note-shell {
     transition: width 150ms ease;
   }
   ```

4. **可访问性增强**
   ```html
   <!-- 为芯片添加语义 -->
   <span class="chip chip--mention" role="button" tabindex="0" aria-label="提及用户 @maya">
     @maya
   </span>
   
   <!-- 为滑块添加 ARIA -->
   <input type="range" aria-label="文本宽度" aria-valuemin="260" aria-valuemax="760">
   ```

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
     .chip--mention { background: rgba(100, 180, 255, 0.15); }
     /* ... 其他芯片颜色调整 */
   }
   ```

6. **选择支持**
   ```css
   .note-body {
     user-select: text;
   }
   .line-row {
     pointer-events: none;  /* 让事件穿透到子元素 */
   }
   .frag, .chip {
     pointer-events: auto;
   }
   ```

7. **复制功能**
   ```javascript
   // 添加复制按钮，将富文本转换为纯文本或 Markdown
   function copyAsPlainText() { /* ... */ }
   function copyAsMarkdown() { /* ... */ }
   ```

8. **编辑模式**
   - 允许用户点击编辑文本内容
   - 实时预览布局变化
   - 导出编辑后的内联规范

### 维护注意事项

- 修改芯片颜色时，需同时更新 HTML 中的 CSS 和 `rich-note.ts` 中的 `CHIP_CLASS_NAMES`
- 添加新的芯片类型时，需要在 `ChipTone` 类型、样式类和渲染逻辑中同步更新
- 调整行高时，需同步更新 CSS 中的 `.line-row` 和 TS 中的 `LINE_HEIGHT` 常量
- 确保演示文本中的特殊字符（CJK、阿拉伯文）在目标字体中有良好支持
