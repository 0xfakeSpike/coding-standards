# 项目检查

本文用于检查整个项目或某个目录中已有的问题。审查一次改动按 [SKILL.md](../SKILL.md) 的审查流程，门禁模式不用本文。

项目检查不受“一次读 3～6 个细则文件”的限制，但要按下面的顺序读：先用规则标题筛选，再只读命中规则的正文，不通读所有细则。

## 流程

1. **摸清项目。** 看依赖清单、目录结构、部署和 CI 配置、`CLAUDE.md` 和项目文档，确定语言和框架、用到的中间件和外部服务、部署方式，以及适用场景（多人协作、线上服务、高风险）。项目没有声明适用场景时，根据代码推测，并在结果里写明推测依据。
2. **列出要读的细则。** 对照下面的“检查入口”，把项目中出现的文件和依赖对应到细则文件，列成清单。
3. **按检查顺序切片。** 每一片是一个检查维度加对应的代码范围，比如“安全和数据：`api/` 和 `migrations/`”。各片互不依赖。支持子 agent 时并行检查，每个子 agent 只拿到自己这片的细则路径和代码范围。不支持时按顺序做，候选清单随时写下来。
4. **用规则标题筛选候选。** 每片先列出相关细则的规则标题，比如 `grep -n '^### ' 文件`，先看【必须】再看【推荐】。按标题到代码里搜可疑位置，只对命中的规则读正文，记下候选问题。
5. **按横切关注点补查。** 用下面“横切关注点”的关键词跨文件搜规则标题，把每个关注点在代码里过一遍，补上按片检查时漏掉的问题。
6. **逐条验证。** 同 SKILL.md 审查流程的第 3 步。
7. **用底线规则收尾。** 同 SKILL.md 审查流程的第 4 步。

## 检查顺序

存量问题已经在影响线上，所以先查后果最重的，和审查改动时的顺序不同：

1. **安全和数据：** 认证和权限、外部输入和注入、密钥、敏感数据、资金、迁移和数据丢失。
2. **可靠性：** 超时、重试、幂等、事务和并发、错误处理。
3. **设计：** 职责和分层、重复实现、无用代码、结构是否绕。
4. **测试：** 关键路径有没有测试，出错时测试能不能失败。
5. **工程和交付：** 依赖、CI、容器、发布。
6. **风格：** 只检查是否接入了 formatter 和 lint，不逐条列出格式问题。

时间不够查完时，按顺序做到哪一片就报告到哪一片。

## 检查入口

每个项目都读：[code-design.md](general/code-design.md)、[responsibilities.md](general/responsibilities.md)、[design-patterns.md](general/design-patterns.md)、[error-handling.md](general/error-handling.md)、[deletion.md](general/deletion.md)、[testing-strategy.md](general/testing-strategy.md)，以及所用语言目录下的文件，见 [languages/index.md](languages/index.md)。

其余细则按项目中出现的内容选：

