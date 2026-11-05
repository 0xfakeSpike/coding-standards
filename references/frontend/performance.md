# 前端性能

本文讲浏览器端的性能目标和优化手段。通用测量方法见 [性能](../general/performance.md)，图片和字体的处理见 [响应式与资源](responsive-and-assets.md)，打包配置见 [构建与兼容](build-and-compat.md)，线上采集见 [可观测性](observability.md)。

## Core Web Vitals

### 以 Core Web Vitals 作为体验目标 【推荐】

**适用：** 线上服务

| 指标 | 衡量内容 | 良好 | 需改进 | 较差 |
|---|---|---|---|---|
| LCP | 加载 | ≤ 2.5 秒 | 2.5～4.0 秒 | > 4.0 秒 |
| INP | 交互响应 | ≤ 200 毫秒 | 200～500 毫秒 | > 500 毫秒 |
| CLS | 视觉稳定 | ≤ 0.1 | 0.1～0.25 | > 0.25 |

按第 75 百分位统计，移动端和桌面端分开评估，三项都达到“良好”才算通过。

**来源：** web.dev Core Web Vitals、web.dev LCP、web.dev INP、web.dev CLS

### 以真实用户数据为准 【推荐】

**适用：** 线上服务

是否达标看线上真实用户数据，采集方式见 [可观测性](observability.md)。Lighthouse、浏览器 Performance 面板和 React、Vue DevTools 用于定位瓶颈，不凭感觉优化。Lighthouse 测不了 INP，只能用 TBT 作为替代参考。

**原因：** 实验室的设备和网络固定，看不到真实用户的分布和真实交互。

**来源：** web.dev Core Web Vitals

### 性能改动前后用同一方法测量 【必须】

改动前后使用同一环境、同一数据和同一步骤，并记录实际数字。做法见 [性能](../general/performance.md)。

**来源：** 团队

## 加载优化

### LCP 资源要能被尽早发现 【推荐】

LCP 图片直接写在初始 HTML 中。只在 CSS 或 JavaScript 中引用时，用 `preload` 提前加载。给它设置 `fetchpriority="high"`，这个属性只用在一两张图片上。

反例：

```html
<div class="hero" data-bg="/hero.avif"></div>
```

正例：

```html
<img src="/hero.avif" width="1200" height="600" fetchpriority="high" alt="产品主图">
```

**原因：** 浏览器要等脚本或样式执行后才发现图片，加载延迟会直接计入 LCP。

**来源：** web.dev 优化 LCP

### 首屏图片不懒加载 【推荐】

`loading="lazy"` 只用于初始视口外的图片，可能进入首屏的图片保持默认加载方式。

**来源：** web.dev 优化 LCP、web.dev 图片懒加载

### 大型依赖按需加载 【推荐】

图表、编辑器这类大型依赖在使用时动态加载，第三方库只导入用到的模块。路由的分割见 [routing.md](routing.md)。

**来源：** web.dev 代码分割

### 减少阻塞渲染的资源 【推荐】

`<head>` 中不放同步脚本，改用 `defer`。缩小阻塞渲染的样式表。非首屏的嵌入内容滚动到附近时再加载。`preconnect` 只用于最关键的第三方来源，次要来源用 `dns-prefetch`。

**来源：** web.dev 优化 LCP、web.dev 第三方脚本加载

## 渲染优化

### 为后到的内容预留空间 【推荐】

图片尺寸见 [响应式与资源](responsive-and-assets.md)。广告、嵌入内容和动态横幅用 `min-height` 或 `aspect-ratio` 占位。没有内容返回时保留占位，折叠同样会造成偏移。动画用 `transform`，不用 `top`、`left` 这类触发布局的属性。

**来源：** web.dev 优化 CLS

### 长列表用虚拟滚动或分页 【推荐】

列表数据量大时（通常上千条），只渲染可见部分，或者分页。

**原因：** 一次渲染大量节点会造成明显卡顿。

**来源：** web.dev 长列表虚拟化

### 不在同一任务中交替读写布局 【推荐】

先集中读取，再集中写入。

**原因：** 修改样式后立刻读取布局值会强制同步布局。

**来源：** web.dev 优化 INP

### 控制 DOM 规模 【推荐】

减少嵌套层级，交互时再添加元素。屏幕外的大块内容可用 `content-visibility` 推迟渲染。

**来源：** web.dev 优化 INP

## 长任务

### 拆分长任务 【推荐】

超过 50 毫秒的任务是长任务。把工作分批，累计约 50 毫秒后让出主线程，不要每个小任务都让出。优先用 `scheduler.yield()`，不支持时退到 `setTimeout`。

```ts
function yieldToMain(): Promise<void> {
  if (globalThis.scheduler?.yield) {
    return scheduler.yield();
  }
  return new Promise((resolve) => setTimeout(resolve, 0));
}

async function processAll(items: Item[]) {
  let deadline = performance.now() + 50;
  for (const item of items) {
    processItem(item);
    if (performance.now() >= deadline) {
      await yieldToMain();
      deadline = performance.now() + 50;
    }
  }
}
```

**原因：** 主线程被占用时，用户输入得不到响应，INP 变差。

**来源：** web.dev 长任务优化

### 事件回调先更新界面 【推荐】

回调只做更新界面必需的工作。保存、统计这类工作放到下一个任务。

**来源：** web.dev 优化 INP

### 计算密集的工作放到 Web Worker 【推荐】

解析大文件、复杂计算这类不操作 DOM 的工作放到 Worker，避免阻塞主线程。

## 缓存

### 带 hash 的资源长期缓存，HTML 每次验证 【必须】

**适用：** 线上服务

文件名带内容 hash 的静态资源设置一年缓存。HTML 入口文件名不变，每次使用前重新验证。发布时的配置见 [构建与兼容](build-and-compat.md)。

| 资源 | 响应头 |
|---|---|
| 带 hash 的静态资源 | `Cache-Control: max-age=31536000, immutable` |
| 静态 HTML 入口 | `Cache-Control: no-cache` |
| 含个人数据的 SSR 页面 | `Cache-Control: no-store`，见 [Web 风险](../backend/security/web-risks.md) |

**原因：** HTML 被长期缓存后，用户拿不到新版本。

**来源：** web.dev HTTP 缓存

### 保持页面可进入往返缓存 【推荐】

**适用：** 线上服务

不使用 `unload` 事件，改用 `pagehide`。`beforeunload` 只在有未保存内容时注册，保存后移除。`no-store` 只用于含个人数据或其他敏感数据的页面。在 `pageshow` 中根据 `event.persisted` 刷新过期数据。

**原因：** 往返缓存让前进和后退几乎瞬时完成，也避免重复的布局偏移。

**来源：** web.dev 往返缓存

## 性能预算

### 按页面类型设定预算并进入 CI 【推荐】

**适用：** 线上服务、多人协作

同时限制资源大小和面向用户的时间指标，不同类型的页面分开设定，比如商品页限制 JavaScript 体积，搜索页限制图片体积。超出预算时构建失败，优化或移除资源，不直接放宽预算。

```js
module.exports = {
  ci: {
    collect: { numberOfRuns: 5 },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
      },
    },
  },
};
```

**工具：** Lighthouse CI

**来源：** web.dev 性能预算、web.dev Lighthouse CI
