# CSS

本文规定样式方案、Tailwind、设计变量、层叠、布局、动画和深色模式。断点和移动端见 [responsive-and-assets.md](responsive-and-assets.md)，对比度和焦点样式见 [accessibility.md](accessibility.md)，RTL 用的逻辑属性见 [i18n.md](i18n.md)。

## 方案选型

### 一个项目只选一种主方案 【推荐】

在 Tailwind、CSS Modules、Vue scoped 中选一种作为主方案。

**原因：** 混用后难以判断覆盖关系，同一视觉会有多种写法。

**例外：** 覆盖第三方组件样式。

### 样式默认隔离作用域 【推荐】

全局样式只放 reset、设计变量和基础排版，其余样式限定在组件内。确实需要全局类名时使用 BEM，比如 `.card__title--active`。

## Tailwind 规范

本节的 `@theme`、`@reference`、`@custom-variant` 和 `bg-(--x)` 写法需要 Tailwind CSS 4 及以上。

### 类名写完整，不动态拼接 【必须】

Tailwind 把源文件当纯文本扫描。条件分支和 props 要映射到完整的类名字符串。

**原因：** 拼接出的类名不会出现在源码里，对应 CSS 不会生成，样式在运行时静默丢失。

**示例：**

反例：

```tsx
<button className={`bg-${color}-600`}>保存</button>
```

正例：

```tsx
const colorVariants = {
  primary: 'bg-blue-600 hover:bg-blue-500',
  danger: 'bg-red-600 hover:bg-red-500',
};

<button className={colorVariants[color]}>保存</button>;
```

**来源：** Tailwind CSS 文档

### 设计系统的值进 `@theme` 【推荐】

会复用的颜色、间距、字体、断点写进 `@theme`。只有一次性的像素调整才用任意值，比如 `top-[117px]`。不需要生成工具类的普通变量写在 `:root`。

**原因：** `@theme` 变量同时生成工具类和 CSS 变量。到处写 `bg-[#2563eb]` 等于绕过设计变量。

**来源：** Tailwind CSS 文档

### 不写互相冲突的工具类 【必须】

同一属性只保留一个工具类，用条件判断选择，比如 `isGrid ? 'grid' : 'flex'`。组件库暴露 props，不让调用方追加冲突的类名。

**原因：** 生效的是生成样式表中靠后的类，不是 `class` 属性中靠后的类。

**来源：** Tailwind CSS 文档

### 复用靠组件，不靠自造类名 【推荐】

跨文件复用的样式抽成组件或模板片段。单元素的小样式可以放进 `@layer components`，工具类仍能覆盖它。`@apply` 主要用于覆盖第三方样式。在组件样式块或 CSS Modules 中使用 `@apply` 时，先写 `@reference`。运行时的值用 `style` 传入 CSS 变量，再由工具类引用，比如 `bg-(--brand-color)`。类名排序交给 `prettier-plugin-tailwindcss`。

**来源：** Tailwind CSS 文档

## 设计变量

### 视觉值使用设计变量 【推荐】

颜色、字号、间距、圆角、阴影使用设计变量，不到处写字面值，一次性的调整除外。变量按语义命名，用 `--color-danger`，不用 `--color-red`。`var()` 只能用在属性值中，不能用在选择器和媒体查询条件中。

**原因：** 主题和改版才能集中修改，深色模式时只替换变量的值。

**示例：**

```css
:root {
  --color-text: #111827;
  --space-4: 1rem;
}

.card {
  padding: var(--space-4);
  color: var(--color-text);
}
```

**来源：** MDN

### `z-index` 用分层变量管理 【推荐】

用 `--z-dropdown`、`--z-modal`、`--z-toast` 这样的分层变量，不写 `9999` 这类竞赛式数值。层级不生效时先检查堆叠上下文。`opacity`、`transform`、`filter`、`will-change` 都会创建堆叠上下文，子元素无法越过父级的上下文。

**来源：** web.dev Learn CSS

## 选择器和层叠

