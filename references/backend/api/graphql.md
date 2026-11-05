# GraphQL 规范

本文讲 GraphQL 服务端的设计。授权模型见 [authorization.md](../security/authorization.md)，数据库侧的 N+1 见 [indexes-and-queries.md](../data/indexes-and-queries.md)。

## Schema 设计

### 按客户端用途设计 Schema 【推荐】

Schema 反映业务概念和客户端场景，不照搬表结构。Mutation 按具体动作命名，比如 `updateHumanName`，这样参数可以声明为非空，由 Schema 直接约束。

**来源：** GraphQL 官方文档

### 按字段判断可空性 【推荐】

字段默认可空。只有确定不会失败时才声明 `!`。非空字段出错时，`null` 向上传递到最近的可空父字段，一个小错误可能让整块数据消失。

**示例：**

```graphql
type Order {
  id: ID!
  status: OrderStatus!
  shippingEstimate: ShippingEstimate
}
```

**原因：** `shippingEstimate` 依赖外部服务，它失败时不应让整个订单为 `null`。

**来源：** GraphQL 官方文档

### 写操作只放在 Mutation 顶层字段 【必须】

Query 字段和嵌套字段不产生副作用。参数用 `input` 类型，返回被修改的对象，让客户端拿到最新状态。

**示例：**

```graphql
input CreateReviewInput {
  stars: Int!
  commentary: String
}

type Mutation {
  createReview(orderId: ID!, input: CreateReviewInput!): Review
}
```

**来源：** GraphQL 官方文档

### 不把多个 Mutation 当作事务 【必须】

同一请求的顶层 Mutation 串行执行，但中途失败时不会回滚已完成的字段。需要原子性时，设计成一个 Mutation，在业务层开启事务。

**来源：** GraphQL 官方文档

### 授权放在业务逻辑层 【必须】

认证在 GraphQL 执行前完成，身份放入请求上下文。权限判断放在业务层，不写在单个解析器中，否则同一数据的不同访问路径会漏检。每个节点都要做对象级检查，不能只检查入口字段。

**来源：** GraphQL 官方文档、OWASP GraphQL 速查表

## N+1 与 DataLoader

### 关联字段批量加载 【必须】

**适用：** 线上服务

解析器逐条查询关联数据时，一次请求会产生大量下游调用。用 DataLoader 把同一执行周期内的 key 合并为一次批量查询。

**示例：**

反例：

```sql
SELECT id, name FROM users WHERE id = 1;
SELECT id, name FROM users WHERE id = 2;
SELECT id, name FROM users WHERE id = 3;
```

正例：

```sql
SELECT id, name FROM users WHERE id IN (1, 2, 3);
```

**来源：** GraphQL 官方文档

### DataLoader 实例按请求创建 【必须】

批量函数返回的结果顺序和数量与输入 key 一一对应，缺失的 key 返回空值。

**原因：** 跨请求共享时，缓存可能把一个用户的数据返回给另一个用户，也会读到过期数据。

## 分页

### 可能很大的列表使用 Connection 【推荐】

遵循 Cursor Connections 规范。连接类型包含 `edges` 和非空的 `pageInfo`，边类型包含 `node` 和 `cursor`，向前翻页用 `first` 和 `after`，向后翻页用 `last` 和 `before`。`first` 和 `last` 设上限。

**示例：**

```graphql
type OrderConnection {
  edges: [OrderEdge]
  pageInfo: PageInfo!
}

type OrderEdge {
  cursor: String!
  node: Order
}

type PageInfo {
  hasPreviousPage: Boolean!
  hasNextPage: Boolean!
  startCursor: String
  endCursor: String
}

type User {
  orders(first: Int, after: String): OrderConnection
}
```

**来源：** GraphQL Cursor Connections、GraphQL 官方文档

### 游标不透明且排序稳定 【推荐】

客户端不解析游标。每页的排序保持一致，`last` 和 `before` 不反转边的顺序。

**来源：** GraphQL Cursor Connections

## 查询复杂度

### 执行前限制查询规模 【必须】

**适用：** 高风险

一条查询可以触发任意多的下游调用，需要在执行前拒绝过大的请求。

| 限制 | 说明 |
|---|---|
| 深度 | 限制嵌套层数，列表嵌套的限制更严 |
| 广度 | 限制顶层字段和别名数量 |
| 批量 | 限制单个 HTTP 请求内的操作数 |
| 成本 | 给字段设权重，估算总成本，超限时拒绝 |
| 超时 | 在应用层或基础设施层设超时 |

**来源：** GraphQL 官方文档、OWASP GraphQL 速查表

### 按查询成本限流 【推荐】

只统计 HTTP 请求数反映不出真实开销，要在业务层按成本扣减配额。别名和批量可以把多次登录尝试放进一个请求，敏感操作要按操作次数限流。

**来源：** GraphQL 官方文档、OWASP GraphQL 速查表

### 第一方客户端使用可信文档 【推荐】

构建时登记查询文档，运行时只接受已登记文档的哈希。这种方式不适用于公开 API。

**来源：** GraphQL 官方文档

### 非开发环境关闭内省和调试界面 【推荐】

面向第一方客户端的服务关闭内省。这只能增加探测难度，不能代替授权和查询限制。

**例外：** 公开 API 可以开放内省。

**来源：** GraphQL 官方文档、OWASP GraphQL 速查表

## 错误和传输

### 错误通过 errors 返回并带稳定标识 【推荐】

字段错误带 `message` 和 `path`，机器可读的标识放在 `extensions.code`。客户端不解析 `message`，并要能处理 `data` 和 `errors` 同时存在的部分响应。

**示例：**

```json
{
  "data": { "order": null },
  "errors": [
    {
      "message": "订单不存在",
      "path": ["order"],
      "extensions": { "code": "ORDER_NOT_FOUND" }
    }
  ]
}
```

**来源：** GraphQL 官方文档

### 生产环境隐藏错误细节 【必须】

解析器的未知异常统一转换为通用信息。校验错误中的字段建议也要关闭，否则即使关了内省，也能推断出 Schema。

**来源：** GraphQL 官方文档、OWASP GraphQL 速查表

### 使用正确的 HTTP 语义 【必须】

服务端支持 POST，GET 只能用于 query，Mutation 必须用 POST。`data` 非空时返回 2xx，请求未通过校验时通常返回 400。

**来源：** GraphQL 官方文档
