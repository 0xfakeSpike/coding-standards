# 数据访问

本文讲数据访问代码的组织方式：表定义和使用方式集中在哪里，业务代码如何使用。分层总则见 [layering.md](../architecture/layering.md)，查询优化见 [indexes-and-queries.md](indexes-and-queries.md)。

## 统一入口

字段或表没有读取方时，model、repo 和迁移一起处理，见 [schema-design.md](schema-design.md)。

一张表只归属一个模块，跨模块只读查询的例外见 [service-boundaries.md](../architecture/service-boundaries.md)。

### 每张表有统一的 model/repo 入口 【必须】

每张表的定义和全部读写集中在一处。model 描述表结构，repo 提供这张表的全部操作。Go 项目放在业务包的 `store.go`，见 [project.md](../../languages/go/project.md)。其他语言的目录名可以不同，结构相同。

**原因：** 表的读写分散时，改一个字段要全局搜索，也看不清这张表到底被怎么用。

**示例：**

反例：

```text
handlers/order_handler     内嵌 SELECT 和 UPDATE 语句
jobs/close_orders          内嵌另一份 UPDATE 语句
pages/order_list           内嵌 JOIN 语句
```

正例：

```text
order/order               orders 和 order_items 的字段定义
order/store               orders 和 order_items 的全部查询和写入
handlers/order_handler     调用业务层
jobs/close_orders          调用业务层
```

**来源：** 团队

### 表的定义和使用方式可追踪 【必须】

从表名出发，能在一个位置看到字段定义，并从 repo 的方法列表看到所有使用方式。再从方法的调用方看到谁在使用。

为此：

- SQL 和 ORM 调用只出现在 repo 中。
- 表名不靠字符串动态拼接，否则搜索不到。
- 分表的路由逻辑也集中在 repo。

**工具：** 在 CI 中搜索 repo 目录以外的 SQL 关键字和 ORM 调用，发现后拒绝合入。

**来源：** 团队

### repo 方法按用途命名 【推荐】

方法名说明查什么、按什么条件，不向业务层暴露通用的条件拼装入口。

| 反例 | 正例 |
|---|---|
| `query(table, conditions)` | `find_unpaid_by_user(user_id)` |
| `update(id, fields)` | `mark_paid(order_id, paid_at)` |

**原因：** 通用入口让所有调用都长得一样，等于没有入口，也无法从方法列表看出使用方式。

## 职责边界

接口响应使用单独的结构，见 [layering.md](../architecture/layering.md)。

### 业务判断不放 repo 【推荐】

repo 只做数据存取，不判断业务状态，不调用外部服务。业务规则在业务层。

**示例：**

反例：`mark_paid` 内部判断订单是否过期，并发送通知。

正例：业务层判断订单能否支付，repo 只执行条件更新，返回影响行数。

### 横切过滤在 repo 统一处理 【必须】

租户过滤和软删除过滤由 repo 或其基础设施统一加上，业务层不重复写。绕过过滤的方法要在名字里写明，比如 `find_including_deleted`。

**原因：** 漏掉一处过滤会造成数据泄露或读到已删除的数据。

### 事务边界由业务层决定 【推荐】

repo 提供可以加入事务的方法，不自行决定跨操作的事务。业务层通过事务句柄或工作单元开启事务并传给 repo，这不算把 ORM 调用带出 repo。事务写法见 [transactions.md](transactions.md)。

**原因：** 一个业务操作可能涉及多个 repo，只有业务层知道哪些写入必须一起成功。

## ORM 边界

### ORM 查询对象不离开 repo 【推荐】

ORM 的查询构造器和懒加载对象不传到 repo 之外，repo 返回已经加载完整的数据。事务句柄或工作单元可以由业务层持有。

**原因：** 懒加载对象离开 repo 后，一次字段访问就可能触发查询，造成 N+1，或者在连接关闭后报错。

### ORM 不强制覆盖所有场景 【推荐】

简单读写用 ORM，复杂查询和批量操作在 repo 中直接写 SQL，见 [indexes-and-queries.md](indexes-and-queries.md)。两种方式共用同一个入口，调用方不需要知道底层用的是哪一种。

### 表结构以迁移文件为准 【必须】

**适用：** 线上服务

生产环境的结构由迁移工具管理，不用 ORM 启动时自动建表或改表。model 定义要与迁移结果一致。

**工具：** 在 CI 的空库验证后，比对 model 定义和实际表结构，见 [migrations.md](migrations.md)。

## 读副本

### 读副本的选择放在 repo 【推荐】

使用读写分离时，读主库还是读副本由 repo 的方法决定，业务层不直接选择连接。写入后马上要读到结果的路径读主库。

**原因：** 副本有复制延迟，写后立即读副本可能读不到刚写入的数据。

## 测试

### repo 用真实数据库测试 【推荐】

SQL 方言和事务行为因数据库而异，repo 的测试使用与生产相同的数据库类型，见 [testing.md](../testing.md)。
