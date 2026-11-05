# Python 测试

测试策略见 [testing-strategy.md](../../general/testing-strategy.md)。本文只讲 Python 的工具和写法。

## 框架

### 使用 pytest 【推荐】

测试用普通函数和 `assert`，不继承 `unittest.TestCase`。Django 项目的数据库测试可以使用框架自带的 `TestCase`。

**来源：** pytest 文档

### 开启严格模式 【推荐】

配置放在 `pyproject.toml`，开启 `strict`。新项目用 `importlib` 导入模式。

**原因：** 拼错的 marker 会被静默接受，标记 `xfail` 的测试意外通过时也没人知道。

**示例：**

```toml
[tool.pytest]
minversion = "9.0"
testpaths = ["tests"]
addopts = ["--import-mode=importlib"]
strict = true
```

**例外：** pytest 9.0 以下用 `[tool.pytest.ini_options]`，并分别设置 `--strict-markers` 和 `xfail_strict`。

**来源：** pytest 文档

## 组织方式

### 测试放在包外并镜像结构 【推荐】

`tests/` 目录的结构跟随 `src/`，文件命名为 `test_*.py`，函数命名为 `test_<行为>_<条件>`。

**来源：** pytest 文档、Google Python Style Guide

### 用 fixture 准备依赖 【推荐】

共享的准备逻辑写成 fixture，放在 `conftest.py`。需要清理时用 `yield`，清理代码写在 `yield` 之后。不用模块级变量在测试之间共享状态。测试生成的文件写到 `tmp_path`，不写到仓库目录。

**示例：**

```python
@pytest.fixture
def account(db: Database) -> Iterator[Account]:
    created = db.create_account()
    yield created
    db.delete_account(created.id)
```

**来源：** pytest 文档

### fixture 范围从小开始 【推荐】

默认用 `function` 范围。只有创建代价高且内容不被修改的资源才用 `module` 或 `session`。`autouse=True` 只用于所有测试都需要的准备，否则依赖关系不可见。

**来源：** pytest 文档

### 异常断言写明类型 【推荐】

用 `pytest.raises(具体类型)`，代码块内只放一条语句，需要的信息断言异常属性。消息不是稳定接口，只有 `ValueError` 这类宽泛异常才用 `match=` 区分。不断言 `Exception`。浮点比较用 `pytest.approx`。

**示例：**

```python
def test_withdraw_rejects_overdraft(account: Account) -> None:
    with pytest.raises(InsufficientFundsError) as exc_info:
        account.withdraw(Decimal("100"))
    assert exc_info.value.shortfall == Decimal("20")
```

**工具：** `PT011`、`PT012`、`B017`。

**来源：** pytest 文档、Ruff 文档

## 参数化

### 多组输入用 parametrize 【推荐】

同一行为的多组输入用 `@pytest.mark.parametrize`，不在测试里写循环。数据本身看不出含义时写 `id`，单个用例的预期失败用 `marks=pytest.mark.xfail`。

**原因：** 每组输入独立报告结果，一组失败不影响其他组运行。

**示例：**

```python
@pytest.mark.parametrize(
    ("amount", "expected"),
    [
        pytest.param(9999, 800, id="below-free-threshold"),
        pytest.param(10000, 0, id="at-free-threshold"),
    ],
)
def test_shipping_fee(amount: int, expected: int) -> None:
    assert calculate_shipping_fee(amount) == expected
```

**工具：** `PT006`、`PT007`。

**来源：** pytest 文档

## 替身

### 环境和属性用 monkeypatch 【推荐】

改环境变量、字典项和属性用 `monkeypatch`，它在测试结束时自动还原。不直接赋值 `os.environ`。

**来源：** pytest 文档

### mock 必须带规格 【推荐】

用 `create_autospec` 或 `patch(..., autospec=True)`，不用裸 `Mock()`。

**原因：** 裸 mock 接受任何方法和参数，被测代码调错了也能通过。

**示例：**

反例：

```python
gateway = Mock()
```

正例：

```python
gateway = create_autospec(PaymentGateway, instance=True)
```

**来源：** Python 标准库文档

### 在使用处打补丁 【推荐】

`patch` 的目标是名字被查找的地方。`orders.py` 里写了 `from gateway import charge`，就要打 `orders.charge`，不打 `gateway.charge`。

**来源：** Python 标准库文档

### 优先注入替身 【推荐】

依赖通过参数传入时，直接传一个符合 `Protocol` 的假实现，比打补丁更简单，也不绑定内部模块路径。

### 时间通过注入控制 【推荐】

需要控制时间的代码接收 `now: Callable[[], datetime]` 这样的参数，测试传入固定时钟，不写真实的 `time.sleep` 等待。

## 集成测试

### 用 marker 区分集成测试 【推荐】

集成测试标记 `@pytest.mark.integration`，并在配置中注册。提交阶段可以用 `pytest -m "not integration"` 跳过，合并前全量运行。

**来源：** pytest 文档

### 用真实依赖验证 SQL 和协议 【推荐】

数据库、消息队列的集成测试连接真实服务，不用 mock 模拟 SQL。每个测试在事务中运行并回滚，或使用独立的库。

### 异步测试用 pytest-asyncio 或 anyio 【推荐】

`pytest-asyncio` 默认是严格模式，测试要加 `@pytest.mark.asyncio`。项目全是异步代码时可以设 `asyncio_mode = "auto"`。FastAPI 项目可以用 `@pytest.mark.anyio`。

**来源：** pytest-asyncio 文档、FastAPI 文档

## 基准测试

### 用 pytest-benchmark 测量 【推荐】

性能断言不靠手写计时。用 `benchmark` fixture 测量，并校验返回值。基准放在单独目录，在固定环境中运行，用 `--benchmark-autosave` 保存结果，用 `--benchmark-compare` 与上次比较。

**示例：**

```python
def test_parse_invoice_speed(benchmark: BenchmarkFixture) -> None:
    result = benchmark(parse_invoice, SAMPLE_INVOICE)
    assert result.total == Decimal("120.00")
```

**来源：** pytest-benchmark 文档

