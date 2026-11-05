# Python 项目结构

本文规定目录、包管理、依赖锁定和构建。通用依赖原则见 [dependencies.md](../../general/dependencies.md)。

## 目录结构

### 使用 src 布局 【推荐】

可导入的代码放在 `src/<包名>/`，测试放在顶层 `tests/`。

**原因：** 测试跑的是安装后的包，打包遗漏能被提前发现，当前目录下的代码也不会意外遮蔽已安装的包。

**示例：**

```text
project/
├── pyproject.toml
├── uv.lock
├── src/
│   └── billing/
│       ├── __init__.py
│       └── invoices.py
└── tests/
    └── test_invoices.py
```

**例外：** Django 项目使用 `manage.py` 所在目录作为根。

**来源：** Python 打包指南、pytest 文档

### __init__.py 保持轻量 【推荐】

`__init__.py` 只放必要的公共导出，不执行 I/O、不建立连接、不读取配置。

**原因：** 导入包就会执行这个文件，副作用会拖慢启动，并容易引起循环导入。

### 命令入口用 project.scripts 声明 【推荐】

命令行工具在 `[project.scripts]` 中声明，不让用户运行源码路径。这要求项目定义构建系统。

**来源：** uv 文档

## 包管理

### 使用虚拟环境 【推荐】

依赖只装进项目的虚拟环境，不装进系统 Python。`.venv` 不提交到版本控制。

**来源：** uv 文档

### 新项目用 uv，现有 Poetry 项目继续用 Poetry 【推荐】

一个项目只用一种包管理器。依赖通过命令增删，不手工改锁定文件。

| 操作 | uv | Poetry |
|---|---|---|
| 添加依赖 | `uv add httpx` | `poetry add httpx` |
| 添加开发依赖 | `uv add --dev pytest` | `poetry add -G dev pytest` |
| 按锁定文件同步 | `uv sync --locked` | `poetry sync` |
| 运行命令 | `uv run pytest` | `poetry run pytest` |
| 校验锁定文件 | `uv lock --check` | `poetry check --lock` |

**来源：** uv 文档、Poetry 文档

### 依赖声明遵循标准表 【推荐】

运行时依赖写在 `[project] dependencies`，开发依赖写在 `[dependency-groups]`，发布给用户的可选功能写在 `[project.optional-dependencies]`。

**原因：** 这些表都有标准定义，换工具时不用重写。

**示例：**

```toml
[project]
name = "billing"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = ["httpx>=0.27,<1"]

[dependency-groups]
dev = ["pytest>=9", "ruff>=0.17"]
```

**来源：** uv 文档、Poetry 文档

### 只有依赖的项目关闭打包 【推荐】

不需要构建分发包的应用，uv 用 `uv init --no-package` 创建，Poetry 设 `package-mode = false`。

**来源：** uv 文档、Poetry 文档

## 版本与锁定

### 声明 requires-python 【推荐】

`pyproject.toml` 写明支持的 Python 版本下限。Ruff 在未设置 `target-version` 时会据此推断目标版本。

**来源：** uv 文档、Ruff 文档

### 应用提交锁定文件 【必须】

应用提交 `uv.lock` 或 `poetry.lock`，也可以使用带完整版本的 requirements 文件。CI 和部署都严格按锁定文件安装。

**原因：** 不提交锁定文件时，每次构建都可能拿到不同版本，问题无法复现。

**工具：** CI 运行 `uv sync --locked`，锁定文件过期时立即失败。

**来源：** uv 文档、Poetry 文档

### 库声明范围，应用锁定版本 【推荐】

库在 `dependencies` 中声明版本范围，不钉死版本。库的锁定文件只影响自身开发，对使用方没有作用。

**来源：** Poetry 文档

### 显式升级依赖 【推荐】

锁定文件只在有意升级时变化。用 `uv lock --upgrade-package <包名>` 单独升级。

**来源：** uv 文档

## 构建

### 用标准后端构建分发包 【推荐】

项目在 `[build-system]` 中声明后端，用 `uv build` 或 `poetry build` 生成 sdist 和 wheel。不手写 `setup.py`。

**来源：** uv 文档、Poetry 文档

### 容器镜像先装依赖再装项目 【推荐】

先复制 `pyproject.toml` 和锁定文件，安装依赖，再复制源码安装项目。这样修改代码时不用重建依赖层。生产镜像不装开发依赖。

**示例：**

```dockerfile
COPY pyproject.toml uv.lock ./
RUN uv sync --locked --no-install-project --no-dev
COPY . .
RUN uv sync --locked --no-dev --no-editable
```

**来源：** uv 文档

## 多模块或 monorepo

### 紧密关联的包用 uv workspace 【推荐】

同仓库多个包需要共用一组依赖版本时，用 `[tool.uv.workspace]` 声明成员。所有成员共用一个锁定文件，用 `uv run --package <成员>` 操作单个成员。

**示例：**

```toml
[tool.uv.workspace]
members = ["packages/*"]

[tool.uv.sources]
billing-core = { workspace = true }
```

**来源：** uv 文档

### 依赖冲突的包不放进同一 workspace 【推荐】

成员的依赖要求互相冲突，或者需要独立虚拟环境时，不用 workspace。各自建项目，用路径依赖互相引用。Poetry 项目同样使用路径依赖。

**来源：** uv 文档
