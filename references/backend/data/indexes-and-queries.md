# 索引和查询

以 MySQL 和 PostgreSQL 为主。索引命名见 [schema-design.md](schema-design.md)，线上建索引的风险见 [migrations.md](migrations.md)。

## 索引设计

业务唯一性用唯一索引保证，见 [consistency.md](../reliability/consistency.md)。

### 索引根据真实查询设计 【推荐】

先列出实际查询，再设计索引，并用执行计划验证。不给每个字段都建索引。

**原因：** 每个索引都占用空间，增加优化器的选择成本，并拖慢写入。

**来源：** MySQL 手册

### 联合索引等值在前、范围在后 【推荐】

遵守最左前缀。等值条件的字段放前面，范围条件的字段放后面，同时考虑排序字段。第一个范围条件之后的字段不再缩小扫描范围。

**示例：**

查询：

```sql
SELECT id, status FROM orders
WHERE user_id = ? AND status = ?
ORDER BY created_at DESC
LIMIT 20;
```

反例：

```sql
CREATE INDEX idx_orders_created_user ON orders (created_at, user_id, status);
```

正例：

```sql
CREATE INDEX idx_orders_user_status_created ON orders (user_id, status, created_at);
```

**来源：** PostgreSQL 文档、Use The Index, Luke

### 控制联合索引的列数 【推荐】

单列索引往往已经够用。超过三列的索引很少有收益，除非查询模式非常固定。`(a, b)` 已存在时，`(a)` 通常是冗余索引。

**来源：** PostgreSQL 文档

### 不让索引失效 【推荐】

不在索引列上使用函数或计算，避免隐式类型转换，避免前导通配符 `LIKE '%abc'`。

**示例：**

反例：

```sql
SELECT id FROM orders WHERE DATE(created_at) = '2026-11-05';
SELECT id FROM users WHERE phone = 13800000000;
```

正例：

```sql
SELECT id FROM orders
WHERE created_at >= '2026-11-05' AND created_at < '2026-11-06';
SELECT id FROM users WHERE phone = '13800000000';
```

### 少量热数据用部分索引 【推荐】

PostgreSQL 中，经常访问的行只占少数时，只为这部分行建索引。查询条件必须能推出索引条件才会用上，带参数的条件可能匹配不上。

**来源：** PostgreSQL 文档

### 删除不再使用的索引 【必须】

没有读取方的旧索引要删除，见 [deletion.md](../../general/deletion.md)。主键和唯一索引用于保证约束，扫描次数为 0 也不能据此删除。

PostgreSQL 查 `pg_stat_user_indexes.idx_scan`，并确认统计的起始时间足够长、覆盖周期性任务，各副本的统计彼此独立。MySQL 可以先把索引设为不可见，观察后再删除：

```sql
ALTER TABLE orders ALTER INDEX idx_orders_status INVISIBLE;
```

不可见索引写入时仍然维护，唯一索引仍然生效。

**来源：** 团队、MySQL 手册、PostgreSQL 文档

## 执行计划

### 上线前看执行计划 【推荐】

**适用：** 线上服务

新增或修改的重要查询，在接近生产规模的数据上查看执行计划。小表上的结果不能推到大表。

PostgreSQL 重点比较 `EXPLAIN ANALYZE` 的估算行数和实际行数。差距很大通常说明统计信息过期，先运行 `ANALYZE`。循环节点的实际值是每次循环的平均值，要乘以 `loops`。

**来源：** MySQL 手册、PostgreSQL 文档

### EXPLAIN ANALYZE 会真实执行 【必须】

对写语句使用时放进事务并回滚。

**示例：**

```sql
BEGIN;
EXPLAIN ANALYZE UPDATE orders SET status = 'closed' WHERE id = 1001;
ROLLBACK;
```

**来源：** PostgreSQL 文档

## 查询写法

SQL 一律参数化，见 [input-and-injection.md](../security/input-and-injection.md)。

### 不写 SELECT * 【推荐】

生产代码明确列出字段，减少传输，也避免字段变更带来的意外。

### 查询限制返回行数 【必须】

**适用：** 线上服务

大结果集使用分页或流式处理，不一次性载入内存。

### 避免 N+1 查询 【推荐】

**适用：** 线上服务

用 JOIN、批量 `IN` 查询或 ORM 的显式预加载。

**原因：** 主要开销是每次查询的网络往返，返回的数据量相同也比一次查询慢得多。

**示例：**

反例：循环中每条订单查一次用户，共 1 + N 次。

```sql
SELECT id, name FROM users WHERE id = ?;
```

正例：收集 ID 后批量查询，共 2 次。

```sql
SELECT id, name FROM users WHERE id IN (?, ?, ?);
```

**工具：** 开发环境打开 SQL 日志，或在测试中断言查询次数。

**来源：** Use The Index, Luke

## 分页

### 深分页使用游标条件 【推荐】

用上一页最后一条的排序值作为边界，避免大 `OFFSET`。代价是不能跳到任意页。接口层的分页约定见 [rest.md](../api/rest.md)。

**原因：** `OFFSET` 要数完所有被跳过的行，越往后越慢，新增数据还会让翻页错位。

**示例：**

反例：

```sql
SELECT id, created_at FROM orders
ORDER BY created_at DESC
LIMIT 20 OFFSET 100000;
```

正例：

```sql
SELECT id, created_at FROM orders
WHERE created_at < ? OR (created_at = ? AND id < ?)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

PostgreSQL 可以写成 `(created_at, id) < (?, ?)` 并用上索引。MySQL 不能把这种行值比较用于索引访问，用上面的展开写法。

**来源：** Use The Index, Luke

### 分页排序必须唯一确定 【必须】

排序字段可能重复时，追加主键作为末位排序字段，并让索引匹配。否则相同值的行会在翻页时丢失或重复。

**来源：** Use The Index, Luke

## 慢查询

### 开启慢查询日志并定期治理 【推荐】

MySQL 的 `slow_query_log` 默认关闭，`long_query_time` 默认 10 秒，要按项目的延迟目标调低。治理时按总耗时排序，优先处理占比最高的语句，不只看单次最慢的。

**工具：** MySQL 用 `mysqldumpslow` 汇总慢日志，可开启 `log_queries_not_using_indexes` 并配合限流参数。PostgreSQL 用 `log_min_duration_statement` 和 `pg_stat_statements`。

**来源：** MySQL 手册

## ORM 使用边界

ORM 和原生 SQL 的分工见 [data-access.md](data-access.md)。

### 关联加载在查询时显式声明 【推荐】

需要关联数据时，在具体查询里声明预加载或连接。不在实体映射里全局开启预加载，否则不需要关联数据的查询也要付出代价。

**来源：** Use The Index, Luke

### 关注 ORM 实际生成的 SQL 【推荐】

重要查询要看生成的 SQL 和执行计划，不能只看 ORM 调用的写法。语言层的具体用法见 `languages/` 下的相关文件。