| 项目中出现 | 读取 |
|---|---|
| 依赖清单，比如 `package.json`、`go.mod`、`pyproject.toml` | 语言层 `project.md`、[dependencies.md](general/dependencies.md) |
| `Dockerfile`、Kubernetes 清单 | [containers.md](general/containers.md) |
| CI 配置，比如 `.github/workflows/` | [ci-cd.md](general/ci-cd.md) |
| 发布脚本、CHANGELOG、功能开关 | [release.md](general/release.md)、[configuration.md](backend/architecture/configuration.md) |
| 环境变量、配置文件 | [configuration.md](backend/architecture/configuration.md) |
| HTTP 路由和处理函数 | [rest.md](backend/api/rest.md)、[layering.md](backend/architecture/layering.md)、[authorization.md](backend/security/authorization.md) |
| OpenAPI 或 JSON Schema 文件 | [contracts.md](backend/api/contracts.md) |
| `.proto` 文件 | [grpc.md](backend/api/grpc.md) |
| GraphQL Schema | [graphql.md](backend/api/graphql.md) |
| WebSocket、SSE | [realtime.md](backend/api/realtime.md) |
| 发送或接收 Webhook | [webhooks.md](backend/api/webhooks.md) |
| 登录、会话、JWT、OAuth | [authentication.md](backend/security/authentication.md) |
| 加解密、签名、TLS 配置 | [crypto-and-secrets.md](backend/security/crypto-and-secrets.md) |
| 文件上传和下载 | [file-handling.md](backend/security/file-handling.md) |
| 拼接 SQL、命令、路径或外部 URL | [input-and-injection.md](backend/security/input-and-injection.md)、[web-risks.md](backend/security/web-risks.md) |
| 个人信息 | [privacy.md](backend/security/privacy.md) |
| 审计日志、限流、防刷 | [audit-and-abuse.md](backend/security/audit-and-abuse.md) |
| 调用外部服务 | [resilience.md](backend/reliability/resilience.md) |
| 跨服务或跨表的写操作 | [consistency.md](backend/reliability/consistency.md) |
| 健康检查、优雅启停 | [operations.md](backend/reliability/operations.md) |
| 连接池、压测、性能问题 | [performance.md](backend/reliability/performance.md)、[general/performance.md](general/performance.md) |
| 日志、指标、链路追踪 | [logging.md](backend/observability/logging.md)、[metrics-and-alerts.md](backend/observability/metrics-and-alerts.md)、[tracing.md](backend/observability/tracing.md) |
| 表结构、ORM 模型 | [schema-design.md](backend/data/schema-design.md)、[data-access.md](backend/data/data-access.md) |
| SQL 和查询构造 | [indexes-and-queries.md](backend/data/indexes-and-queries.md)、[transactions.md](backend/data/transactions.md) |
| 数据库迁移目录 | [migrations.md](backend/data/migrations.md) |
| Redis 或其他缓存 | [cache-and-redis.md](backend/data/cache-and-redis.md) |
| MongoDB、Elasticsearch | [document-and-search.md](backend/data/document-and-search.md) |
| 消息队列客户端 | [messaging.md](backend/async/messaging.md) |
| 定时任务、任务队列 | [background-jobs.md](backend/async/background-jobs.md) |
| 调用大模型、Agent、工具调用 | [agents.md](backend/ai/agents.md) |
| 向量检索、RAG | [rag.md](backend/ai/rag.md) |
| 后端集成测试 | [backend/testing.md](backend/testing.md) |
| 前端组件文件 | [components.md](frontend/components.md)，再按框架读 [react.md](frontend/react.md) 或 [vue.md](frontend/vue.md) |
| `next.config.*`、`nuxt.config.*` | [nextjs.md](frontend/nextjs.md)、[nuxt.md](frontend/nuxt.md) |
| 前端目录结构、monorepo | [frontend/architecture.md](frontend/architecture.md) |
| 样式文件、Tailwind 配置 | [css.md](frontend/css.md)、[responsive-and-assets.md](frontend/responsive-and-assets.md) |
| HTML 模板、页面元信息 | [html.md](frontend/html.md)、[accessibility.md](frontend/accessibility.md) |
| 表单 | [forms.md](frontend/forms.md) |
| 前端请求封装 | [data-fetching.md](frontend/data-fetching.md) |
| 前端路由配置 | [routing.md](frontend/routing.md) |
| Store、本地存储 | [state-management.md](frontend/state-management.md) |
| 多语言资源 | [i18n.md](frontend/i18n.md) |
| 前端构建配置，比如 Vite、webpack | [build-and-compat.md](frontend/build-and-compat.md)、[frontend/performance.md](frontend/performance.md) |
| 前端第三方脚本、CSP | [frontend/security.md](frontend/security.md) |
| 前端错误监控、埋点 | [frontend/observability.md](frontend/observability.md) |
| 前端测试 | [frontend/testing.md](frontend/testing.md) |

## 横切关注点

下面这些关注点的规则分散在多个文件中。在本手册的 `references/` 目录下用关键词搜规则标题，多个关键词用 `|` 连成一个正则，比如 `grep -rnE '^### .*(重试|退避)' references/`。只看与项目语言和领域相关的结果。

| 关注点 | 标题关键词 |
|---|---|
| 敏感数据 | `敏感`、`脱敏`、`隐私`、`个人` |
| 密钥和凭证 | `密钥`、`凭证`、`Token`、`令牌` |
| 权限 | `权限`、`授权`、`鉴权`、`租户` |
| 外部输入 | `校验`、`注入`、`净化`、`转义` |
| 超时 | `超时` |
| 重试 | `重试`、`退避` |
| 幂等 | `幂等` |
| 并发和锁 | `并发`、`竞态`、`锁` |
| 分页 | `分页` |
| 时间和时区 | `UTC`、`时区` |
| 金额 | `金额`、`货币` |

## 输出

按 SKILL.md 的审查输出格式输出，在审查结论之后加一节检查范围：

```markdown
## 检查范围
- 适用场景：项目声明的场景，或推测的场景和依据
- 已检查：检查过的片和代码范围
- 未检查：没检查的片和原因
```
