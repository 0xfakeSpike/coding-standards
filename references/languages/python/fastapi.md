# FastAPI

## 路由组织

### 按领域拆分 APIRouter 【推荐】

接口变多后按领域拆分，每个领域一个 `APIRouter`，放在对应的业务模块内，小项目可以只有一个文件。`main.py` 只创建应用、注册路由和配置生命周期。多个模块都导出 `router` 时，用 `from app import items, users`，再注册 `items.router`，避免重名。

**来源：** FastAPI 文档

### 公共配置写在路由器上 【推荐】

`prefix`、`tags` 和鉴权依赖写在 `APIRouter(...)` 上，不在每个路径操作上重复。

**来源：** FastAPI 文档

### 路由函数只处理 HTTP 【必须】

路由函数负责解析入参、调用业务函数、返回结果。业务规则放在业务模块，不依赖 `Request`、`HTTPException`。业务层抛领域异常，用 `@app.exception_handler()` 转成响应，不在每个路由写 `try/except`。分层原则见 [layering.md](../../backend/architecture/layering.md)。

**来源：** FastAPI 文档

### 启动和关闭用 lifespan 【推荐】

连接池等资源用 `lifespan` 异步上下文管理器管理。不用 `@app.on_event`，传入 `lifespan` 后事件处理器不再被调用。

**来源：** FastAPI 文档

## Pydantic

### 输入和输出分开建模 【必须】

为创建、更新和响应分别定义模型。响应模型不含密码等内部字段。优先用返回类型声明响应，FastAPI 按它过滤和校验输出。

**原因：** 输入输出共用一个模型时，敏感字段很容易被原样返回。

**示例：**

```python
class UserCreate(BaseModel):
    email: EmailStr
    password: SecretStr


class UserPublic(BaseModel):
    id: int
    email: EmailStr


@router.post("/users")
async def create_user(data: UserCreate, session: SessionDep) -> UserPublic:
    return await register_user(session, data)
```

**工具：** `FAST001`。

**来源：** FastAPI 文档

### 校验声明在模型上 【推荐】

长度和范围用 `Annotated[int, Field(ge=1)]` 表达。复杂校验用 `AfterValidator`，失败时抛 `ValueError`，不用 `assert`，`python -O` 下断言会被跳过。入参模型设 `model_config = ConfigDict(extra="forbid")`，拼错的字段直接报错。Pydantic 默认会转换类型，比如把 `"123"` 转成 `123`，接口要求类型精确时用 `strict=True`。

**来源：** Pydantic 文档、FastAPI 文档

### 敏感值和 ORM 对象用模型能力处理 【推荐】

密码和令牌字段用 `SecretStr`，`repr` 和 `str` 都显示为掩码，取值时显式调用 `get_secret_value()`。从 ORM 对象构造响应模型时设 `from_attributes=True`，不手工逐字段拷贝。

**来源：** Pydantic 文档

### 配置用 BaseSettings 【推荐】

配置类继承 `pydantic_settings.BaseSettings`，用 `@lru_cache` 修饰的 `get_settings()` 提供，并作为依赖注入，测试时可以覆盖。

**来源：** FastAPI 文档

## 依赖注入

### 依赖用 Annotated 声明 【推荐】

写 `Annotated[T, Depends(fn)]`，不写 `param: T = Depends(fn)`。重复使用时定义成类型别名。

**示例：**

```python
SessionDep = Annotated[AsyncSession, Depends(get_session)]
CurrentUser = Annotated[User, Depends(get_current_user)]
```

**工具：** `FAST002`。

**来源：** FastAPI 文档

### 需要清理的资源用 yield 依赖 【必须】

资源在 `yield` 前创建，用 `try/finally` 释放。依赖里捕获异常后必须重新抛出，否则 FastAPI 感知不到错误，服务端也不留记录。

**示例：**

```python
async def get_session() -> AsyncIterator[AsyncSession]:
    async with session_factory() as session:
        yield session
```

**来源：** FastAPI 文档

### 了解 yield 依赖的退出时机 【推荐】

默认在响应发送之后才执行退出代码。不需要等响应发送完就能释放的资源，用 `Depends(get_x, scope="function")` 提前释放。`scope="request"` 的依赖不能依赖 `scope="function"` 的依赖。同一依赖在一次请求中只执行一次，需要重新计算时写 `use_cache=False`。

**来源：** FastAPI 文档

### 测试用 dependency_overrides 替换依赖 【推荐】

测试通过 `app.dependency_overrides[get_session]` 替换依赖，结束后清空。不用 `patch` 改模块内部。

**来源：** FastAPI 文档

## 异步数据库

### 路由类型匹配驱动类型 【必须】

异步驱动配 `async def`。只有同步驱动时写普通 `def`，FastAPI 会放进线程池运行。`async def` 里不调用同步驱动，否则会阻塞事件循环。

**工具：** `ASYNC` 规则组。

**来源：** FastAPI 文档

### 全局一个引擎，每个请求一个会话 【必须】

用 `create_async_engine` 创建一个引擎，用 `async_sessionmaker` 创建工厂。`AsyncSession` 通过 yield 依赖按请求提供。在 `lifespan` 退出阶段调用 `await engine.dispose()`，异步连接无法在垃圾回收时正常关闭。

**来源：** SQLAlchemy 文档、FastAPI 文档

### 会话不跨任务共享 【必须】

一个 `AsyncSession` 不能在多个并发任务中使用。`TaskGroup` 或 `gather` 中的每个任务都要使用自己的会话。

**原因：** 会话是有状态对象，代表一个进行中的事务。

**来源：** SQLAlchemy 文档

### 避免隐式懒加载 【必须】

关联用 `selectinload` 显式加载。工厂写 `async_sessionmaker(engine, expire_on_commit=False)`，避免提交后访问属性时触发 I/O。

**原因：** 异步模式下，属性访问不能隐式发起查询。

**来源：** SQLAlchemy 文档

### 表结构由迁移工具管理 【必须】

**适用：** 线上服务

生产环境用 Alembic 管理表结构，应用启动时不调用 `create_all`。

**来源：** FastAPI 文档

## 后台任务

### BackgroundTasks 只做轻量工作 【推荐】

`BackgroundTasks` 适合发通知这类小任务，在响应发送后于同一进程运行。计算量大或要跨机器运行的任务交给 Celery 这类任务队列。任务只存在于进程内存中，没有重试，必须完成的工作写入持久化队列。

**示例：**

```python
@router.post("/reports", status_code=202)
async def request_report(user: CurrentUser, tasks: BackgroundTasks) -> None:
    tasks.add_task(send_report, user.id)
```

**来源：** FastAPI 文档

### 后台任务自己获取资源并记录异常 【必须】

任务函数接收 ID 这类普通数据，自己创建数据库会话，不接收请求依赖的会话。响应已经发出，异常无法再告诉调用方，要在任务函数最外层记录。

**原因：** 请求级资源的生命周期由依赖管理，后台任务不应依赖它的退出时机。

