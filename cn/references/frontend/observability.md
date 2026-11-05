# 前端可观测性

本文讲浏览器端的错误监控、性能采集和埋点。性能目标见 [性能](performance.md)，Source Map 的安全要求见 [安全](security.md)，个人数据的要求见 [隐私](../backend/security/privacy.md)，服务端日志和链路见 [后端规范](../backend/index.md)。

**适用：** 线上服务（本文全部规则）

## 错误监控

### 接入错误监控平台 【推荐】

线上接入 Sentry 这类平台，尽早初始化，捕获同步异常和未处理的 Promise 拒绝。自己实现时同时监听 `error` 和 `unhandledrejection`，前者收不到未处理的 Promise 拒绝。

**来源：** Sentry JavaScript 文档、MDN

### 错误绑定版本和环境 【推荐】

每次部署明确设置 `release`，用统一的名称设置 `environment`。

```ts
Sentry.init({
  dsn: import.meta.env.VITE_SENTRY_DSN,
  release: import.meta.env.VITE_RELEASE,
  environment: import.meta.env.MODE,
});
```

**原因：** 没有版本就无法判断错误由哪次发布引入。

**来源：** Sentry JavaScript 文档

### 构建时上传 Source Map 【推荐】

用构建插件在 CI 中上传，只处理生产构建。上传要先于错误发生。上传令牌放在 CI 的密钥中，不用 `VITE_` 前缀。上传后用插件的 `sourcemaps.filesToDeleteAfterUpload` 删除 `.map` 文件。

```ts
export default defineConfig({
  build: { sourcemap: 'hidden' },
  plugins: [
    sentryVitePlugin({
      org: 'example-org',
      project: 'example-web',
      authToken: process.env.SENTRY_AUTH_TOKEN,
      sourcemaps: { filesToDeleteAfterUpload: ['dist/assets/*.map'] },
    }),
  ],
});
```

**来源：** Sentry JavaScript 文档

### 过滤噪音，不靠采样控量 【推荐】

用 `ignoreErrors`、`allowUrls`、`beforeSend` 排除浏览器扩展和第三方脚本的错误。错误量过大时优先用平台的限流，SDK 的 `sampleRate` 会同时降低对真实问题的可见度。

**来源：** Sentry JavaScript 文档

### 只上报需要处理的异常 【推荐】

表单校验失败、用户取消请求这类预期内的结果不上报为错误。401、404 这类已处理的响应按 [数据请求](data-fetching.md) 处理。

### 上报前移除敏感数据 【必须】

用 `beforeSend` 和 `beforeBreadcrumb` 在离开浏览器前清理。用户只上报内部 ID。平台侧的清理发生在数据发送之后，不能代替这一步。

反例：

```ts
Sentry.setUser({ email: user.email });
```

正例：

```ts
Sentry.setUser({ id: user.id });
```

**来源：** Sentry JavaScript 文档

### 链路头只发给自己的后端 【推荐】

`tracePropagationTargets` 只列出自己的后端域名，避免把追踪数据发给第三方。跨域时确认后端 CORS 允许这些请求头。确认 SDK 发送的链路头与后端使用的格式一致，后端入口按不可信输入处理，见 [链路追踪](../backend/observability/tracing.md)。

**来源：** Sentry JavaScript 文档

## 性能监控

### 采集真实用户的 Core Web Vitals 【推荐】

用 `web-vitals` 库采集 LCP、INP 和 CLS。每个页面只注册一次回调。定位原因时用 `web-vitals/attribution`。

```ts
function send(metric: Metric) {
  navigator.sendBeacon('/analytics', JSON.stringify({
    name: metric.name,
    value: metric.value,
    id: metric.id,
    route: currentRouteTemplate(),
  }));
}

onLCP(send);
onINP(send);
onCLS(send);
```

**原因：** 重复注册会创建多余的观察器。

**来源：** web-vitals 文档、web.dev Core Web Vitals

### 考虑指标的上报时机 【推荐】

CLS 和 INP 在页面隐藏时才上报，从往返缓存恢复后会再次上报。用户没有交互时没有 INP。服务端按 `id` 去重，用最后的值。

**来源：** web-vitals 文档

### 按路由模板和设备分组 【推荐】

按第 75 百分位看数据，移动端和桌面端分开。路由用模板，比如 `/orders/:id`，不用实际地址。

**原因：** 实际地址的基数过高，还可能带上用户数据。

**来源：** web.dev Core Web Vitals

### 单页应用单独看路由切换 【推荐】

路由切换不算完整页面导航，Core Web Vitals 反映不了它。需要时单独记录路由切换耗时。

**来源：** web-vitals 文档

### 性能追踪从低采样率开始 【推荐】

`tracesSampleRate` 先设低值，了解流量后再调整。`tracesSampler` 可以按操作设置不同比例。

**来源：** Sentry JavaScript 文档

## 埋点规范

### 事件有统一定义 【推荐】

事件名和属性集中定义类型。命名用固定格式，比如 `对象_动作`。

反例：

```ts
track('click1', { p: 2 });
```

正例：

```ts
track('order_submitted', { itemCount: 2, paymentType: 'card' });
```

**原因：** 没有统一定义的事件很快失去分析价值。

### 通过统一入口上报 【推荐】

业务代码只调用项目封装的 `track`，不直接调用第三方 SDK。事件和属性用类型约束。同意状态判断和字段过滤放在这一层。

**原因：** 分散的调用无法统一治理，更换平台时要改遍全部代码。

### 埋点不影响业务 【必须】

上报失败不抛错，也不阻塞用户操作。不延迟页面跳转等待上报，离开页面时用 `navigator.sendBeacon()`。

**来源：** MDN

### 环境隔离 【推荐】

开发和测试环境的事件不进入生产数据。

## 隐私要求

### 采集前确认依据 【必须】

需要同意的分析和第三方 SDK，在用户同意后才加载。撤回同意后停止采集。按用途记录同意，监控和分析平台作为委托处理方纳入数据清单，见 [隐私](../backend/security/privacy.md)。

### 不采集直接标识符和输入内容 【必须】

事件属性、错误信息和面包屑中不放姓名、手机号、邮箱、令牌和表单输入值。上报前从 URL 中去掉查询参数，或只上报路由模板。用户标识用内部 ID 或假名，它们仍属于个人信息，按 [隐私](../backend/security/privacy.md) 管理。

### 会话录制保持默认遮盖 【必须】

保持 `maskAllText`、`maskAllInputs` 和 `blockAllMedia` 开启，确有需要时用 `unmask`、`unblock` 精确放开。遮盖属性写在头像、输入框这类公共组件上。CSS 背景图不在媒体拦截范围内，要手动加 `data-sentry-block`。修改配置或升级 SDK 后重新验证。

**来源：** Sentry JavaScript 文档
