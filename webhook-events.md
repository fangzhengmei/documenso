# Documenso Webhook 事件推送机制

## 概述

Documenso 的 webhook 系统实现了文档状态变更的异步推送，包含三大核心机制：**请求签名验证**、**失败重试退避**和**事件版本兼容**。

---

## 1. 签名字段机制

### 1.1 签名实现原理

Webhook 通过 HTTP 请求头 `X-Documenso-Secret` 传递签名密钥，用于验证请求来源的合法性。

**代码位置**: `packages/lib/server-only/webhooks/execute-webhook-call.ts:40`

```typescript
headers: {
  'Content-Type': 'application/json',
  'X-Documenso-Secret': secret ?? '',
},
```

### 1.2 密钥配置流程

1. 用户在创建 webhook 时可选择配置 `secret` 字段（`Webhook.secret`）
2. 密钥存储在数据库中，与 webhook 绑定
3. 每次推送事件时，系统自动将密钥注入 HTTP 头
4. 接收方通过比对 `X-Documenso-Secret` 头与本地存储的密钥完成验证

### 1.3 验证步骤

```
接收方验证流程:
1. 从 HTTP 请求头提取 X-Documenso-Secret
2. 与本地配置的 webhook secret 比对
3. 若匹配则处理请求，否则拒绝
```

---

## 2. 失败重试与退避机制

### 2.1 重试配置参数

系统使用 BullMQ 作为异步任务队列，配置如下：

**代码位置**: `packages/lib/jobs/client/bullmq.ts:24-25, 158-162`

| 参数 | 值 | 说明 |
|------|-----|------|
| `DEFAULT_MAX_RETRIES` | 3 次 | 最大重试次数 |
| `DEFAULT_BACKOFF_DELAY` | 1000ms | 初始退避延迟 |
| 退避策略 | `exponential` | 指数退避 |

### 2.2 指数退避算法

```
退避公式: delay = DEFAULT_BACKOFF_DELAY * (2 ^ attempt)

第 1 次重试: 1000ms * 2^0 = 1秒
第 2 次重试: 1000ms * 2^1 = 2秒
第 3 次重试: 1000ms * 2^2 = 4秒
```

### 2.3 任务状态流转

**代码位置**: `packages/lib/jobs/client/bullmq.ts:247-295`

```
PENDING → PROCESSING → { SUCCESS, FAILED }
           ↓ 失败
         PENDING (重试计数 +1)
           ↓ 达到最大重试
         FAILED (最终失败)
```

### 2.4 重试触发条件

Webhook 执行失败时抛出异常，触发 BullMQ 的重试机制：

**代码位置**: `packages/lib/jobs/definitions/internal/execute-webhook.handler.ts:43-45`

```typescript
if (!result.success) {
  throw new Error(`Webhook execution failed with status ${result.responseCode}`);
}
```

---

## 3. 事件版本号兼容策略

### 3.1 兼容字段设计

为保证向后兼容性，payload 中同时提供新旧两种字段命名：

**代码位置**: `packages/lib/types/webhook-payload.ts:84-87, 162-163`

```typescript
export const ZWebhookDocumentSchema = z.object({
  // ... 其他字段
  recipients: z.array(ZWebhookRecipientSchema),
  
  /**
   * Legacy field for backwards compatibility.
   */
  Recipient: z.array(ZWebhookRecipientSchema),
});
```

### 3.2 字段映射关系

| 新字段名 (小写开头) | 旧字段名 (大写开头) | 兼容性说明 |
|---------------------|---------------------|------------|
| `recipients` | `Recipient` | 接收人列表，两者内容完全相同 |

### 3.3 迁移建议

1. **新集成**: 推荐使用 `recipients` 字段（驼峰命名，符合 JavaScript 惯例）
2. **现有集成**: 可继续使用 `Recipient`，但建议逐步迁移到新字段
3. **过渡方案**: 同时支持两种字段名，确保平滑升级

---

## 4. 完整调用链路

### 4.1 事件触发流程

```
1. 文档状态变更
   ↓
2. triggerWebhook() 被调用
   packages/lib/server-only/webhooks/trigger/trigger-webhook.ts:13
   ↓
3. 查询匹配的 webhook 配置
   getAllWebhooksByEventTrigger()
   ↓
4. 为每个 webhook 创建异步任务
   jobs.triggerJob({ name: 'internal.execute-webhook' })
   ↓
5. BullMQ 队列调度执行
   packages/lib/jobs/client/bullmq.ts:134
   ↓
6. 执行 HTTP POST 请求
   executeWebhookCall()
   packages/lib/server-only/webhooks/execute-webhook-call.ts:23
   ↓
7. 记录调用结果到 WebhookCall
   packages/lib/jobs/definitions/internal/execute-webhook.handler.ts:29
```

