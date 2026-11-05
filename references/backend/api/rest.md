# REST 接口规范

本文讲 HTTP 资源接口的设计。契约管理见 [contracts.md](contracts.md)，鉴权见 [authentication.md](../security/authentication.md) 和 [authorization.md](../security/authorization.md)，服务端缓存见 [cache-and-redis.md](../data/cache-and-redis.md)。

## 资源建模

### URL 表示资源，动作由方法表达 【推荐】

路径用名词复数，小写 `kebab-case`，不带动词和结尾斜杠。

**示例：**

反例：

```text
POST /getOrderList
POST /order_items/delete
```

正例：

```text
GET    /orders
POST   /orders
GET    /orders/{id}
PATCH  /orders/{id}
DELETE /orders/{id}
GET    /users/{id}/orders
```

**来源：** Zalando RESTful API Guidelines

### 路径嵌套不超过 2 层 【推荐】

子资源的生命周期依附于父资源时才嵌套。层级过深时，让子资源成为顶层资源。

### 动作优先建模为资源 【推荐】

先考虑能否表达为状态字段或子资源。确实无法映射为增删改查时，使用 `POST /orders/{id}/cancel`。

**来源：** Zalando RESTful API Guidelines

## 方法和状态码

### 方法符合 HTTP 语义 【必须】

| 方法 | 用途 | 安全 | 幂等 |
|---|---|---|---|
| GET | 查询 | 是 | 是 |
| POST | 创建或执行动作 | 否 | 否 |
| PUT | 整体替换 | 否 | 是 |
| PATCH | 部分更新 | 否 | 视实现而定 |
| DELETE | 删除 | 否 | 是 |

GET 不产生业务副作用，也不带请求体。查询条件超出 URL 长度时改用 POST，并在契约中说明。

**原因：** 爬虫、预取和缓存都可能触发 GET，客户端和代理也会自动重发幂等请求。

**来源：** RFC 9110、Zalando RESTful API Guidelines

### PATCH 使用明确的补丁格式 【推荐】

每个接口只选一种，优先 `application/merge-patch+json`，需要操作数组元素时用 `application/json-patch+json`。

**来源：** Zalando RESTful API Guidelines

### 使用语义正确的状态码 【必须】

不要所有错误都返回 200，否则网关、监控和客户端重试逻辑都无法工作。

| 状态码 | 场景 | 要求 |
|---|---|---|
| 200 / 204 | 成功 / 成功且无响应体 | |
| 201 | 已创建 | 用 `Location` 指向新资源 |
| 202 | 已接受，尚未完成 | 提供查询进度的地址 |
| 400 | 请求无法解析 | |
| 401 | 未认证 | 必须带 `WWW-Authenticate` |
| 403 | 已认证但无权限 | |
| 404 | 资源不存在，或不愿暴露其存在 | |
| 405 | 资源不支持该方法 | 必须带 `Allow` |
| 409 | 与当前状态冲突 | 响应说明冲突原因，不带字段错误 |
| 412 | 条件请求的前提不成立 | |
| 415 | 媒体类型不支持 | |
| 422 | Schema、字段或业务校验失败 | 响应带 `errors` |
| 429 | 超过频率限制 | 带 `Retry-After` |
| 500 / 502 / 503 / 504 | 服务端或上游错误 | 503 可带 `Retry-After` |

**来源：** RFC 9110、Zalando RESTful API Guidelines

## 错误格式

### 错误响应使用 Problem Details 【推荐】

新项目的错误响应使用 RFC 9457，媒体类型为 `application/problem+json`。`type` 用绝对 URI，`status` 必须与实际状态码相同，校验明细和请求标识放在扩展字段。

**示例：**

```http
HTTP/1.1 409 Conflict
Content-Type: application/problem+json

{
  "type": "https://api.example.com/problems/order-already-paid",
  "title": "订单已支付",
  "status": 409,
  "detail": "订单已支付，无法取消",
  "requestId": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json

{
  "type": "https://api.example.com/problems/validation-failed",
  "title": "校验失败",
  "status": 422,
  "requestId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "errors": [
    { "pointer": "#/couponId", "detail": "优惠券已过期" }
  ]
}
```

**例外：** 项目已统一其他错误格式时，按下一条执行。

**来源：** RFC 9457、Zalando RESTful API Guidelines

### 已统一的响应包装继续沿用 【推荐】

项目已统一使用 `{ code, message, data }` 等包装时沿用，不在同一项目中混用两种错误格式。字段错误放在 `errors` 中，结构与 Problem Details 相同。HTTP 状态码仍必须反映真实结果，不能用 200 加业务码表示失败。

**示例：**

反例：

```http
HTTP/1.1 200 OK

{ "code": 40901, "message": "订单已支付", "data": null }
```

正例：

```http
HTTP/1.1 409 Conflict

{ "code": "ORDER_ALREADY_PAID", "message": "订单已支付，无法取消", "data": null }
```

### 错误有稳定的机器可读标识 【推荐】

Problem Details 中是 `type`，响应包装中是 `code`。客户端基于标识判断，不解析 `detail` 或 `message`。

**来源：** RFC 9457

### 错误响应不暴露内部细节 【必须】

