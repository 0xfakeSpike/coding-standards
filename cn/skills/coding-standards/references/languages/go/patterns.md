# Go 设计模式写法

本文写设计模式在 Go 里的惯用形态。什么时候用哪个模式见 [design-patterns.md](../../general/design-patterns.md)。Go 没有类和继承，照搬 Java 式结构在 Go 里最显别扭。接口的规则见 [types.md](types.md) 的“接口由使用方定义”和“不提前定义接口”，可选参数见“可选参数用配置结构体或函数式选项”。

## 结构

### 不写单例，依赖在 `main` 中创建并传入 【推荐】

不写 `GetInstance()` 加 `sync.Once` 的全局实例。连接池、客户端这类对象在 `main` 或组装依赖的函数里创建一次，通过构造函数传给使用方，见 [idioms.md](idioms.md) 的“不用可变的包级变量”。`sync.Once` 只用于包内部的延迟初始化，不对外暴露全局实例。

**来源：** Uber Go 风格指南、Google Go 风格指南

### 构造用 `New` 函数，不写工厂类型和建造者 【推荐】

构造函数是普通的 `NewX` 函数，返回具体类型。运行时才决定具体类型时，工厂函数返回接口，见 [types.md](types.md) 的“接口由使用方定义”的例外。不写 `XxxFactory` 结构体，也不写链式的建造者。

### 不用嵌入模拟继承 【推荐】

Java 式的模板方法让子类覆盖父类的步骤，但 Go 的嵌入不是继承：被嵌入类型的方法调用不到外层类型的同名方法，“覆盖”不会生效。流程相同、步骤不同时，把不同的步骤作为函数或小接口传入。嵌入的规则见 [types.md](types.md) 的“谨慎嵌入”。

**示例：**

```go
type exporter struct {
	format func(rows []Row) ([]byte, error)
}

func (e exporter) Export(ctx context.Context, q Query) error {
	rows, err := query(ctx, q)
	if err != nil {
		return fmt.Errorf("query rows: %w", err)
	}
	data, err := e.format(rows)
	if err != nil {
		return fmt.Errorf("format rows: %w", err)
	}
	return upload(ctx, data)
}
```

**来源：** Effective Go

### getter 不加 `Get` 前缀，不写只转发字段的方法 【推荐】

字段 `owner` 的读取方法叫 `Owner()`，写入方法叫 `SetOwner()`。没有不变量要保护时，直接导出字段，不写只转发字段的方法。

**来源：** Effective Go

## 行为

### 策略用函数类型 【推荐】

每种情况一个函数，用一个函数按类型返回对应的实现。`switch` 配合 `exhaustive` 检查，新增类型时漏处理的地方会被指出，见 [types.md](types.md) 的“用具名类型加 iota，从 1 开始”。策略需要多个方法时，由使用方定义一个小接口。

**示例：**

```go
type exportFunc func(rows []Row) ([]byte, error)

func exporterFor(f Format) (exportFunc, error) {
	switch f {
	case FormatCSV:
		return toCSV, nil
	case FormatXLSX:
		return toXLSX, nil
	default:
		return nil, fmt.Errorf("unsupported export format %v", f)
	}
}
```

### 状态机用具名类型加 `switch` 【推荐】

状态用具名整数类型加 `iota` 表达，迁移方法按状态 `switch`，每个分支做完这一步、确认没有出错后，再设置下一个状态。未知状态返回错误。

**示例：**

```go
type step int

const (
	stepAwaitingToken step = iota + 1
	stepUploading
	stepMerging
	stepDone
)

func (u *upload) advance(ctx context.Context) error {
	switch u.step {
	case stepAwaitingToken:
		token, err := fetchToken(ctx)
		if err != nil {
			return fmt.Errorf("fetch token: %w", err)
		}
		u.token, u.step = token, stepUploading
	case stepUploading:
		parts, err := uploadParts(ctx, u.token)
		if err != nil {
			return fmt.Errorf("upload parts: %w", err)
		}
		u.parts, u.step = parts, stepMerging
	case stepMerging:
		fileID, err := mergeParts(ctx, u.token, u.parts)
		if err != nil {
			return fmt.Errorf("merge parts: %w", err)
		}
		u.fileID, u.step = fileID, stepDone
	case stepDone:
	default:
		return fmt.Errorf("unknown upload step %d", u.step)
	}
	return nil
}
```

### 横切逻辑用包装函数 【推荐】

HTTP 的横切逻辑写成 `func(http.Handler) http.Handler` 中间件。其他接口用一个实现同一接口、内部持有被包装对象的结构体。

**示例：**

```go
func withTiming(name string, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		metrics.Observe(name, time.Since(start))
	})
}
```

### 遍历返回 `iter.Seq` 【推荐】

Go 1.23 起，自定义集合和分页数据的遍历返回 `iter.Seq` 或 `iter.Seq2`，调用方直接 `range`。不写带 `HasNext()`、`Next()` 的迭代器类型。需要处理错误时用 `iter.Seq2[T, error]`。

**来源：** Go 发行说明、Go 标准库文档
