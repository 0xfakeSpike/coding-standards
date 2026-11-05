# 指标和告警

本文规定监控什么、指标如何命名、如何控制基数和设计告警。SLO 的定义见 [运维就绪](../reliability/operations.md)。命名示例按 Prometheus 约定。

**适用：** 线上服务（本文全部规则）

## 监控什么

### 服务用 RED，资源用 USE 【推荐】

| 方法 | 对象 | 指标 |
|---|---|---|
| RED | 服务和接口 | 请求速率（Rate）、错误数（Errors）、耗时分布（Duration） |
| USE | CPU、内存、磁盘、连接池、线程池 | 使用率（Utilization）、饱和度（Saturation）、错误数（Errors） |

RED 回答“用户体验如何”，USE 回答“哪个资源是瓶颈”。饱和度通常体现为排队长度。

**来源：** Grafana RED 方法、Gregg USE 方法、Google SRE 监控

### 按服务类型选择核心指标 【推荐】

| 类型 | 核心指标 |
|---|---|
| 在线服务 | 请求数、错误数、延迟（P95、P99）、进行中的请求数 |
| 异步处理 | 进出数量、队列积压、最老消息年龄、端到端耗时 |
| 批任务 | 上次成功时间、运行时长、处理条数 |
| 资源池 | 连接池使用率、等待数、线程池排队数 |

**来源：** Prometheus 埋点

### 失败指标配总量指标 【推荐】

记录失败次数时，同时要有总尝试次数，否则算不出失败率。请求数在结束时计数，这样和错误数、延迟保持一致。在线调用要在调用方和被调用方两侧都埋点，两侧不一致时问题往往在网络或排队。

**来源：** Prometheus 埋点

### 看分布不看平均 【推荐】

延迟使用直方图，桶按指数分布，并覆盖 SLO 阈值。需要跨实例聚合时不用摘要类型，分位数无法合并。

**来源：** Prometheus 直方图、Google SRE 监控

## 指标命名

### 遵循统一的命名规则 【推荐】

| 规则 | 反例 | 正例 |
|---|---|---|
| 带应用或领域前缀 | `requests_total` | `payment_requests_total` |
| 使用基本单位 | `latency_ms` | `request_duration_seconds` |
| 单位放在名字里，用复数 | `request_duration` | `request_duration_seconds` |
| 累加计数器以 `_total` 结尾 | `http_requests` | `http_requests_total` |
| 时间戳用 `_timestamp_seconds` | `last_run` | `last_success_timestamp_seconds` |
| 比例取 0 到 1 | `disk_usage_percent` | `disk_usage_ratio` |

字节统一用 bytes，不用 bits。

**工具：** `promtool check metrics` 可检查命名。

**来源：** Prometheus 命名

### 一个指标只表达一种量 【推荐】

不混用单位和含义。对所有维度求和或求平均应该有意义，否则拆成多个指标。

**来源：** Prometheus 命名

### 用标签区分维度，不把值写进名字 【推荐】

指标名不能动态生成，标签名也不重复写进指标名。

**示例：**

反例：

```text
http_responses_500_total
http_responses_403_total
```

正例：

```text
http_responses_total{code="500"}
http_responses_total{code="403"}
```

**来源：** Prometheus 埋点、Prometheus 命名

### 选对指标类型和表达方式 【推荐】

值可能减小就用 Gauge，只增不减用 Counter，不对 Gauge 求速率。记录上次发生时间时导出 Unix 时间戳，不导出时间差。可预知的序列先导出 0，事件发生后才出现的序列无法用常规方式计算和告警。

**来源：** Prometheus 埋点

### 全项目选定一套命名体系 【推荐】

OpenTelemetry 语义约定使用点分命名，且不用 `_total` 后缀，和 Prometheus 风格不同。项目内统一一种，由导出层转换，不在代码中混用。

**来源：** OpenTelemetry 命名、Prometheus 命名

## 基数控制

