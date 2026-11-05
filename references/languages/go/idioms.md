# Go 惯用写法

## 惯用写法

### 优先用标准库的现成写法 【推荐】

核心语法够用就不引入库。Go 1.21+ 用内置 `min`、`max`、`clear`，以及 `slices`、`maps`、`cmp`。Go 1.22+ 可写 `for i := range n`。集合判断用 `map[string]bool` 通常就够了。

**工具：** `go fix`（Go 1.26+）

**来源：** Google Go 风格指南、Go 发行说明

### 用 defer 释放资源 【必须】

获取资源成功后立即写 `defer`，多个返回路径都能释放。`defer` 的开销不是不用它的理由。

**来源：** Uber Go 风格指南、Effective Go

### 不在循环体内 defer 每次迭代的资源 【推荐】

`defer` 要到函数结束才执行，资源会一直累积，把循环体提取成函数。

**示例：**

反例：

```go
for _, name := range names {
	f, err := os.Open(name)
	if err != nil {
		return err
	}
	defer f.Close()
	if err := scan(f); err != nil {
		return err
	}
}
```

正例：

```go
func scanFile(name string) error {
	f, err := os.Open(name)
	if err != nil {
		return err
	}
	defer f.Close()
	return scan(f)
}
```

### 缩小变量作用域 【推荐】

只在 `if` 内使用的结果写成 `if err := f(); err != nil`。局部赋值用 `:=`，要表达零值时用 `var`。具名返回值只在同类型结果需要区分或 `defer` 要修改时使用，只有很短的函数才用裸 `return`。

**来源：** Uber Go 风格指南、Google Go 风格指南、Go Code Review Comments

### 不用可变的包级变量 【推荐】

依赖通过构造函数或参数传入，时钟也一样，用 `now func() time.Time` 字段代替全局替换。库不让使用方依赖全局状态。

**原因：** 全局状态让测试互相影响，也无法在同一进程运行多个实例。

**来源：** Uber Go 风格指南、Google Go 风格指南

## 集合

### 切片共享底层数组 【必须】

子切片和原切片互相可见，容量足够时 `append` 会覆盖后面的元素。需要隔离时用 `slices.Clone` 或 `copy`，要继续追加时用完整切片表达式 `s[:n:n]`。`slices.Delete` 等函数原地修改，要使用它的返回值。从大缓冲区截取少量数据长期保存时也要复制，否则整个数组无法回收。

**示例：**

反例：

```go
head := items[:2]
head = append(head, extra)
```

正例：

```go
head := slices.Clone(items[:2])
head = append(head, extra)
```

**来源：** Effective Go

### 在边界复制切片和 map 【推荐】

保存入参或返回内部字段时要复制，否则外部修改会绕过你的不变量和锁。

**示例：**

```go
func (s *Stats) Snapshot() map[string]int {
	s.mu.Lock()
	defer s.mu.Unlock()
	return maps.Clone(s.countByName)
}
```

**来源：** Uber Go 风格指南

### 已知大小时才预分配 【推荐】

大小确定时写 `make([]T, 0, n)`、`make(map[K]V, n)`。多数代码不需要，预分配过大反而浪费，在性能敏感处用基准测试判断。

**来源：** Uber Go 风格指南、Google Go 风格指南

## 字符串

### 区分字节和字符 【必须】

`len(s)` 是字节数，`s[i]` 是字节，`for range s` 才按 rune 遍历。截断含中文的字符串要按 rune 处理。

**来源：** Effective Go

### 格式字符串声明为常量 【推荐】

放在字面量之外的格式串写成 `const`，自定义的 Printf 风格函数以 `f` 结尾，`go vet` 才能检查。

**工具：** `go vet`

**来源：** Uber Go 风格指南

## 日期时间

### 时刻用 time.Time，时长用 time.Duration 【推荐】

不用裸整数加单位后缀。外部格式不支持时，字段名写明单位，比如 `IntervalMillis`；时间戳字符串用 RFC 3339。

**来源：** Uber Go 风格指南

### 比较用 Equal，不用 == 【必须】

`==` 同时比较时区和单调时钟读数，同一时刻也可能不相等。先后比较用 `Before`、`After`。

**来源：** Go 标准库文档

### 区分日历加法和时长加法 【必须】

下一个日历日用 `AddDate(0, 0, 1)`，整 24 小时后用 `Add(24 * time.Hour)`。夏令时切换日两者不同。

**来源：** Uber Go 风格指南

## 数值与金额

### 金额不用浮点数 【必须】

用整数表示最小货币单位，比如 `int64` 的分，或用十进制库。类型或字段名写明单位。

**原因：** 二进制浮点无法精确表示大多数十进制小数，累加后会出现误差。

### 注意整数溢出和截断 【必须】

Go 的整数溢出不报错，直接回绕；`int64` 转 `int32` 直接截断。来自外部的数值转换前检查范围。

**工具：** `gosec`

## 常见陷阱

### HTTP 客户端要设超时并关闭响应体 【必须】

`http.DefaultClient` 的 `Timeout` 为零，表示没有超时。`Client` 创建一次后复用。检查 `err` 后立刻 `defer resp.Body.Close()`。

**工具：** `bodyclose`、`noctx`

**来源：** Go 标准库文档

### 写文件时检查 Close 的错误 【必须】

缓冲写入的失败可能在 `Close` 时才出现，只写 `defer f.Close()` 会丢掉它。

