# FastAPI

## Route organization

### Split APIRouter by domain [Recommended]

As endpoints grow, split them by domain: one `APIRouter` per domain, in the matching business module. A small project may have just one file. `main.py` only creates the app, registers routers, and configures the lifespan. When several modules export `router`, use `from app import items, users` and register `items.router` to avoid name clashes.

**Source:** FastAPI Docs

### Put shared configuration on the router [Recommended]

Put `prefix`, `tags`, and authorization dependencies on `APIRouter(...)`. Do not repeat them on each path operation.

**Source:** FastAPI Docs

### Route functions handle only HTTP [Recommended]

Route functions parse input, call business functions, and return results. Business rules live in business modules and do not depend on `Request` or `HTTPException`. The business layer raises domain exceptions, and `@app.exception_handler()` converts them into responses. Do not write `try/except` in every route. For layering principles, see [layering.md](../../backend/architecture/layering.md).

**Source:** FastAPI Docs

### Use lifespan for startup and shutdown [Recommended]

Manage resources such as connection pools with the `lifespan` async context manager. Do not use `@app.on_event`: event handlers are no longer called once `lifespan` is passed.

**Source:** FastAPI Docs

## Pydantic

### Model input and output separately [Required]

Define separate models for create, update, and response. Response models do not include internal fields such as passwords. Prefer declaring the response through the return type. FastAPI filters and validates output by it.

**Why:** When input and output share a model, sensitive fields are easily returned as-is.

**Example:**

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

**Tooling:** `FAST001`.

**Source:** FastAPI Docs

### Declare validation on the model [Recommended]

Express length and range with `Annotated[int, Field(ge=1)]`. For complex validation, use `AfterValidator` and raise `ValueError` on failure. Do not use `assert`: assertions are skipped under `python -O`. Set `model_config = ConfigDict(extra="forbid")` on input models so misspelled fields raise errors. Pydantic converts types by default, for example `"123"` to `123`. Use `strict=True` when the endpoint requires exact types.

**Source:** Pydantic Docs, FastAPI Docs

### Use model features for sensitive values and ORM objects [Recommended]

Use `SecretStr` for password and token fields. `repr` and `str` both show a mask. Call `get_secret_value()` explicitly to read the value. When building response models from ORM objects, set `from_attributes=True`. Do not copy fields one by one by hand.

**Source:** Pydantic Docs

### Use BaseSettings for configuration [Recommended]

Configuration classes inherit from `pydantic_settings.BaseSettings`, are provided by a `get_settings()` decorated with `@lru_cache`, and are injected as a dependency so tests can override them.

**Source:** FastAPI Docs

## Dependency injection

### Declare dependencies with Annotated [Recommended]

Write `Annotated[T, Depends(fn)]`, not `param: T = Depends(fn)`. For repeated use, define a type alias.

**Example:**

```python
SessionDep = Annotated[AsyncSession, Depends(get_session)]
CurrentUser = Annotated[User, Depends(get_current_user)]
```

**Tooling:** `FAST002`.

**Source:** FastAPI Docs

### Use yield dependencies for resources that need cleanup [Required]

Create the resource before `yield` and release it with `try/finally`. After catching an exception in a dependency, you must re-raise it. Otherwise FastAPI does not see the error and the server leaves no record.

**Example:**

```python
async def get_session() -> AsyncIterator[AsyncSession]:
    async with session_factory() as session:
        yield session
```

**Source:** FastAPI Docs

### Know when yield dependencies exit [Recommended]

By default, exit code runs after the response is sent. For resources that can be released without waiting for the response to finish sending, use `Depends(get_x, scope="function")` to release them early. A `scope="request"` dependency cannot depend on a `scope="function"` dependency. The same dependency runs only once per request. Write `use_cache=False` when it must be recomputed.

**Source:** FastAPI Docs

### Replace dependencies in tests with dependency_overrides [Recommended]

Tests replace dependencies through `app.dependency_overrides[get_session]` and clear it afterwards. Do not use `patch` to change module internals.

**Source:** FastAPI Docs

## Async database

### Match the route type to the driver type [Required]

Pair async drivers with `async def`. With only a sync driver, write plain `def`, and FastAPI runs it in a thread pool. Do not call sync drivers in `async def`, or you block the event loop.

**Tooling:** `ASYNC` rule group.

**Source:** FastAPI Docs

### One global engine, one session per request [Required]

Create one engine with `create_async_engine` and a factory with `async_sessionmaker`. Provide `AsyncSession` per request through a yield dependency. Call `await engine.dispose()` in the `lifespan` exit phase, because async connections cannot close properly during garbage collection.

**Source:** SQLAlchemy Docs, FastAPI Docs

### Do not share sessions across tasks [Required]

One `AsyncSession` cannot be used in multiple concurrent tasks. Each task in a `TaskGroup` or `gather` must use its own session.

**Why:** A session is a stateful object that represents one in-progress transaction.

**Source:** SQLAlchemy Docs

### Avoid implicit lazy loading [Required]

Load relationships explicitly with `selectinload`. Write the factory as `async_sessionmaker(engine, expire_on_commit=False)` to avoid triggering I/O on attribute access after commit.

**Why:** In async mode, attribute access cannot implicitly issue queries.

**Source:** SQLAlchemy Docs

### Manage table schemas with a migration tool [Required]

**Applies to:** production service

In production, manage table schemas with Alembic. Do not call `create_all` at application startup.

**Source:** FastAPI Docs

## Background tasks

### Use BackgroundTasks only for light work [Recommended]

`BackgroundTasks` suits small jobs such as sending notifications, and runs in the same process after the response is sent. Hand compute-heavy jobs and jobs that run across machines to a task queue such as Celery. Tasks live only in process memory and have no retries. Write work that must complete to a durable queue.

**Example:**

```python
@router.post("/reports", status_code=202)
async def request_report(user: CurrentUser, tasks: BackgroundTasks) -> None:
    tasks.add_task(send_report, user.id)
```

**Source:** FastAPI Docs

### Background tasks acquire their own resources and log exceptions [Required]

Task functions receive plain data such as IDs and create their own database sessions. They do not receive sessions from request dependencies. The response has already gone out, so exceptions can no longer reach the caller. Log them at the outermost level of the task function.

**Why:** Dependencies manage the lifecycle of request-scoped resources, and background tasks should not depend on when they exit.
