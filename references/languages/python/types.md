# Python 类型

本文规定类型注解、空值、泛型、枚举、数据类和边界校验。版本要求写在规则里，项目的最低版本以 `requires-python` 为准。

## 类型系统用法

### 公共接口写完整注解并在 CI 检查 【推荐】

公共函数和方法写完整的参数和返回类型。CI 运行 `mypy --strict` 或 pyright 的 `strict` 模式，strict 同样检查内部函数，依赖缺少类型时按模块放宽。`self`、`cls` 不用注解。

**原因：** 注解不在运行时生效，没有检查器就只是可能过期的文字。

**工具：** `mypy --strict`；pyright 的 `typeCheckingMode = "strict"`。

**来源：** Google Python Style Guide、mypy 文档、pyright 文档

### 使用现代写法 【必须】

写 `list[str]`、`dict[str, int]`、`X | Y`，不写 `List`、`Dict`、`Union`、`Optional`。Python 3.10+。

**工具：** `UP006`、`UP007`、`UP045`。

**来源：** Python typing 文档、Ruff 文档

### 参数宽、返回具体 【推荐】

参数用能满足需求的最宽抽象类型，比如 `Iterable[str]`、`Sequence[str]`、`Mapping[str, int]`，返回值用具体类型。

**来源：** Google Python Style Guide

### 用 Protocol 描述依赖的能力 【推荐】

调用方只关心某几个方法时，用 `typing.Protocol` 声明，由使用方定义。需要共享实现时才用 `abc.ABC`。`@runtime_checkable` 只检查属性是否存在，不检查签名和类型，不能用作校验。这一能力不继承，子协议要用时必须重新标注。

**原因：** 结构化类型不要求实现方继承，模块之间不产生导入依赖。

**示例：**

```python
from typing import Protocol


class Notifier(Protocol):
    def send(self, recipient: str, body: str) -> None: ...


def notify_owner(notifier: Notifier, owner: str) -> None:
    notifier.send(owner, "任务已完成")
```

**来源：** Python typing 文档

### 少用 Any、cast 和类型忽略 【必须】

`Any` 只用于确实无法表达的类型。`cast` 不做运行时检查，只在检查器无法推断时使用。忽略类型错误时写具体错误码，比如 `# type: ignore[no-untyped-call]`，不写裸的 `# type: ignore`。

**工具：** `PGH003`；mypy 的 `warn_unused_ignores`。

**来源：** mypy 文档、Ruff 文档

### 子类方法标注 override 【推荐】

覆写父类方法时加 `@typing.override`。Python 3.12+。

**原因：** 父类方法改名后，检查器会报告子类，而不是让它悄悄失效。

**来源：** Python typing 文档

### 只为注解使用的导入放进 TYPE_CHECKING 【推荐】

解决循环导入或避免昂贵导入时，把导入放到 `if TYPE_CHECKING:` 下。注解中用到这些名字时写成引号或加 `from __future__ import annotations`，Python 3.14 及以上不需要。

**例外：** Pydantic、FastAPI 在运行时解析注解，模型和路由签名用到的类型必须真实导入。

**工具：** `TC001`、`TC004`。

**来源：** Python typing 文档、Ruff 文档

## 空值

### 可空类型显式写出并先收窄 【必须】

默认值为 `None` 时写 `x: int | None = None`，不写 `x: int = None`，PEP 484 已不再推荐隐式可空。使用前先用 `is None` 收窄，不用 `assert` 或 `cast` 绕过检查器。

**工具：** `RUF013`。

**来源：** PEP 484、Ruff 文档

## 泛型

### 用类型参数语法 【推荐】

Python 3.12+ 用 `def first[T](...)`、`class Box[T]`、`type Pairs[T] = ...`。需要支持 3.11 及以下时用 `TypeVar`，内部类型变量以下划线开头。类型变量只在签名中出现一次时没有意义，直接写具体类型或 `object`。

**示例：**

```python
from collections.abc import Callable, Sequence


def first[T](items: Sequence[T]) -> T:
    return items[0]


def logged[**P, R](func: Callable[P, R]) -> Callable[P, R]:
    return func


type UserById[T] = dict[int, T]
```

**工具：** `UP040`、`UP046`、`UP047`。

**来源：** Python typing 文档、Ruff 文档

## 枚举

### 固定取值用枚举 【推荐】

状态和类型值用 `enum.Enum` 表达，需要序列化成字符串时用 `StrEnum`（Python 3.11+）。值只在类型层面约束时可以用 `Literal`。对枚举或联合类型做分支时，最后一支调用 `typing.assert_never`（Python 3.11+），新增成员后检查器会指出遗漏。

**来源：** Python 标准库文档、Python typing 文档

## 数据类

### 结构化数据不传裸 dict 【推荐】

内部数据用 `dataclass`，外部输入用 Pydantic。`TypedDict` 只用于形状确实是字典的数据，比如 JSON 片段。不需要修改的数据写 `frozen=True, slots=True`，字段较多时加 `kw_only=True`。后两个参数要求 Python 3.10+。

**原因：** 裸 `dict` 的字段拼写和类型都得不到检查。

**示例：**

```python
from dataclasses import dataclass, field


@dataclass(frozen=True, slots=True, kw_only=True)
class Report:
    title: str
    tags: list[str] = field(default_factory=list)
```

**工具：** `RUF008`、`RUF009`。

**来源：** Python 标准库文档

## 类型边界校验

### 在边界把外部数据转成已校验类型 【必须】

HTTP 请求、文件、消息和环境变量进入系统时，立即解析成 Pydantic 模型或 dataclass。内部代码只接收已校验的类型，不重复校验。

**原因：** 类型注解不做运行时检查，外部数据不会因为写了注解就变得可信。

**示例：**

```python
from pydantic import BaseModel


class SignupForm(BaseModel):
    email: str
    age: int


def parse_signup(payload: bytes) -> SignupForm:
    return SignupForm.model_validate_json(payload)
```

**来源：** Pydantic 文档

### 用 NewType 区分同构标识 【推荐】

类型相同但含义不同的值用 `typing.NewType` 区分，比如 `UserId` 和 `OrderId`。它没有运行时开销，也不做校验。

**来源：** Python typing 文档
