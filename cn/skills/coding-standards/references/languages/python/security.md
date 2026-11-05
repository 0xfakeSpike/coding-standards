# Python 安全

通用后端安全见 [后端安全规范](../../backend/index.md)。本文只讲 Python 特有的风险。

## 语言特有风险

### 不对外部输入使用 eval 和 exec 【必须】

`eval`、`exec`、`compile` 不接收任何外部数据。解析字面量用 `ast.literal_eval`，解析数据用 `json`。

**原因：** 这些函数会以当前进程权限执行任意代码。

**工具：** `S307`、`S102`。

**来源：** Ruff 文档、Bandit 文档

### SQL 和模板不拼接外部输入 【必须】

不用 f-string、`%` 或 `format` 拼 SQL，用驱动或 ORM 的参数绑定。Jinja2 同样不关闭 `autoescape`，Django 的相关规则见 [django.md](django.md)。

**示例：**

反例：

```python
cursor.execute(f"SELECT * FROM users WHERE email = '{email}'")
```

正例：

```python
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
```

**工具：** `S608`、`S701`。

**来源：** Ruff 文档、Bandit 文档

### 安全地处理文件 【必须】

`tarfile` 的 `extractall` 显式传 `filter="data"`。临时文件用 `NamedTemporaryFile`、`mkstemp`、`TemporaryDirectory`，不用 `tempfile.mktemp`，也不写固定的 `/tmp` 路径。用户提供的路径按 [file-handling.md](../../backend/security/file-handling.md) 处理，`Path.resolve()` 会解析符号链接。

**原因：** 恶意归档可以用路径穿越覆盖目标目录外的文件。`mktemp` 在生成文件名和创建文件之间有时间窗口。

**工具：** `S202`、`S306`、`S108`。

**来源：** Python 标准库文档

### HTTP 调用设超时并校验证书 【必须】

`requests` 调用显式传 `timeout`，它默认不会超时。不写 `verify=False`。

**工具：** `S113`、`S501`。

**来源：** Requests 文档、Ruff 文档

### 解析不可信 XML 前确认 Expat 版本 【必须】

攻击者控制的 XML 可用于拒绝服务。用 `pyexpat.EXPAT_VERSION` 确认运行环境的 Expat 已修复已知问题，或者换用 JSON。`xmlrpc` 有解压炸弹风险。

**工具：** `S314` 及同组规则。

**来源：** Python 标准库文档

## 随机数

### 安全用途用 secrets 【必须】

令牌、密码、验证码用 `secrets`，比如 `secrets.token_urlsafe(32)`。`random` 只用于模拟和测试数据。比较令牌和签名用 `secrets.compare_digest`，不用 `==`，后者的耗时可能被用于时序攻击。

**工具：** `S311`。

**来源：** Python secrets 文档

### 密码用专用算法 【必须】

算法顺序和参数见 [authentication.md](../../backend/security/authentication.md)，Argon2id 用专门的密码库。MD5 和 SHA-1 不用于安全场景，只做校验和时传 `usedforsecurity=False`。

**工具：** `S324`。

**来源：** Python 标准库文档

## 序列化

### 不反序列化不可信数据 【必须】

`pickle`、`shelve`、`marshal` 不处理不可信数据。跨信任边界用 JSON，确实要用 pickle 时用 `hmac` 签名防止篡改。YAML 用 `yaml.safe_load`，不用默认加载器的 `yaml.load`。

**原因：** 加载恶意 pickle 数据时会执行任意代码，不安全的 YAML 加载器可以实例化任意对象。

**工具：** `S301`、`S403`、`S506`。

**来源：** Python pickle 文档、Ruff 文档

## 命令执行

### 用参数列表调用子进程 【必须】

`subprocess` 传参数列表，不开启 `shell=True`，不用 `os.system`。外部文件名前加 `--`，防止被解析成选项。写 `check=True` 和 `timeout`，超时后子进程会被终止，并抛 `TimeoutExpired`。

**原因：** 不经过 shell 时，元字符会原样传给子进程。开启 shell 后，转义完全由应用负责。

**示例：**

反例：

```python
subprocess.run(f"convert {filename} out.png", shell=True)
```

正例：

```python
subprocess.run(["/usr/bin/convert", "--", filename, "out.png"], check=True, timeout=30)
```

**工具：** `S602`、`S604`、`S605`。

**来源：** Python subprocess 文档

### 可执行文件用完整路径 【推荐】

使用绝对路径，或者先用 `shutil.which` 解析，按 `PATH` 查找的行为随平台变化。Windows 的 `.bat`、`.cmd` 可能被系统交给 shell 解析，即使没有开启 `shell=True`，Python 也不做转义，不向它们传外部参数。

**工具：** `S607`。

**来源：** Python subprocess 文档

## 静态扫描

### CI 运行安全扫描 【推荐】

**适用：** 多人协作

用 Ruff 的 `S` 规则组做日常检查，或用 `bandit -c pyproject.toml -r src` 做独立扫描，两者规则重叠，选其一即可。抑制时写明规则编号，比如 `# nosec B602`、`# noqa: S603`，否则同一行以后新增的问题也会被隐藏。

**工具：** `PGH004`。

**来源：** Bandit 文档、Ruff 文档

### 依赖漏洞用 pip-audit 扫描 【推荐】

**适用：** 多人协作

CI 对锁定的依赖运行 `pip-audit -r requirements.txt`，它查询 PyPI 或 OSV 的漏洞数据。uv 项目先用 `uv export` 导出。

**来源：** pip-audit 文档
