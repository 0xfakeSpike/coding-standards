# Python 惯用写法

本文收录语言用法和常见陷阱。设计层面的原则见 [code-design.md](../../general/code-design.md)。

## 惯用写法

### 不写只转调一次的函数 【必须】

函数体只是把参数原样转给另一个函数时，直接调用目标函数，不加这层封装。

**示例：**

反例：

```python
def get_user(user_id: int) -> User:
    return repository.find_user(user_id)
```

正例：

```python
user = repository.find_user(user_id)
```

**来源：** 团队

### 能用函数就不写类 【推荐】

没有状态的逻辑写成模块级函数，不写只有静态方法的类。

### 资源用 with 管理 【必须】

文件、锁、连接都用 `with` 或 `async with` 管理，不靠析构函数释放。

**工具：** `SIM115`。

**来源：** PEP 8、Google Python Style Guide

### 空值判断用 is 【推荐】

写 `x is None`、`x is not None`，不写 `x == None`。序列判空用 `if not items:`。默认值为 `None` 的参数必须用 `is None` 判断，因为 `0`、`""`、`[]` 也是假值，不写 `x = x or []`。

**工具：** `E711`。

**来源：** PEP 8、Google Python Style Guide

### 容易混淆的参数用仅关键字参数 【推荐】

布尔标志和含义不明的参数放在 `*` 之后，比如 `def create_user(data: UserCreate, *, send_email: bool = False) -> User: ...`。

**工具：** `FBT001`、`FBT002`。

## 集合

### 用内置工具和合适的容器 【推荐】

推导式只用于一层循环加简单条件，多层 `for` 或多个条件时改写成普通循环。用 `enumerate` 取下标，用 `any`、`all` 做判断，`zip` 的输入应等长时写 `strict=True`（Python 3.10+）。成员判断用 `set`，计数用 `Counter`，分组用 `defaultdict`，队列用 `deque`。

**工具：** `B905`。

**来源：** Google Python Style Guide

## 字符串

### 字符串和路径用现代写法 【推荐】

格式化用 f-string，日志例外，见 [errors.md](errors.md)。循环拼接用 `"".join(parts)`。前后缀用 `startswith`、`removeprefix`，不用切片比较。路径用 `pathlib.Path`，不用字符串拼接。文本读写显式传 `encoding="utf-8"`，默认值可能取决于系统区域设置。

**工具：** `PTH` 规则组、`PLW1514`。

**来源：** PEP 8、Python 标准库文档

## 日期时间

### 使用带时区的时间 【必须】

用 `datetime.now(UTC)`（`datetime.UTC` 要求 Python 3.11+），不用 `datetime.utcnow()`，它返回不带时区的值。具名时区用 `zoneinfo.ZoneInfo("Asia/Shanghai")`，不手写固定偏移。没有系统时区库的环境（如 Windows）要声明 `tzdata` 依赖。计算耗时用 `time.monotonic()`，`time.time()` 会受系统校时影响。

**原因：** 很多 `datetime` 方法把不带时区的值当作本地时间，换一台机器结果就变了。

**工具：** `DTZ` 规则组。

**来源：** Python 标准库文档

## 数值与金额

### 金额不用浮点数 【必须】

用 `decimal.Decimal` 或整数表示的最小货币单位。`Decimal` 从字符串构造，用 `quantize` 显式指定精度和舍入方式。

**原因：** `Decimal(0.1)` 会把二进制浮点误差原样带进来。

**示例：**

反例：

```python
total = Decimal(19.99) * 3
```

正例：

```python
total = (Decimal("19.99") * 3).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
```

**来源：** Python 标准库文档

## 常见陷阱

### 不用 assert 做运行时校验 【必须】

`python -O` 会移除 `assert`。校验参数和业务条件时用 `if` 并抛异常。测试代码中的 `assert` 不受此限。

**工具：** `S101`。

**来源：** Google Python Style Guide

