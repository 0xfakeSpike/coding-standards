# Python 设计模式写法

本文写设计模式在 Python 里的惯用形态。什么时候用哪个模式见 [design-patterns.md](../../general/design-patterns.md)。函数、模块和生成器都是一等对象，大多数模式不需要类，见 [idioms.md](idioms.md) 的“能用函数就不写类”。接口用 `Protocol`，见 [types.md](types.md) 的“用 Protocol 描述依赖的能力”。

## 结构

### 单例用模块，不写 `__new__` 或元类 【推荐】

模块只导入一次，模块级对象天然只有一份。需要延迟创建时，用 `functools.cache` 修饰一个无参函数。可变的业务状态不放模块级，见 [style.md](style.md) 的“避免可变全局状态”。

**示例：**

```python
import functools


@functools.cache
def get_settings() -> Settings:
    return Settings.from_env()
```

**来源：** Python 标准库文档

### 备选构造用 `classmethod`，可选参数用关键字参数 【推荐】

不写 `XxxFactory` 类和建造者。从其他格式构造对象时写 `from_dict`、`from_row` 这样的 `classmethod`。可选参数用带默认值的仅关键字参数或 `dataclass`，见 [idioms.md](idioms.md) 的“容易混淆的参数用仅关键字参数”。基于不可变对象改几个字段时用 `dataclasses.replace`。

**示例：**

```python
from dataclasses import dataclass
from typing import Self


@dataclass(frozen=True, slots=True)
class Money:
    cents: int
    currency: str

    @classmethod
    def from_row(cls, row: Row) -> Self:
        return cls(cents=row.amount_cents, currency=row.currency)
```

### 属性直接访问，需要计算时才用 `property` 【推荐】

不写 `get_name()`、`set_name()`。字段直接访问，需要计算、校验或保持不变量时改成 `@property`，调用方不用改。

**来源：** Google Python Style Guide

## 行为

### 策略用函数或函数字典 【推荐】

每种情况一个函数，按枚举放进字典。字典没有编译期的完整性检查，用一个测试确认每个枚举成员都有对应的函数。策略需要多个方法时，用 `Protocol` 描述，由各自的类实现。

**示例：**

```python
from collections.abc import Callable

EXPORTERS: dict[ExportFormat, Callable[[list[Row]], bytes]] = {
    ExportFormat.CSV: to_csv,
    ExportFormat.XLSX: to_xlsx,
}


def test_every_format_has_exporter() -> None:
    assert set(EXPORTERS) == set(ExportFormat)
```

### 状态机用枚举加 `match` 【推荐】

状态用 `enum.Enum` 表达，迁移函数用 `match` 按状态分支，每个分支返回下一个状态，最后用 `assert_never` 让类型检查器发现漏处理的状态。各状态需要不同的数据时，用几个 `dataclass` 的联合代替枚举，按类模式匹配。

**示例：**

```python
from enum import Enum, auto
from typing import assert_never


class Step(Enum):
    AWAITING_TOKEN = auto()
    UPLOADING = auto()
    MERGING = auto()
    DONE = auto()


def advance(step: Step, job: UploadJob) -> Step:
    match step:
        case Step.AWAITING_TOKEN:
            job.token = fetch_token()
            return Step.UPLOADING
        case Step.UPLOADING:
            job.parts = upload_parts(job.token)
            return Step.MERGING
        case Step.MERGING:
            job.file_id = merge_parts(job.token, job.parts)
            return Step.DONE
        case Step.DONE:
            return step
        case _:
            assert_never(step)
```

**来源：** Python typing 文档

### 横切逻辑用装饰器，成对的准备和清理用上下文管理器 【推荐】

计时、重试、缓存写成函数装饰器，用 `functools.wraps` 保留原函数的信息。“先准备、再执行、最后清理”这类模板方法，写成 `contextlib.contextmanager`，不写抽象基类让子类覆盖步骤。

**示例：**

```python
import functools
import time
from collections.abc import Callable


def timed[**P, R](fn: Callable[P, R]) -> Callable[P, R]:
    @functools.wraps(fn)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        start = time.perf_counter()
        try:
            return fn(*args, **kwargs)
        finally:
            metrics.observe(fn.__qualname__, time.perf_counter() - start)

    return wrapper
```

**来源：** Python 标准库文档

### 遍历用生成器 【推荐】

分页数据和自定义集合的遍历用 `yield` 写成生成器，组合时用 `itertools`。不写实现 `__next__` 的迭代器类，除非对象本身要维护复杂的遍历状态。

**示例：**

```python
from collections.abc import Iterator


def all_orders(client: Client) -> Iterator[Order]:
    cursor: str | None = None
    while True:
        page = client.list_orders(cursor)
        yield from page.items
        if page.next_cursor is None:
            return
        cursor = page.next_cursor
```

**来源：** Python 标准库文档
