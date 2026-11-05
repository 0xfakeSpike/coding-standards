# Go 项目

## 目录结构

### 从最简单的布局开始 【推荐】

小项目所有代码放在根目录，有多个可执行入口或支撑包时再分层。不为了目录规整强行使用 `pkg/`。

**示例：**

有两个可执行入口和两个业务包时：

```text
go.mod
cmd/
  api/
    main.go
  worker/
    main.go
internal/
  order/
    order.go
    store.go
  payment/
    payment.go
    store.go
migrations/
```

**来源：** Go 模块布局

### 可执行入口放 cmd，内部代码放 internal 【推荐】

`cmd/<name>/main.go` 只做解析配置、组装依赖和启动。支撑包默认放 `internal/`，只有确实要给其他模块用才放在外面。

**原因：** `internal` 由 Go 工具链强制，外部模块无法导入，内部结构可以自由调整。

**来源：** Go 模块布局

### 按业务职责分包 【推荐】

包名说明提供什么，比如 `order`、`payment`。避免 `controllers`、`services` 这样的按类型分包，否则一次改动横跨多个包，也容易产生循环依赖。紧密耦合的类型放同一个包。

**来源：** Google Go 风格指南

### 数据表有统一入口 【必须】

每张表归属一个业务包。表对应的类型放在业务包内，对这张表的全部读写放在该包的 `store.go`，业务代码不直接写 SQL。其他包需要这份数据时调用归属包的导出方法，不写这张表的 SQL，只读 JOIN 的例外见下文。全部表结构在 `migrations/` 中统一查看。代码都在一个包时，这个包就是归属包。

`DBTX` 和 `WithTx(*sql.Tx)` 是 `database/sql` 下的示例，不是结构要求。其他驱动保持同样的结构：pgx 的接口列 `Exec`、`QueryRow`，`WithTx` 接收 `pgx.Tx`；GORM 的 `Store` 持有 `*gorm.DB`，`WithTx` 传入事务内的 `*gorm.DB`；sqlc 用生成的 `Queries.WithTx`。具体签名以所用版本文档为准。

**原因：** 改一张表时只需要看归属包和只读 JOIN 包，也能从 `Store` 的方法列表看清这张表的全部用法。

**示例：**

```go
package order

const (
	orderByIDSQL = "SELECT id, user_id, total_cents, created_at FROM orders WHERE id = ?"
	markPaidSQL  = "UPDATE orders SET status = 'paid' WHERE id = ? AND status = 'pending'"
)

var (
	ErrNotFound   = errors.New("order not found")
	ErrNotPending = errors.New("order not pending")
)

type Order struct {
	ID         int64
	UserID     int64
	TotalCents int64
	CreatedAt  time.Time
}

type DBTX interface {
	ExecContext(ctx context.Context, query string, args ...any) (sql.Result, error)
	QueryRowContext(ctx context.Context, query string, args ...any) *sql.Row
}

type Store struct {
	db DBTX
}

func NewStore(db DBTX) *Store {
	return &Store{db: db}
}

func (s *Store) WithTx(tx *sql.Tx) *Store {
	return &Store{db: tx}
}

func (s *Store) ByID(ctx context.Context, id int64) (Order, error) {
	var o Order
	err := s.db.QueryRowContext(ctx, orderByIDSQL, id).
		Scan(&o.ID, &o.UserID, &o.TotalCents, &o.CreatedAt)
	if errors.Is(err, sql.ErrNoRows) {
		return Order{}, ErrNotFound
	}
	if err != nil {
		return Order{}, fmt.Errorf("order by id %d: %w", id, err)
	}
	return o, nil
}

func (s *Store) MarkPaid(ctx context.Context, id int64) error {
	res, err := s.db.ExecContext(ctx, markPaidSQL, id)
	if err != nil {
		return fmt.Errorf("mark order %d paid: %w", id, err)
	}
	n, err := res.RowsAffected()
	if err != nil {
		return fmt.Errorf("mark order %d paid: %w", id, err)
	}
	if n == 0 {
		return ErrNotPending
	}
	return nil
}
```

**来源：** 团队

## 跨业务包

本节规则以项目有两个以上业务包为前提。

### 跨业务流程放在上层业务包 【推荐】

业务包之间可以单向 import，方向由领域依赖决定。比如 `invoice` 需要订单数据时，直接调用 `order` 的导出方法。只有流程要协调互不依赖的包，或双方都需要对方时，才新建以流程命名的上层包，比如 `internal/checkout`。上层包依赖 `order` 和 `payment`，这两个包不互相 import，SQL 仍留在各自的 `store.go`。

**原因：** Go 不允许循环 import，流程也不应分散在多个包里。

**示例：**

```text
internal/
  checkout/
    checkout.go
  order/
  payment/
```

### 跨业务事务由编排包开启 【推荐】

要求原子写入的表属于多个业务包时，由上层包开启事务，通过 `WithTx` 传给各个 `Store`。`Store` 依赖 `DBTX` 接口，`*sql.DB` 和 `*sql.Tx` 都满足它。业务包不开启跨包事务，事务内不调用外部服务。

单包内的多表事务由包内业务函数开启，写法相同。`DBTX` 没有 `BeginTx`，开启事务的类型另外持有 `*sql.DB`。`*sql.DB` 和 `*sql.Tx` 只出现在这类类型和 `Store` 中。

先用条件更新占住状态，再写其他表。重复或并发的请求在这一步得到 `ErrNotPending`，不会写出第二条支付记录。

