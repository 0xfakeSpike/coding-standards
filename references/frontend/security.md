# 前端安全

本文只讲浏览器端要做的事。CSRF 服务端防护和安全响应头见 [Web 风险](../backend/security/web-risks.md)，Cookie 属性见 [认证](../backend/security/authentication.md)，`eval` 和原型污染见 [TypeScript 安全](../languages/typescript/security.md)，依赖漏洞扫描也在该文件。

## XSS

### 优先用框架转义和文本写入 【必须】

不可信数据用框架默认的模板输出，或写入 `textContent`，不写入 `innerHTML`。

反例：

```ts
element.innerHTML = comment;
```

正例：

```ts
element.textContent = comment;
```

**来源：** OWASP XSS 防护速查表、OWASP DOM XSS 防护速查表

### 不把未净化的内容写入 HTML 【必须】

`dangerouslySetInnerHTML`、`v-html`、`innerHTML` 都是 XSS 入口。确需渲染富文本时，先用 DOMPurify 净化。

```tsx
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(html) }} />
```

净化是写入 DOM 前的最后一步，净化后再修改内容，净化就失效了。净化库要及时升级。在服务端运行时使用最新的 jsdom，不用 happy-dom。

**工具：** `eslint-plugin-react` 的 `no-danger`、`eslint-plugin-vue` 的 `no-v-html`

**来源：** OWASP XSS 防护速查表、DOMPurify 文档

### 校验用户提供的 URL 协议 【必须】

写入 `href`、`src` 前解析 URL，只放行白名单协议。框架对 `javascript:` 的拦截不能代替校验。

```ts
const ALLOWED_PROTOCOLS = new Set(['http:', 'https:']);

function toSafeHref(input: string): string | undefined {
  try {
    const url = new URL(input, window.location.origin);
    return ALLOWED_PROTOCOLS.has(url.protocol) ? url.href : undefined;
  } catch {
    return undefined;
  }
}
```

**来源：** OWASP XSS 防护速查表

### 不把不可信数据写入事件属性和样式源码 【必须】

不用 `setAttribute` 写 `onclick` 这类属性，事件用 `addEventListener` 绑定。不把不可信数据拼进 CSS 选择器或样式块。

**原因：** 事件属性的值会被当作代码执行，编码也挡不住。

**来源：** OWASP DOM XSS 防护速查表、OWASP XSS 防护速查表

### 用 Trusted Types 收紧写入点 【推荐】

在 CSP 中设置 `require-trusted-types-for 'script'`，让写入点只接受策略生成的值。

**来源：** MDN

## CSP

### 所有 HTML 响应下发 CSP 【必须】

**适用：** 线上服务

策略至少包含 `frame-ancestors`、`object-src` 和 `base-uri`，响应头要求见 [Web 风险](../backend/security/web-risks.md)。

### 脚本使用严格 CSP 【推荐】

优先用基于 nonce 或 hash 的策略，不维护域名白名单。nonce 每次响应重新生成，需要动态生成 HTML。纯静态部署用 hash。代码不依赖 `unsafe-inline` 和 `unsafe-eval`：内联事件属性和 `javascript:` 链接改为脚本绑定，不用字符串执行代码。CSP 是纵深防御，不能代替上面的 XSS 规则。

```http
Content-Security-Policy: script-src 'nonce-{RANDOM}' 'strict-dynamic'; object-src 'none'; base-uri 'none'; frame-ancestors 'none'
```

**来源：** web.dev 严格 CSP、OWASP 内容安全策略速查表、MDN

### 先用 Report-Only 上线 【推荐】

新策略先通过 `Content-Security-Policy-Report-Only` 响应头观察违规，修完再强制执行。`<meta>` 不支持 Report-Only，也不支持 `frame-ancestors`。

**来源：** web.dev 严格 CSP、OWASP 内容安全策略速查表

## CSRF

### Cookie 认证的写请求按后端选定的 CSRF 方案处理 【必须】

在统一的请求层处理，不逐个接口处理。后端用令牌方案时，把令牌写入自定义请求头，令牌不放进 URL。后端用 Fetch Metadata 加 Origin 校验时，前端不需要令牌。服务端防护见 [Web 风险](../backend/security/web-risks.md)。

