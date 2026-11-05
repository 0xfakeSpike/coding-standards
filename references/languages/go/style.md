# Go 风格

能交给工具的格式问题不靠人工评审。

## 工具配置

### 用 gofmt 和 goimports 格式化 【必须】

**适用：** 多人协作

提交前格式化，缩进、对齐和导入分组不讨论。

**工具：** `gofmt -l .`、`goimports -l .`

**来源：** Go Code Review Comments、Effective Go

### 静态检查进入门禁 【必须】

**适用：** 多人协作

CI 运行 `go vet ./...` 和 `golangci-lint run`，后者至少包含 `staticcheck`。Go 1.26+ 可用 `go fix` 批量更新到新写法，改动单独提交。

**来源：** Go 安全最佳实践、Go 发行说明

### 关闭文档注释类规则 【必须】

Go 惯例要求导出标识符有文档注释，团队规则是不写注释，以团队规则为准。对外发布的库保留文档注释和这些检查。`revive` 的 `exported`、`package-comments` 要关闭，`staticcheck` 的 ST1000、ST1020、ST1021、ST1022 不启用。还在用 `golint` 的项目直接移除，由 `revive` 代替。显式配置任一 `revive` 规则后默认规则会失效，所以要同时打开 `enable-default-rules`，配置项以所用 golangci-lint 版本文档为准。

**示例：**

```yaml
version: "2"
linters:
  enable:
    - bodyclose
    - errorlint
    - gosec
    - noctx
    - revive
    - sqlclosecheck
  settings:
    revive:
      enable-default-rules: true
      rules:
        - name: exported
          disabled: true
        - name: package-comments
          disabled: true
  exclusions:
    presets:
      - comments
      - std-error-handling
formatters:
  enable:
    - gofmt
    - goimports
```

**工具：** `//nolint` 的原因写在指令后，比如 `//nolint:gosec // 路径来自内部配置`。

**来源：** 团队、golangci-lint 文档、revive 文档、staticcheck 文档

## 格式

### 代码中不写注释 【必须】

包括导出标识符和包声明，对外发布的库的 API 文档和许可证头除外。工具指令除外，比如 `//go:build`、`//go:generate`、`//go:embed`、`//nolint`、`// Deprecated:`。忽略错误、类型断言、抑制 lint 这类单点取舍，原因就近写一句注释。需要记录的注意事项放在项目文档。

**来源：** 团队

### 按语义换行 【推荐】

Go 没有固定行宽。行太长时先改名字和结构，不在调用中间机械折行。

**来源：** Go Code Review Comments

### 正常路径不缩进 【推荐】

先处理异常分支并返回，分支以 `return`、`continue` 结束时不写 `else`。变量要在 `if` 之后继续使用时，声明单独写一行。

**来源：** Go Code Review Comments、Google Go 风格指南、Uber Go 风格指南

## 命名

| 对象 | 规则 | 示例 |
|---|---|---|
| 包 | 简短小写单词，不用下划线和驼峰，不用复数 | `billing`、`tabwriter` |
| 导出标识符 | `MixedCaps` | `OrderService` |
| 非导出标识符 | `mixedCaps` | `orderTotal` |
| 常量 | 同上，不用全大写，按作用命名 | `MaxRetryCount` |
| 缩写词 | 大小写保持一致 | `userID`、`HTTPServer`、`parseURL` |
| 错误变量 | `Err` 或 `err` 开头 | `ErrNotFound` |
| 错误类型 | `Error` 结尾 | `ValidationError` |
| 单方法接口 | 方法名加 `er` | `Reader`、`Stringer` |
| 文件 | `snake_case.go` | `order_service.go` |

**来源：** Effective Go、Go Code Review Comments、Google Go 风格指南、Uber Go 风格指南

### 映射变量说明 key 是什么 【必须】

名字写成 `值ByKey`，或者用具名类型作 key，让读者不看赋值就知道按什么索引。

**示例：**

反例：

```go
var cache map[string]Order
prices := map[string]int64{}
```

正例：

```go
var orderByID map[OrderID]Order
priceBySKU := map[string]int64{}
```

**来源：** 团队

### 名字不重复包名和类型 【推荐】

调用方写的是 `order.Service`，不要写成 `order.OrderService`；`widget.New` 优于 `widget.NewWidget`。变量不带类型词，写 `users`，不写 `userSlice`。

**来源：** Go Code Review Comments、Google Go 风格指南

### Getter 不加 Get 前缀 【推荐】

字段 `owner` 的 getter 写 `Owner()`，setter 写 `SetOwner()`。有远程调用或耗时计算时用 `Fetch`、`Compute` 表明代价。

**来源：** Effective Go、Google Go 风格指南

### 接收者用类型的缩写 【推荐】

用一两个字母，同一类型的所有方法保持一致，不用 `self`、`this`、`me`。

**来源：** Go Code Review Comments、Google Go 风格指南

### 不用无信息的包名 【推荐】

不用 `util`、`common`、`helpers`、`misc`、`types`、`interfaces`。`model` 也不用作包名，类型放在所属业务包内，见 [project.md](project.md)。

**来源：** Go Code Review Comments、Google Go 风格指南、Uber Go 风格指南

### 名字长度与作用域匹配 【推荐】

循环中的 `i`、读取器 `r`、写入器 `w` 用短名字，距离声明越远名字越完整。缩写不靠删字母，`Sandbox` 优于 `Sbx`。

**来源：** Go Code Review Comments、Google Go 风格指南

### 不遮蔽预声明标识符和常用包名 【推荐】

变量不叫 `error`、`string`、`len`，较大作用域内不叫 `url`、`time`。

**来源：** Uber Go 风格指南、Google Go 风格指南

## 导入排序

### 标准库在第一组 【推荐】

标准库一组，其余一组，组间空一行。需要第三组时由 `goimports -local` 统一。

**来源：** Go Code Review Comments、Uber Go 风格指南

### 只在冲突时重命名导入 【推荐】

包名与路径末段不一致时才补别名。冲突时重命名本地或项目内的包。Proto 生成包用 `pb` 后缀的完整词，比如 `orderpb`。

**来源：** Go Code Review Comments、Google Go 风格指南、Uber Go 风格指南

### 不用点导入，空白导入留在入口 【推荐】

不写 `import . "pkg"`。`import _ "pkg"` 只出现在 `main` 包或测试中。

**例外：** 因循环依赖只能放在 `foo_test` 包的测试，可以点导入被测包。

**来源：** Go Code Review Comments、Google Go 风格指南

## 文件组织

### 按阅读顺序排列 【推荐】

类型和常量在前，随后是构造函数和该类型的方法，普通辅助函数放在最后。方法按接收者聚集，大致按调用顺序排列。Go 没有一个类型一个文件的约定，紧密相关的类型放一起。

**来源：** Uber Go 风格指南、Google Go 风格指南

### 避免 init 函数 【推荐】

`init` 中不做 I/O，不读环境变量和工作目录，不启动 goroutine，也不依赖其他 `init` 的顺序。这些工作放进 `main` 或显式调用的函数。

**例外：** `database/sql` 驱动这类注册钩子。

**来源：** Uber Go 风格指南
