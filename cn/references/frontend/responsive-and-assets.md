# 响应式与静态资源

本文规定断点、移动端适配、安全区、图片、字体、图标和触控。`alt` 的写法见 [html.md](html.md)，首屏图片的加载优先级见 [performance.md](performance.md)。

## 断点

### 断点常量只留一份 【必须】

断点在项目中只定义一处，CSS 和脚本都从这里取值，不在各处写 `768px` 这类数字。Tailwind 项目定义在 `@theme` 的 `--breakpoint-*`，全部用同一单位。其他方案用一份常量在构建时生成 CSS，因为 `var()` 不能写进媒体查询条件，Sass 变量或 `@custom-media` 等项目已有方式都算一份。脚本中用 `matchMedia` 判断，不读 `window.innerWidth` 后自己比较。

**原因：** 多份断点迟早不一致，页面会在某些宽度下同时命中两套布局。Tailwind 断点的单位不一致时，生成的样式排序会出错。

**示例：**

```css
@theme {
  --breakpoint-*: initial;
  --breakpoint-tablet: 48rem;
  --breakpoint-desktop: 80rem;
}
```

**来源：** 团队、Tailwind CSS 文档、MDN

### 移动优先，断点由内容决定 【推荐】

基础样式写窄屏，再用 `min-width` 向上叠加。Tailwind 的 `sm:` 含义是从该断点起生效，不是“在小屏上”，窄屏样式写在无前缀的类上。在内容开始难读的宽度设断点，不追逐热门设备尺寸。

**来源：** Tailwind CSS 文档、web.dev Learn Design

### 组件按容器尺寸响应 【推荐】

可复用组件用容器查询，页面级布局用媒体查询。容器的 `container-type` 通常用 `inline-size`。

**原因：** 同一组件放进侧栏或主栏时，视口宽度不能说明它的可用空间。容器不能依赖子元素撑开尺寸，否则会塌陷。

**来源：** web.dev Learn CSS、Tailwind CSS 文档

## 移动端适配

### 设置 viewport，不禁止缩放 【必须】

**适用：** 线上服务

使用 `<meta name="viewport" content="width=device-width, initial-scale=1">`。不写 `user-scalable=no`，不用 `maximum-scale` 限制缩放。

**原因：** 低视力用户依赖缩放阅读。文字要能放大到 200%（WCAG 2.2 的 1.4.4）。

**来源：** MDN、WCAG 2.2

### 窄屏不出现横向滚动 【推荐】

**适用：** 线上服务

内容在 320 CSS 像素宽时仍可用，不需要同时朝两个方向滚动。地图、数据表和视频这类内容可以除外。

**原因：** 这是 WCAG 2.2 的 1.4.10。放大 400% 的桌面用户也会落到这个宽度。

**来源：** WCAG 2.2

### 全屏高度用新视口单位 【推荐】

内容不能被截断时用 `svh`。必须贴合可见区域时用 `dvh`，接受滚动时尺寸变化。不直接用 `100vh`，它在手机上等同 `lvh`，地址栏展开时底部会被遮住。

**原因：** `dvh` 的值会随浏览器工具栏变化，滚动时可能引起重新布局，所以优先用稳定的 `svh`。

**来源：** MDN

### 按输入能力判断，不按屏幕宽度 【推荐】

用 `pointer`、`hover` 媒体特性判断输入方式。只靠悬停出现的内容，同时支持 `:focus`，并提供触摸入口。

**原因：** 笔记本可能有触屏，平板可能接鼠标。

**来源：** web.dev Learn Design

## 安全区

### 铺满屏幕时避开安全区 【推荐】

viewport 写 `viewport-fit=cover` 时，贴边的导航栏、底部按钮和浮层用 `env(safe-area-inset-*)` 加内边距。和原有间距相加时用 `calc()`，并写默认值。

**原因：** 否则内容会被刘海、圆角或系统手势条遮挡。

**示例：**

```css
.bottom-bar {
  position: sticky;
  bottom: 0;
  padding-block-end: calc(1rem + env(safe-area-inset-bottom, 0px));
}
```

**来源：** MDN

## 图片

