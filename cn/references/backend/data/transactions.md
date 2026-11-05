# 事务和锁

以 MySQL InnoDB 和 PostgreSQL 为主。幂等、分布式锁、Outbox 和 Saga 见 [consistency.md](../reliability/consistency.md)。

## 事务边界

### 事务尽量短 【必须】

只把必须一起成功的写操作放进事务，完成后立即提交。

**原因：** 事务越长，持锁越久，冲突和死锁的概率越高。

**来源：** MySQL 手册、PostgreSQL 文档

### 事务内不做外部操作 【必须】

事务内不做网络调用、文件处理，也不等待用户操作。

**原因：** 外部调用会拉长持锁时间，而且无法随事务回滚。需要同时写库和发消息时，见 [consistency.md](../reliability/consistency.md) 的 Outbox。

**示例：**

反例：开启事务，更新订单，调用支付接口，等待返回，提交。

正例：事务 1 写入 `pending` 状态和幂等键并提交；事务外用同一幂等键调用支付接口；事务 2 用条件更新写入结果。调用超时视为状态未知，保持“处理中”，用同一幂等键重试或查询。是否增加定时对账任务由用户决定，不默认添加。

## 隔离级别

### 明确所用的隔离级别 【推荐】

两种数据库的默认级别和行为不同，迁移数据库或依赖快照语义时要逐项确认。

| 项目 | MySQL InnoDB | PostgreSQL |
|---|---|---|
| 默认级别 | `REPEATABLE READ` | `READ COMMITTED` |
| 默认级别的普通读 | 事务内沿用第一次读建立的快照 | 每条语句一个新快照 |
| 默认级别的范围加锁 | 使用间隙锁和临键锁阻止插入 | 不使用间隙锁 |
| `READ COMMITTED` | 不加间隙锁，幻读可能出现 | 同左，同一事务两次读取结果可能不同 |
| `REPEATABLE READ` 的写冲突 | 等待锁后继续 | 报序列化失败，需要重试 |

**来源：** MySQL 手册、PostgreSQL 文档

### 序列化失败要重试整个事务 【必须】

PostgreSQL 使用 `REPEATABLE READ` 或 `SERIALIZABLE` 时，并发冲突会让事务失败，错误码 `40001`。应用必须从头重试整个事务，不能只重试最后一条语句。

**来源：** PostgreSQL 文档

## 锁

### 更新和加锁都走索引 【必须】

**适用：** 线上服务

`UPDATE`、`DELETE` 和 `SELECT ... FOR UPDATE` 的条件要命中合适的索引，最好是唯一索引。

**原因：** InnoDB 的行锁实际加在扫描到的索引记录上。没有合适索引时扫描范围变大，锁住的行和间隙也更多。

**示例：**

反例：`note` 字段没有索引，语句会扫描并锁住大量记录。

```sql
UPDATE orders SET status = 'closed' WHERE note = 'expired';
```

正例：

```sql
UPDATE orders SET status = 'closed' WHERE id = ?;
```

**来源：** MySQL 手册

### 能不加锁就不加锁 【推荐】

读取旧快照可以接受时，不写 `FOR UPDATE` 或 `FOR SHARE`。确实要加锁时，用能满足需求的最弱锁。PostgreSQL 不改键字段时，`FOR NO KEY UPDATE` 比 `FOR UPDATE` 冲突更少。

**来源：** MySQL 手册、PostgreSQL 文档

### 锁等待设置上限 【推荐】

PostgreSQL 默认会一直等待锁，用 `lock_timeout` 设置上限，并按角色或会话配置，不写进全局配置。MySQL 用 `innodb_lock_wait_timeout` 控制行锁等待。

**来源：** PostgreSQL 文档

## 死锁

### 多行更新按固定顺序加锁 【必须】

**适用：** 线上服务

所有事务按同样的顺序访问表和行，比如按主键升序。这样事务会排队，而不是互相等待。

**示例：**

反例：转账时总是先更新转出账户，两笔方向相反的转账会互相等待。

正例：无论转账方向，都先更新 ID 较小的账户。

```sql
SELECT id, balance FROM accounts WHERE id IN (?, ?) ORDER BY id FOR UPDATE;
```

**来源：** MySQL 手册、PostgreSQL 文档

### 死锁错误有限次重试 【推荐】

数据库会回滚其中一个事务，被回滚的事务可以整体重试，并设置次数上限。MySQL 错误码是 1213，PostgreSQL 的 SQLSTATE 是 `40P01`。重试规则见 [resilience.md](../reliability/resilience.md)。

**原因：** 固定加锁顺序只能降低概率，不能完全消除死锁。重试时事务内不能有无法撤销的外部副作用，这也是事务内不做外部操作的原因之一。

**工具：** MySQL 用 `SHOW ENGINE INNODB STATUS` 查看最近一次死锁，频繁发生时临时开启 `innodb_print_all_deadlocks`。

**来源：** MySQL 手册、PostgreSQL 文档

## 长事务

### 监控并限制长事务 【推荐】

**适用：** 线上服务

长事务不只影响自己：

- MySQL 中，它阻止清理其他事务产生的旧版本数据，其他读取还要重建旧版本。
- PostgreSQL 中，它阻止 `VACUUM` 回收死元组，造成表膨胀。
- 它持有元数据锁时会阻塞表结构变更，见 [migrations.md](migrations.md)。

**工具：** MySQL 查 `information_schema.INNODB_TRX`。PostgreSQL 查 `pg_stat_activity`，并为应用角色设置 `idle_in_transaction_session_timeout`。

**来源：** MySQL 手册、PostgreSQL 文档

### 大批量更新和删除分批执行 【必须】

**适用：** 线上服务

按主键范围或固定行数分批，每批单独提交，批次之间留出间隔。

**原因：** 单条大语句会造成长事务、锁等待和主从延迟。失败时的回滚耗时可能是执行耗时的数倍，杀掉进程也不能取消回滚。

**示例：**

反例：

```sql
DELETE FROM event_logs WHERE created_at < '2026-01-01';
```

正例（MySQL），重复执行直到影响行数为 0：

```sql
DELETE FROM event_logs WHERE created_at < '2026-01-01' LIMIT 5000;
```

正例（PostgreSQL，`DELETE` 不支持 `LIMIT`）：

```sql
DELETE FROM event_logs
WHERE id IN (
  SELECT id FROM event_logs WHERE created_at < '2026-01-01' LIMIT 5000
);
```

**来源：** MySQL 手册

## 乐观锁

### 并发修改用乐观锁或条件更新 【推荐】

冲突较少时，用版本字段或业务条件代替长时间持锁，把判断和写入合成一条语句。

**示例：**

版本字段：

```sql
UPDATE orders SET status = 'paid', version = version + 1
WHERE id = ? AND version = ?;
```

业务条件，保证库存不超卖：

```sql
UPDATE inventory SET stock = stock - 1
WHERE sku_id = ? AND stock >= 1;
```

### 检查条件更新的影响行数 【必须】

影响行数为 0 表示版本冲突或条件不满足。要作为明确结果处理，重新读取后重试或向调用方返回失败，不能当作成功。

**原因：** 不检查时，更新丢失不会有任何报错。
