# 接口契约

本文讲接口契约的编写、管理和验证。REST 设计见 [rest.md](rest.md)，测试分层见 [testing-strategy.md](../../general/testing-strategy.md)。

**适用：** 多人协作（本文全部规则）

## OpenAPI 优先

### 接口有机器可读的契约 【推荐】

HTTP 接口用 OpenAPI 描述，并存放在代码仓库。契约由代码生成时，在 CI 中校验，避免文档过期。

**来源：** Zalando RESTful API Guidelines

### 先写契约再实现 【推荐】

先定义契约并让调用方评审，再写实现，前后端和服务之间可以并行开发。

**例外：** 只有单一调用方的内部接口，可以由代码生成契约。

**来源：** Zalando RESTful API Guidelines

### 契约只保留一个事实来源 【必须】

选定手写契约或代码生成其中一种。不同时手工维护两份描述。

### 每个操作有唯一的 operationId 【必须】

`operationId` 在整份契约内唯一，区分大小写，并遵循常见的编程命名习惯。代码生成工具通常用它命名方法。

**来源：** OpenAPI 规范

### 声明成功和错误响应 【推荐】

每个操作列出所有成功响应，以及调用方需要区别处理的错误响应。错误响应引用统一的错误 Schema。

**示例：**

```yaml
paths:
  /orders/{id}:
    get:
      operationId: getOrder
      parameters:
        - $ref: "#/components/parameters/OrderId"
      responses:
        "200":
          description: 订单详情
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Order"
        "404":
          $ref: "#/components/responses/Problem"
```

**来源：** OpenAPI 规范、Zalando RESTful API Guidelines

## Schema 管理

### 公共结构放入 components 【推荐】

多处使用的 Schema、参数和响应定义在 `components`，用 `$ref` 引用。只用一次的结构不必提取。

**来源：** OpenAPI 规范

### Schema 写全约束 【推荐】

写明 `required`、`format`、`enum`、取值范围和长度上限。不用无约束的 `object` 表示已知结构。

**示例：**

反例：

```yaml
Order:
  type: object
```

正例：

```yaml
Order:
  type: object
  required: [id, status, createdAt]
  properties:
    id:
      type: string
    status:
      type: string
      enum: [pending, paid, cancelled]
    createdAt:
      type: string
      format: date-time
```

**原因：** 约束不全时，生成的代码和校验都没有保护作用。

### 可空字段用 JSON Schema 语法 【推荐】

可空写成 `type: [string, "null"]`，不用 `nullable`，需要 OpenAPI 3.1 及以上。

**来源：** OpenAPI 规范、Zalando RESTful API Guidelines

### 契约在 CI 中做静态检查 【推荐】

校验语法，并用规则集检查命名、错误响应和分页约定。

**工具：** Spectral、Redocly CLI

## 生成代码

### 不手工修改生成代码 【必须】

需要调整时改契约或生成配置，然后重新生成。

**原因：** 手工修改会在下次生成时丢失。

### 生成过程可重复 【推荐】

锁定生成器版本，用一条命令完成生成。提交生成物时，CI 重新生成并检查没有差异。

**工具：** `git diff --exit-code`

### 生成类型停留在接口边界 【推荐】

生成的类型只用于接口层，进入业务层前转换为领域类型。

## 契约测试

### 按场景选择验证方式 【推荐】

| 场景 | 方式 |
|---|---|
| 消费者和提供者都由组织内团队开发，消费者数量可管理 | 消费者驱动的契约测试，如 Pact |
| 公开接口，消费者无法逐一识别 | 用 OpenAPI 校验真实响应 |
| 对方团队不参与，或无法控制提供者数据 | 用 OpenAPI 校验，或维护第三方替身 |
| 只透传请求、不校验内容的接口 | 在消费者和下游之间建立契约 |

**来源：** Pact 文档

### 契约测试不代替功能测试 【推荐】

契约测试只验证双方对请求和响应的理解一致。校验规则、副作用和业务逻辑由提供者自己的测试覆盖。每类错误响应保留一个代表性场景即可。

**原因：** 把业务规则写进契约，提供者调整规则时会破坏本来不受影响的消费者测试。

**来源：** Pact 文档

### 消费者只断言自己使用的内容 【推荐】

响应字段默认按类型匹配，内容确实影响消费者时才精确匹配。不断言用不到的字段和提供者的格式规则。

**来源：** Pact 文档

### 消费者测试覆盖真实的客户端代码 【推荐】

测试范围是负责发起调用的客户端模块，不走 UI，也不只测原始 HTTP 调用。业务代码中的所有调用都经过这个模块。

**来源：** Pact 文档

### 写操作的响应回显更新后的内容 【推荐】

每个交互独立验证，提供者忽略拼错的字段时测试仍会通过。让响应包含更新后的值，或在读写两侧共用测试数据。查询参数同理，可让响应回显实际生效的参数。

**来源：** Pact 文档

### 提供者验证在本地实例上运行 【推荐】

验证针对本地或 CI 中启动的提供者，并随其他测试一起运行，不针对已部署环境。下游依赖可以替换，但只替换请求解析和校验之后的层。

**原因：** 针对已部署环境运行时，反馈慢且难调试，也无法并行。

**来源：** Pact 文档
