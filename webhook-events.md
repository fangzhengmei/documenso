# Documenso Webhook 事件推送机制

## 概述

Documenso 的 webhook 系统实现了文档状态变更的异步推送，包含三大核心机制：**请求签名验证**、**失败重试退避**和**隐式版本兼容策略**。

---

## 1. 版本号字段检查结论

**payload 中没有显式的事件版本号字段。**

根据 `packages/lib/types/webhook-payload.ts:94-99` 的 Schema 定义，完整的 Payload 结构仅包含 4 个字段：

```typescript
export const ZWebhookPayloadSchema = z.object({
  event: z.nativeEnum(WebhookTriggerEvents),      // 事件类型
  payload: ZWebhookDocumentSchema,                // 文档数据
  createdAt: z.string(),                          // 创建时间
  webhookEndpoint: z.string(),                    // 推送端点
});
```

**没有 version/apiVersion 等版本标识字段。** 版本兼容性通过字段级别的向后兼容机制实现。

---

## 2. 版本兼容机制详解

### 2.1 Recipient/recipients 双字段兼容

**代码位置**：`packages/lib/types/webhook-payload.ts:82-87, 162-163`

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

**实现机制**：
- **服务端组装阶段**：两个字段指向**完全相同的数组对象**（同一个引用）
  - 代码位置：`packages/lib/types/webhook-payload.ts:162-163`
  - 内存中 `Recipient === recipients` 为 `true`
- **网络传输后**：接收方拿到的 JSON 经过序列化/反序列化，两个字段是**值相同但独立的对象**
  - `JSON.parse(JSON.stringify(payload)).Recipient !== JSON.parse(JSON.stringify(payload)).recipients`
- 内容、顺序、元素始终完全一致
- 唯一区别是字段命名的大小写风格
  - `recipients`：驼峰命名，新 API 标准风格
  - `Recipient`：首字母大写，旧 API 遗留风格

**边界情况处理**：

| 场景 | 行为 | 风险 |
|------|------|------|
| 接收方只读取 `Recipient` | 正常工作 ✅ | 未来此字段可能被废弃 |
| 接收方只读取 `recipients` | 正常工作 ✅ | 推荐方式 |
| 服务端内部修改数组元素 | 两个字段都会同步变化 ⚠️ | 仅影响服务端内部逻辑 |
| 接收方修改任一字段 | 另一字段不受影响 ✅ | JSON 反序列化后对象独立 |
| JSON 序列化后反序列化 | 两个字段独立存在 ✅ | 增加传输体积，但无功能风险 |

**迁移建议**：
- 新集成：统一使用 `recipients` 字段
- 现有系统：尽快从 `Recipient` 迁移到 `recipients`
- 过渡期间：不要依赖字段的引用相等性（接收端永远不相等）

---

### 2.2 legacyId 映射兼容机制

**代码位置**：`packages/lib/utils/envelope.ts:177-233` + `packages/lib/types/webhook-payload.ts:113-116, 138-139`

#### 背景

Documenso 经历了架构演进：
- **旧架构**：使用自增数字 ID（document.id = 123）
- **新架构**：使用 Envelope 模型，ID 格式为 `document_123` 或 `template_123`（称为 secondaryId）

#### 映射函数

```typescript
// 数字 ID → 字符串 secondaryId
const mapDocumentIdToSecondaryId = (documentId: number) => `document_${documentId}`;
const mapTemplateIdToSecondaryId = (templateId: number) => `template_${templateId}`;

// 字符串 secondaryId → 数字 ID（webhook 中使用）
export const mapSecondaryIdToDocumentId = (secondaryId: string) => {
  const parsed = ZDocumentIdSchema.safeParse(secondaryId);
  if (!parsed.success) throw new AppError(AppErrorCode.INVALID_BODY, { message: 'Invalid document ID' });
  return parseInt(parsed.data.split('_')[1]);
};
```

#### Webhook 中的应用

```typescript
// webhook-payload.ts:113-116
const legacyId =
  envelope.type === EnvelopeType.DOCUMENT
    ? mapSecondaryIdToDocumentId(envelope.secondaryId)  // document_123 → 123
    : mapSecondaryIdToTemplateId(envelope.secondaryId); // template_123 → 123

// 返回给接收方的是数字 ID，与旧 API 一致
return {
  id: legacyId,  // 123，不是 document_123
  // ...
};
```

#### 边界情况与风险

