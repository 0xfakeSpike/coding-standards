# Project check

Use this file to check a whole project, or one directory, for problems that already exist. To review a change, follow the review workflow in [SKILL.md](../SKILL.md). Gate mode does not use this file.

A project check is not bound by the "3 to 6 rule files per task" limit, but read in the order below: filter by rule titles first, then read the body only of rules that match. Do not read every rule file through.

## Workflow

1. **Get to know the project.** Look at the dependency manifests, directory layout, deployment and CI configuration, `CLAUDE.md` and project docs. Establish the languages and frameworks, the middleware and external services in use, how it is deployed, and which scenarios apply (team project, production service, high risk). If the project does not declare its scenarios, infer them from the code and state your evidence in the result.
2. **List the rule files to read.** Use "Entry points" below to map the files and dependencies found in the project to rule files, and write them down as a list.
3. **Slice by check order.** Each slice is one check dimension plus its code scope, for example "security and data: `api/` and `migrations/`". The slices do not depend on each other. If you can use subagents, check them in parallel, giving each subagent only its slice's rule file paths and code scope. If you cannot, work through them in order and write the candidate list down as you go.
4. **Filter candidates by rule title.** For each slice, list the rule titles of the relevant files first, for example with `grep -n '^### ' file`. Look at [Required] before [Recommended]. Use the titles to search the code for suspicious places, read the body only of rules that match, and note candidate problems.
5. **Sweep the cross-cutting concerns.** Search rule titles across files with the keywords under "Cross-cutting concerns" below, go through the code for each concern, and add problems the slices missed.
6. **Verify each candidate.** Same as step 3 of the review workflow in SKILL.md.
7. **Finish with the baseline rules.** Same as step 4 of the review workflow in SKILL.md.

## Check order

Existing problems already affect production, so check the most serious consequences first. This differs from the order for reviewing a change:

1. **Security and data:** authentication and permissions, external input and injection, secrets, sensitive data, money, migrations and data loss.
2. **Reliability:** timeouts, retries, idempotency, transactions and concurrency, error handling.
3. **Design:** responsibilities and layers, duplicate implementations, unused code, convoluted structure.
4. **Tests:** whether critical paths have tests, and whether the tests fail when something breaks.
5. **Engineering and delivery:** dependencies, CI, containers, releases.
6. **Style:** check only whether a formatter and linter are set up. Do not list formatting issues one by one.

If there is not enough time to finish, report up to the slice you reached, in order.

## Entry points

Read for every project: [code-design.md](general/code-design.md), [responsibilities.md](general/responsibilities.md), [design-patterns.md](general/design-patterns.md), [error-handling.md](general/error-handling.md), [deletion.md](general/deletion.md), [testing-strategy.md](general/testing-strategy.md), and the files for each language in use, see [languages/index.md](languages/index.md).

Choose the other rule files by what appears in the project:

| Found in the project | Read |
|---|---|
| Dependency manifests, such as `package.json`, `go.mod`, `pyproject.toml` | The language layer's `project.md`, [dependencies.md](general/dependencies.md) |
| `Dockerfile`, Kubernetes manifests | [containers.md](general/containers.md) |
| CI configuration, such as `.github/workflows/` | [ci-cd.md](general/ci-cd.md) |
| Release scripts, CHANGELOG, feature flags | [release.md](general/release.md), [configuration.md](backend/architecture/configuration.md) |
| Environment variables, configuration files | [configuration.md](backend/architecture/configuration.md) |
| HTTP routes and handlers | [rest.md](backend/api/rest.md), [layering.md](backend/architecture/layering.md), [authorization.md](backend/security/authorization.md) |
| OpenAPI or JSON Schema files | [contracts.md](backend/api/contracts.md) |
| `.proto` files | [grpc.md](backend/api/grpc.md) |
| GraphQL schema | [graphql.md](backend/api/graphql.md) |
| WebSocket, SSE | [realtime.md](backend/api/realtime.md) |
| Sending or receiving webhooks | [webhooks.md](backend/api/webhooks.md) |
| Login, sessions, JWT, OAuth | [authentication.md](backend/security/authentication.md) |
| Encryption, signing, TLS configuration | [crypto-and-secrets.md](backend/security/crypto-and-secrets.md) |
| File upload and download | [file-handling.md](backend/security/file-handling.md) |
| SQL, commands, paths or external URLs built from strings | [input-and-injection.md](backend/security/input-and-injection.md), [web-risks.md](backend/security/web-risks.md) |
| Personal data | [privacy.md](backend/security/privacy.md) |
| Audit logs, rate limiting, abuse prevention | [audit-and-abuse.md](backend/security/audit-and-abuse.md) |
| Calls to external services | [resilience.md](backend/reliability/resilience.md) |
| Writes across services or tables | [consistency.md](backend/reliability/consistency.md) |
| Health checks, graceful startup and shutdown | [operations.md](backend/reliability/operations.md) |
| Connection pools, load tests, performance problems | [performance.md](backend/reliability/performance.md), [general/performance.md](general/performance.md) |
| Logs, metrics, tracing | [logging.md](backend/observability/logging.md), [metrics-and-alerts.md](backend/observability/metrics-and-alerts.md), [tracing.md](backend/observability/tracing.md) |
| Table schemas, ORM models | [schema-design.md](backend/data/schema-design.md), [data-access.md](backend/data/data-access.md) |
| SQL and query building | [indexes-and-queries.md](backend/data/indexes-and-queries.md), [transactions.md](backend/data/transactions.md) |
| A database migrations directory | [migrations.md](backend/data/migrations.md) |
| Redis or another cache | [cache-and-redis.md](backend/data/cache-and-redis.md) |
| MongoDB, Elasticsearch | [document-and-search.md](backend/data/document-and-search.md) |
| Message queue clients | [messaging.md](backend/async/messaging.md) |
| Scheduled jobs, task queues | [background-jobs.md](backend/async/background-jobs.md) |
| LLM calls, agents, tool calling | [agents.md](backend/ai/agents.md) |
| Vector search, RAG | [rag.md](backend/ai/rag.md) |
| Backend integration tests | [backend/testing.md](backend/testing.md) |
| Frontend component files | [components.md](frontend/components.md), then [react.md](frontend/react.md) or [vue.md](frontend/vue.md) by framework |
| `next.config.*`, `nuxt.config.*` | [nextjs.md](frontend/nextjs.md), [nuxt.md](frontend/nuxt.md) |
| Frontend directory layout, monorepo | [frontend/architecture.md](frontend/architecture.md) |
| Style files, Tailwind configuration | [css.md](frontend/css.md), [responsive-and-assets.md](frontend/responsive-and-assets.md) |
| HTML templates, page metadata | [html.md](frontend/html.md), [accessibility.md](frontend/accessibility.md) |
| Forms | [forms.md](frontend/forms.md) |
| Frontend request wrappers | [data-fetching.md](frontend/data-fetching.md) |
| Frontend route configuration | [routing.md](frontend/routing.md) |
| Stores, local storage | [state-management.md](frontend/state-management.md) |
| Translation resources | [i18n.md](frontend/i18n.md) |
| Frontend build configuration, such as Vite or webpack | [build-and-compat.md](frontend/build-and-compat.md), [frontend/performance.md](frontend/performance.md) |
| Frontend third-party scripts, CSP | [frontend/security.md](frontend/security.md) |
| Frontend error monitoring, analytics | [frontend/observability.md](frontend/observability.md) |
| Frontend tests | [frontend/testing.md](frontend/testing.md) |

## Cross-cutting concerns

The rules for these concerns are spread across many files. Search the rule titles under this handbook's `references/` directory by keyword, ignoring case, and join several keywords with `|` into one pattern, for example `grep -rniE '^### .*(retry|retries|backoff)' references/`. Keep only results relevant to the project's languages and domains.

| Concern | Title keywords |
|---|---|
| Sensitive data | `sensitive`, `redact`, `privacy`, `personal` |
| Secrets and credentials | `secret`, `credential`, `token`, `API key` |
| Permissions | `permission`, `authoriz`, `tenant` |
| External input | `validat`, `inject`, `sanitiz`, `escap` |
| Timeouts | `timeout` |
| Retries | `retry`, `retries`, `backoff` |
| Idempotency | `idempot` |
| Concurrency and locks | `concurren`, `race`, `lock` |
| Pagination | `paginat` |
| Time and time zones | `UTC`, `time zone` |
| Money | `money`, `amount`, `currenc` |

## Output

Use the review output format in SKILL.md, and add a check scope section after the review conclusion:

```markdown
## Check scope
- Scenarios: the scenarios the project declares, or the inferred ones and the evidence
- Checked: the slices and code scopes that were checked
- Not checked: the slices that were not checked, and why
```