### 不使用 `!important` 【推荐】

包括 Tailwind 的 `!` 后缀。

**原因：** `!important` 不提高优先级，而是进入更高的层叠来源，之后只能用另一个 `!important` 对抗。

**例外：** 覆盖第三方样式且确实无法用其他办法时，就近写一句注释说明原因。

**来源：** web.dev Learn CSS、Tailwind CSS 文档

### 不用 ID 选择器写样式 【推荐】

样式选择器用类名，ID 只用于锚点和 `label` 关联。

**原因：** 一个 ID 的优先级高于任意数量的类名，之后难以覆盖。

**来源：** web.dev Learn CSS

### 优先级保持最低够用 【推荐】

尽量用单个类名，嵌套不超过 3 层。不用 `.btn.btn` 抬高优先级，不写 `.a > div > span` 这类依赖 DOM 结构的选择器。需要被轻松覆盖的基础样式用 `:where()`，它的优先级为零。

**来源：** web.dev Learn CSS

### 用 `@layer` 管理层叠顺序 【推荐】

存在多个样式来源时，在入口用一条语句声明层的顺序，比如 `@layer reset, vendor, components, utilities;`，后声明的优先级高。

**原因：** 层的顺序先于选择器优先级。未分层的样式会压过所有层，第三方样式要放进层里。层内的 `!important` 顺序相反。

**来源：** MDN

## 布局

### 组件不带外边距 【推荐】

组件之间的间距用 `gap` 或单方向 `margin`，由父级控制排布。

### 字号使用 `rem` 【推荐】

`rem` 基于根元素，能跟随用户的浏览器字号设置。边框等细节可以用 `px`。长文本用 `max-width: 60ch` 这类单位限制行宽。

**来源：** web.dev Learn CSS

## 动画

### 优先动画 `transform` 和 `opacity` 【推荐】

**原因：** 这两个属性只走合成阶段。动画 `width`、`top` 等属性会触发布局和绘制，容易卡顿。

**来源：** web.dev 动画指南

### 尊重 `prefers-reduced-motion` 【推荐】

**适用：** 线上服务

对位移、缩放和视差这类大幅动画提供减弱版本。保留必要反馈时，用淡入淡出代替运动。

**原因：** 大幅运动可能让前庭障碍用户不适。对应 WCAG 2.2 的 2.3.3（AAA 级）。

**示例：**

```css
.panel {
  transition: transform 300ms ease;
}

@media (prefers-reduced-motion: reduce) {
  .panel {
    transition: opacity 150ms linear;
  }
}
```

**来源：** MDN、WCAG 2.2

### `will-change` 只用于已确认的性能问题 【推荐】

不写进全局样式，不加在大范围的元素上。需要时在动画前用脚本设置，结束后重置。

**原因：** 滥用会增加内存并让页面变慢，还会提前创建堆叠上下文。

**来源：** MDN

## 深色模式

### 通过语义变量切换主题 【推荐】

组件只引用语义变量，主题切换只改变量的值，比如在 `prefers-color-scheme: dark` 下重写 `:root` 的变量。不在组件里成对写浅色和深色的字面颜色。两套主题都要满足 [accessibility.md](accessibility.md) 的对比度要求。

### 声明 `color-scheme` 【推荐】

支持深色模式时，在 `:root` 写 `color-scheme: light dark`，并在 `<head>` 的 CSS 之前写 `<meta name="color-scheme">`。提供手动切换时，选定主题后把 `color-scheme` 改成对应的单一值。

**原因：** 表单控件和滚动条的默认颜色由它决定。不声明时控件会和页面主题不一致，加载时也可能闪烁。

**来源：** MDN

### 手动切换时避免首屏闪烁 【推荐】

用户选择由服务端渲染到 `<html>`，或由 `<head>` 内联脚本在绘制前写入，内联脚本要符合 [security.md](security.md) 的 CSP 要求。Tailwind 项目用 `@custom-variant` 让 `dark:` 改为匹配类名或属性。

**来源：** Tailwind CSS 文档
