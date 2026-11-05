# Go 测试

竞态检测的要求见 [concurrency.md](concurrency.md)。

## 框架

### 优先使用标准库 testing 【推荐】

用 `if got != want` 加 `t.Errorf`，结构比较用 `github.com/google/go-cmp/cmp`。项目已统一使用 `testify` 时保持一致，不混用。不写自家断言框架。

**原因：** 断言库容易产出信息不足的失败提示，也会提前终止测试。

**来源：** Google Go 风格指南、Go Code Review Comments

### 失败信息足以定位问题 【推荐】

写出函数名、输入、实际值和期望值，实际值在前，格式为 `Foo(%q) = %v, want %v`。不用读测试源码就能判断哪里错了。

**来源：** Go Code Review Comments、Google Go 风格指南

### 比较语义，不比较不稳定的输出 【推荐】

结构体和切片用 `cmp.Diff` 整体比较，不逐字段手写。JSON 解析后再比，不比字节。错误用 `errors.Is` 判断，不比较错误文本。

**来源：** Google Go 风格指南

## 组织方式

### 测试文件和被测代码放在一起 【推荐】

`foo.go` 的测试放在 `foo_test.go`，测试数据放 `testdata/`。需要访问未导出标识符时用 `package foo`，只测公开 API 或要避开循环依赖时用 `package foo_test`。

**来源：** Google Go 风格指南

### 用 t.Error 继续，用 t.Fatal 止损 【推荐】

默认用 `t.Error` 让一次运行报出全部问题。后续检查没有意义时才用 `t.Fatal`。测试中不用 `panic`。

**来源：** Google Go 风格指南、Uber Go 风格指南

### 不在新 goroutine 中调用 t.Fatal 【必须】

`t.FailNow`、`t.Fatal` 只能在运行测试的 goroutine 中调用，其他 goroutine 用 `t.Error` 后返回。

**工具：** `go vet` 的 `testinggoroutine`

**来源：** Google Go 风格指南

### 辅助函数调用 t.Helper 【推荐】

测试辅助函数第一行写 `t.Helper()`，让失败定位到调用处。`t` 放在 `ctx` 之后、其他参数之前。环境准备失败直接 `t.Fatal`。

**来源：** Google Go 风格指南

### 清理交给 testing 【推荐】

用 `t.Cleanup`、`t.TempDir`、`t.Setenv`，用 `t.Context()`（Go 1.24+）作为测试的 context。这个 context 在 `Cleanup` 执行前就已取消，清理时改用 `context.Background()`。

**来源：** Google Go 风格指南、Go 标准库文档

### TestMain 只放全部用例都需要的准备 【推荐】

测试独立性的要求见 [testing-strategy.md](../../general/testing-strategy.md)。子测试要能单独运行，不在 `TestMain` 或包级变量里放只有部分用例需要的准备。

**来源：** Google Go 风格指南

## 参数化

### 输入输出变化时用表驱动测试 【推荐】

用 `t.Run` 给每行命名，名字要可读。

**示例：**

```go
func TestShippingFee(t *testing.T) {
	tests := []struct {
		name        string
		amountCents int64
		wantCents   int64
	}{
		{"未满免运费门槛", 9999, 800},
		{"满足免运费门槛", 10000, 0},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := ShippingFee(tt.amountCents); got != tt.wantCents {
				t.Errorf("ShippingFee(%d) = %d, want %d", tt.amountCents, got, tt.wantCents)
			}
		})
	}
}
```

**来源：** Google Go 风格指南、Uber Go 风格指南、Go Code Review Comments

### 表驱动内不写分支逻辑 【推荐】

循环体出现条件断言，或表字段只有部分用例使用时，拆成多张表或多个测试函数。

**来源：** Uber Go 风格指南

## 替身

### 优先用真实实现 【推荐】

能用真实类型就不用替身。HTTP 用 `httptest`，RPC 用真实传输连到本地测试服务。

**来源：** Google Go 风格指南、Gin 文档

### 替身由使用方的小接口承接 【推荐】

被测包定义只含所需方法的接口，手写的 fake 放在 `_test.go`。多个包共用时放专门的包，命名为 `xxxtest`，比如 `creditcardtest`。不在生产类型上留测试后门。

**来源：** Go Code Review Comments、Google Go 风格指南

### 时间和随机数通过注入控制 【推荐】

用字段 `now func() time.Time` 注入，不改包级变量，不靠真实的 `time.Sleep` 等待。并发和定时逻辑用 `testing/synctest`（Go 1.25+），在隔离环境中使用虚拟时钟。

**示例：**

```go
func TestExpiry(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		c := NewCache(time.Minute)
		c.Put("k", 1)
		time.Sleep(2 * time.Minute)
		synctest.Wait()
		if got, ok := c.Get("k"); ok {
			t.Errorf(`Get("k") = %v, true, want expired`, got)
		}
	})
}
```

**来源：** Uber Go 风格指南、Go 发行说明

## 集成测试

### 集成测试与单元测试分开运行 【推荐】

用 `//go:build integration` 标记，或在开头 `if testing.Short() { t.Skip() }`。默认的 `go test ./...` 不依赖外部服务。

### 使用真实依赖 【推荐】

数据库、消息队列用容器里的真实实例，`Store` 中的 SQL 要在真实数据库上跑过。每个用例用事务或独立数据隔离，不依赖执行顺序。

### 检查 goroutine 泄漏 【推荐】

启动后台 goroutine 的包在测试中用 `goleak` 检查。

**来源：** Uber Go 风格指南

## 基准测试

### 基准测试用 b.Loop 【推荐】

Go 1.24+ 写 `for b.Loop()`，代替 `for range b.N`。循环前的准备不计入耗时，也不用手动 `ResetTimer`。用 `b.ReportAllocs()` 看分配。

**示例：**

```go
func BenchmarkEncode(b *testing.B) {
	order := sampleOrder()
	b.ReportAllocs()
	for b.Loop() {
		if _, err := Encode(order); err != nil {
			b.Fatal(err)
		}
	}
}
```

**来源：** Go 发行说明、Go 标准库文档

### 用数据证明优化 【推荐】

性能改动附上前后基准结果，用 `benchstat` 比较多次运行。只优化热路径。

**来源：** Uber Go 风格指南、Google Go 风格指南
