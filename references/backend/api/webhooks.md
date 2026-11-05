# Webhook

本文讲 Webhook 的发送和接收。密钥管理见 [crypto-and-secrets.md](../security/crypto-and-secrets.md)，SSRF 的通用防护见 [web-risks.md](../security/web-risks.md)。

## 签名

比较签名用常量时间比较，见 [crypto-and-secrets.md](../security/crypto-and-secrets.md)。

### 每次投递都签名 【必须】

签名内容包含消息 ID、时间戳和请求体，用 `.` 连接，通过请求头传递。

| 请求头 | 内容 |
|---|---|
| `webhook-id` | 事件的唯一标识，重试时不变 |
| `webhook-timestamp` | 本次尝试的 Unix 秒级时间戳 |
| `webhook-signature` | 签名列表，空格分隔 |

**示例：**

```text
签名内容：msg_2KWPBgLl.1674087231.{"type":"invoice.paid"}
```

```http
POST /webhooks/billing HTTP/1.1
Content-Type: application/json
webhook-id: msg_2KWPBgLl
webhook-timestamp: 1674087231
webhook-signature: v1,K5oZfzN95Z9UVu1EsfQmfVNQhnkZ2pj9o9NDN/H/pI4=

{"type":"invoice.paid"}
```

**原因：** 只签请求体时，时间戳和 ID 可以被篡改，重放防护就失效了。

**来源：** Standard Webhooks

### 选择合适的签名算法 【推荐】

无法同时控制收发两端的安全时，优先用非对称签名，接收方只持有公钥。

| 类型 | 算法 | 标识 | 密钥格式 |
|---|---|---|---|
| 对称 | HMAC-SHA256 | `v1` | 24～64 字节随机数，base64 编码，前缀 `whsec_` |
| 非对称 | ed25519 | `v1a` | 前缀 `whsk_`（私钥）、`whpk_`（公钥） |

**来源：** Standard Webhooks

### 每个端点使用独立的密钥 【必须】

对称密钥按端点隔离。非对称密钥至少按客户隔离。

**来源：** Standard Webhooks

### 用原始请求体验证签名 【必须】

验证前不解析 JSON。发送方签名的字节必须与实际发送的完全相同。

**示例：**

反例：解析 JSON 后重新序列化，再计算签名。

正例：读取原始字节，验证通过后再解析。

**原因：** 重新序列化可能改变空格和字段顺序，导致验证失败。

**来源：** Standard Webhooks

### 只信任预先配置的公钥 【必须】

不使用请求中携带的公钥。

**来源：** Standard Webhooks

### 支持密钥无中断轮换 【推荐】

轮换期间用新旧密钥各签一次，放在同一个请求头。接收方逐个验证，任意一个通过即可。

**来源：** Standard Webhooks

## 重放防护

### 校验时间戳容差 【必须】

`webhook-timestamp` 与当前时间的差超出容差时拒绝，比如 5 分钟。时间戳必须在签名验证通过后才可信。发送方每次重试都更新时间戳并重新签名。

**来源：** Standard Webhooks

### 消息 ID 和时间戳不接受用户控制 【必须】

至少禁止其中出现 `.`，否则攻击者可以构造相同的签名内容。

**来源：** Standard Webhooks

## 投递和重试

发送方必须设置超时，建议 15～30 秒，规则见 [resilience.md](../reliability/resilience.md)。

### 按状态码判定结果 【推荐】

| 响应 | 发送方处理 |
|---|---|
| 2xx | 成功 |
| 3xx | 失败，不跟随重定向 |
| 410 | 禁用端点，停止发送 |
| 429、502、504 | 失败并降低发送速率 |
| 其他状态码、超时、连接中断 | 失败，按计划重试 |

响应带 `retry-after` 时，下次重试不早于该时间。

**来源：** Standard Webhooks

### 重试使用指数退避和抖动 【必须】

**适用：** 线上服务

重试计划覆盖数天，并加随机抖动。

**示例：**

| 重试间隔 | 距首次发送 |
|---|---|
| 立即 | 0 |
| 5 秒 | 5 秒 |
| 5 分钟 | 约 5 分钟 |
| 30 分钟 | 约 35 分钟 |
| 2 小时 | 约 2.5 小时 |
| 5 小时 | 约 7.5 小时 |
| 10 小时 | 约 17.5 小时 |
| 14 小时 | 约 31.5 小时 |

**来源：** Standard Webhooks

### 投递与业务写入解耦 【必须】

**适用：** 线上服务

事件先持久化，再由后台任务发送，不在业务事务内调用对方地址。可靠写入见 [consistency.md](../reliability/consistency.md)。

### 接收方先确认再处理 【推荐】

验证签名并持久化后立即返回 2xx，耗时处理放到后台任务。

**原因：** 同步处理容易超时，触发重复投递。

### 长期失败时通知并禁用 【推荐】

端点持续失败时，停止投递并通知对方。

**来源：** Standard Webhooks

## 幂等

### 接收方按事件 ID 去重 【必须】

`webhook-id` 作为幂等键，用唯一约束记录已处理的事件，与业务写入放在同一事务内。去重记录的保留时间要覆盖发送方的完整重试周期。

**示例：**

```sql
INSERT INTO webhook_events (event_id, received_at)
VALUES ('msg_2KWPBgLl', now())
ON CONFLICT (event_id) DO NOTHING;
```

**原因：** 网络问题、重试和恶意重放都会造成重复投递。

**来源：** Standard Webhooks

### 不依赖投递顺序 【必须】

重试会打乱顺序。接收方用事件时间或资源版本判断新旧，或收到通知后回查最新状态。

## 消息内容

### 事件结构统一 【推荐】

包含 `type`、`timestamp` 和 `data`。`timestamp` 是事件发生时间，不是投递时间。类型用点分隔的层级名称，同一类型的结构固定，并写进契约。

**示例：**

```json
{
  "type": "invoice.paid",
  "timestamp": "2026-11-05T08:30:00Z",
  "data": { "id": "inv_1001" }
}
```

**来源：** Standard Webhooks

### 控制消息体大小 【推荐】

建议小于 20 KB。大数据放链接，或只发标识，由接收方回查。只发标识也能减少敏感数据外发，接收方回查时还要经过正常鉴权。

**来源：** Standard Webhooks

## 发送方安全

回调地址由用户提供，发送时按 [web-risks.md](../security/web-risks.md) 防 SSRF，发送进程放在无法访问内部服务的独立子网。

### 回调地址使用 HTTPS 【推荐】

签名只保证完整性，不加密内容。

**来源：** Standard Webhooks
