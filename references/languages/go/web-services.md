# Go Web 服务

**适用：** 线上服务（本文全部规则）

接口设计见 [backend/api/rest.md](../../backend/api/rest.md)，本文只写 Go 的实现要求。

## net/http

### 服务端显式配置 http.Server 【必须】

不用 `http.ListenAndServe`，它的超时都是零值，表示没有限制。至少设置 `ReadHeaderTimeout`，并按接口特点设置 `ReadTimeout`、`WriteTimeout`、`IdleTimeout`。流式响应用 `http.ResponseController` 单独调整截止时间。

**示例：**

```go
srv := &http.Server{
	Addr:              cfg.Addr,
	Handler:           mux,
	ReadHeaderTimeout: 5 * time.Second,
	ReadTimeout:       30 * time.Second,
	WriteTimeout:      30 * time.Second,
	IdleTimeout:       2 * time.Minute,
}
```

**来源：** Go 标准库文档

### 用自己的 ServeMux 【推荐】

创建 `http.NewServeMux()`，不用 `http.DefaultServeMux`，任何导入的包都能向后者注册路由。Go 1.22+ 直接写方法和路径参数，简单服务不需要第三方路由。

**示例：**

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /orders/{id}", h.getOrder)
mux.HandleFunc("POST /orders", h.createOrder)
```

**来源：** Go 发行说明、Go 标准库文档

### Handler 只做协议转换 【推荐】

解析请求、调用业务、写响应。业务逻辑不依赖 `http.Request` 和 `gin.Context`，下游调用传 `r.Context()`。

### 限制请求体大小 【必须】

用 `http.MaxBytesReader` 或 `http.MaxBytesHandler` 限制，超限时读取返回 `*http.MaxBytesError`。

**来源：** Go 标准库文档

### 映射错误时不暴露内部信息 【必须】

包错误映射到状态码，未知错误统一返回 500 和通用信息，详情写日志。

## 中间件

### 中间件用标准签名 【推荐】

写成 `func(http.Handler) http.Handler`，顺序在组装处一次写清。请求范围的数据通过 `context` 传递，key 的要求见 [concurrency.md](concurrency.md)。

### 包装 ResponseWriter 时实现 Unwrap 【推荐】

包装类型提供 `Unwrap() http.ResponseWriter`，否则 `http.ResponseController` 无法使用底层的 `Flush`、`SetWriteDeadline` 等能力。

**来源：** Go 标准库文档

### 恢复 panic 要放在最外层 【推荐】

`net/http` 会恢复 Handler 的 panic 并断开连接。需要返回 500 或上报时，用最外层中间件 `recover`，并继续抛出 `http.ErrAbortHandler`。

**来源：** Go 标准库文档

## Gin

### 用 gin.New 显式组装中间件 【推荐】

`gin.Default()` 自带 Logger 和 Recovery。需要结构化日志时用 `gin.New()`，再 `Use` 自己的日志和 `gin.Recovery()`。生产环境设置 `GIN_MODE=release`。Gin 同样放进 `http.Server`，用 `Handler: router` 设置超时，不用 `router.Run`。

**来源：** Gin 文档

### 绑定用 ShouldBind 系列 【推荐】

`ShouldBindJSON` 把错误交回代码统一处理。`BindJSON` 这类方法会自行写 400 和 `text/plain` 响应。校验规则写 `binding` tag，错误转成接口的统一格式。

**来源：** Gin 文档

### 下游调用传 c.Request.Context() 【必须】

默认配置下 `gin.Context` 的 `Done` 返回 nil，不传递取消。传给业务层的是 `c.Request.Context()`，业务层不接收 `*gin.Context`。

**来源：** Gin 文档

### goroutine 里用 c.Copy() 【必须】

`gin.Context` 会被复用，在 Handler 之外使用要先 `c.Copy()`，并且只读。

**来源：** Gin 文档

### 不信任客户端提供的信息 【必须】

Gin 默认信任所有代理，`ClientIP()` 可被请求头伪造，用 `SetTrustedProxies` 列出代理地址，不经过代理时传 `nil`。上传的 `file.Filename` 不能作为保存路径，由服务端生成文件名。

**来源：** Gin 文档

## gRPC 服务

### 客户端用 grpc.NewClient 【推荐】

grpc-go 1.63+ 用 `grpc.NewClient`，不用 `grpc.Dial` 和 `WithBlock`。`ClientConn` 是自动重连的虚拟连接，不需要先确认就绪，错误以 RPC 结果为准。

**来源：** grpc-go 文档

### 每次调用都设截止时间 【必须】

gRPC 默认没有截止时间，客户端可能一直等待，用 `context.WithTimeout` 设置。服务端再调下游时传递入站 ctx，长任务定期检查 `ctx.Err()`。

**来源：** gRPC 文档

### 服务端返回 status 错误 【推荐】

在 RPC 方法或拦截器中把包错误映射为 `status.Error(codes.NotFound, "order not found")`。普通错误到客户端是 `Unknown`。客户端用 `status.Code(err)` 判断，它能识别被包装的 status 错误。业务包不引用 `codes`。

**来源：** gRPC 文档、grpc-go 文档

### 用拦截器处理横切逻辑 【推荐】

日志、认证、指标用 `grpc.ChainUnaryInterceptor` 和流式对应项组装。

**来源：** grpc-go 文档

### gRPC 服务加恢复拦截器 【必须】

gRPC 不会恢复 Handler 的 panic，一次 panic 会让进程退出。

**来源：** grpc-go 文档

## 优雅关闭

### 用 signal.NotifyContext 监听信号 【必须】

Shutdown 的 ctx 必须新建，因为信号 ctx 此时已取消。`ListenAndServe` 在 `Shutdown` 调用时立刻返回 `http.ErrServerClosed`，进程要等 `Shutdown` 返回后才能退出。

**示例：**

```go
func run(ctx context.Context, srv *http.Server) error {
	ctx, stop := signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)
	defer stop()

	serveErr := make(chan error, 1)
	go func() {
		serveErr <- srv.ListenAndServe()
	}()

	select {
	case err := <-serveErr:
		return err
	case <-ctx.Done():
	}
	stop()

	shutdownCtx, cancel := context.WithTimeout(context.Background(), shutdownTimeout)
	defer cancel()
	if err := srv.Shutdown(shutdownCtx); err != nil {
		return fmt.Errorf("shutdown http server: %w", err)
	}
	if err := <-serveErr; !errors.Is(err, http.ErrServerClosed) {
		return err
	}
	return nil
}
```

**来源：** Go 标准库文档、Gin 文档

### 长连接单独处理 【推荐】

`Shutdown` 不关闭也不等待 WebSocket 这类被劫持的连接，用 `RegisterOnShutdown` 通知它们。

**来源：** Go 标准库文档

### gRPC 的优雅关闭要有兜底 【必须】

`GracefulStop` 会一直阻塞到 RPC 全部结束，用定时器在超时后调用 `Stop`。注册了 `health/v1` 时，关闭前调用健康服务的 `Shutdown`，把状态设为 `NOT_SERVING`。

**示例：**

```go
healthSrv.Shutdown()
stopped := make(chan struct{})
go func() {
	grpcServer.GracefulStop()
	close(stopped)
}()
select {
case <-stopped:
case <-time.After(shutdownTimeout):
	grpcServer.Stop()
	<-stopped
}
```

**来源：** gRPC 文档、grpc-go 文档

### 按依赖的反序关闭 【推荐】

先停止接收请求，再等后台 worker 退出，最后关闭数据库和消息连接。总耗时要小于平台的强制终止宽限期。

## 配置

### 启动时加载一次并校验 【推荐】

`main` 中把环境变量解析进配置结构体，校验失败立即退出，再显式传给各组件。业务代码不调用 `os.Getenv`，`init` 中不读配置。时长用 `time.ParseDuration` 解析。启动时打印配置要脱敏。

**来源：** Uber Go 风格指南、Google Go 风格指南

### 随环境变化的配置与代码分离 【推荐】

地址、凭据等随部署变化的值来自环境变量或配置注入，不写成代码常量。

**来源：** Twelve-Factor App

