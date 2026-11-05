# HTML

本文规定 HTML 的语义、文档结构、元信息、表单元素、媒体和链接。ARIA 和键盘行为见 [accessibility.md](accessibility.md)，图片的尺寸和格式见 [responsive-and-assets.md](responsive-and-assets.md)，表单校验见 [forms.md](forms.md)。

## 语义化

### 按含义选元素 【推荐】

元素按内容的含义选，不按外观选。`<div>` 和 `<span>` 没有语义，只在没有合适元素时使用。外观交给 CSS。

**原因：** 读屏软件和搜索引擎依赖元素语义。`role` 只补语义，不带来焦点和键盘行为。

**示例：**

反例：

```html
<div class="btn" onclick="save()">保存</div>
```

正例：

```html
<button type="button" id="save">保存</button>
```

**工具：** `jsx-a11y/no-static-element-interactions`、`jsx-a11y/click-events-have-key-events`。

**来源：** web.dev Learn HTML、web.dev Learn Accessibility

### 用地标元素划分页面 【推荐】

页面用 `<header>`、`<nav>`、`<main>`、`<aside>`、`<footer>` 划分。`<main>` 每页只有一个。`<section>` 要有标题，`<article>` 用于可独立成篇的内容。多个 `<nav>` 用 `aria-label` 区分。

**原因：** 地标是读屏用户跳转的入口。只有顶层的 `<header>` 和 `<footer>` 才算地标，嵌在 `<article>`、`<section>` 里的不算。

**来源：** web.dev Learn HTML

### 标题层级连续 【推荐】

用 `<h1>` 作页面主标题，往下逐级使用，不跳级。视觉大小用 CSS 控制，不靠标题级别。

**原因：** 浏览器不会根据嵌套自动计算大纲，层级只能由作者写对。

**来源：** web.dev Learn HTML

### 数据表格写全表头 【推荐】

数据表用 `<caption>` 说明用途，`<th>` 写 `scope="col"` 或 `scope="row"`，并用 `<thead>`、`<tbody>` 分区。

**原因：** 部分辅助技术无法自行判断表头对应的单元格。

**来源：** MDN

## 文档结构

### 页面骨架完整 【推荐】

第一行写 doctype。`<html>` 写 `lang`。`<meta charset>` 是 `<head>` 的第一个元素，之后是 `<title>` 和 viewport。正文中的外语片段在所在元素上单独写 `lang`。`lang` 的取值规则见 [i18n.md](i18n.md)。

**原因：** 缺少 doctype 会进入怪异模式。缺少 `lang` 时读屏软件读音会出错，这也违反 WCAG 2.2 的 3.1.1 和 3.1.2。

**示例：**

```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="utf-8" />
    <title>订单详情 - 示例商城</title>
    <meta name="viewport" content="width=device-width, initial-scale=1" />
  </head>
  <body></body>
</html>
```

**来源：** web.dev Learn HTML、MDN、WCAG 2.2

## SEO 元信息

### 每个页面有唯一的标题 【推荐】

`<title>` 说清页面内容，不用“首页”这类空泛词，不堆砌关键词。需要收录的页面用 `<meta name="description">` 一两句话概括本页。客户端渲染的页面在路由切换时也要更新。不写 `keywords`，Google 搜索不使用它。需要分享的页面写 `og:title`、`og:description`、`og:image`，图片用绝对地址。

**原因：** 标题相同时，用户无法在搜索结果和标签页中区分页面。这也是 WCAG 2.2 的 2.4.2。

**来源：** Google 搜索中心、web.dev Learn HTML、WCAG 2.2

### 声明规范地址 【推荐】

重复内容的页面用 `<link rel="canonical">` 指向首选地址，规范页面也指向自己。写完整的绝对地址，每页只保留一个。优先在 HTML 中输出，不靠脚本注入。不用 `robots.txt` 做规范化。

**来源：** Google 搜索中心

### 不收录的页面用 `noindex` 【推荐】

用 `<meta name="robots" content="noindex">` 或 `X-Robots-Tag` 响应头。这类页面不能同时被 `robots.txt` 屏蔽。需要收录的页面不要在初始 HTML 中写 `noindex` 再靠脚本移除。

**原因：** 爬虫读不到被屏蔽页面上的 `noindex`，页面仍可能进入结果。看到 `noindex` 后，Google 也可能跳过脚本执行。

**来源：** Google 搜索中心

### 可索引的页面用真实路径 【推荐】

