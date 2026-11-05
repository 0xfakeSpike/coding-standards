# Go 安全

通用安全要求见 [后端安全规范](../../backend/index.md)，本文只写 Go 特有的做法。

## 语言特有风险

### 生成 HTML 用 html/template 【必须】

`html/template` 按上下文自动转义，`text/template` 不转义。不把用户输入转成 `template.HTML` 绕过转义。

**来源：** Go 标准库文档

### SQL 用占位符 【必须】

用 `database/sql` 的占位符传参，不用 `fmt.Sprintf` 拼接。占位符语法取决于驱动，比如 `?` 或 `$1`。动态表名和排序字段用白名单。

**示例：**

反例：

```go
query := fmt.Sprintf("SELECT id FROM users WHERE email = '%s'", email)
rows, err := db.QueryContext(ctx, query)
```

正例：

```go
rows, err := db.QueryContext(ctx, "SELECT id FROM users WHERE email = ?", email)
```

**工具：** `gosec`

### 用户路径不直接拼接 【必须】

Go 1.24+ 用 `os.Root` 把文件操作限制在目录内，它能挡住指向外部的符号链接。`filepath.IsLocal` 只做词法判断。解压归档时逐项检查路径。

**示例：**

```go
root, err := os.OpenRoot(uploadDir)
if err != nil {
	return err
}
defer root.Close()
f, err := root.Open(name)
```

**工具：** `gosec` 的 G304

**来源：** Go 发行说明、Go 标准库文档

### 不关闭 TLS 证书校验 【必须】

生产代码不设 `InsecureSkipVerify: true`。测试用 `httptest.NewTLSServer` 的 `Client()`。

**工具：** `gosec` 的 G402

**来源：** gosec 文档

### 比较密钥用常量时间函数 【必须】

token、签名用 `crypto/subtle.ConstantTimeCompare`，不用 `==` 或 `bytes.Equal`。

**来源：** Go 标准库文档

### 敏感数据不进日志和错误 【必须】

密码、token 不写入错误文本和日志。含敏感字段的类型实现 `slog.LogValuer` 或 `String` 脱敏，`%+v` 打印结构体时也要留意。

**来源：** Google Go 风格指南

### 不暴露 pprof 【必须】

`net/http/pprof` 导入后注册到 `http.DefaultServeMux`。业务端口不用默认 mux，调试接口放在只监听内网的独立端口。

**来源：** Go 标准库文档

### 非必要不用 unsafe 和 cgo 【推荐】

它们绕过类型和内存安全。没有性能数据或必须对接 C 库的理由时不用。

## 随机数

### 安全用途用 crypto/rand 【必须】

密钥、token、验证码、会话 ID 用 `crypto/rand`，即使是一次性的也不用 `math/rand` 或 `math/rand/v2`。需要文本时用 `rand.Text()`（Go 1.24+）。

**工具：** `gosec`

**来源：** Go Code Review Comments、Google Go 风格指南、Go 标准库文档

### 模拟和抖动用 math/rand/v2 【推荐】

非安全场景用 `math/rand/v2`（Go 1.22+），全局生成器已随机播种，不用手动 `Seed`。

**来源：** Go 发行说明

### UUID 用标准库 【推荐】

Go 1.27+ 用标准库 `uuid` 包，随机部分来自安全随机数生成器，低版本用成熟的第三方库。

**来源：** Go 发行说明

## 序列化

### 解码前限制输入大小 【必须】

不可信输入先用 `http.MaxBytesReader` 或 `io.LimitReader` 限制，再解码，防止内存耗尽。

**来源：** Go 标准库文档

### JSON 解码要明确字段 【推荐】

严格接口用 `Decoder.DisallowUnknownFields`。`encoding/json` 的 key 匹配不区分大小写，不要靠它区分字段。Go 1.27+ 的 `encoding/json/v2` 默认拒绝无效 UTF-8 和重复键，解析不可信 JSON 时可以优先考虑。

**来源：** Go 标准库文档、Go 发行说明

### 密钥字段不参与序列化 【必须】

敏感字段写 `json:"-"`。响应结构和数据库实体的关系见 [layering.md](../../backend/architecture/layering.md)。

### 不用 gob 解析不可信数据 【必须】

`encoding/gob` 不针对恶意输入加固，也不在 Go 安全策略范围内。

**来源：** Go 标准库文档

### 解析不可信输入的函数写模糊测试 【推荐】

`FuzzXxx` 随 `go test` 运行种子语料，`go test -fuzz=FuzzParse` 开始变异。目标要快且确定，崩溃样本留在 `testdata/fuzz/`。

**来源：** Go 模糊测试、Go 安全最佳实践

## 命令执行

### exec.Command 传参数列表 【必须】

不把用户输入拼进 `sh -c`。`os/exec` 不经过 shell，参数原样传递。

**示例：**

反例：

```go
cmd := exec.CommandContext(ctx, "sh", "-c", "convert "+name+" out.png")
```

正例：

```go
cmd := exec.CommandContext(ctx, "convert", "--", name, "out.png")
```

**工具：** `gosec` 的 G204

**来源：** Go 标准库文档

### 用户输入不作程序名 【必须】

程序名用常量或白名单。参数以 `-` 开头会被目标程序当成选项，目标程序支持时用 `--` 分隔。经 PATH 解析到当前目录的程序会返回 `exec.ErrDot`，要运行时写 `./prog`，不关闭这项保护。

**来源：** Go 标准库文档

### 子进程带 context 【必须】

**适用：** 线上服务

用 `exec.CommandContext` 设置超时，防止子进程无限挂起。

**来源：** Go 标准库文档

## 静态扫描

### CI 运行 govulncheck 【推荐】

**适用：** 多人协作

`govulncheck ./...` 结合调用图，只报告实际调用到的漏洞，噪音小。也扫描发布的二进制。

**工具：** `go get -tool golang.org/x/vuln/cmd/govulncheck` 登记后用 `go tool govulncheck ./...` 运行。

**来源：** govulncheck 教程、Go 安全最佳实践

### CI 运行 gosec 【推荐】

**适用：** 多人协作

通过 [style.md](style.md) 的 `golangci-lint` 配置启用。误报用 `//nolint:gosec` 或 `#nosec` 标记，原因写在指令后。

**来源：** Go 安全最佳实践、gosec 文档
