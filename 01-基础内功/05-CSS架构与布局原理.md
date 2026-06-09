# CSS 架构与布局原理

## 学习目标

- 理解 CSS 盒模型、BFC、层叠与继承
- 掌握 Flexbox 与 Grid 的布局原理及适用场景
- 了解 CSS 架构方案：BEM、CSS Modules、CSS-in-JS、原子化
- 理解响应式设计与多端适配策略
- 能从架构角度选择 CSS 组织方式

## 为什么需要

CSS 看似简单，却是大型前端项目维护成本的来源：

- 全局样式污染、特异性战争
- 组件样式难以复用与隔离
- 多端适配（PC/移动/大屏）策略混乱
- 设计系统与主题化缺乏统一 token

架构师需要为团队选定 CSS 方案，平衡开发效率、可维护性、运行时性能。

## 核心原理

### 1. 盒模型

```mermaid
flowchart TB
    subgraph marginBox [Margin 区域]
        subgraph borderBox [Border 区域]
            subgraph paddingBox [Padding 区域]
                content[Content 内容区]
            end
        end
    end
```

```css
/* 标准盒模型：width = content */
.box {
  box-sizing: content-box;
  width: 200px;
  padding: 10px;
  border: 2px solid;
  /* 实际占用宽度 = 200 + 20 + 4 = 224px */
}

/* 推荐：border-box，width 包含 padding 和 border */
.box {
  box-sizing: border-box;
  width: 200px;
  padding: 10px;
  border: 2px solid;
  /* 实际占用宽度 = 200px */
}

*, *::before, *::after {
  box-sizing: border-box;
}
```

### 2. BFC（块级格式化上下文）

BFC 是独立布局区域，内部布局不影响外部。

**触发 BFC 的条件：**

- `float` 不为 none
- `position` 为 absolute/fixed
- `display: flow-root`（推荐）
- `overflow` 不为 visible

**应用：**

```css
/* 清除浮动 */
.clearfix::after {
  content: '';
  display: block;
  clear: both;
}
/* 或使用 display: flow-root 触发 BFC */

/* 防止 margin 折叠 */
.wrapper {
  display: flow-root;
}

/* 自适应两栏布局 */
.sidebar { float: left; width: 200px; }
.main { overflow: hidden; } /* 触发 BFC，不与 float 重叠 */
```

### 3. Flexbox 布局原理

Flex 是一维布局：主轴 + 交叉轴。

```mermaid
flowchart LR
    subgraph flexContainer [Flex Container]
        direction[flex-direction: row]
        item1[Item 1 flex: 1]
        item2[Item 2 flex: 2]
        item3[Item 3 flex: 0 0 100px]
    end
```

```css
.container {
  display: flex;
  flex-direction: row;      /* 主轴方向 */
  justify-content: center;  /* 主轴对齐 */
  align-items: center;      /* 交叉轴对齐 */
  gap: 16px;
}

.item {
  flex: 1 1 auto; /* flex-grow flex-shrink flex-basis */
  /* flex: 1 等价于 flex: 1 1 0% */
}
```

**适用场景：** 导航栏、卡片内对齐、等分布局、垂直居中

### 4. Grid 布局原理

Grid 是二维布局：行 + 列。

```css
.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: auto 1fr auto;
  gap: 24px;
}

.header { grid-column: 1 / -1; } /* 跨所有列 */
.sidebar { grid-row: 2; grid-column: 1; }
.main { grid-row: 2; grid-column: 2 / 4; }
```

**适用场景：** 页面整体布局、复杂二维网格、仪表盘

| 对比 | Flex | Grid |
|------|------|------|
| 维度 | 一维 | 二维 |
| 适用 | 组件内部、线性排列 | 页面级、复杂网格 |
| 对齐 | 强 | 更强 |

### 5. 层叠、继承与特异性

**特异性（Specificity）：** 内联 > ID > 类/属性/伪类 > 元素

```
!important > 内联 style > #id > .class > element
```

**架构启示：** 避免高特异性选择器，组件样式保持扁平。

### 6. CSS 架构方案对比

```mermaid
flowchart TD
    Global[全局 CSS 问题] --> Solutions{架构方案}
    Solutions --> BEM[BEM 命名约定]
    Solutions --> Modules[CSS Modules]
    Solutions --> CIJS[CSS-in-JS]
    Solutions --> Atomic[原子化 Tailwind]
```

| 方案 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| **BEM** | Block__Element--Modifier 命名 | 无运行时开销、简单 | 类名冗长、靠约定 |
| **CSS Modules** | 构建时 hash 类名，局部作用域 | 真正隔离、零运行时 | 需构建工具 |
| **CSS-in-JS** | JS 中写样式，运行时/编译时生成 | 动态主题、组件化 | 运行时成本（styled-components） |
| **Tailwind** | 原子类组合 | 开发快、设计一致 | HTML 类名多、学习曲线 |

**BEM 示例：**

```html
<div class="card card--featured">
  <h2 class="card__title">Title</h2>
  <button class="card__button card__button--primary">Action</button>
</div>
```