| 场景 | 行为 | 风险等级 |
|------|------|----------|
| secondaryId 格式错误（如 `doc_123`） | 抛出 AppError，webhook 发送失败 ⚠️ | 中 |
| 接收方需要真实的 Envelope ID | 无法从 payload 直接获取，需额外调用 API | 低 |
| document 与 template ID 冲突 | 数字 ID 可能重复（document_123 和 template_123 都映射到 123） | 中 |
| 未来引入新的 envelope type | 映射函数需要同步更新，否则失败 | 高 |

**注意**：接收方无法通过 `id` 字段区分是 document 还是 template，需要结合 `event` 类型或 `templateId` 字段判断。

---

## 3. 失败重试与退避机制（基于代码推导）

### 3.1 核心配置参数

**代码位置**：`packages/lib/jobs/client/bullmq.ts:24, 158-162`

```typescript
const DEFAULT_MAX_RETRIES = 3;
const DEFAULT_BACKOFF_DELAY = 1000; // 1秒

// 任务入队时配置
await this._queue.add(job.id, jobData, {
  attempts: DEFAULT_MAX_RETRIES,
  backoff: {
    type: 'exponential',
    delay: DEFAULT_BACKOFF_DELAY,
  },
});
```

### 3.2 总尝试次数推导

根据 BullMQ 语义：
- `attempts` = **总尝试次数**（包含首次执行）
- 重试次数 = `attempts - 1`

因此：
- **总尝试次数**：3 次（首次 + 2 次重试）
- **重试次数**：2 次

**验证代码**：`packages/lib/jobs/client/bullmq.ts:284`

```typescript
// 判断是否是最后一次尝试
const isFinalAttempt = job.attemptsMade >= (job.opts.attempts ?? DEFAULT_MAX_RETRIES) - 1;

// 当 attemptsMade = 0 → 首次执行
// 当 attemptsMade = 1 → 第1次重试
// 当 attemptsMade = 2 → 第2次重试（最后一次，isFinalAttempt = true）
```

### 3.3 指数退避间隔计算

BullMQ 指数退避公式：
```
延迟 = delay * (2 ^ (attemptsMade - 1))
```

其中：
- `attemptsMade` = 0 时：首次执行，**无延迟，立即执行**
- `attemptsMade` = 1 时：第 1 次重试

**实际重试时间表**：

| 次数 | attemptsMade | 状态 | 延迟计算公式 | 实际延迟 |
|-----:|-------------:|------|-------------:|---------:|
| 1 | 0 | 首次执行 | 无 | 立即 |
| 2 | 1 | 第 1 次重试 | 1000ms * 2^(1-1) = 1000ms * 1 | **1 秒** |
| 3 | 2 | 第 2 次重试 | 1000ms * 2^(2-1) = 1000ms * 2 | **2 秒** |

> **注意**：此为代码层面配置。官方文档（setup.mdx:323-329）描述的 5 次重试（1分钟/5分钟/30分钟/2小时）可能是生产环境的其他配置或 BullMQ 的自定义策略。

### 3.4 重试触发条件

**代码位置**：`packages/lib/jobs/definitions/internal/execute-webhook.handler.ts:43-45`

```typescript
if (!result.success) {
  throw new Error(`Webhook execution failed with status ${result.responseCode}`);
}
```

触发重试的情况：
1. HTTP 响应状态码非 2xx（`result.success = false`）
2. 请求超时（10 秒超时，见 `execute-webhook-call.ts:6`）
3. 网络错误或异常
4. 私有 URL 校验失败

### 3.5 任务状态流转

```
                     ┌──────────────────┐
                     │   入队 PENDING   │
                     └────────┬─────────┘
                              ↓
                     ┌──────────────────┐
                     │  执行 PROCESSING │ attemptsMade = 0
                     └────────┬─────────┘
                              │
              ┌───────────────┴───────────────┐
              ↓                               ↓
    ┌──────────────────┐            ┌──────────────────┐
    │  成功 COMPLETED  │            │  失败，重试 1    │ attemptsMade = 1
    └──────────────────┘            └────────┬─────────┘  延迟 1 秒
                                             ↓
                                    ┌──────────────────┐
                                    │  失败，重试 2    │ attemptsMade = 2
                                    └────────┬─────────┘  延迟 2 秒
                                             ↓
                                    ┌──────────────────┐
                                    │  最终 FAILED     │
                                    └──────────────────┘
```

---

## 4. 签名验证机制

### 4.1 实现原理

**代码位置**：`packages/lib/server-only/webhooks/execute-webhook-call.ts:38-41`

```typescript
headers: {
  'Content-Type': 'application/json',
  'X-Documenso-Secret': secret ?? '',  // 配置的 secret 原样传输
},
```

