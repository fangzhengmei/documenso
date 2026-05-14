# 多收件人签署顺序状态机分析报告

## 1. 核心数据模型

### 1.1 信封状态枚举 (envelope.status)

```prisma
enum DocumentStatus {
  DRAFT       # 草稿状态
  PENDING     # 待签署状态
  COMPLETED   # 已完成（封存后）
  REJECTED    # 已拒绝（封存后）
}
```

### 1.2 收件人签署状态枚举 (recipient.signingStatus)

```prisma
enum SigningStatus {
  NOT_SIGNED  # 未签署
  SIGNED      # 已签署
  REJECTED    # 已拒绝
}
```

### 1.3 签署顺序类型枚举

```prisma
enum DocumentSigningOrder {
  PARALLEL      # 并行签署：所有收件人可同时签署
  SEQUENTIAL    # 顺序签署：必须按指定顺序依次签署
}
```

---

## 2. signingStatus 与 envelope.status 的真实关系

### 2.1 状态独立性

| 字段 | 所属实体 | 更新时机 |
|------|----------|----------|
| `recipient.signingStatus` | 单个收件人 | 收件人点击"完成"或"拒绝"时立即更新 |
| `envelope.status` | 整个文档 | **异步封存任务完成后**才更新 |

### 2.2 关键差异

- **收件人状态是实时的**：每个收件人完成签署时，其 `signingStatus` 立即变为 `SIGNED`
- **信封状态是延迟的**：所有收件人完成后，信封不会立即变成 `COMPLETED`，而是需要等待异步封存任务执行完毕
- **过渡状态 PROCESSING**：当所有收件人完成签署（或有人拒绝）但封存未完成时，信封的实际状态是 PENDING，但 API 会返回 PROCESSING

---

## 3. Webhook / 任务触发边界

### 3.1 触发事件定义

| Webhook 事件 | 触发时机 | 触发位置 |
|-------------|----------|----------|
| `DOCUMENT_RECIPIENT_COMPLETED` | 单个收件人完成签署时 | `complete-document-with-token.ts` 事务内 |
| `DOCUMENT_SIGNED` | **任一收件人完成签署时立即触发** | `complete-document-with-token.ts` 末尾 |
| `DOCUMENT_COMPLETED` | **所有收件人完成 + 封存任务完成后** | `seal-document.handler.ts` 异步任务末尾 |
| `DOCUMENT_REJECTED` | **任一收件人拒绝 + 封存任务完成后** | `seal-document.handler.ts` 异步任务末尾 |

### 3.2 关键边界说明

⚠️ **重要：DOCUMENT_SIGNED ≠ DOCUMENT_COMPLETED**

- `DOCUMENT_SIGNED` 会被触发 **N 次**（N 为签署人数），每次有人签署就触发一次
- `DOCUMENT_COMPLETED` 只会被触发 **1 次**（封存完成后）
- 拒绝文档不会触发 `DOCUMENT_SIGNED`，直接触发 `DOCUMENT_REJECTED`（封存后）

---

## 4. PROCESSING 过渡链路详解

### 4.1 PROCESSING 状态判定逻辑

**文件位置**: `packages/trpc/server/envelope-router/signing-status-envelope.ts:65-75`

```typescript
// 判定逻辑
const isComplete =
  // 有任一收件人拒绝
  envelope.recipients.some(r => r.signingStatus === SigningStatus.REJECTED) ||
  // 所有非CC收件人都已签署
  envelope.recipients.every(r => r.role === RecipientRole.CC || r.signingStatus === SigningStatus.SIGNED);

if (isComplete) {
  // envelope.status 仍为 PENDING，但所有收件人都完成了
  // 说明正在封存过程中
  return { status: 'PROCESSING' };
}
```

### 4.2 过渡链路的产生原因

1. **收件人完成签署** → `recipient.signingStatus = SIGNED`（数据库立即更新）
2. **触发异步封存任务** → `jobs.triggerJob('internal.seal-document')`（排队执行）
3. **封存任务执行中** → 此时查询信封状态：
   - `envelope.status` 仍为 `PENDING`（数据库未更新）
   - 所有 `recipient.signingStatus` 都是 `SIGNED` 或 `REJECTED`
   - API 返回 `PROCESSING` 过渡状态
4. **封存任务完成** → `envelope.status = COMPLETED` 或 `REJECTED`

---

## 5. 完整时序表（核心）

### 5.1 正常签署完成时序（以 2 个顺序签署人为例）

| 时间点 | 操作 | recipient1.signingStatus | recipient2.signingStatus | envelope.status | 触发的 Webhook / 任务 |
|--------|------|---------------------------|---------------------------|-----------------|---------------------|
| T0 | 文档发送 | NOT_SIGNED | NOT_SIGNED | PENDING | `DOCUMENT_SENT` |
| T1 | 收件人1开始签署 | NOT_SIGNED | NOT_SIGNED | PENDING | - |
| T2 | 收件人1完成签署 | **SIGNED** | NOT_SIGNED | PENDING | `DOCUMENT_RECIPIENT_COMPLETED` → `DOCUMENT_SIGNED` |
| T3 | 收件人2开始签署（顺序模式下需等收件人1完成） | SIGNED | NOT_SIGNED | PENDING | - |
| T4 | 收件人2完成签署 | SIGNED | **SIGNED** | **PENDING** | `DOCUMENT_RECIPIENT_COMPLETED` → `DOCUMENT_SIGNED` → **触发 `internal.seal-document` 任务** |
| T5 | ⏳ 封存任务执行中（PDF处理、证书生成、数字签名） | SIGNED | SIGNED | **PENDING** | API 查询返回 `PROCESSING` |
| T6 | 封存任务完成 | SIGNED | SIGNED | **COMPLETED** | `DOCUMENT_COMPLETED` → 发送完成邮件 |

### 5.2 拒绝文档时序

| 时间点 | 操作 | recipient.signingStatus | envelope.status | 触发的 Webhook / 任务 |
|--------|------|--------------------------|-----------------|---------------------|
| T0 | 文档发送 | NOT_SIGNED | PENDING | `DOCUMENT_SENT` |
| T1 | 收件人点击拒绝 | NOT_SIGNED | PENDING | - |
| T2 | 确认拒绝 | **REJECTED** | **PENDING** | 创建审计日志 → **触发 `internal.seal-document` 任务** → 触发拒绝邮件 → 触发取消邮件 |
| T3 | ⏳ 封存任务执行中 | REJECTED | PENDING | API 查询返回 `PROCESSING` |
| T4 | 封存任务完成 | REJECTED | **REJECTED** | `DOCUMENT_REJECT