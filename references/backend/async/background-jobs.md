# 后台任务

本文讲定时任务和任务队列。消息中间件的投递语义和死信配置见 [messaging.md](messaging.md)，接口层的长任务见 [rest.md](../api/rest.md)。

## 基本要求

### 任务按可能重复执行设计 【必须】

同一输入执行多次，结果相同。任务重复执行是常态：队列至少投递一次，调度可能重叠，重启会重放做了一半的工作。发邮件这类做不到幂等的任务，写明可能重复的取舍。

**示例：**

反例：任务每次运行都给所有用户发放积分。

正例：用唯一索引记录发放批次，重复运行时跳过。

```sql
INSERT INTO point_grants (user_id, batch_date, points) VALUES (?, ?, ?)
ON CONFLICT (user_id, batch_date) DO NOTHING;
```

**来源：** Azure 后台任务、Kubernetes CronJob

### 任务持久化，不只放在内存 【必须】

不能丢的任务放进持久队列或任务表，不放在进程内存队列中。

**原因：** 进程重启或发布时，内存中的任务会全部丢失，也没有记录。

### 任务参数只传标识 【推荐】

任务参数放 ID 这类小数据，执行时读取最新状态，并再次确认任务是否仍需执行。

**示例：**

反例：

```json
{ "type": "order.close", "order": { "id": "1001", "status": "pending", "items": [] } }
```

正例：

```json
{ "type": "order.close", "order_id": "1001" }
```

**原因：** 任务排队期间数据可能已经变化，参数里的快照会让判断出错。

### 后台任务与接口服务分开部署 【推荐】

负载较重的任务用独立进程运行，可以单独扩容和发布，也不会抢占接口服务的资源。接口服务按请求量扩容，后台任务按队列深度扩容。

**来源：** Azure 后台任务

## 定时任务

定时任务不放进数据库调度器，见 [schema-design.md](../data/schema-design.md)。

### 多实例部署时防止重复执行 【必须】

调度只保留一个入口，或者用数据库唯一约束抢占本次执行权。任务运行时间超过调度间隔时，也会与下一次重叠，要明确处理。分布式锁的注意事项见 [consistency.md](../reliability/consistency.md)。

**示例：**

反例：每个服务实例内都启动同样的定时器。

正例：用任务名和计划时间抢占，插入成功的实例才执行。

```sql
INSERT INTO job_runs (job_name, scheduled_at, status) VALUES (?, ?, 'running')
ON CONFLICT (job_name, scheduled_at) DO NOTHING;
```

Kubernetes 中使用 `CronJob`，关键配置如下：

```yaml
spec:
  schedule: "*/10 * * * *"
  timeZone: "Etc/UTC"
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 300
```

`CronJob` 可能创建两次，也可能一次都不创建，所以任务本身仍要幂等。`concurrencyPolicy` 默认是 `Allow`，允许并发运行。`Forbid` 跳过重叠的这次运行。

**来源：** Azure 后台任务、Kubernetes CronJob

### 明确调度时区 【必须】

调度时区写在配置里，默认使用 UTC，不依赖服务器本地时区。Kubernetes 用 `.spec.timeZone`，不支持在 `schedule` 中写 `TZ` 或 `CRON_TZ`。

**来源：** Kubernetes CronJob

### 按计划时间处理数据窗口 【推荐】

任务处理的范围由计划时间决定，不由实际启动时间决定。这样延迟启动或重新执行时，处理的是同一批数据。

**示例：**

反例：

```sql
SELECT id FROM orders WHERE created_at >= NOW() - INTERVAL '1 day';
```

正例：

```sql
SELECT id FROM orders WHERE created_at >= ? AND created_at < ?;
```

## 任务队列

### 用数据库做任务队列时安全地抢占 【推荐】

任务量不大时，一张任务表就够用，不必引入新中间件。抢占时跳过已被锁定的行，避免多个执行者互相等待。

**示例：**

```sql
SELECT id, payload FROM jobs
WHERE attempts < ?
  AND ((status = 'pending' AND run_at <= ?)
    OR (status = 'running' AND lease_until <= ?))
ORDER BY run_at
LIMIT 10
FOR UPDATE SKIP LOCKED;

UPDATE jobs SET status = 'running', lease_until = ?, attempts = attempts + 1 WHERE id IN (?);
```

抢占事务只做三件事：选中任务、写入 `running` 和租约到期时间、提交。任务在事务外执行。

任务表至少包含状态、下次执行时间、重试次数和租约到期时间。执行者崩溃后，租约到期的任务由抢占条件重新领取，领取时累加重试次数，达到上限的任务不再领取，按失败处理。租约时长要大于任务的超时时间。

### 不同优先级的任务分开队列 【推荐】

耗时的批量任务和短小的实时任务使用不同队列和执行者，防止前者阻塞后者。

### 消费能力匹配下游容量 【推荐】

并发数按数据库和下游服务的承受能力设置，不只按队列积压扩容。

**原因：** 积压时盲目扩容消费者，会把压力转移到数据库或下游服务上。

## 分片

### 大任务拆成可独立重试的分片 【推荐】

按主键范围或哈希把数据切成分片，每片独立执行、记录状态和重试。一个分片失败时，不需要重跑全部数据。

| 方式 | 适用 |
|---|---|
| 主键范围 | 主键大致连续的表，切分简单 |
| 哈希取模 | 主键不连续，或者需要均匀分配 |
| 按租户 | 租户之间互不影响，注意大租户造成倾斜 |

### 长任务保存进度 【推荐】

长任务或多步骤任务定期把进度写入持久存储，重启后从检查点继续。批量写入按 [transactions.md](../data/transactions.md) 分批提交。

**示例：** 处理到主键 50000 时写入 `last_id = 50000`，下次从 `id > 50000` 开始。

**来源：** Azure 后台任务

## 重试和终止

暂时失败才重试，永久失败直接标记失败，规则见 [resilience.md](../reliability/resilience.md) 和 [error-handling.md](../../general/error-handling.md)。

### 失败任务有上限和死信 【必须】

**适用：** 线上服务

重试耗尽的任务进入死信队列或标记为失败，保留输入和失败原因，并配置告警。修复后可以重新提交。

**来源：** Azure 后台任务

### 任务设置执行超时 【必须】

每种任务都有最长执行时间，超时后中止并按失败处理。外部调用也要设置超时。

**原因：** 卡住的任务不报错，却一直占用执行槽位和租约。

### 收到终止信号时优雅退出 【必须】

**适用：** 线上服务

收到 `SIGTERM` 后不再领取新任务，完成当前任务或保存进度后退出。平台的宽限时长要大于单个工作项的典型耗时，比如 Kubernetes 的 `terminationGracePeriodSeconds`。

**来源：** Azure 后台任务

## 可观测性

### 记录任务的完整生命周期 【推荐】

**适用：** 线上服务

记录开始、完成和失败，包含任务类型、关联 ID 和耗时，并保存执行结果。只记录开始时，卡死或崩溃的任务看起来像在正常运行。多步骤任务全程传递关联 ID，见 [tracing.md](../observability/tracing.md)。

**来源：** Azure 后台任务

### 监控任务没有运行的情况 【推荐】

**适用：** 线上服务

对比实际运行和预期调度，缺失时告警。任务没有触发时不会产生任何错误，只看错误日志发现不了。

**来源：** Azure 后台任务

### 关注排队等待时间 【推荐】

除了执行耗时，还要统计从入队到完成的总耗时、队列深度和死信数量。业务感受到的是总耗时。指标和告警见 [metrics-and-alerts.md](../observability/metrics-and-alerts.md)。

**来源：** Azure 后台任务