### 4.2 验证流程

```
接收方验证步骤：
1. 从 HTTP Header 中提取 `X-Documenso-Secret`
2. 与本地保存的 webhook 配置密钥进行字符串比对
3. 若匹配则继续处理，否则拒绝请求
```

### 4.3 安全说明

- **没有 HMAC 签名**：secret 是明文传输，不是 payload 的哈希签名
- **依赖 HTTPS**：必须使用 HTTPS 加密传输，否则 secret 可能被窃取
- **空值处理**：未配置 secret 时 header 值为空字符串

---

## 5. 完整调用链路

```
1. 文档状态变更事件触发
   ↓
2. triggerWebhook() 被调用
   packages/lib/server-only/webhooks/trigger/trigger-webhook.ts:13
   ↓
3. 查询所有匹配的 webhook 配置（按事件类型过滤）
   getAllWebhooksByEventTrigger()
   ↓
4. 为每个 webhook 创建异步任务
   jobs.triggerJob({ name: 'internal.execute-webhook' })
   ↓
5. BullMQ 队列调度执行（配置 3 次尝试 + 指数退避）
   packages/lib/jobs/client/bullmq.ts:149-164
   ↓
6. 执行 HTTP POST 请求（10 秒超时，附带 X-Documenso-Secret header）
   executeWebhookCall()
   packages/lib/server-only/webhooks/execute-webhook-call.ts:23
   ↓
7. 记录调用结果到 WebhookCall 表（持久化请求/响应）
   packages/lib/jobs/definitions/internal/execute-webhook.handler.ts:29-41
   ↓
8. 若失败则抛出异常，触发 BullMQ 重试机制
```

---

## 6. 支持的事件类型

**代码位置**：`packages/prisma/schema.prisma:167-182`

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
    "externalId": "your-external-ref-456",
    "userId": 789,
    "title": "服务合同",
    "status": "PENDING",
    "createdAt": "2024-01-15T10:30:00.000Z",
    "updatedAt": "2024-01-15T10:35:00.000Z",
    "completedAt": null,
    "deletedAt": null,
    "teamId": 1,
    "templateId": null,
    "source": "DOCUMENT",
    "recipients": [
      {
        "id": 1,
        "documentId": 123,
        "templateId": null,
        "email": "signer@example.com",
        "name": "张三",
        "token": "abc123def456",
        "signedAt": "2024-01-15T10:35:00.000Z",
        "signingStatus": "SIGNED",
        "role": "SIGNER",
        "readStatus": "OPENED",
        "sendStatus": "SENT"
      }
    ],
    "Recipient": [
      { "同 recipients 数组完全一致" }
    ]
  },
  "createdAt": "2024-01-15T10:35:00.000Z",
  "webhookEndpoint": "https://your-api.com/webhook/documenso"
}
```

---

## 8. 最佳实践与注意事项

### 集成建议

1. **字段选择**：优先使用 `recipients`（驼峰命名），避免依赖 `Recipient` 字段
2. **幂等处理**：由于重试机制可能导致重复推送，建议使用 `event + payload.id + createdAt` 作为幂等键
3. **快速响应**：10 秒内返回 2xx 状态码，业务逻辑异步处理
4. **Secret 验证**：始终验证 `X-Documenso-Secret` header，防止伪造请求

### 版本兼容风险

1. **无版本标识**：接收方无法通过版本号判断字段变更，需关注文档更新
2. **legacyId 映射**：`id` 字段是数字（兼容旧 API），不是真实的 Envelope ID
3. **双字段冗余**：`Recipient` 和 `recipients` 增加 payload 体积，未来可能移除旧字段

### 监控与调试

1. 检查 WebhookCall 表记录所有请求/响应详情
2. 监控 BackgroundJob 表查看重试状态
3. 注意 `attemptsMade` 字段记录已尝试次数

---

## 总结

| 机制 | 实现方式 | 关键参数 |
|------|---------|---------|
| **签名验证** | HTTP Header `X-Documenso-Secret` 明文传输 | 无加密，依赖 HTTPS |
| **重试策略** | BullMQ 指数退避 | attempts=3（总3次，重试2次），delay=1000ms |
| **重试间隔** | 第1次重试：1秒，第2次重试：2秒 | 公式：`delay * 2^(attemptsMade - 1)` |
| **版本兼容** | 双字段（`recipients`/`Recipient`）+ legacyId 映射 | 无显式 version 字段 |
| **超时配置** | fetch timeout | 10 秒 |
