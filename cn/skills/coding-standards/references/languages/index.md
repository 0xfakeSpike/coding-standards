# 语言层入口

根据文件扩展名和项目配置判断语言，再按任务只读对应目录下的相关文件。

| 语言 | 目录 | 专项文件 |
|---|---|---|
| TypeScript / JavaScript（含 Node.js） | `typescript/` | [node.md](typescript/node.md) |
| Python | `python/` | [django.md](python/django.md)、[fastapi.md](python/fastapi.md) |
| Go | `go/` | [web-services.md](go/web-services.md) |

每门语言的目录下都有以下文件：

| 文件 | 何时读取 |
|---|---|
| `style.md` | 工具配置、格式、命名、导入 |
| `types.md` | 类型设计 |
| `idioms.md` | 语言惯用法 |
| `errors.md` | 错误和异常处理 |
| `concurrency.md` | 异步和并发 |
| `project.md` | 项目结构、依赖和构建 |
| `testing.md` | 编写测试 |
| `security.md` | 语言相关的安全问题 |

**未收录的语言：** 遵循该语言的官方或社区风格指南和主流格式化工具，并告诉用户本手册没有这门语言的细则。领域层和通用层仍然适用。
