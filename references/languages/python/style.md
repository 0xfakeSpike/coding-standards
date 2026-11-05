# Python 风格

本文规定工具配置、格式、命名、导入和文件组织。通用命名原则见 [naming.md](../../general/naming.md)。

## 工具配置

### 用 Ruff 统一格式化和静态检查 【推荐】

**适用：** 多人协作

格式化用 `ruff format`，静态检查和导入排序用 `ruff check`，类型检查另用 mypy 或 pyright。这些工具的配置都放在 `pyproject.toml`，不分散到多个文件。

**原因：** 配置分散后，本地和 CI 容易读到不同的规则。

**例外：** 已经使用 Black、Flake8 和 isort 的项目可以继续使用，但同一职责只保留一个工具。

**来源：** Ruff 文档

### 显式选择 lint 规则 【推荐】

用 `lint.select` 列出规则组，不用 `ALL`。先用下面的基线，再按需增减。使用 `ruff format` 时不启用与它冲突的规则：`W191`、`E111`、`E114`、`E117`、`Q`、`COM812`、`COM819`。

**原因：** `ALL` 会在升级 Ruff 时隐式启用新规则，同样的代码可能突然不过门禁。

**示例：**

```toml
[tool.ruff]
line-length = 88

[tool.ruff.lint]
select = [
    "E", "F", "I", "N", "UP", "B", "A", "SIM", "RUF",
    "ASYNC", "BLE", "DTZ", "G", "LOG", "PT", "S",
    "ERA001", "FIX", "PGH", "TRY201", "TRY400", "TRY401",
]
ignore = ["E111", "E114", "E117"]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101"]
```

**例外：** Django 项目追加 `DJ`，FastAPI 项目追加 `FAST`。

**来源：** Ruff 文档

### 门禁只检查不改写 【推荐】

CI 运行 `ruff format --check` 和 `ruff check`，不在 CI 里自动修复。`--unsafe-fixes` 只在本地使用，并逐条审查结果。

**原因：** 不安全修复可能改变运行时行为。

**来源：** Ruff 文档

## 格式

### 格式以工具输出为准 【推荐】

行宽默认用 Ruff 的 88，PEP 8 建议 79，团队约定时可放宽到 99，一个项目只设一个值。不手工对抗格式化器，`# fmt: off` 只用于确实需要对齐的数据表，并且作用于整条语句。

**来源：** PEP 8、Ruff 文档

### 注释只写代码表达不了的原因 【必须】

不写复述代码的注释，包括只重复名字和类型的 docstring，对外发布的库的 API 文档和许可证头除外。靠命名和结构表达意图。工具指令除外，比如 `# type: ignore[...]`、`# noqa: ...`、`# fmt: off`、`# nosec`。代码表达不了的原因就近写一句，比如外部系统的限制、反直觉的行为，以及忽略错误、类型断言、抑制 lint 这类单点取舍。跨多处代码的注意事项记入项目文档，见 [principles.md](../../general/principles.md)。

**工具：** 用 `ERA001` 检查注释掉的代码，用 `FIX` 检查待办注释。

**来源：** 团队

## 命名

### 按对象类型命名 【推荐】

| 对象 | 规则 | 示例 |
|---|---|---|
| 变量、函数、方法 | `snake_case` | `order_total`、`fetch_user` |
| 类 | `CapWords`，缩写全大写 | `OrderService`、`HTTPClient` |
| 异常类 | `CapWords`，以 `Error` 结尾 | `PaymentError` |
| 常量 | `UPPER_SNAKE_CASE`，定义在模块级 | `MAX_RETRY_COUNT` |
| 模块 | 简短的 `snake_case`，不含连字符 | `order_service.py` |
| 包 | 简短小写，尽量不用下划线 | `billing` |
| 内部成员 | 单下划线开头 | `_parse_row` |
| 布尔值 | `is_`、`has_`、`can_` 开头 | `is_active` |
| 类型变量 | 短 `CapWords`，协变加 `_co`，逆变加 `_contra` | `T`、`KT_contra` |

**工具：** `N` 规则组。

**来源：** PEP 8、Google Python Style Guide

### 映射变量的名字说明 key 【必须】

字典变量用 `<值>_by_<key>` 命名。名字不重复类型，不写 `user_dict`。

**原因：** 注释不解释代码在做什么，只有名字能告诉读者 key 是什么。

**示例：**

反例：

```python
prices = {"A100": Decimal("9.90")}
```

正例：

```python
price_by_sku: dict[str, Decimal] = {"A100": Decimal("9.90")}
```

**来源：** 团队

## 导入排序

### 分组排序 【推荐】

导入放在文件顶部，按标准库、第三方、本项目分三组，组间空一行。一条 `import` 语句只导入一个模块。`typing` 和 `collections.abc` 的符号可以直接导入，容器抽象类型从 `collections.abc` 导入。

**工具：** `I` 规则组、`UP035`。

**来源：** PEP 8、Google Python Style Guide

### 不用通配导入 【推荐】

不写 `from x import *`。

**原因：** 名字来源不清，容易互相覆盖，工具也无法检查未定义的名字。

**工具：** `F403`。

**来源：** PEP 8

### 优先绝对导入 【推荐】

跨包导入写完整路径。同一包内的兄弟模块可以用单点相对导入，不用 `..` 向上跳。

**工具：** `TID252`。

**来源：** PEP 8

## 文件组织

### 相关内容放在同一模块 【推荐】

Python 不要求一个类一个文件。关系紧密的类和函数放在一起，模块按职责划分，不按语法种类划分。

**来源：** Google Python Style Guide

### 入口放在 main 函数 【推荐】

可执行文件的逻辑放在 `main()`，由 `if __name__ == "__main__":` 调用。导入模块时不产生副作用。

**原因：** 否则测试和工具导入模块时就会执行程序。多进程的 `spawn` 和 `forkserver` 也会重新导入主模块。

**示例：**

```python
def main() -> None:
    run_server()


if __name__ == "__main__":
    main()
```

**来源：** Google Python Style Guide、Python 标准库文档

### 避免可变全局状态 【推荐】

模块级不放可变业务状态。logger、框架的应用和路由对象、数据库引擎、`ContextVar`、类型别名这类随进程存在的对象可以放在模块级。确实需要全局状态时，用单下划线命名，并通过函数访问。

**原因：** 全局状态让导入顺序影响行为，测试之间也会互相污染。

**工具：** `PLW0603`。

**来源：** Google Python Style Guide
