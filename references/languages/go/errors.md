# Go 错误处理

语言无关的原则见 [general/error-handling.md](../../general/error-handling.md)，本文只写 Go 的机制。

## 错误机制

### error 是最后一个返回值，并立即检查 【必须】

返回类型写 `error`，不写具体错误类型。`err != nil` 时其他返回值视为无效。接收 `context.Context` 的函数通常也要返回 `error`。

**来源：** Google Go 风格指南

### 不用 _ 丢弃错误 【必须】

要么处理，要么返回。确实可以忽略时，在旁边写一句注释说明原因。只读资源的 `Close` 和 `defer tx.Rollback()` 可以不检查，事务提交后 `Rollback` 返回 `sql.ErrTxDone`。写入路径按 [idioms.md](idioms.md) 检查。

**工具：** `errcheck`

**来源：** Go Code Review Comments、Google Go 风格指南

### 普通错误不用 panic 【必须】

`panic` 只用于无法继续的程序缺陷，或启动阶段的致命失败。`MustXxx` 只用于包级变量和启动初始化，不用于用户输入和请求处理。`recover` 不是错误处理手段，只在请求边界、后台任务入口和工具执行入口使用。

**来源：** Effective Go、Go Code Review Comments、Uber Go 风格指南

### 只在 main 退出进程 【推荐】

`os.Exit` 和 `log.Fatal` 只在 `main` 中调用一次，它们会跳过 `defer`。其余逻辑放进返回 `error` 的 `run`。

**来源：** Uber Go 风格指南

### 错误信息小写开头，不以标点结尾 【推荐】

错误信息通常拼进更长的信息。专有名词和缩写保持原样。这条不适用于日志。

**工具：** `staticcheck` 的 ST1005

**来源：** Go Code Review Comments、Google Go 风格指南

## 自定义错误及其归属

### 每个包定义自己需要的 error 【必须】

包返回的错误定义在这个包内，不设所有包共用的 `errs`、`errors`、`common` 错误包。上层需要区分时，引用下层包的错误，或者在自己的包内定义并转换。

**示例：**

反例：

```go
package errs

var (
	ErrOrderNotFound = errors.New("order not found")
	ErrUserNotFound  = errors.New("user not found")
)
```

正例：

```go
package order

var ErrNotFound = errors.New("order not found")

package user

var ErrNotFound = errors.New("user not found")
```

**来源：** 团队

### 按调用方是否要判断来选错误形式 【推荐】

| 调用方要判断 | 信息 | 写法 |
|---|---|---|
| 否 | 固定 | `errors.New` |
| 否 | 动态 | `fmt.Errorf` |
| 是 | 固定 | 包级 `var ErrXxx = errors.New(...)` |
| 是 | 动态 | 自定义类型 `XxxError` |

没有调用方判断的错误不导出。导出的错误变量和类型都是包的 API。

**来源：** Uber Go 风格指南、Google Go 风格指南

### 错误带上结构化字段 【推荐】

调用方需要从错误里取数据时，放进自定义类型的字段，比如 `os.PathError` 的路径，不让调用方解析字符串。包住下层错误时实现 `Unwrap`。

**来源：** Google Go 风格指南、Go 博客：错误处理

## 包装与判断

### 补充上下文时用 %w 【必须】

写清正在做的动作和对象。`%w` 放在末尾，让文字顺序和错误链一致。

**示例：**

```go
if err := gateway.Charge(ctx, order); err != nil {
	return fmt.Errorf("charge order %s: %w", order.ID, err)
}
```

**例外：** 哨兵错误作为分类标签时可以放在开头，写成 `%w: 详情`。

**工具：** `errorlint`

**来源：** Google Go 风格指南、Uber Go 风格指南、Go 发行说明

### 上下文不写废话 【推荐】

不写 `failed to`、`unable to`，否则层层堆积。不重复下层已有的信息，`os.Open` 的错误已含路径。没有新信息时直接 `return err`。

**来源：** Google Go 风格指南、Uber Go 风格指南

### 实现细节的错误不穿过包边界 【推荐】

用 `%w` 包装等于把下层错误变成自己的 API。调用方需要判断的情况转换成本包错误，不让调用方依赖 `sql.ErrNoRows` 这类实现细节。要隐藏全部下层错误时用 `%v`。

**示例：**

反例：

```go
func (s *Store) ByID(ctx context.Context, id int64) (Order, error) {
	var o Order
	err := s.db.QueryRowContext(ctx, orderByIDSQL, id).Scan(&o.ID, &o.TotalCents)
	if err != nil {
		return Order{}, fmt.Errorf("order by id %d: %w", id, err)
	}
	return o, nil
}
```

正例：

```go
func (s *Store) ByID(ctx context.Context, id int64) (Order, error) {
	var o Order
	err := s.db.QueryRowContext(ctx, orderByIDSQL, id).Scan(&o.ID, &o.TotalCents)
	if errors.Is(err, sql.ErrNoRows) {
		return Order{}, ErrNotFound
	}
	if err != nil {
		return Order{}, fmt.Errorf("order by id %d: %w", id, err)
	}
	return o, nil
}
```

**来源：** Go 博客：错误处理

### 用 errors.Is 和 errors.As 判断 【必须】

不用 `==` 比较可能被包装的错误，不比较错误字符串。Go 1.26+ 可用类型安全的 `errors.AsType`。多个错误合并用 `errors.Join`（Go 1.20+）。

**示例：**

```go
if errors.Is(err, order.ErrNotFound) {
	return nil
}
if ve, ok := errors.AsType[*ValidationError](err); ok {
	return badField(ve.Field)
}
```

**工具：** `errorlint`

**来源：** Go 博客：错误处理、Google Go 风格指南、Go 发行说明

## 日志配合

### 在请求边界用 slog 记录错误 【推荐】

记录和返回只选一个，原则见 [general/error-handling.md](../../general/error-handling.md)。记录放在请求边界或后台任务入口，用 `log/slog` 把错误作为字段传入。

**来源：** Uber Go 风格指南、Google Go 风格指南

## 框架边界

### 在传输层统一映射错误 【推荐】

业务包不引用 HTTP 状态码或 gRPC `codes`。Handler 用 `errors.Is`、`errors.As` 把包错误映射成响应，对外信息的要求见 [web-services.md](web-services.md)。

### 长期运行的 goroutine 要接住 panic 【必须】

**适用：** 线上服务

`net/http` 只恢复 Handler 所在 goroutine 的 panic。自己启动的 goroutine 发生 panic 会让整个进程退出，长期运行的后台任务和工具执行入口用 `recover` 接住 panic，转成错误并记录。gRPC 的要求见 [web-services.md](web-services.md)。

**来源：** Go 标准库文档