不返回堆栈、SQL 和内部服务地址。通用原则见 [error-handling.md](../../general/error-handling.md)。

**来源：** RFC 9457、Zalando RESTful API Guidelines

### 错误中返回请求标识 【推荐】

在响应头或错误体返回 `requestId`，取值为入口生成的 `traceId`，便于用户反馈时定位日志。

## 字段约定

### 字段表示全项目统一 【推荐】

| 内容 | 要求 |
|---|---|
| 命名 | 全项目选定 `camelCase` 或 `snake_case`，与语言命名不同时用序列化配置映射 |
| 时间 | 表示时刻的字段用 RFC 3339 UTC，带 `Z`，如 `2026-11-05T08:30:00Z`；本地日期用 `YYYY-MM-DD`；未来事件用本地时间加 IANA 时区 |
| 金额 | 不用浮点数，全项目选定最小货币单位整数或十进制字符串，并且必须带 ISO 4217 币种 |
| 64 位整数 ID | 契约一律声明为 string，与当前数值大小无关，否则 JavaScript 解析时丢失精度 |
| 枚举 | 使用字符串，如 `"paid"`，不用 `3` |
| 空列表 | 返回 `[]`，不返回 `null` |

## 分页、过滤和排序

### 列表接口默认分页并限制上限 【必须】

**适用：** 线上服务

比如默认 20、最大 100。上限写入契约。超过上限时按上限处理，响应返回实际生效的 `limit`。负数返回 422。

**原因：** 不分页的列表会随数据增长拖垮服务。

**来源：** Google AIP、Zalando RESTful API Guidelines

### 大数据量或频繁变动的列表用游标分页 【推荐】

`OFFSET` 在深分页时性能差，数据变动时可能重复或遗漏。后台管理类小数据量可以用页码分页。

**示例：**

```text
GET /orders?status=paid&sort=-createdAt&limit=20&cursor=eyJpZCI6MTAwfQ
```

**来源：** Zalando RESTful API Guidelines

### 游标对客户端不透明 【推荐】

客户端不解析也不构造游标。翻页时过滤条件变化要返回 422。游标只表示位置，每次请求照常鉴权。最后一页不返回下一页游标。

**来源：** Google AIP、Zalando RESTful API Guidelines

### 不默认返回总数 【推荐】

复杂查询的计数成本很高。确有需要时单独提供，估算值要在契约中注明。

**来源：** Zalando RESTful API Guidelines

### 过滤和排序参数有明确语义 【推荐】

简单过滤用查询参数，多个条件之间是“与”关系。排序用 `sort`，`-` 前缀表示降序。每个参数写明比较方式和取值范围。

**来源：** Zalando RESTful API Guidelines

## 批量和长任务

### 批量接口声明原子性和上限 【推荐】

默认全部成功或全部失败。允许部分成功时返回 207，响应中逐项给出标识和状态。批量大小设上限。

**来源：** Google AIP、Zalando RESTful API Guidelines

### 长任务返回任务资源 【推荐】

请求可能超过网关超时时，返回 202 和 `Location`。任务资源包含状态、结果或错误，未完成时用 `Retry-After` 提示轮询间隔，完成后保留一段公开说明的时间。任务失败时，查询接口仍返回 200，失败写在任务的 `error` 字段。

**示例：**

```http
HTTP/1.1 202 Accepted
Location: /report-jobs/8f3a
Retry-After: 5
```

**来源：** Microsoft REST API Guidelines、Google AIP、Zalando RESTful API Guidelines

## 幂等和并发

### 关键写操作支持幂等 【必须】

**适用：** 高风险

支付、下单等操作由客户端传 `Idempotency-Key`。服务端保存 key、请求摘要和响应，有效期内相同 key 返回同一结果。key 记录与本地状态写入在同一事务内完成，需要调用外部服务时按 [transactions.md](../data/transactions.md) 的顺序执行。天然存在业务唯一键时，用唯一约束实现。

**原因：** 客户端在超时后无法判断请求是否成功，只能重试。

**来源：** Zalando RESTful API Guidelines、Google AIP

### 幂等键的异常情况有明确响应 【必须】

**适用：** 高风险

| 情况 | 响应 |
|---|---|
| 要求带 key 却缺失 | 400 |
| key 相同、请求摘要不同 | 422 |
| 同 key 的请求仍在执行 | 409 |

只回放已完成的结果。事务回滚的失败不留记录，客户端可以用同一个 key 重试。

### 并发更新使用乐观锁 【推荐】

通过 `ETag` 加 `If-Match`，或 `version` 字段。条件请求失败返回 412，`version` 冲突返回 409。

**来源：** RFC 9110、Zalando RESTful API Guidelines

## 缓存和限流

公开接口的限流见 [resilience.md](../reliability/resilience.md)。

### 默认不缓存，可缓存的接口显式声明 【推荐】

动态和用户相关的响应默认使用 `Cache-Control: no-store`，含个人数据的响应必须使用，见 [privacy.md](../security/privacy.md)。可缓存的接口声明 `Cache-Control`、`ETag` 和 `Vary`，并支持 `If-None-Match` 返回 304。

**来源：** RFC 9110、Zalando RESTful API Guidelines
