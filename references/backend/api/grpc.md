# gRPC 规范

本文讲 Protobuf 接口定义和 gRPC 调用的约定。语言相关的用法见语言层，超时预算和重试退避的通用策略见 [resilience.md](../reliability/resilience.md)。

## Proto 风格

### 命名遵循官方风格 【必须】

| 对象 | 风格 | 示例 |
|---|---|---|
| 文件 | `lower_snake_case.proto` | `order_service.proto` |
| 包 | 点分隔的小写名称，不用反向域名 | `shop.orders` |
| 消息、服务、方法 | `TitleCase` | `CreateOrderRequest` |
| 字段、oneof | `lower_snake_case` | `order_id` |
| repeated 字段 | 复数 | `items` |
| 枚举类型 | `TitleCase` | `OrderStatus` |
| 枚举值 | `UPPER_SNAKE_CASE` | `ORDER_STATUS_PAID` |

下划线后面必须跟字母，不用作名称的首尾字符。字段名避免 `has_`、`get_`、`set_`、`clear_` 前缀和 `_value` 后缀，它们可能与生成代码冲突。

**工具：** `buf lint`

**来源：** Protobuf 风格指南

### 枚举有零值并带类型前缀 【必须】

第一个值为 0，以 `_UNSPECIFIED` 或 `_UNKNOWN` 结尾，不赋予业务含义。其他值用枚举名作前缀。

**示例：**

反例：

```proto
enum OrderStatus {
  PAID = 0;
  CANCELLED = 1;
}
```

正例：

```proto
enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PAID = 1;
  ORDER_STATUS_CANCELLED = 2;
}
```

**原因：** 枚举值的作用域不限于枚举本身，同一包内的同名值会冲突。

**来源：** Protobuf 风格指南、Protobuf 最佳实践

### 每个方法使用专属的请求和响应消息 【推荐】

消息按方法命名，不在多个方法之间共用，也不直接用资源消息作为请求。

**示例：**

```proto
service OrderService {
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
  rpc ListOrders(ListOrdersRequest) returns (ListOrdersResponse);
}
```

**来源：** Protobuf 风格指南、Google AIP

### 优先使用公共类型 【推荐】

时间用 `Timestamp`，时长用 `Duration`，部分更新用 `FieldMask`，金额用 `Money`。不用 `int64 timeout_millis` 这类自定义表示。

**来源：** Protobuf 最佳实践

### 接口消息和存储消息分开 【推荐】

RPC 消息不用于长期存储。两者的变化节奏不同，由转换层衔接。

**来源：** Protobuf 最佳实践

### 列表方法从一开始就分页 【推荐】

使用 `page_size`、`page_token` 和 `next_page_token`，不用流式响应代替分页。`next_page_token` 为空是到达末尾的唯一信号。令牌规则与 [rest.md](rest.md) 的游标相同。

**来源：** Google AIP

## 错误码

### 使用标准状态码 【必须】

业务失败通过 status 返回，不在成功响应里放错误字段。应用只使用 gRPC 定义的状态码。

| 场景 | 状态码 |
|---|---|
| 参数本身非法，与系统状态无关 | `INVALID_ARGUMENT` |
| 超出当前有效范围，比如读到文件末尾之后 | `OUT_OF_RANGE` |
| 资源不存在 | `NOT_FOUND` |
| 创建的资源已存在 | `ALREADY_EXISTS` |
| 无法识别调用方 | `UNAUTHENTICATED` |
| 调用方无权限 | `PERMISSION_DENIED` |
| 配额或资源耗尽 | `RESOURCE_EXHAUSTED` |
| 系统状态不满足，修复前不应重试 | `FAILED_PRECONDITION` |
| 并发冲突，应从读—改—写开始重试 | `ABORTED` |
| 暂时不可用，可重试本次调用 | `UNAVAILABLE` |
| 内部不变量被破坏 | `INTERNAL` |

**原因：** 客户端靠状态码决定是否重试，以及在哪一层重试。

**来源：** gRPC 文档

### 错误详情使用结构化消息 【推荐】

使用 Protobuf 时，用 `google.rpc.Status` 的 `details` 携带 `ErrorInfo`，`reason` 是稳定的 `UPPER_SNAKE_CASE` 标识，动态内容放 `metadata`。客户端不解析 `message`。

**例外：** 详情放在尾部元数据，代理和日志组件看不到，各语言的支持程度也不同。需要被这些组件观察的信息放在状态码里。

**来源：** gRPC 文档、Google AIP

### 错误信息不暴露内部细节 【必须】

未知错误返回 `INTERNAL` 和通用信息，完整上下文记录在日志中。不把堆栈放进错误详情。

## Deadline 和重试

### 客户端必须设置 Deadline 【必须】

gRPC 默认没有 Deadline，调用可能无限等待。根据网络延迟和处理耗时估算。

**来源：** gRPC 文档

### 向下游传播 Deadline 和取消 【推荐】

服务端调用下游时沿用入站调用的剩余时间，不重新设置更长的超时。部分语言需要显式开启传播。

**来源：** gRPC 文档

### 服务端响应取消 【推荐】

Deadline 到期后，框架取消调用，但不会中断业务处理。耗时任务要定期检查调用是否已取消，并停止自己启动的工作。

**来源：** gRPC 文档

### 超时后不假定操作失败 【必须】

`DEADLINE_EXCEEDED` 时，写操作可能已经成功。需要重试的写操作要有幂等保证，比如请求中带 `request_id`。

**来源：** gRPC 文档、Google AIP

### 重试策略按状态码显式配置 【推荐】

只自动重试幂等的一元调用，通常只配置 `UNAVAILABLE`，并限制次数和使用指数退避。`ABORTED` 在事务层重试，`INVALID_ARGUMENT`、`DEADLINE_EXCEEDED`、`INTERNAL` 不自动重试。

**来源：** gRPC 文档、Google AIP

## 流式

### 只在收益明确时使用流 【推荐】

长期持续的数据流才用流式。流建立后无法再做负载均衡，排障也更难，普通请求响应用一元调用。

**来源：** gRPC 文档

### 流式调用处理中断和背压 【必须】

**适用：** 线上服务

连接断开时，进行中的流失败，未发送的数据丢失。协议要让客户端能从断点恢复。写入遵守框架的流控，不在应用层无限缓存待发送消息。

**来源：** gRPC 文档

### 长连接的 keepalive 与服务方协商 【推荐】

客户端的 keepalive 间隔不要远低于 1 分钟，也不要在没有调用时开启。

**来源：** gRPC 文档

### 复用 Channel 和 Stub 【必须】

Channel 长期持有，不为每次调用新建。

**来源：** gRPC 文档

## 拦截器

### 横切逻辑放在拦截器 【推荐】

元数据、日志、指标、服务端认证和授权检查放在拦截器，业务规则留在方法实现。客户端认证优先用调用凭证（call credentials）。

**来源：** gRPC 文档

### 明确拦截器顺序 【推荐】

靠近网络的拦截器看到真实的网络调用，靠近应用的能看到应用行为。比如日志拦截器要放在缓存拦截器靠近应用的一侧，才能记录命中缓存的调用。认证要在依赖身份的拦截器之前执行。

**来源：** gRPC 文档

### 不用拦截器管理连接 【推荐】

拦截器按调用执行，TCP、端口和 TLS 在 Channel 或服务端配置中设置。

**来源：** gRPC 文档