需要收录的客户端路由用 History API，不用 `#/path`。站内链接必须是带 `href` 的 `<a>`。客户端渲染的错误页加 `noindex`，错误页不跳转，见 [routing.md](routing.md)。路由实现见 [routing.md](routing.md)。

**原因：** 只有 `onclick` 的元素不是可靠的抓取入口，片段路由也不能被可靠解析。

**来源：** Google 搜索中心

### 多语言页面互相声明 `hreflang` 【推荐】

每个语言版本列出全部版本，包括自身，集合在各版本上保持一致。地址写完整的协议和域名。语言选择页用 `x-default`。

**原因：** 两个页面没有互相指向时，标注会被忽略。

**来源：** Google 搜索中心

## 表单元素

### 每个控件关联 `<label>` 【必须】

**适用：** 线上服务

用 `for` 对应控件的 `id`，或把控件包在 `<label>` 里。`placeholder` 和 `title` 不能代替标签。`<label>` 内不放其他可交互元素。相关的单选框和复选框放进 `<fieldset>`，用 `<legend>` 作组标题。

**原因：** 标签提供无障碍名称，还能扩大点击区域。占位符在输入后消失。这是 WCAG 2.2 的 3.3.2。

**工具：** `jsx-a11y/label-has-associated-control`。

**来源：** web.dev Learn HTML、WAI-ARIA APG、WCAG 2.2

### 输入类型和用途写准确 【推荐】

用 `type`、`inputmode` 匹配数据，手机会显示对应键盘。采集用户信息的字段写 `autocomplete`，比如 `email`、`tel`、`current-password`、`new-password`、`one-time-code`。不用 `autocomplete="off"` 关闭这些字段的自动填充。

**原因：** 这能减少输入错误，也是 WCAG 2.2 的 1.3.5。`type="tel"` 不校验格式，需要时另加 `pattern`。

**来源：** web.dev Learn HTML、web.dev Learn Design、MDN、WCAG 2.2

## 媒体

### 图片写替代文本 【必须】

**适用：** 线上服务

有含义的图片用 `alt` 说明内容或用途，不写“图片”和文件名。触发操作的图片写操作，比如“下一页”。装饰性图片写 `alt=""`，不能省略该属性。`title` 不能代替 `alt`。

**原因：** 这是 WCAG 2.2 的 1.1.1。省略 `alt` 时读屏软件可能读出文件名。

**工具：** `jsx-a11y/alt-text`。

**来源：** MDN、Google 搜索中心、WCAG 2.2

### 音视频提供控件和字幕 【推荐】

**适用：** 线上服务

`<video>` 和 `<audio>` 写 `controls`，或提供完整的自定义控件。有语音的视频用 `<track>` 提供 WebVTT 字幕。自动播放只用于静音的装饰性背景视频。自动开始、持续超过 5 秒的动态内容必须能暂停、停止或隐藏。

**原因：** 部分浏览器在未设置 `muted` 时忽略 `autoplay`。不能暂停则违反 WCAG 2.2 的 2.2.2。

**来源：** web.dev Learn HTML、WCAG 2.2

## 链接

### 跳转用 `<a>`，操作用 `<button>` 【必须】

有目标地址时用带 `href` 的 `<a>`。其余操作用 `<button>`。不写 `href="#"`、`href="javascript:void(0)"`。不把按钮嵌进链接，也不把链接嵌进按钮。

**原因：** 假链接无法复制、在新标签打开或收藏，语义也是错的。

**工具：** `jsx-a11y/anchor-is-valid`。

**来源：** MDN、web.dev Learn HTML

### 链接文字说明去向 【推荐】

脱离上下文也能看懂去向，写“查看产品说明”，不写“点击这里”“更多”。图片链接用 `alt` 作名称。

**原因：** 读屏用户常按链接列表浏览，搜索引擎也用锚文本理解目标页。这是 WCAG 2.2 的 2.4.4。

**来源：** MDN、Google 搜索中心、WCAG 2.2

### 提示新窗口和下载，按用途标注外链 【推荐】

打开新标签页或下载文件时，在链接文字中说明，比如“（在新标签页打开）”“（PDF）”。`opener` 的安全要求见 [security.md](security.md)。付费链接用 `rel="sponsored"`，用户提交的链接用 `rel="ugc"`，不信任的来源用 `rel="nofollow"`，不给所有外链一律加 `nofollow`。

**来源：** MDN、web.dev Learn HTML、Google 搜索中心
