# Django

## 应用划分

### 按业务能力划分应用 【推荐】

每个应用对应一块业务能力，包含自己的模型、视图、迁移和测试，不建堆放杂物的 `common`、`utils` 应用。`AppConfig` 放在 `apps.py`，信号在 `ready()` 中注册。

**来源：** Django 文档

### 新项目先定义自定义用户模型 【推荐】

项目开始时就继承 `AbstractUser`，并设置 `AUTH_USER_MODEL`。外键写 `settings.AUTH_USER_MODEL`，其他代码用 `get_user_model()`。

**原因：** 项目中途更换用户模型的代价很高。

**来源：** Django 文档

## 模型

### 字段类型表达语义 【推荐】

`CharField`、`TextField` 用空字符串表示无数据，不用 `null=True`，需要唯一约束且允许留空时例外。金额用 `DecimalField`，写明 `max_digits` 和 `decimal_places`，不用 `FloatField`。状态字段用 `models.TextChoices` 或 `IntegerChoices`，代码中引用枚举成员。

**工具：** `DJ001`。

**来源：** Django 文档

### 不变量用数据库约束表达 【推荐】

唯一性和取值范围写成 `Meta.constraints` 中的 `UniqueConstraint`、`CheckConstraint`，索引写在 `Meta.indexes`。

**原因：** 表单和 `save()` 中的校验挡不住并发写入和绕过它们的批量操作。

**来源：** Django 文档

## QuerySet 与 N+1

### 循环访问关联对象前先预取 【必须】

**适用：** 线上服务

外键和一对一用 `select_related`，多对多和反向外键用 `prefetch_related`。Django 6.1+ 难以预测访问字段时，可以用 `fetch_mode(models.FETCH_PEERS)` 按需批量加载。

**原因：** 否则每次循环都发一条查询，请求耗时随数据量线性增长。

**示例：**

反例：

```python
for book in Book.objects.all():
    print(book.author.name)
```

正例：

```python
for book in Book.objects.select_related("author"):
    print(book.author.name)
```

**工具：** 列表类接口的测试用 `assertNumQueries` 固定查询次数。

**来源：** Django 文档

### 预取后不再链式过滤 【推荐】

`book.reviews.filter(...)` 会忽略预取缓存并重新查询。需要筛选时用 `Prefetch` 对象写明查询集。

**来源：** Django 文档

### 只取需要的数据 【推荐】

只要字段值时用 `values()`、`values_list()`。外键用 `obj.author_id`，不用 `obj.author.id`。大结果集用 `iterator()`。不需要顺序时不排序。只判断存在时用 `exists()`，只计数时用 `count()`，后面还要用结果时直接求值并复用。

**来源：** Django 文档

### 批量写入用批量方法 【推荐】

用 `bulk_create`、`bulk_update`、`QuerySet.update()`。这些方法不调用 `save()`，也不发送 `pre_save` 和 `post_save` 信号，依赖这些逻辑时不能用。

**来源：** Django 文档

## 事务

### 多步写入用 atomic 【必须】

需要一起成功的写操作放在 `transaction.atomic()` 中，Django 默认每条语句单独提交。事务在异步模式下尚不可用，要把事务逻辑写成同步函数，用 `sync_to_async` 调用。

**来源：** Django 文档

### 不在 atomic 内捕获数据库异常 【必须】

捕获 `IntegrityError` 这类异常时，`try` 写在 `atomic` 外层，或者再包一层 `atomic`。

**原因：** 捕获后事务已经损坏，继续查询会抛 `TransactionManagementError`。

**示例：**

```python
try:
    with transaction.atomic():
        create_membership(user, team)
except IntegrityError:
    raise AlreadyMemberError(user.id) from None
```

**来源：** Django 文档

### 事务外副作用用 on_commit 【必须】

发邮件、投递任务、刷新缓存放进 `transaction.on_commit()`。需要绑定参数时用 `functools.partial`。

**原因：** 事务回滚后已发出的副作用收不回来，任务也可能先于提交读到旧数据。

**来源：** Django 文档

### 并发更新用 F 表达式或行锁 【必须】

计数更新用 `F("count") + 1`，由数据库完成。读后写用 `select_for_update()`，必须放在 `atomic` 内，否则抛 `TransactionManagementError`。`get_or_create` 依赖字段有唯一约束才是原子的。

**来源：** Django 文档

## 迁移

### 迁移和模型改动同一提交 【必须】

**适用：** 多人协作

运行 `makemigrations` 后，把迁移文件和模型改动放进同一次提交，用 `-n` 起描述性名字。执行前用 `sqlmigrate <应用> <编号>` 查看 SQL，用 `migrate --plan` 查看计划。

**工具：** CI 运行 `python manage.py makemigrations --check` 和 `migrate --check`。

**来源：** Django 文档

### 不修改已应用的迁移 【必须】

已在共享环境应用的迁移不再编辑或删除，变更写成新的迁移。两个分支产生冲突时用 `makemigrations --merge` 合并。迁移过多时用 `squashmigrations <应用> <编号>` 压缩，确认所有环境都已应用后，再删除被替换的文件并去掉 `replaces` 属性。

**来源：** Django 文档

### 迁移中用历史模型 【必须】

`RunPython` 里用 `apps.get_model()`，不直接导入模型。

**原因：** 直接导入得到的是当前代码的模型，与该迁移时的表结构不一致。

**来源：** Django 文档

## 安全配置

### 密钥和开关来自环境 【必须】

`SECRET_KEY` 从环境变量或密钥文件读取，比如 `os.environ["SECRET_KEY"]`，生产密钥不与其他环境共用。生产环境 `DEBUG = False`，`ALLOWED_HOSTS` 列出实际域名。

**工具：** `S105`。

**来源：** Django 文档

### 部署配置过检查 【必须】

**适用：** 线上服务

CI 使用生产配置运行 `python manage.py check --deploy`，不用 `runserver` 部署。设置 `SESSION_COOKIE_SECURE`、`CSRF_COOKIE_SECURE`、`SECURE_SSL_REDIRECT` 和 `SECURE_HSTS_SECONDS`。在代理后面部署时，理解风险后再配置 `SECURE_PROXY_SSL_HEADER`。

**来源：** Django 文档

### 谨慎绕过框架防护 【必须】

`mark_safe`、`csrf_exempt`、`raw()`、`extra()`、`RawSQL` 只在确有必要时使用，用户输入必须走参数绑定。`ModelForm` 用 `fields` 列出字段，不用 `__all__` 和 `exclude`。

**工具：** `S308`、`S610`、`S611`、`DJ006`、`DJ007`。

**来源：** Django 文档、Ruff 文档

### 上传内容按不可信处理 【必须】

上传大小在 Web 服务器层限制。上传文件用独立域名提供，只用子域名不够，并限制允许的扩展名。

**来源：** Django 文档