### 标签只用有限枚举值 【必须】

每种标签组合都是一条时间序列，消耗内存、CPU 和存储。下列值不能作为标签：

- 用户 ID、邮箱、订单号、请求 ID、`traceId`
- 未模板化的 URL 路径
- 错误消息原文、时间戳

**示例：**

反例：

```text
http_requests_total{path="/orders/1001",user_id="42"}
```

正例：

```text
http_requests_total{route="/orders/{id}",method="GET",code="200"}
```

**原因：** 无界标签会让序列数量暴涨，压垮监控系统，而且往往在故障时才爆发。

**来源：** Prometheus 命名、Prometheus 埋点

### 控制单个指标的基数 【推荐】

大多数指标不需要标签。单个指标的基数尽量在 10 以内，超过 100 时要换方案，比如减少维度。拿不准时先不加标签，有具体用途再加。需要按用户或订单查问题时，使用日志和链路追踪。

**来源：** Prometheus 埋点

## 告警设计

### 告警针对用户可感知的症状 【推荐】

告警首选高错误率和高延迟这类症状，并尽量设在调用栈高层。CPU 高这类原因类信号用于排查，不直接呼叫人。

**原因：** 原因类告警数量多、误报多，还总有漏掉的原因。

**来源：** Prometheus 告警、Google SRE 监控

### 每条呼叫都要可操作 【推荐】

呼叫只用于紧急且需要人工判断的情况。能自动处理的就自动处理，没有动作可做的告警要删除。告警内容包含影响范围、看板链接和处理手册链接。

| 级别 | 条件 | 通知 |
|---|---|---|
| 呼叫 | 正在或即将影响用户，需要立即处理 | 电话、即时呼叫 |
| 工单 | 需要处理，但可以等到工作时间 | 工单 |
| 看板 | 仅供排查 | 不通知 |

**原因：** 噪声告警会导致疲劳，真实故障也会被忽略。

**来源：** Google SRE 监控、Prometheus 告警

### 基于 SLO 的燃烧率告警 【推荐】

燃烧率是错误预算的消耗速度，1 表示窗口结束时恰好耗尽。快速燃烧用小时级窗口并呼叫，缓慢燃烧用天级窗口并开工单。同时使用长窗口和短窗口，长窗口确认影响足够大，短窗口确认问题仍在发生，修复后告警能很快恢复。

**示例：** 30 天窗口、99.9% 目标，要在 1 小时内耗掉 2% 预算时告警，阈值为 `0.02 × 720 ÷ 1 = 14.4`，也就是错误率超过 1.44%。

**例外：** 请求很少时，一次失败就会触发高燃烧率。可以加入合成探测流量，合并相关服务一起统计，或者放宽窗口。

**来源：** Google SRE 工作手册告警

### 不同服务类型用不同告警 【推荐】

| 类型 | 告警条件 |
|---|---|
| 在线服务 | 用户可见的错误率和延迟 |
| 异步处理 | 数据通过系统的耗时影响到用户 |
| 批任务 | 太久没有成功，阈值至少覆盖两次完整运行 |
| 容量 | 接近上限，需要人工介入 |

低流量的重要接口单独告警，否则会被高流量接口淹没。

**来源：** Prometheus 告警

### 告警要容忍短暂波动 【推荐】

设置持续时间，避免瞬时毛刺触发。

**示例：**

```yaml
- alert: PaymentHighErrorRate
  expr: |
    sum(rate(payment_requests_total{code=~"5.."}[5m]))
      / sum(rate(payment_requests_total[5m])) > 0.05
  for: 5m
  labels:
    severity: page
  annotations:
    summary: 支付接口错误率超过 5%
    runbook: https://example.com/runbooks/payment-errors
```

**来源：** Prometheus 告警

### 监控监控系统本身 【推荐】

为告警链路设置端到端探测，并用外部黑盒探测兜底，内部监控整体失效时也能发现。

**来源：** Prometheus 告警
