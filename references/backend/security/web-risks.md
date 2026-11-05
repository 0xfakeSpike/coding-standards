# Web 风险

本文讲浏览器和服务端交互中的常见风险。Cookie 属性见 [认证](authentication.md)，TLS 配置见 [加密与密钥](crypto-and-secrets.md)，回调和 Webhook 的签名与防重放见 [Webhook](../api/webhooks.md)。

## CORS

### 使用明确的来源白名单 【必须】

`Access-Control-Allow-Origin` 用固定值，或用 `Origin` 与白名单做完整字符串比较后返回。不反射任意 `Origin`，不做前缀或后缀匹配，不放行 `null`。

反例：

```http
Access-Control-Allow-Origin: <请求中的 Origin 原样返回>
Access-Control-Allow-Credentials: true
```

正例：

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Vary: Origin
```

**原因：** 反射来源加上凭证，任何网站都能以用户身份读取响应。

**来源：** OWASP ASVS、MDN

### 带凭证时不用通配符 【必须】

允许凭证时，来源、方法和请求头都写明确的值。`*` 只用于不含敏感信息的公开资源。动态返回来源时加 `Vary: Origin`，避免缓存串用。

**来源：** MDN、OWASP ASVS、OWASP HTTP 响应头速查表

### CORS 不是访问控制 【必须】

CORS 只限制浏览器读取响应，简单请求仍会到达服务端并执行。服务端必须独立完成认证、授权和 CSRF 防护。

**来源：** OWASP ASVS

### 暴露前端要读取的响应头 【推荐】

跨域时脚本只能读取 CORS 安全响应头。`Retry-After`、`ETag`、`Location` 和请求标识这类响应头要列入 `Access-Control-Expose-Headers`。

## CSRF

### 基于 Cookie 的认证必须防 CSRF 【必须】

优先用框架内置的防护。没有时，按下表选择，并用于所有改变状态的请求。

| 方案 | 适用场景 |
|---|---|
| 同步令牌 | 有服务端会话 |
| 签名的双提交 Cookie，令牌绑定会话 | 无状态服务 |
| Fetch Metadata 加 Origin 校验 | 只面向现代浏览器，全站 HTTPS |
| 自定义请求头 | 不使用表单的 API |

Fetch Metadata 的做法：非安全方法按 `Sec-Fetch-Site` 判断，放行 `same-origin`，拒绝 `cross-site`。`same-site` 默认拒绝，除非完全控制所有同站子域。`none` 只放行用户直接发起的顶层导航。请求不带该头时改用 `Origin` 或 `Referer` 校验，敏感接口直接拒绝。

**原因：** 浏览器会自动携带 Cookie，第三方页面能让用户在不知情时发起请求。

**例外：** 只用 `Authorization` 头传令牌、不用 Cookie 认证的接口不受 CSRF 影响。

**来源：** OWASP CSRF 速查表

### `GET` 不改变状态 【必须】

改变状态的操作用 `POST`、`PUT`、`PATCH`、`DELETE`。

**原因：** `SameSite=Lax` 和多数 CSRF 防护都放行安全方法。

**来源：** OWASP CSRF 速查表、OWASP ASVS

### `SameSite` 只是纵深防御 【推荐】

`SameSite` 按可注册域判断，同站的子域不受限制，各浏览器的默认值也不一致。除非完全控制所有同站子域，否则仍要配合令牌使用。不用未签名的简单双提交 Cookie。XSS 能绕过所有 CSRF 防护，两者不能互相替代。

**来源：** OWASP CSRF 速查表

## SSRF

### 目标已知时用白名单 【必须】

服务端只访问已知服务时，让用户选择标识，由服务端映射为地址，不接受 URL。必须接受时，校验协议、主机、端口和路径白名单，主机用严格比较。

**来源：** OWASP SSRF 速查表、OWASP ASVS

### 目标不确定时拦截内部地址 【必须】

像 Webhook 这样可访问任意外部地址的场景，只允许 `http` 和 `https`，解析全部 A 和 AAAA 记录，拒绝以下地址：

| 类型 | 示例 |
|---|---|
| 回环 | `127.0.0.0/8`、`::1` |
| 私有网段 | `10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`、`fc00::/7` |
| 链路本地 | `169.254.0.0/16`、`fe80::/10` |
| 云元数据 | `169.254.169.254`、`metadata.google.internal` |

用成熟的库解析 IP，防止十六进制、八进制这类编码绕过。

**来源：** OWASP SSRF 速查表

### 防重定向和 DNS 重绑定 【必须】

HTTP 客户端关闭自动跳转，确需跳转时逐跳重新校验。校验后直接连接已校验的 IP，同时保留原主机名用于 `Host`、SNI 和证书校验。校验和连接之间不能再做一次未经检查的 DNS 解析。

**原因：** 只校验域名时，攻击者可以让域名在校验后解析到内网地址。

**来源：** OWASP SSRF 速查表

### 在网络层限制出站 【推荐】

发起外部请求的组件放在受限网段，用防火墙或出站代理只开放必要目标。云主机启用要求令牌的元数据服务，比如 AWS 的 IMDSv2。不把抓取到的响应原样返回给用户。

**来源：** OWASP SSRF 速查表

## 开放重定向

### 不用用户输入作为跳转目标 【必须】

优先让用户传短标识，由服务端映射为地址。站内跳转只接受本地路径，用框架的校验方法。

反例：

```http
GET /login?next=https://evil.example/phish HTTP/1.1
```

正例：

```http
GET /login?next=orders HTTP/1.1
```

**原因：** 开放重定向可用于钓鱼，还能窃取 OAuth 授权码和令牌。

**来源：** OWASP 未校验重定向速查表、RFC 9700

### 外部目标解析后比对白名单 【必须】

用维护中的 URL 解析器，比对协议、规范化后的主机和端口。拒绝带用户信息的地址，比如 `https://good.example@evil.example`。不用前缀或后缀匹配。跳转时使用校验过的 URL，只允许 `http` 和 `https`。跳到站外时优先经过中间页，由用户确认。

