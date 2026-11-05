# 数据库迁移

本文讲迁移工具、迁移文件管理、空库验证、大表变更风险和回滚。发布流程见 [release.md](../../general/release.md)。

## 迁移工具

### 表结构变更通过迁移工具管理 【必须】

**适用：** 线上服务

所有表结构变更写成迁移文件，纳入版本控制，由工具记录执行历史。常用工具有 Flyway、Liquibase、Alembic、Prisma Migrate、golang-migrate。

**原因：** 手工变更无法追踪，环境之间的结构会逐渐不一致。

**来源：** Flyway 文档、Azure 多租户存储

### 不手工修改生产表结构 【必须】

**适用：** 线上服务

生产环境的结构变更只通过迁移工具执行。线上紧急处理过的变更，要补成迁移文件并让各环境一致。

### 已应用的迁移文件不再修改 【必须】

需要改动时写新的迁移。工具用校验和发现已执行文件的变化。

**原因：** 修改已执行的文件后，新环境和老环境的结构不再相同。

**工具：** Flyway 用 `validate` 比对本地文件和已执行记录的校验和。

**来源：** Flyway 文档

## 迁移文件

### 文件名表达顺序和内容 【必须】

文件名包含版本和描述，版本唯一，描述说明这次改了什么。

**示例：**

反例：

```text
update.sql
fix2.sql
```

正例：

```text
V20261105093000__add_status_to_orders.sql
V20261105101500__create_order_items.sql
```

多人并行开发时用时间戳作为版本，减少编号冲突。

**来源：** Flyway 文档

### 一个迁移只做一件事 【推荐】

一个文件只包含一个逻辑变更。结构变更和数据修改分开写。

**原因：** 迁移可能在任意一条语句失败。文件越小，失败后越容易判断停在哪里。

**来源：** Flyway 文档

### 了解数据库的 DDL 事务行为 【推荐】

PostgreSQL 的 DDL 可以在事务内回滚，失败的迁移会被撤销。MySQL 在 DDL 前后隐式提交，失败时已执行的语句不会撤销，工具只能标记失败，需要人工清理并修复历史记录。

不能在事务内执行的语句要单独成文件，并配置为非事务执行，比如 PostgreSQL 的 `CREATE INDEX CONCURRENTLY`。

**工具：** Flyway 用脚本配置 `executeInTransaction=false` 控制是否在事务内执行，用 `repair` 清理失败记录。

**来源：** Flyway 文档、PostgreSQL 文档

## 空库验证

### CI 从空库执行全部迁移 【必须】

**适用：** 多人协作

每次变更迁移文件时，CI 在空数据库上从头执行到最新版本，并运行依赖数据库的测试。使用与生产相同的数据库类型，不用内存数据库替代。

**原因：** 迁移集合只有能从空库重建，才能用于新环境、测试环境和灾难恢复。

**示例：**

```bash
flyway clean
flyway migrate
flyway validate
```

### 清库命令不用于生产 【必须】

`flyway clean` 这类命令会删除所有对象，只在开发和测试环境使用，并在生产配置中禁用。

**来源：** Flyway 文档

## 大表变更

### 大表变更先评估锁和重写 【必须】

**适用：** 线上服务

上线前确认语句是否阻塞读写、是否重写全表，并在接近生产规模的数据上演练，在低峰期执行。

| 操作 | MySQL InnoDB | PostgreSQL |
|---|---|---|
| 加二级索引 | `INPLACE`，允许并发读写 | 普通方式阻塞写入，用 `CONCURRENTLY` |
| 加字段 | 默认 `INSTANT`，只改元数据，需要 MySQL 8.0.12 及以上 | 非易变默认值不重写，需要 PostgreSQL 11 及以上；易变默认值重写全表 |
| 删字段 | 默认 `INSTANT`，需要 MySQL 8.0.29 及以上 | 只标记删除，不重写表，但需要短暂的排他锁 |
| 改字段类型 | `COPY`，阻塞写入并重建表 | 通常重写表和索引，临时需要约两倍磁盘 |
| 加约束 | 取决于约束类型 | 用 `NOT VALID` 加约束，再 `VALIDATE CONSTRAINT` |

**来源：** MySQL 手册、PostgreSQL 文档