### 4.2 请求超时配置

**代码位置**: `packages/lib/server-only/webhooks/execute-webhook-call.ts:6`

```typescript
const WEBHOOK_TIMEOUT_MS = 10_000; // 10秒超时
```

---

## 5. 数据模型

### 5.1 Webhook 配置表

**代码位置**: `packages/prisma/schema.prisma:184-197`

```prisma
model Webhook {
  id            String                 @id @default(cuid())
  webhookUrl    String
  eventTriggers WebhookTriggerEvents[]
  secret        String?
  enabled       Boolean                @default(true)
  createdAt     DateTime               @default(now())
  updatedAt     DateTime               @default(now()) @updatedAt
  userId        Int
  teamId        Int
  webhookCalls  WebhookCall[]
}
```

### 5.2 调用日志表

```prisma
enum WebhookCallStatus {
  SUCCESS
  FAILED
}

model WebhookCall {
  id             String            @id @default(cuid())
  url            String
  event          WebhookTriggerEvents
  status         WebhookCallStatus
  requestBody    Json
  responseCode   Int
  responseBody   Json
  responseHeaders Json
  createdAt      DateTime          @default(now())
  webhookId      String
  webhook        Webhook           @relation(fields: [webhookId], references: [id], onDelete: Cascade)
}
```

---

## 6. 支持的事件类型

**代码位置**: `packages/prisma/schema.prisma:167-182`

| 事件类型 | 触发时机 |
|---------|---------|
| `DOCUMENT_CREATED` | 文档创建 |
| `DOCUMENT_SENT` | 文档发送给接收人 |
| `DOCUMENT_OPENED` | 接收人首次打开文档 |
| `DOCUMENT_SIGNED` | 接收人签署文档 |
| `DOCUMENT_COMPLETED` | 所有接收人完成签署 |
| `DOCUMENT_REJECTED` | 接收人拒绝文档 |
| `DOCUMENT_CANCELLED` | 文档所有者删除文档 |
| `RECIPIENT_EXPIRED` | 接收人签署过期 |
| `DOCUMENT_RECIPIENT_COMPLETED` | 单个接收人完成操作 |
| `DOCUMENT_REMINDER_SENT` | 提醒邮件已发送 |
| `TEMPLATE_CREATED` | 模板创建 |
| `TEMPLATE_UPDATED` | 模板更新 |
| `TEMPLATE_DELETED` | 模板删除 |
| `TEMPLATE_USED` | 模板被使用创建文档 |

---

## 7. Payload 结构示例

```json
{
  "event": "DOCUMENT_SIGNED",
  "payload": {
    "id": 123,
    "externalId": "external-ref-456",
    "userId": 789,
    "title": "服务合同",
    "status": "PENDING",
    "createdAt": "2024-01-15T10:30:00.000Z",
    "updatedAt": "2024-01-15T10:35:00.000Z",
    "recipients": [
      {
        "id": 1,
        "email": "signer@example.com",
        "name": "张三",
        "token": "abc123",
        "signedAt": "2024-01-15T10:35:00.000Z",
        "signingStatus": "SIGNED",
        "role": "SIGNER"
      }
    ],
    "Recipient": [
      // 与 recipients 内容完全相同，用于兼容旧版本
    ]
  },
  "createdAt": "2024-01-15T10:35:00.000Z",
  "webhookEndpoint": "https://your-api.com/webhook"
}
```

---

## 8. 安全最佳实践

1. **始终启用 HTTPS**: 确保 webhook endpoint 使用 HTTPS 加密传输
2. **配置签名密钥**: 为每个 webhook 配置唯一的 secret 并验证 `X-Documenso-Secret` 头
3. **快速响应**: 30秒内返回 2xx 状态码，异步处理业务逻辑
4. **幂等处理**: 由于重试机制可能导致重复推送，确保业务逻辑幂等

---

## 总结

Documenso 的 webhook 系统通过以下机制保证可靠性：

1. **签名验证**: `X-Documenso-Secret` 头确保请求来源可信
2. **重试机制**: BullMQ + 指数退避（最多3次重试）保障交付
3. **版本兼容**: 双字段命名策略确保平滑升级不破坏现有集成

这套机制共同构成了一个健壮、可扩展的异步事件推送系统。