**CSS Modules 示例：**

```css
/* Button.module.css */
.primary { background: blue; }
```

```tsx
import styles from './Button.module.css';
<button className={styles.primary}>Click</button>
// 编译后 class="Button_primary_x7f2a"
```

**Design Token 与主题化：**

```css
:root {
  --color-primary: #0066cc;
  --color-text: #333;
  --spacing-md: 16px;
  --radius-sm: 4px;
}

[data-theme="dark"] {
  --color-primary: #4da6ff;
  --color-text: #eee;
}

.button {
  background: var(--color-primary);
  padding: var(--spacing-md);
  border-radius: var(--radius-sm);
}
```

### 7. 响应式设计

```css
/* Mobile First */
.container { padding: 16px; }

@media (min-width: 768px) {
  .container { padding: 24px; max-width: 720px; }
}

@media (min-width: 1024px) {
  .container { max-width: 960px; }
}

/* 容器查询 — 组件级响应式 */
.card-container {
  container-type: inline-size;
}
@container (min-width: 400px) {
  .card { flex-direction: row; }
}
```

### 8. 定位与层叠上下文

```css
/* position 取值 */
.static   { position: static; }   /* 默认 */
.relative { position: relative; top: 10px; } /* 相对自身偏移 */
.absolute { position: absolute; } /* 相对最近定位祖先 */
.fixed    { position: fixed; }    /* 相对视口 */
.sticky   { position: sticky; top: 0; } /* 滚动到阈值后 fixed */
```

**层叠上下文：** 由 `position`+`z-index`、`opacity<1`、`transform`、`filter` 等创建。`z-index` 仅在同一层叠上下文内比较。

### 9. 单位体系

| 单位 | 说明 |
|------|------|
| `px` | 绝对像素 |
| `rem` | 相对根元素 `html` font-size |
| `em` | 相对当前元素 font-size |
| `vw/vh` | 视口宽/高 1% |
| `%` | 相对父元素 |

```css
html { font-size: 16px; }
.title { font-size: 1.5rem; } /* 24px */
.fluid { font-size: clamp(1rem, 2vw + 1rem, 2rem); } /* 流体排版 */
```

### 10. 居中方案汇总

```css
/* Flex */
.center-flex { display: flex; justify-content: center; align-items: center; }

/* Grid */
.center-grid { display: grid; place-items: center; }

/* 绝对定位 + transform */
.center-abs {
  position: absolute;
  top: 50%; left: 50%;
  transform: translate(-50%, -50%);
}
```

### 11. transition / animation 与性能

```css
.box {
  transition: transform 0.3s ease, opacity 0.3s;
}
.box:hover { transform: scale(1.05); } /* 仅合成层，性能好 */

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}
.animate { animation: fadeIn 0.5s ease; }
```

避免动画 `width`/`height`/`top`/`left`，优先 `transform`/`opacity`。

### 12. 现代 CSS 特性

```css
/* :has() — 父选择子 */
.card:has(img) { padding-top: 0; }

/* 原生嵌套（现代浏览器） */
.card {
  & .title { font-weight: bold; }
}

/* @layer — 层叠顺序控制 */
@layer reset, base, components, utilities;

/* 容器查询深入 */
@container card (min-width: 400px) {
  .card-body { display: grid; grid-template-columns: 1fr 1fr; }
}
```

### 13. 预处理器与 PostCSS

| 工具 | 作用 |
|------|------|
| Sass/Less | 变量、嵌套、混入，编译为 CSS |
| PostCSS | 插件生态：Autoprefixer、cssnano、preset-env |

现代项目常直接用 CSS Variables + 原生嵌套，PostCSS 仅做 autoprefixer 和压缩。

## 常见误区与最佳实践

| 误区 | 正确理解 |
|------|---------|
| "!important 解决问题" | 破坏层叠秩序，应降低特异性 |
| "Flex 可以替代 Grid" | 一维 vs 二维，场景不同 |
| "Tailwind 不是 CSS" | 是原子化 CSS 架构，仍是 CSS |
| "CSS-in-JS 一定慢" | 编译时方案（Vanilla Extract）无运行时 |

**架构选型建议：**

- 小团队/快速迭代：Tailwind + Design Token
- 大型组件库：CSS Modules 或 CSS-in-JS（编译时）
- 设计系统：Token + CSS Variables，支持主题切换
- 避免全局 CSS 无约束增长，建立 Stylelint 规则

## 小结

- 盒模型用 `border-box`，BFC 解决浮动与 margin 问题
- Flex 一维、Grid 二维，按场景选择
- CSS 架构核心是**作用域隔离**与**命名规范**
- Design Token 是设计系统的基础
- 定位、层叠上下文、单位体系是布局基础
- Mobile First + 容器查询 + 现代 CSS 实现响应式

## 延伸阅读

- [MDN: Flexbox](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_flexible_box_layout)
- [MDN: Grid](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_grid_layout)
- [BEM 方法论](https://en.bem.info/)
- [Tailwind CSS 文档](https://tailwindcss.com/docs)
- 《CSS 揭秘》