```ts
await fetch('/api/orders', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'X-CSRF-Token': csrfToken },
  body: JSON.stringify(order),
});
```

**来源：** OWASP CSRF 速查表

### 跨域请求才带 `credentials: 'include'` 【推荐】

`fetch` 默认只在同源请求中带 Cookie。只有确实要跨域携带凭证的接口才改成 `include`，不要全局设置。

## 凭证与信任边界

### 登录态优先用 `HttpOnly` Cookie 【推荐】

使用带 `HttpOnly`、`Secure`、`SameSite` 的 Cookie。这时要同时防范 CSRF。

**原因：** `localStorage` 中的 Token 在发生 XSS 时可以被直接读取。

**来源：** OWASP HTML5 安全速查表、MDN

### Web Storage 不存敏感信息 【必须】

`localStorage`、`sessionStorage` 和 IndexedDB 不存会话标识、令牌和个人信息。

**例外：** 必须用 Bearer 令牌时，令牌只放在内存中。

**来源：** OWASP HTML5 安全速查表

### 前端不放密钥，权限以后端为准 【必须】

打包产物对所有人可见，需要密钥的调用走后端，环境变量见 [构建与兼容](build-and-compat.md)。前端隐藏按钮只是体验优化，不是安全措施。

## 第三方脚本

### 第三方脚本按需异步加载 【推荐】

用 `async` 或 `defer` 加载，只在需要的页面引入。

**来源：** web.dev 第三方脚本加载、OWASP 第三方 JavaScript 管理速查表

### CDN 脚本配置 SRI 【推荐】

**适用：** 线上服务

从 CDN 加载的脚本配置 `integrity` 和 `crossorigin`，并使用固定版本的地址。内容会变动而无法配置 SRI 的关键脚本，改为自托管。

```html
<script src="https://cdn.example.com/lib@1.2.3/lib.min.js" integrity="sha384-{HASH}" crossorigin="anonymous"></script>
```

**原因：** CDN 内容被篡改后，脚本会以页面权限运行。

**来源：** MDN、OWASP 第三方 JavaScript 管理速查表

### 限制第三方能看到的数据 【推荐】

用自己定义的数据层向标签管理器提供数据，不让它任意读取 DOM。高风险的第三方代码放进不同域名的 `sandbox` iframe，通过 `postMessage` 通信。

**来源：** OWASP 第三方 JavaScript 管理速查表

## Clickjacking

### 确认页面不能被外站嵌入 【必须】

**适用：** 线上服务

用响应头同时设置 CSP 的 `frame-ancestors` 和 `X-Frame-Options`，取值见 [Web 风险](../backend/security/web-risks.md)。写在 `<meta>` 中无效，JavaScript 防嵌入脚本可以被绕过，都不能作为防护手段。需要被嵌入的页面只列出可信来源。

**来源：** OWASP 点击劫持防护速查表、MDN

## Source Map

### Source Map 不公开部署 【推荐】

**适用：** 线上服务

生产构建的 `build.sourcemap` 不用 `'inline'`，也不用 `true` 直接部署。Source Map 只上传到错误监控平台，上传后从部署目录删除，或在服务器上禁止访问 `.map` 文件。Vite 的 `build.sourcemap: 'hidden'` 只去掉文件中的引用注释，`.map` 文件仍会生成。配置见 [可观测性](observability.md)。

**原因：** 公开的 Source Map 会泄露源码。

**来源：** Sentry JavaScript 文档、Vite 文档

## 跨窗口通信

### `postMessage` 校验来源 【必须】

发送时指定目标 origin，不用 `*`。接收时校验 `event.origin`，并校验消息内容。不需要接收外部消息时不注册监听。

```ts
window.addEventListener('message', (event) => {
  if (event.origin !== TRUSTED_ORIGIN) return;
  handleMessage(parseMessage(event.data));
});
```

**来源：** MDN

### 打开不可信站点时切断 `opener` 【推荐】

`target="_blank"` 打开不可信站点时加 `rel="noopener noreferrer"`，`window.open` 传入 `noopener`。

**来源：** OWASP HTML5 安全速查表、MDN