### MySQL 显式声明算法 【推荐】

在 `ALTER TABLE` 中写明 `ALGORITHM` 和 `LOCK`。条件不满足时语句直接报错，不会悄悄退化成更重的方式。

**示例：**

```sql
ALTER TABLE orders ADD COLUMN source VARCHAR(20) NOT NULL DEFAULT '', ALGORITHM=INSTANT;
ALTER TABLE orders ADD INDEX idx_orders_source (source), ALGORITHM=INPLACE, LOCK=NONE;
```

`INSTANT` 增删字段的次数有上限，达到后要重建表。压缩行格式和带全文索引的表不支持即时加字段。

**来源：** MySQL 手册

### DDL 设置锁等待超时 【必须】

**适用：** 线上服务

DDL 要等持有表元数据锁的事务结束。等待期间，它后面的普通查询也会排队，一条简单的 DDL 就可能让整张表不可用。执行前检查长事务，并设置较短的超时，失败后重试。

**示例：**

MySQL：

```sql
SET SESSION lock_wait_timeout = 5;
ALTER TABLE orders ADD COLUMN source VARCHAR(20) NOT NULL DEFAULT '', ALGORITHM=INSTANT;
```

PostgreSQL：

```sql
SET lock_timeout = '5s';
ALTER TABLE orders ADD COLUMN source VARCHAR(20) NOT NULL DEFAULT '';
```

**工具：** MySQL 查 `performance_schema.metadata_locks`，PostgreSQL 查 `pg_locks`。

**来源：** MySQL 手册、PostgreSQL 文档

### 并发建索引失败后要清理 【推荐】

`CREATE INDEX CONCURRENTLY` 比普通建索引慢，要扫描两遍表并等待已有事务结束，同一张表同时只能有一个。失败时会留下 `INVALID` 索引：查询不使用它，写入却仍要维护。删除后重建。

**示例：**

```sql
DROP INDEX CONCURRENTLY idx_orders_source;
CREATE INDEX CONCURRENTLY idx_orders_source ON orders (source);
```

**来源：** PostgreSQL 文档

### 在线工具用于原生 DDL 无法承受的变更 【推荐】

**适用：** 线上服务

MySQL 上阻塞写入的大表变更，使用 `gh-ost` 或 `pt-online-schema-change`。这类工具建影子表，分批复制数据，最后切换表名。

使用 `gh-ost` 前确认：

- 有行格式的二进制日志。
- 表上没有外键和触发器。
- 变更前后的表有相同的主键或非空唯一键。
- 磁盘能放下一份影子表。

先用 `--test-on-replica` 在副本上验证。用限流控制对主库的影响，用 `--postpone-cut-over-flag-file` 把切换推迟到有人值守的时间。

**来源：** gh-ost 文档

## 数据修复

### 数据修复脚本先备份和演练 【必须】

**适用：** 线上服务

修改线上数据的脚本要先备份，在副本或演练环境执行，执行前用 `SELECT` 确认影响行数，并分批执行，见 [transactions.md](transactions.md)。删除生产数据前按 [deletion.md](../../general/deletion.md) 确认。

## 回滚

### 发布前写明回滚方式 【必须】

**适用：** 线上服务

每个迁移在发布前确定失败时怎么处理：撤销语句是什么，是否可逆，需要多长时间。删除字段、删除表和改类型这类操作不可逆，只能靠备份恢复，执行前必须确认备份可用。

### 不把撤销脚本当作唯一的回滚手段 【推荐】

撤销迁移只能处理完整执行成功的迁移。不支持 DDL 事务的数据库可能失败在任意一条语句，撤销脚本帮不上忙。遇到删除类变更时，撤销脚本也恢复不了数据。

**来源：** Flyway 文档

### 备份和恢复要定期演练 【必须】

**适用：** 线上服务

备份恢复不依赖具体的表结构，是所有迁移失败时的最后手段。只有实际恢复成功过，备份才算可用。数据量大时优先使用存储快照。

**来源：** Flyway 文档

### 已上线的迁移出错时写新迁移修复 【推荐】

已经在生产执行成功的迁移有问题时，写一个新的迁移修正，不修改旧文件，也不手工改库。
