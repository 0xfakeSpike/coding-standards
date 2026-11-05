# 无障碍

本文规定键盘、焦点、ARIA、对比度和常用组件的要求。元素语义、`label`、`alt` 见 [html.md](html.md)，点击目标见 [responsive-and-assets.md](responsive-and-assets.md)，表单错误见 [forms.md](forms.md)，测试写法见 [testing.md](testing.md)。

**适用：** 线上服务（本文全部规则）

## WCAG 2.2 要点

### 以 WCAG 2.2 AA 级为基线 【推荐】

AA 级要满足全部 A 级和 AA 级成功标准。除本文规则外，重点看 2.2 新增的标准。

| 编号 | 名称 | 级别 | 做法 |
|---|---|---|---|
| 2.4.11 | Focus Not Obscured (Minimum) | AA | 固定层不完全遮住焦点元素 |
| 2.5.7 | Dragging Movements | AA | 拖拽有单指针替代方式 |
| 2.5.8 | Target Size (Minimum) | AA | 目标至少 24×24 CSS 像素 |
| 3.2.6 | Consistent Help | A | 帮助入口在各页面位置一致 |
| 3.3.7 | Redundant Entry | A | 流程内已填内容自动带入或可选择，确认新密码等情况除外 |
| 3.3.8 | Accessible Authentication (Minimum) | AA | 登录不依赖记忆和转录 |

**来源：** WCAG 2.2、Understanding WCAG 2.2

### 登录流程不阻止粘贴和自动填充 【必须】

密码和验证码框允许粘贴，不阻止密码管理器。拆成多格的验证码要接受一次粘贴全部内容。转录型验证码和算术题要有不依赖认知测试的替代方式。

**原因：** 阻止粘贴会迫使用户转录，不满足 3.3.8。

**来源：** Understanding WCAG 2.2

## 键盘

### 所有功能可以用键盘完成 【必须】

鼠标能做的操作，键盘都能做，且焦点不会困在某个组件里出不来（2.1.1、2.1.2）。用 `role` 自定义的控件要自己实现键盘行为。

**工具：** `jsx-a11y/click-events-have-key-events`。

**来源：** WCAG 2.2、WAI-ARIA APG

### 焦点顺序跟随 DOM 【推荐】

`tabindex` 只用 `0` 和 `-1`，不用正数。需要改顺序时调整 DOM，不让视觉顺序和 DOM 顺序脱节（2.4.3）。`<body>` 内第一个可聚焦元素是跳到 `<main>` 的链接，可以在获得焦点前隐藏（2.4.1）。

**工具：** `jsx-a11y/tabindex-no-positive`。

**来源：** WAI-ARIA APG、web.dev Learn Accessibility、MDN、WCAG 2.2

## 焦点

### 焦点样式始终可见 【必须】

不用 `outline: none` 去掉焦点样式而不提供替代。自定义样式用 `:focus-visible`，与相邻颜色的对比度至少 3:1，且要和选中态有区别。

**原因：** 去掉焦点指示违反 2.4.7（AA 级），键盘用户将不知道自己在哪。

**示例：**

```css
button:focus-visible {
  outline: 2px solid var(--color-focus);
  outline-offset: 2px;
}
```

**来源：** Understanding WCAG 2.2、WAI-ARIA APG、MDN

### 固定层不遮挡焦点 【推荐】

有吸顶栏、底部栏或 Cookie 横幅时，用 `scroll-padding-block` 留出空间。

**来源：** Understanding WCAG 2.2

### 焦点不能丢失 【推荐】

元素被删除或隐藏前，把焦点移到相邻的合理位置，不让它落回 `<body>`。单页应用切换路由后，把焦点移到新页面的主标题或 `<main>`，目标元素设 `tabindex="-1"`。

**原因：** 焦点丢失后，键盘和读屏用户要从页面顶部重新开始。

**来源：** WAI-ARIA APG

## ARIA

### 优先使用原生元素 【推荐】

原生元素能满足时不用 ARIA。不写多余的角色，比如 `<ul role="list">`，也不改变原生语义。`role` 不提供键盘行为和样式，这些都要作者实现。

**原因：** 错误的 ARIA 比没有 ARIA 更糟。

**来源：** WAI-ARIA APG、web.dev Learn Accessibility

### 不隐藏可聚焦元素 【必须】

`aria-hidden="true"` 和 `role="presentation"` 不用在可聚焦元素及其祖先上。

**原因：** 用户会聚焦到读屏软件读不出的“空白”。

**来源：** web.dev Learn Accessibility

### 每个控件有准确的名称 【必须】

优先用可见文字，有可见文字可引用时用 `aria-labelledby`，都没有时才用 `aria-label`，比如纯图标按钮 `<button aria-label="关闭">`。名称要包含可见标签的文字（2.5.3），一到三个词即可，不写“按钮”这类角色词。补充说明用 `aria-describedby`，不用 `title`。

**原因：** 控件要能通过程序读出名称、角色和值（4.1.2）。在按钮上写 `aria-label` 会覆盖按钮里的文字。

**来源：** WAI-ARIA APG、WCAG 2.2

