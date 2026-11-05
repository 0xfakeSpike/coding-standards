# 链路追踪

本文规定分布式追踪的接入、上下文传播、Span 规范和采样。规则以 OpenTelemetry 为基准，语言相关的接入方式见各语言文档。

**适用：** 线上服务（本文全部规则）

## 接入

### 使用 OpenTelemetry 【推荐】

新服务用 OpenTelemetry 产生追踪数据，不绑定某个厂商的 SDK。框架、HTTP 客户端和数据库驱动优先使用现成的自动埋点，业务关键步骤再手工补充。

### 统一服务标识 【推荐】

每个服务设置 `service.name`，和日志、指标的服务名一致。这样三种信号才能按来源关联。

**来源：** OpenTelemetry 日志规范

### 遵循语义约定 【推荐】

HTTP、数据库和消息等常见操作使用语义约定中已有的属性名，不自己发明。

**原因：** 名称统一后，不同服务和库产生的数据才能放在一起查询。

**来源：** OpenTelemetry 语义约定

## 上下文传播

### 跨服务调用传递上下文 【推荐】

让 `traceId` 贯穿调用链。默认使用 W3C Trace Context 的 `traceparent` 和 `tracestate` 头。所有中间层都要转发，否则链路会断开。

**示例：**

```http
GET /orders/1001 HTTP/1.1
Host: order.internal
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

**来源：** OpenTelemetry 上下文传播、W3C Trace Context

### 进程内不手工传递标识 【推荐】

上下文通过语言提供的上下文机制传递，不在业务函数的参数中传 `traceId`。线程池和异步任务切换时，要确认上下文没有丢失。

### 异步消息也要携带上下文 【推荐】

上下文写入消息属性，不写入业务载荷。生产者注入，消费者提取。

消费侧的 Span 通过链接（Link）关联消息的创建上下文。批量消费时一个 Span 对应多条消息，只能用链接表达。

**来源：** OpenTelemetry 消息 Span

### 不信任外部传入的上下文 【必须】

来自公网的 `traceparent` 和 Baggage 可能是伪造的。入口新建 trace，把浏览器传来的上下文作为 link 保存，不作为可信父节点，只信任内部调用方传入的值。

**原因：** 伪造的头可能污染追踪数据，或利用解析缺陷。

**来源：** OpenTelemetry 上下文传播、W3C Trace Context

### 不向外部系统发送内部上下文 【推荐】

调用第三方时，`traceId`、Baggage 可能暴露内部结构，要评估是否发送。

**来源：** OpenTelemetry 上下文传播

### Baggage 不放敏感数据 【必须】

Baggage 会传给所有下游，也可能被记录。不放凭据、API Key 和个人信息。`tracestate` 同样不放个人身份信息。

**来源：** OpenTelemetry 上下文传播、W3C Trace Context

## Span 规范

### Span 名称保持低基数 【推荐】

名称用操作的类别，不含 ID 等可变值。可变值放进属性。

**示例：**

| 名称 | 判断 |
|---|---|
| `get` | 反例：太笼统 |
| `GET /orders/1001` | 反例：含具体 ID |
| `GET /orders/{id}` | 正例 |
| `order.reserve_stock` | 正例 |

HTTP Span 使用 `{method} {route}`，没有低基数的路由时只用 `{method}`，不用原始路径。

**来源：** OpenTelemetry Trace API、OpenTelemetry HTTP Span

### 只为有意义的操作创建 Span 【推荐】

为跨进程调用、数据库操作和重要业务步骤创建 Span。不给每个函数创建。

**原因：** Span 太多会增加开销，也让链路难以阅读。

### 按语义约定设置状态 【推荐】

| 情况 | 状态 |
|---|---|
| 操作成功 | 保持 `Unset` |
| 操作失败 | `Error`，并设置 `error.type` |
| HTTP 服务端返回 4xx | `Unset` |
| HTTP 服务端返回 5xx | `Error` |
| HTTP 客户端收到 4xx 或 5xx | `Error` |
| 调用方主动取消 | `Unset` |

`Ok` 只在应用明确判定成功时使用，它会覆盖其他状态。

**来源：** OpenTelemetry Trace API、OpenTelemetry HTTP Span

### 已处理的错误不标记为失败 【推荐】

操作经过重试后成功，或错误已被妥善处理时，描述这个操作的 Span 不记录错误。

**来源：** OpenTelemetry 错误记录

### 自定义属性使用命名空间 【推荐】

属性名小写，用点分隔命名空间，单词之间用下划线，比如 `order.item_count`。不占用 `otel.` 和已有语义约定的命名空间，公司或应用自定义的属性带上自己的前缀。

**来源：** OpenTelemetry 命名

### Span 属性不放敏感数据 【必须】

Span 的属性和事件和日志适用同样的脱敏规则，见 [日志](logging.md)。不记录请求体、SQL 参数值和完整的带查询串 URL。

## 采样

### 先判断是否需要采样 【推荐】

数据量很小或只看聚合结果时，不需要采样。法规要求不能丢数据时也不能采样。流量大，并且大部分请求都正常时才考虑。

**来源：** OpenTelemetry 采样

### 按需求选择采样位置 【推荐】

| 方式 | 决策时机 | 优点 | 缺点 |
|---|---|---|---|
| 头部采样 | 请求开始时 | 简单、开销低 | 不能保证留下错误链路 |
| 尾部采样 | 收集完整链路后 | 可以保留错误和慢请求 | 有状态组件，运维成本高 |

默认从头部采样开始。确实需要保留错误和慢请求时再加尾部采样。

**来源：** OpenTelemetry 采样

### 一条链路的采样决定保持一致 【推荐】

基于 `traceId` 做确定性采样，下游沿用上游的决定，否则链路会缺少片段。

**来源：** OpenTelemetry 采样

### 采样率写入配置 【推荐】

采样率放在配置中，并能运行时调整，见 [配置](../architecture/configuration.md)。

### 指标不依赖采样后的链路 【推荐】

请求量和错误率来自指标，在采样之前统计。链路用于定位单次请求，见 [指标和告警](metrics-and-alerts.md)。