**来源：** OWASP 未校验重定向速查表、OWASP ASVS

## 安全响应头

### 设置基础响应头 【必须】

**适用：** 线上服务

| 响应头 | 建议值 | 作用 |
|---|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | 强制 HTTPS，至少一年 |
| `X-Content-Type-Options` | `nosniff` | 禁止猜测内容类型 |
| `Content-Security-Policy` | HTML 响应必须含 `frame-ancestors`、`object-src`、`base-uri`，严格的 `script-src` 见 [前端安全](../../frontend/security.md) | 防点击劫持，限制插件和基准地址，收紧脚本来源 |
| `X-Frame-Options` | `DENY`，其他取值见下一条 | 为不支持 `frame-ancestors` 的浏览器兜底 |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | 限制来源信息外泄 |
| `Content-Type` | 与内容一致，带 `charset` | 防止错误渲染 |
| `Cache-Control` | 含个人数据或其他敏感数据的响应用 `no-store` | 防止被缓存 |

`no-cache` 不阻止存储，这类响应不能用它代替 `no-store`。HSTS 的 `includeSubDomains` 和 `preload` 配错后难以快速撤回，确认子域都支持 HTTPS 后再开启。

**工具：** OWASP ZAP

**来源：** OWASP HTTP 响应头速查表、OWASP ASVS

### 同时设置 `frame-ancestors` 和 `X-Frame-Options` 【必须】

**适用：** 线上服务

`frame-ancestors` 是主防护，`X-Frame-Options` 为旧浏览器兜底。两者都通过响应头设置，同时存在时，支持 `frame-ancestors` 的浏览器忽略 `X-Frame-Options`。

| 嵌入需求 | `frame-ancestors` | `X-Frame-Options` |
|---|---|---|
| 禁止嵌入（默认） | `'none'` | `DENY` |
| 仅同源嵌入 | `'self'` | `SAMEORIGIN` |
| 跨源可信嵌入 | 列出可信来源，比如 `https://partner.example` | 不发送，现代浏览器不支持 `ALLOW-FROM` |

**来源：** MDN

### 文档响应设置隔离头 【推荐】

HTML 响应设置 `Cross-Origin-Opener-Policy: same-origin`。用 `Permissions-Policy` 关闭不用的浏览器能力，比如 `geolocation=(), camera=()`。

**来源：** OWASP HTTP 响应头速查表、OWASP ASVS

### 清理多余的响应头 【推荐】

`X-XSS-Protection` 不设置或设为 `0`，靠 CSP 防护。不使用 `Expect-CT`。删除 `X-Powered-By`，`Server` 不带版本信息。

**来源：** OWASP HTTP 响应头速查表
