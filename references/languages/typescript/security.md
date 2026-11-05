# TypeScript / JavaScript 安全

通用后端安全见 [后端安全规范](../../backend/index.md)。本文只写语言和运行时特有的风险。

## 语言特有风险

### 不执行动态字符串代码 【必须】

不用 `eval`、`new Function`、字符串形式的 `setTimeout`。`vm` 模块不是安全边界，不能运行不可信代码。

**工具：** ESLint `no-eval`、`no-new-func`，`@typescript-eslint/no-implied-eval`。

**来源：** Google TypeScript 风格指南、Node.js API 文档、MDN

### 防止原型污染 【必须】

攻击者通过 `__proto__`、`constructor`、`prototype` 改写原型，影响进程内所有对象，可能绕过权限判断或造成拒绝服务。不可信数据不进入递归合并和 `obj[key] = value`。

**示例：**

反例：

```ts
function setByPath(target: Record<string, any>, path: string[], value: unknown): void {
  const lastKey = path.at(-1)!;
  const parent = path.slice(0, -1).reduce((node, key) => (node[key] ??= {}), target);
  parent[lastKey] = value;
}
```

正例：

```ts
const preferencesSchema = z.strictObject({ theme: z.enum(['light', 'dark']) });
const preferences = preferencesSchema.parse(JSON.parse(body));
```

**来源：** Node.js 安全最佳实践、MDN

### 动态键使用安全容器 【必须】

键来自外部时用 `Map`、`Set` 或 `Object.create(null)`。必须写入普通对象时，先校验键在白名单内，或排除 `__proto__`、`constructor`、`prototype`。高风险服务可以加 `--disable-proto=delete`，它不覆盖 `constructor.prototype` 路径，不能替代校验。

**来源：** MDN、Node.js 安全最佳实践

### 权限字段只读自有属性 【推荐】

被污染的原型会让 `user.isAdmin` 这样的读取返回攻击者的值。判断前用 `Object.hasOwn`。遍历用 `Object.keys`，不用 `for...in`。

**来源：** MDN

### 正则不接受不可信模式 【必须】

避免嵌套量词 `(a+)*`、重叠分支 `(a|a)*` 和反向引用。不用用户输入构造正则，简单匹配用 `includes` 或 `indexOf`。ReDoS 会阻塞事件循环，影响所有请求。`JSON.parse` 同样同步执行，先在框架层设置 body 大小上限。

**来源：** Node.js Learn 指南

### 文件路径限制在根目录内 【必须】

用户输入不直接拼路径，要求见 [file-handling.md](../../backend/security/file-handling.md)。`realpath` 解析符号链接后确认结果位于允许的根目录下，再访问。`uploadRoot` 是已解析的绝对路径。

**示例：**

```ts
const filePath = await realpath(path.resolve(uploadRoot, fileName));
if (!filePath.startsWith(uploadRoot + path.sep)) {
  throw new InvalidPathError(fileName);
}
const content = await readFile(filePath);
```

## 随机数

### 安全场景用密码学随机数 【必须】

令牌、会话 ID、验证码和盐值用 `crypto.randomUUID()`、`crypto.randomBytes()`、`crypto.randomInt()` 或 `crypto.getRandomValues()`。`Math.random()` 可预测，只用于无安全要求的场景。

**示例：**

反例：

```ts
const resetToken = Math.random().toString(36).slice(2);
```

正例：

```ts
const resetToken = randomUUID();
```

**来源：** MDN、Node.js API 文档

### 密钥比较用恒定时间函数 【必须】

`===` 比较时间随内容变化，可被用来猜测。用 `crypto.timingSafeEqual`，两个 Buffer 长度必须相同。

**来源：** Node.js 安全最佳实践、Node.js API 文档

### 密码哈希用异步 API 【必须】

算法顺序和参数见 [authentication.md](../../backend/security/authentication.md)。Node.js 中不用同步版本，`scryptSync`、`pbkdf2Sync` 在请求路径上会阻塞事件循环。

**来源：** Node.js 安全最佳实践、Node.js Learn 指南

## 序列化

### 只用 JSON 解析不可信数据 【必须】

`JSON.parse` 只产生数据。`v8.deserialize`、YAML 的自定义类型和能还原函数的库不用于不可信输入。

### 解析结果必须校验 【必须】

`JSON.parse` 返回 `any`，它创建的 `__proto__` 是自有属性，随后 `Object.assign` 或展开合并时可能进入原型。解析后立即用 schema 校验，拒绝多余字段，比如 Zod 的 `z.strictObject`。

**来源：** Node.js 安全最佳实践、MDN

### 日志和响应不序列化完整对象 【必须】

对象可能含密码和令牌。响应用显式字段或序列化 schema 输出，日志用脱敏配置。Fastify 的校验和序列化用 `new Function()` 编译 schema，schema 不能来自用户输入。

**来源：** Fastify 文档

## 命令执行

### 用 `execFile` 或 `spawn` 传参数数组 【必须】

`exec` 和 `shell: true` 把字符串交给 shell，用户输入可以注入命令。以 `-` 开头的值可能被当成选项，程序支持时用 `--` 分隔。设置超时，不在请求路径上用 `execSync`、`spawnSync`。

**示例：**

反例：

```ts
await execAsync(`gzip ${fileName}`);
```

正例：

```ts
await execFileAsync('gzip', ['--', fileName], { signal, timeout: gzipTimeoutMs });
```

**来源：** Node.js API 文档、Node.js Learn 指南

## 静态扫描

### 生产环境不开启调试端口 【必须】

生产环境不开启 `--inspect`。

**来源：** Node.js 安全最佳实践

### 依赖漏洞纳入 CI 【必须】

**适用：** 多人协作

CI 运行 `npm audit --audit-level=high` 或 `pnpm audit`，用 `npm audit signatures` 校验签名。

**来源：** npm 文档、Node.js 安全最佳实践

### 安装依赖时防范供应链攻击 【必须】

**适用：** 线上服务

核对包名避免 typosquatting，用 lock 文件固定全部传递依赖，安装脚本的限制见 [project.md](project.md)。pnpm 可以用 `minimumReleaseAge` 延迟采用新版本。

**来源：** Node.js 安全最佳实践、pnpm 文档

