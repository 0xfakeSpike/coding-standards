# 后端层入口

**取舍：** 本层所有手段，包括重试、回退、补偿、Outbox 和熔断，都按 [不为罕见失败增加补偿机制](reliability/resilience.md) 判断是否值得。

## 接口

| 文件 | 何时读取 |
|---|---|
| [api/rest.md](api/rest.md) | 设计 HTTP 接口、错误格式、分页、幂等 |
| [api/contracts.md](api/contracts.md) | OpenAPI、Schema、代码生成、契约测试 |
| [api/grpc.md](api/grpc.md) | gRPC 服务和 Proto |
| [api/graphql.md](api/graphql.md) | GraphQL Schema 和查询控制 |
| [api/realtime.md](api/realtime.md) | WebSocket 和 SSE |
| [api/webhooks.md](api/webhooks.md) | 发送或接收 Webhook |

## 架构

| 文件 | 何时读取 |
|---|---|
| [architecture/layering.md](architecture/layering.md) | 分层、DTO、业务逻辑放在哪里 |
| [architecture/domain-modeling.md](architecture/domain-modeling.md) | 领域建模、是否使用 DDD |
| [architecture/service-boundaries.md](architecture/service-boundaries.md) | 模块和服务边界、服务拆分 |
| [architecture/configuration.md](architecture/configuration.md) | 配置、功能开关 |

## 可靠性

| 文件 | 何时读取 |
|---|---|
| [reliability/resilience.md](reliability/resilience.md) | 超时、重试、熔断、限流、降级 |
| [reliability/consistency.md](reliability/consistency.md) | 幂等、并发控制、分布式锁、Outbox、Saga |
| [reliability/operations.md](reliability/operations.md) | 健康检查、优雅启停、SLO、容量 |
| [reliability/performance.md](reliability/performance.md) | 性能剖析、连接池、压测 |

## 可观测性

| 文件 | 何时读取 |
|---|---|
| [observability/logging.md](observability/logging.md) | 日志字段、级别、脱敏 |
| [observability/metrics-and-alerts.md](observability/metrics-and-alerts.md) | 指标和告警 |
| [observability/tracing.md](observability/tracing.md) | 链路追踪 |

## 数据

| 文件 | 何时读取 |
|---|---|
| [data/schema-design.md](data/schema-design.md) | 设计表结构 |
| [data/indexes-and-queries.md](data/indexes-and-queries.md) | 索引、SQL、慢查询 |
| [data/transactions.md](data/transactions.md) | 事务、锁、死锁 |
| [data/migrations.md](data/migrations.md) | 数据库迁移 |
| [data/data-access.md](data/data-access.md) | model/repo 入口、ORM 边界 |
| [data/cache-and-redis.md](data/cache-and-redis.md) | 缓存和 Redis |
| [data/document-and-search.md](data/document-and-search.md) | MongoDB 和 Elasticsearch |

## 异步

| 文件 | 何时读取 |
|---|---|
| [async/messaging.md](async/messaging.md) | 消息队列 |
| [async/background-jobs.md](async/background-jobs.md) | 定时任务和后台任务 |

## 安全

| 文件 | 何时读取 |
|---|---|
| [security/authentication.md](security/authentication.md) | 登录、会话、JWT、OAuth |
| [security/authorization.md](security/authorization.md) | 权限、对象级校验、多租户 |
| [security/input-and-injection.md](security/input-and-injection.md) | 输入校验和注入防护 |
| [security/crypto-and-secrets.md](security/crypto-and-secrets.md) | 加密、密钥、TLS |
| [security/web-risks.md](security/web-risks.md) | CORS、CSRF、SSRF、安全响应头 |
| [security/file-handling.md](security/file-handling.md) | 文件上传和下载 |
| [security/privacy.md](security/privacy.md) | 个人信息和隐私合规 |
| [security/audit-and-abuse.md](security/audit-and-abuse.md) | 审计日志、限流、防刷 |

## AI 应用

| 文件 | 何时读取 |
|---|---|
| [ai/agents.md](ai/agents.md) | Agent、工具调用 |
| [ai/rag.md](ai/rag.md) | 检索增强生成 |

## 测试

| 文件 | 何时读取 |
|---|---|
| [testing.md](testing.md) | 后端集成测试、迁移测试、性能测试 |