**示例：**

```go
package checkout

type Service struct {
	db       *sql.DB
	orders   *order.Store
	payments *payment.Store
}

func (s *Service) Pay(ctx context.Context, orderID int64) error {
	tx, err := s.db.BeginTx(ctx, nil)
	if err != nil {
		return fmt.Errorf("begin pay tx: %w", err)
	}
	defer tx.Rollback()

	orders := s.orders.WithTx(tx)
	o, err := orders.ByID(ctx, orderID)
	if err != nil {
		return err
	}
	if err := orders.MarkPaid(ctx, o.ID); err != nil {
		return err
	}
	if err := s.payments.WithTx(tx).Create(ctx, o.ID, o.TotalCents); err != nil {
		return err
	}
	if err := tx.Commit(); err != nil {
		return fmt.Errorf("commit pay tx: %w", err)
	}
	return nil
}
```

### 跨业务读取优先分别调用 【推荐】

数据量小时，分别调用两个 `Store`，在上层包组合结果。

### 必须 JOIN 的只读查询放专门的包 【推荐】

跨业务的列表或报表确实需要 JOIN 时，放到专门的只读包，比如 `internal/orderreport`。这是“不写其他包表的 SQL”的唯一例外，这类包只查询，不写入。

**原因：** 把 JOIN 放进任一业务包，都会让它依赖另一个业务的表结构。

## 包边界

### 总是一起变化的表考虑合并包 【推荐】

两个包的表总是一起修改、一起查询时，说明边界可能划错了，考虑合并成一个包。

### 避免循环依赖靠调整包边界 【推荐】

出现循环依赖时先检查包划分，把公共部分下沉或合并。用接口硬断循环通常说明包结构不对。

**来源：** Google Go 风格指南

## 包管理

### 提交 go.mod 和 go.sum 【必须】

两个文件都进版本库，`go.sum` 用于校验下载内容的完整性。

**来源：** Go 依赖管理

### CI 检查依赖整洁 【推荐】

运行 `go mod tidy` 后 `go.mod`、`go.sum` 不能有差异。未使用的依赖直接删除。

**工具：** `go mod tidy -diff`

**来源：** Go 依赖管理

### 开发工具用 tool 指令管理 【推荐】

Go 1.24+ 用 `go get -tool` 登记 `stringer` 这类工具，用 `go tool stringer` 运行，版本随 `go.mod` 锁定。

**来源：** Go 依赖管理、Go Modules 参考

### replace 不进入发布版本 【推荐】

给其他模块依赖的模块不靠 `replace` 生效，它只在主模块生效，依赖方会忽略它。本地联调多个模块时优先用 `go.work`。

**例外：** 修复尚未发布的上游缺陷时可以临时指向 fork，原因写在 `replace` 旁的注释里。

**来源：** Go Modules 参考

### 私有模块配置 GOPRIVATE 【推荐】

私有模块设置 `GOPRIVATE`，直接从源仓库下载，模块路径不发送给公共代理。

**来源：** Go Modules 参考

## 版本与锁定

### go 指令写真实需要的最低版本 【推荐】

`go` 指令是强制的最低版本，也决定语言特性和循环变量等语义。升级它是有行为影响的改动，单独提交。`toolchain` 只在主模块生效，用于建议具体工具链版本。

**来源：** Go Modules 参考

### 版本由 go.mod 锁定，升级要有意识 【推荐】

Go 按最小版本选择，不会自动取最新版本。升级用 `go get example.com/pkg@v1.2.3`，不把无关的 `go get -u ./...` 混进功能提交。升级后运行测试和 `govulncheck`。

**来源：** Go Modules 参考、Go 安全最佳实践

### 及时更新 Go 补丁版本 【推荐】

**适用：** 线上服务

安全修复随补丁版本发布，CI 和镜像使用的 Go 版本要跟进，并与本地保持一致。

**来源：** Go 安全最佳实践

## 构建

### 构建默认不改写 go.mod 【推荐】

`go.mod` 需要变动时构建直接报错，这是默认行为。CI 不使用 `-mod=mod`。

**来源：** Go Modules 参考

### 发布构建用可重现参数 【推荐】

发布用 `go build -trimpath`，去掉本机路径。版本信息用 `-ldflags "-X main.version=$VERSION"` 注入，或用 `go version -m` 读取嵌入的 VCS 信息。不依赖 C 库的服务设置 `CGO_ENABLED=0`。

**示例：**

```sh
CGO_ENABLED=0 go build -trimpath -o bin/api ./cmd/api
```

### 平台差异用构建约束 【推荐】

用 `//go:build linux` 或 `_linux.go` 后缀，不在运行时判断。静态资源用 `//go:embed` 打进二进制。

### 生成代码有固定入口 【推荐】

用 `//go:generate` 声明生成命令，生成物是否提交由项目统一决定。提交时 CI 重新生成并检查无差异。

## 多模块或 monorepo

### 优先单模块 【推荐】

一个仓库一个 `go.mod`，用 `internal/` 和目录划分边界。只有发布周期确实独立时才拆模块。

**来源：** Go 模块布局

### 本地多模块联调用 go.work 【推荐】

`go.work` 通常不提交，否则可能覆盖开发者自己的工作区，也会让 CI 测到错误的依赖版本。CI 设置 `GOWORK=off`，或者只在模块互相配套开发时才提交。

**来源：** Go Modules 参考