### 声明宽高或 `aspect-ratio` 【推荐】

`<img>` 和 `<video>` 写 `width` 和 `height`，值为固有像素尺寸。缩放由 CSS 完成：`max-inline-size: 100%; block-size: auto`。

**原因：** 否则加载后页面会跳动，影响 CLS。懒加载图片没有尺寸时可能一开始就全部加载。

**来源：** MDN、web.dev Learn Design、web.dev 图片懒加载

### 按显示尺寸提供图片 【推荐】

同一张图用 `srcset` 的 `w` 描述符加 `sizes`，固定显示尺寸的图用 `x` 描述符。窄屏要换构图时用 `<picture>` 的 `media`。`sizes` 要和实际布局一致。

**原因：** 浏览器在解析 CSS 前就开始选图，只能靠标记声明。用脚本切换地址会多下载一次。

**示例：**

```html
<img
  src="hero-800.jpg"
  srcset="hero-480.jpg 480w, hero-800.jpg 800w, hero-1600.jpg 1600w"
  sizes="(width <= 600px) 100vw, 800px"
  width="800"
  height="450"
  alt="仓库内景"
/>
```

**来源：** MDN、web.dev Learn Design

### 使用 AVIF 或 WebP 【推荐】

用 `<picture>` 的 `<source type="image/avif">` 提供格式，高压缩率的放前面，`<img>` 放通用格式并写 `alt`。初始视口外的图片加 `loading="lazy"`，首屏图片不懒加载，具体要求见 [performance.md](performance.md)。

**来源：** MDN、web.dev Learn Design、web.dev 图片懒加载

## 字体

### 明确字体加载策略 【推荐】

`@font-face` 写 `font-display`。正文一般用 `swap`，文字立即可见但切换时可能偏移。品牌字体不是必需时用 `optional`，没有偏移，但首次访问可能看不到网络字体。

**来源：** MDN、web.dev 字体优化

### 控制字体体积 【推荐】

只加载用到的字重，格式用 WOFF2。中文字体按需子集化，完整字体通常有数 MB。`<link rel="preload" as="font">` 只用于首屏确实使用的字体，并带 `crossorigin`，自托管字体也一样。

**原因：** 预加载过多会抢占关键资源的带宽。

**来源：** web.dev 字体优化

## 图标

### 图标使用 SVG 【推荐】

需要跟随文字变色时用内联 SVG 或 `<use>`，填充写 `currentColor`。设计工具导出的 SVG 先压缩。

**来源：** web.dev Learn Design

### 区分装饰图标和独立图标 【推荐】

**适用：** 线上服务

文字旁的图标是装饰，内联 SVG 加 `aria-hidden="true"`，图片写 `alt=""`。单独表达含义的图标要有无障碍名称，描述含义而不是外观，写法见 [accessibility.md](accessibility.md)。独立图标不用 CSS 背景图，因为背景图无法提供名称。

**来源：** web.dev Learn Design

## 触控

### 点击目标足够大 【推荐】

**适用：** 线上服务

所有指针目标至少 24×24 CSS 像素，这是 WCAG 2.2 的 2.5.8（AA 级）。小于该尺寸时，以目标为圆心的 24 像素圆不能与其他目标相交。行内链接等情况除外。移动端的主要操作做到 44 到 48 像素，相邻目标间隔约 8 像素。图标较小时用内边距扩大点击区域。

**来源：** WCAG 2.2、Understanding WCAG 2.2、MDN、web.dev 触控目标

### 拖拽提供单指针替代方式 【推荐】

**适用：** 线上服务

排序、看板、滑块这类拖拽功能，要同时提供点击方式，比如上下移动按钮或移动到某列的菜单。

**原因：** 这是 WCAG 2.2 的 2.5.7（AA 级）。部分用户无法按住并移动指针。

**来源：** WCAG 2.2、Understanding WCAG 2.2

### 手势限制在局部区域 【推荐】

自定义手势只在对应元素上设置 `touch-action`。不需要自定义手势时用 `manipulation`，它保留平移和双指缩放。`none` 只用于地图、画布这类元素。

**原因：** `none` 可能让用户无法缩放。

**来源：** MDN
