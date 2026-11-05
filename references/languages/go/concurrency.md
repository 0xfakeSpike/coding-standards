# Go 并发

## 并发模型

### 优先写同步函数 【推荐】

函数直接返回结果，并发由调用方决定。调用方很容易加 goroutine，去掉多余的并发却很难。

**来源：** Go Code Review Comments、Google Go 风格指南

### 每个 goroutine 都有退出路径和等待者 【必须】

启动时就要确定它何时结束、谁等待它。垃圾回收不会结束阻塞的 goroutine，泄漏的 goroutine 会一直占内存和资源。

**示例：**

反例：

```go
go func() {
	for {
		flush()
		time.Sleep(delay)
	}
}()
```

正例：

```go
func (w *Worker) Run(ctx context.Context) error {
	ticker := time.NewTicker(w.delay)
	defer ticker.Stop()
	for {
		select {
		case <-ticker.C:
			w.flush(ctx)
		case <-ctx.Done():
			return ctx.Err()
		}
	}
}
```

**工具：** 测试用 `go.uber.org/goleak`。

**来源：** Go Code Review Comments、Google Go 风格指南、Uber Go 风格指南、Go 发行说明

### 函数返回前等待自己启动的 goroutine 【推荐】

goroutine 不活过启动它的函数。多个任务用 `sync.WaitGroup`，Go 1.25+ 用 `wg.Go(f)` 代替 `Add`、`Done` 的配对。需要长期运行时，由一个对象持有它，并提供 `Close` 或 `Shutdown` 来通知退出并等待。

**来源：** Google Go 风格指南、Uber Go 风格指南、Go 标准库文档

### 一组可失败的任务用 errgroup 【推荐】

`errgroup.WithContext` 在第一个错误时取消其余任务，`Wait` 返回第一个错误。派生的 ctx 在 `Wait` 返回后失效，不要带到后面使用。

**示例：**

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(maxFetches)
for _, addr := range addrs {
	g.Go(func() error {
		return fetch(ctx, addr)
	})
}
return g.Wait()
```

**来源：** errgroup 文档、Google Go 风格指南

### 简单共享状态用互斥锁 【推荐】

channel 适合转移数据所有权和发信号，计数器、缓存这类状态用锁更直接，不必强行使用 channel。

**来源：** Effective Go

### 由发送方关闭 channel 【必须】

向已关闭的 channel 发送会 panic。多个发送方时，等全部发送完成后由一个协调者关闭。函数参数写 `<-chan T` 或 `chan<- T`，由编译器防止误用。

**来源：** Go 博客：Pipelines、Google Go 风格指南

## 取消与超时

### ctx 作为第一个参数，不放进结构体 【必须】

函数签名写 `func F(ctx context.Context, ...)`，不用自定义 Context 类型，不传 nil，不确定时用 `context.TODO()`。HTTP Handler 用 `r.Context()`，测试用 `t.Context()`（Go 1.24+）。

**原因：** 结构体里的 ctx 掩盖了生命周期，调用方无法为每次调用设置截止时间和取消。

**例外：** 方法签名必须匹配标准库或第三方接口时。

**工具：** `contextcheck`

**来源：** Go 标准库文档、Go Code Review Comments、Google Go 风格指南、Go 博客：Contexts and structs

### 只在入口创建根 context 【推荐】

`context.Background()` 只出现在 `main` 这类入口函数和 `init` 中，中间层使用调用方传入的 ctx。请求结束后还要继续的工作用 `context.WithoutCancel(ctx)`（Go 1.21+），保留值，断开取消。

**来源：** Google Go 风格指南、Go 标准库文档

### 派生 context 后立刻 defer cancel 【必须】

`ctx, cancel := context.WithTimeout(ctx, d)` 的下一行写 `defer cancel()`，否则资源要等父 context 结束才释放。

**工具：** `go vet` 的 `lostcancel`

**来源：** Go 标准库文档

### context.Value 只传请求范围的数据 【推荐】

只放 traceId、认证信息这类跨进程和 API 的数据，不传函数依赖和可选参数。key 用未导出的自定义类型，比如 `type traceIDKey struct{}`，不用字符串。

**来源：** Go 标准库文档、Go Code Review Comments

### 阻塞操作都要响应取消 【必须】

**适用：** 线上服务

channel 收发放进 `select` 并监听 `ctx.Done()`。长循环定期检查 `ctx.Err()`。I/O 用带 ctx 的 API，比如 `QueryContext`、`http.NewRequestWithContext`。

**来源：** Go 博客：Pipelines、Go 博客：Context

## 共享状态

### 并发访问必须同步 【必须】

一个 goroutine 写、另一个同时读写同一内存就是数据竞争，这是缺陷，不是性能取舍。用 channel、`sync` 或 `sync/atomic` 串行化。原子操作用 `atomic.Int64`、`atomic.Bool`（Go 1.19+），不用 `atomic.AddInt64` 操作裸整数。

**来源：** Go 内存模型

### map 并发读写要加锁 【必须】

并发读写普通 `map` 时运行时可能直接崩溃。`sync.Map` 只适合写一次读多次或键集合互不相交的场景，其余用 `map` 加 `sync.RWMutex` 或 `sync.Mutex`。

**来源：** Go 标准库文档

### 不复制含 sync.Mutex 的结构体 【必须】

含锁的类型用指针接收者和指针传递。`sync.WaitGroup`、`atomic.Int64` 同样不可复制。

**工具：** `go vet` 的 `copylocks`

**来源：** Go 标准库文档、Google Go 风格指南

## 资源限制

### 并发度要有上限 【必须】

**适用：** 线上服务

不为每个输入项无限制地启动 goroutine。用 `errgroup.SetLimit`、固定数量的 worker，或者带缓冲的信号量。

**来源：** errgroup 文档

### channel 缓冲默认 0 或 1 【推荐】

更大的缓冲要说清大小的依据、写满后谁阻塞。缓冲不能修复泄漏，只是推迟它。

**来源：** Uber Go 风格指南、Go 博客：Pipelines

### 容器内不手工设置 GOMAXPROCS 【推荐】

Go 1.25+ 在 Linux 上默认按 cgroup CPU 限制设置，并在限制变化时更新。手工设置会关闭这一行为。

**来源：** Go 发行说明

## 常见竞态

### CI 运行 go test -race 【必须】

**适用：** 多人协作

检测器只发现实际执行到的路径，要配合足够的测试和并发场景。内存约增加 5 到 10 倍，耗时约增加 2 到 20 倍，通常不用于生产构建。

**工具：** `go test -race ./...`

**来源：** Go 竞态检测器