### 动态状态用实时区域播报 【推荐】

保存成功、搜索结果数、加载中这类不移动焦点的提示用 `role="status"`，并写 `aria-live="polite"`。重要且紧急的错误才用 `role="alert"`。容器放在初始 HTML 中，之后再写入内容。计数这类局部更新的文本加 `aria-atomic="true"`。

**原因：** 否则读屏用户收不到提示，违反 4.1.3（AA 级）。动态插入的容器可能不被播报，`role="alert"` 除外。

**来源：** Understanding WCAG 2.2、MDN

### 需要被发现的禁用项用 `aria-disabled` 【推荐】

工具栏按钮、菜单项、标签页这类控件禁用时保留焦点，写 `aria-disabled="true"`，并阻止操作。

**原因：** 原生 `disabled` 会移出 Tab 顺序，读屏用户发现不了它。

**来源：** WAI-ARIA APG

## 对比度

### 对比度达到 AA 级 【必须】

正文和图片中的文字至少 4.5:1。大号文字（至少 18pt，或 14pt 粗体）至少 3:1（1.4.3）。图标、输入框边界和状态指示这类必需图形，与相邻颜色至少 3:1（1.4.11）。禁用控件、纯装饰文字和 Logo 不受限。深色和浅色主题都要检查。

**来源：** WCAG 2.2、Understanding WCAG 2.2

### 不只靠颜色传递信息 【必须】

错误、必填和状态要同时有文字或图标。正文中的链接除颜色外还要有下划线，或与周围文字有 3:1 的对比度（1.4.1）。图表用图案或标注区分系列。

**来源：** Understanding WCAG 2.2、web.dev Learn Accessibility

### 放大和调整间距后内容可用 【推荐】

文字放大到 200% 时内容不丢失（1.4.4）。用户把行高设为字号的 1.5 倍、段间距 2 倍、字间距 0.12 倍、词间距 0.16 倍后，内容不被截断（1.4.12）。放文字的容器不写死高度。

**来源：** WCAG 2.2

### 兼顾强制颜色模式 【推荐】

`forced-colors: active` 下 `box-shadow` 失效，靠阴影表达的边界要补边框。只做局部修补，不另做一套设计。

**来源：** MDN

## 常用组件模式

### 弹窗优先用原生 `<dialog>` 或成熟的无障碍组件库 【推荐】

用 `showModal()` 打开，浏览器负责隔离背景、限制焦点和响应 `Esc`。提供可见的关闭按钮，用 `aria-labelledby` 指向标题。不在 `<dialog>` 上写 `tabindex`。打开时焦点放到内部的合适元素：内容长时放开头，不可逆操作放最不危险的选项。关闭后焦点回到触发元素，它不存在时移到合理位置。完全手写时，要给背景加 `inert`，并设 `role="dialog"` 和 `aria-modal="true"`。

**原因：** 手写这些行为容易出错。

**来源：** MDN、WAI-ARIA APG

### 自定义组件按 APG 模式实现 【推荐】

原生元素不够用时，角色、状态和按键以 APG 为准。组内多项控件只占一个 Tab 位，组内用方向键移动，实现方式是移动 `tabindex="0"` 或 `aria-activedescendant`。

| 组件 | 角色和状态 | 键盘 |
|---|---|---|
| 折叠区 | 按钮的 `aria-expanded` | `Enter`、`Space` 切换 |
| 标签页 | `tablist`、`tab`、`tabpanel`，`aria-selected`，`aria-controls` | 左右方向键切换，`Tab` 进入面板 |
| 菜单按钮 | `aria-haspopup`、`aria-expanded`，容器 `menu` | `Enter`、`Space`、下方向键打开并聚焦首项 |
| 组合框 | `combobox`、`aria-expanded`、`aria-controls`、`aria-activedescendant` | 下方向键进入列表，`Esc` 关闭，`Enter` 选中 |

标签页只有在面板内容无明显延迟时才在聚焦时自动激活。站点导航优先用 `<nav>`、链接和折叠区模式，`menu` 主要用于类似操作系统菜单的命令集合，其中 `Tab` 不在项之间移动。

**来源：** WAI-ARIA APG

### 悬停和聚焦出现的内容要可控 【推荐】

提示内容按 `Esc` 能关闭，指针能移到提示上，并保持显示到用户离开（1.4.13）。提示框用 `role="tooltip"`，触发元素用 `aria-describedby` 关联，焦点留在触发元素上。内容里有可聚焦元素时改用非模态弹窗。

**来源：** Understanding WCAG 2.2、WAI-ARIA APG

## 自动化检查

### lint 进入 CI 【推荐】

React 项目启用 `eslint-plugin-jsx-a11y` 的 `recommended` 配置。axe 的写法见 [testing.md](testing.md)。

**来源：** eslint-plugin-jsx-a11y 文档

### 手工检查关键流程 【推荐】

自动化满分不代表可用。用键盘走完主流程，用读屏软件核对名称和状态，把页面放大到 200%，并在窄屏下检查。

**来源：** web.dev Learn Accessibility、axe-core 文档
