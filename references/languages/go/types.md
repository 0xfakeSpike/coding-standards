# Go 类型

## 类型系统用法

### 接口由使用方定义 【推荐】

接口放在调用它的包，只列实际用到的方法，保持小巧，只在包内使用时不导出。函数接受接口，返回具体类型，这样加方法时不影响使用方。

**示例：**

反例：

```go
package order

type Store interface {
	ByID(ctx context.Context, id int64) (Order, error)
	Save(ctx context.Context, o Order) error
	Delete(ctx context.Context, id int64) error
}

func NewStore(db DBTX) Store {
	return &sqlStore{db: db}
}
```

正例：

```go
package order

type Store struct {
	db DBTX
}

func NewStore(db DBTX) *Store {
	return &Store{db: db}
}

package invoice

type orderFinder interface {
	ByID(ctx context.Context, id int64) (order.Order, error)
}
```

**例外：** 接口本身就是产品时由提供方定义，比如 `io.Writer`。运行时才决定具体类型的工厂函数，以及 `error`，也返回接口。

**来源：** Go Code Review Comments、Google Go 风格指南

### 不提前定义接口 【推荐】

有两个以上实现，或使用方确实需要替换时才抽接口。“服务”“存储”这些概念不等于要有同名接口。不为 mock 在实现方定义接口，也不给生成的 RPC 客户端再包一层手写接口。

**原因：** 没有真实用法时，看不出接口是否必要，也定不准方法集。

**来源：** Go Code Review Comments、Google Go 风格指南

### 需要约定实现关系时做编译期检查 【推荐】

导出类型承诺实现某接口时写 `var _ http.Handler = (*Handler)(nil)`。

**来源：** Effective Go、Uber Go 风格指南

### 不写只包一层的壳类型和转调函数 【必须】

只持有一个字段并转调它的结构体、函数体只转调一次的函数，都直接删掉，调用方使用原对象。

**示例：**

反例：

```go
type UserService struct {
	store *Store
}

func (s *UserService) ByID(ctx context.Context, id int64) (User, error) {
	return s.store.ByID(ctx, id)
}
```

正例：

```go
func NewHandler(users *user.Store) *Handler {
	return &Handler{users: users}
}
```

**来源：** 团队

### 接收者类型保持一致 【推荐】

需要修改接收者、含 `sync.Mutex` 等不可复制字段、结构体较大时用指针接收者。`map`、函数、`chan` 和小的不可变值类型用值接收者。拿不准用指针。同一类型的方法不混用。不为省几个字节传指针，`*string`、`*io.Reader` 这类参数改成值。不复制方法挂在 `*T` 上的 `T`，比如 `bytes.Buffer`。

**来源：** Go Code Review Comments、Google Go 风格指南

### 谨慎嵌入 【推荐】

公开结构体不嵌入实现细节类型，互斥锁不嵌入任何结构体。嵌入的全部导出方法都该出现在外层类型上时才嵌入，否则用具名字段。

**原因：** 嵌入会让内部类型的方法和字段成为外层 API。

**来源：** Uber Go 风格指南

## 空值

### 让零值可用 【推荐】

设计类型时让 `var x T` 可以直接使用，`sync.Mutex` 和 `bytes.Buffer` 是范例。互斥锁用值字段，不用 `*sync.Mutex`。零值无法成立时提供构造函数。

**来源：** Effective Go、Uber Go 风格指南、Google Go 风格指南

### 用 nil 切片表示空，用 len 判断 【推荐】

写 nil map 会 panic，读是安全的。局部空切片写 `var t []string`，判空写 `len(s) == 0`。API 不让调用方区分 nil 和空切片。

**例外：** JSON 需要输出 `[]` 时初始化为 `[]string{}`，nil 切片会序列化为 `null`。

**来源：** Go Code Review Comments、Google Go 风格指南、Uber Go 风格指南

### 不把 nil 指针当 nil 接口返回 【必须】

具体类型的 nil 指针存入接口后不等于 nil，函数返回 `error` 时尤其危险。没有错误就直接写 `return nil`，声明返回类型用 `error`。

**来源：** Go FAQ、Google Go 风格指南

### 缺失值用额外返回值表达 【推荐】

用 `(value, ok)` 或 `error` 表示没有结果，不用 `-1`、`""`、`nil` 这类带内信号。

**来源：** Go Code Review Comments、Google Go 风格指南

## 泛型

### 有多种实际类型时才用泛型 【推荐】

只有一种类型在用时先写具体代码。多个类型共享行为时先考虑接口。新代码用 `any`，不写 `interface{}`。泛型用于替代 `any` 加类型断言，不用它造错误处理框架或断言库。

**来源：** Google Go 风格指南、Go 发行说明

## 枚举

### 用具名类型加 iota，从 1 开始 【推荐】

零值留给“未设置”，防止漏赋值时被当成合法状态。

**示例：**

```go
type Status int

const (
	StatusPending Status = iota + 1
	StatusPaid
	StatusRefunded
)
```

**例外：** 零值本身就是合理默认值时可以从 0 开始。

**工具：** `exhaustive` 检查 `switch` 分支是否完整，`stringer` 生成 `String` 方法。

**来源：** Uber Go 风格指南

## 数据类

### 结构体字面量写字段名 【必须】

跨包类型必须写字段名，零值字段省略。零值结构体写 `var user User`。

**工具：** `go vet` 的 `composites` 检查。

**来源：** Uber Go 风格指南、Google Go 风格指南

### 序列化字段写 tag 【推荐】

所有参与 JSON、YAML 编码的字段写 tag，改 Go 字段名时不影响外部格式。零值需要省略时，Go 1.24+ 可用 `omitzero`，它能正确处理 `time.Time`。

**来源：** Uber Go 风格指南、Go 发行说明

### 可选参数用配置结构体或函数式选项 【推荐】

默认用配置结构体，多数调用方不需要任何选项时才考虑函数式选项，后者代码量更大。`context.Context` 不放进配置结构体。

**来源：** Google Go 风格指南、Uber Go 风格指南

## 类型边界校验

### 入口处解析成领域类型 【必须】

外部输入在边界解析和校验一次，转成领域类型，内部不再重复判断。构造函数失败时返回 `error`。

### 类型断言使用 comma ok 形式 【必须】

单返回值的断言在类型不符时会 panic，写 `v, ok := x.(T)`。

**来源：** Uber Go 风格指南
