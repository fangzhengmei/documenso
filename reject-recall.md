# 拒签与撤回流程协作分析报告

## 概述

本文档分析 Documenso 系统中**拒签（Reject）**与**撤回（Recall/Delete）**两类反向操作的流程协作，包括：
- 状态机转换逻辑
- 通知链触发机制
- 审计记录的生成与关联

---

## 一、文档状态机定义

### 1.1 文档状态枚举（DocumentStatus）

```typescript
enum DocumentStatus {
  DRAFT       // 草稿
  PENDING     // 签署中
  COMPLETED   // 已完成
  REJECTED    // 已拒签
}
```

### 1.2 签署人状态枚举（SigningStatus）

```typescript
enum SigningStatus {
  NOT_SIGNED  // 未签署
  SIGNED      // 已签署
  REJECTED    // 已拒签
}
```

---

## 二、拒签触发与状态回退

### 2.1 拒签入口：`reject-document-with-token.ts`

#### 前置条件检查
- 文档必须处于 `PENDING` 状态
- 签署人未过期

#### 核心流程（原子事务）

```typescript
// 1. 更新签署人状态
prisma.recipient.update({
  where: { id: recipientId },
  data: {
    signedAt: new Date(),
    signingStatus: SigningStatus.REJECTED,
    rejectionReason: reason
  }
})

// 2. 记录审计日志
prisma.documentAuditLog.create({
  type: DOCUMENT_AUDIT_LOG_TYPE.DOCUMENT_RECIPIENT_REJECTED,
  data: {
    recipientEmail,
    recipientName,
    recipientId,
    recipientRole,
    reason
  }
})
```

### 2.2 异步后续处理（Job 触发）

拒签操作后，立即触发三个异步 Job：

| Job 名称 | 用途 | 触发时机 |
|---------|------|---------|
| `internal.seal-document` | 文档密封与最终状态转换 | 拒签后立即 |
| `send.signing.rejected.emails` | 拒签确认邮件与通知 | 拒签后立即 |
| `send.document.cancelled.emails` | 取消通知邮件给其他收件人 | 拒签后立即 |

### 2.3 Seal-Document 中的状态回退逻辑

在 `seal-document.handler.ts` 中处理文档最终状态：

```typescript
// 判断是否有任何收件人拒签
const rejectedRecipient = recipientsWithoutCCers.find(
  recipient => recipient.signingStatus === SigningStatus.REJECTED
);

const isRejected = Boolean(rejectedRecipient);
const rejectionReason = rejectedRecipient?.rejectionReason ?? '';

// 只要有一个人拒签，文档整体状态变为 REJECTED
const finalEnvelopeStatus = isRejected 
  ? DocumentStatus.REJECTED 
  : DocumentStatus.COMPLETED;
```

#### 拒签状态的最终落库

```typescript
await prisma.$transaction(async (tx) => {
  // 更新文档状态为 REJECTED
  await tx.envelope.update({
    where: { id: envelope.id },
    data: {
      status: finalEnvelopeStatus,  // REJECTED
      completedAt: new Date()
    }
  });

  // 记录 DOCUMENT_COMPLETED 审计日志（标记 isRejected）
  await tx.documentAuditLog.create({
    type: DOCUMENT_AUDIT_LOG_TYPE.DOCUMENT_COMPLETED,
    data: {
      transactionId: nanoid(),
      isRejected: true,
      rejectionReason: rejectionReason
    }
  });
});
```

#### PDF 处理的拒签标记

```typescript
// 在 PDF 上添加拒签章
if (isRejected) {
  await addRejectionStampToPdf(pdfDoc, rejectionReason);
}

// 文件名后缀区分
const suffix = isRejected ? '_rejected.pdf' : '_signed.pdf';
```

---

## 三、撤回触发与状态回退

### 3.1 撤回入口：`delete-document.ts`

撤回操作即文档所有者删除文档，分为两种策略：

| 文档状态 | 删除策略 | 结果 |
|---------|---------|------|
| `COMPLETED` | 软删除（Soft Delete） | 设置 `deletedAt` 时间戳，保留记录 |
| `DRAFT/PENDING` | 硬删除（Hard Delete） | 直接从数据库删除 |

#### 权限校验

```typescript
const isUserTeamMember = await getMemberRoles({ teamId, reference });
const isUserOwner = envelope.userId === userId;
const userRecipient = envelope.recipients.find(r => r.email === user.email);

if (!isUserOwner && !isUserTeamMember && !userRecipient) {
  throw new AppError(AppErrorCode.UNAUTHORIZED);
}
```

### 3.2 软删除流程（已完成文档）

```typescript
if (isDocumentCompleted(envelope.status)) {
  await prisma.$transaction(async (tx) => {
    // 审计日志：DOCUMENT_DELETED (SOFT)
    await tx.documentAuditLog.create({
      type: DOCUMENT_AUDIT_LOG_TYPE.DOCUMENT_DELETED,
      data: { type: 'SOFT' }
    });

    // 仅标记删除时间
    await tx.envelope.update({
      where: { id: envelope.id },
      data: { deletedAt: new Date().toISOString() }
    });
  });
}
```

### 3.3 硬删除流程（草稿/签署中）

```typescript
await prisma.$transaction(async (tx) => {
  // 审计日志：DOCUMENT_DELETED (HARD)
  await tx.documentAuditLog.create({
    type: DOCUMENT_AUDIT_LOG_TYPE.DOCUMENT_DELETED,
    data: { type: 'HARD' }
  });

  // 物理删除
  await tx.envelope.delete({
    where: {
      id: envelope.id,
      status: { not: DocumentStatus.COMPLETED }
    }
  });
});
```

### 3.4 撤回后的收件人隐藏处理

对于收件人角色，单独标记其视图中的删除状态：

```typescript
if (userRecipient?.documentDeletedAt === null) {
  await prisma.recipient.update({
    where: { id: userRecipient.id },
    data: { documentDeletedAt: new Date().toISOString() }
  });
}
```

---

## 四、通知链协作机制

### 4.1 拒签通知链

拒签操作触发三类邮件通知：

#### A. 拒签确认邮件（给拒签人）
- 模板：`DocumentRejectionConfirmedEmail`
- 内容：确认拒签成功，显示拒签原因
- 触发：`send.signing.rejected.emails` Job

#### B. 拒签通知邮件（给文档所有者）
- 模板：`DocumentRejectedEmail`
- 内容：告知哪个收件人拒签，附带拒签原因
- 发送人：使用系统内部邮箱 `DOCUMENSO_INTERNAL_EMAIL`

#### C. 取消通知邮件（给其他收件人）
- 模板：`DocumentCancelTemplate`
- 触发：`send.document.cancelled.emails` Job
- 接收范围：已发送/已查看且未拒签的收件人

### 4.2 撤回通知链

撤回操作触发两类通知：

#### A. 取消邮件（给所有收件人）
- 模板：`DocumentCancelTemplate`
- 发送条件：`documentDeleted` 邮件设置开启
- 接收范围：已发送且邮箱有效的收件人

#### B. Webhook 事件通知
- 事件类型：`WebhookTriggerEvents.DOCUMENT_CANCELLED`
- 触发时机：文档所有者删除后
- 接收方：配置的 Webhook 端点

### 4.3 Webhook 事件映射

| 操作 | Webhook 事件 |
|-----|-------------|
| 拒签完成 | `DOCUMENT_REJECTED` |
| 撤回删除 | `DOCUMENT_CANCELLED` |
| 正常完成 | `DOCUMENT_COMPLETED` |

---

## 五、审计记录体系

### 5.1 拒签审计事件链

| 顺序 | 事件类型 | 触发时机 | 记录内容 |
|-----|---------|---------|---------|
| 1 | `DOCUMENT_RECIPIENT_REJECTED` | 收件人点击拒签时 | 收件人信息、拒签原因 |
| 2 | `DOCUMENT_COMPLETED` | Seal 完成后 | `isRejected: true`, rejectionReason, transactionId |

### 5.2 撤回审计事件链

| 顺序 | 事件类型 | 触发时机 | 记录内容 |
|-----|---------|---------|---------|
| 1 | `DOCUMENT_DELETED` | 删除文档时 | `type: 'SOFT'` 或 `'HARD'` |

### 5.3 审计日志数据结构

审计日志统一通过 `createDocumentAuditLogData` 工具函数创建：

```typescript
{
  type: DOCUMENT_AUDIT_LOG_TYPE,
  envelopeId: string,
  userId: number | null,
  email: string | null,
  name: string | null,
  userAgent: string | null,  // 来自 requestMetadata
  ipAddress: string | null,   // 来自 requestMetadata
  data: { /* 事件特定数据 */ }
}
```

---

## 六、两类反向流程的协作对比

| 维度 | 拒签（Reject） | 撤回（Recall/Delete） |
|-----|--------------|---------------------|
| **触发方** | 收件人（签署人） | 文档所有者/团队成员 |
| **状态变更** | PENDING → REJECTED（通过 Seal） | PENDING/DRAFT → 物理删除<br>COMPLETED → 软删除 |
| **状态回退范围** | 先更新单个收件人状态，再异步更新整体文档状态 | 直接更新文档状态（删除） |
| **邮件通知** | 3 类邮件：拒签人确认、所有者通知、其他收件人取消 | 1 类邮件：所有收件人取消通知 |
| **审计日志** | 2 条：收件人拒签 + 文档完成（标记拒签） | 1 条：文档删除 |
| **Webhook** | `DOCUMENT_REJECTED` | `DOCUMENT_CANCELLED` |
| **PDF 处理** | 生成带拒签章的 PDF，文件名 `_rejected.pdf` | 不生成新 PDF |
| **事务边界** | 收件人状态 + 审计日志 原子事务 | 审计日志 + 文档删除 原子事务 |
| **异步处理** | 触发 3 个 Job：seal + 两类邮件 | 同步发送邮件 + Webhook |

---

## 七、关键协作节点

### 7.1 拒签的异步 Seal 触发

```
用户点击拒签
    ↓
[同步事务] 更新收件人状态 → 记录审计日志
    ↓
[异步触发] seal-document Job
    ↓
  检查所有收件人状态 → 发现有人拒签
    ↓
  更新文档整体状态为 REJECTED
    ↓
  生成带拒签章的 PDF
    ↓
  触发 DOCUMENT_REJECTED Webhook
```

### 7.2 邮件通知的复用设计

- `send.document.cancelled.emails` Job 被**两个流程复用**：
  1. 拒签流程：通知其他收件人文档因拒签而终止
  2. 撤回流程：通知所有收件人文档已被删除

---

## 八、代码文件索引

| 功能 | 文件路径 |
|-----|---------|
| 拒签入口 | `packages/lib/server-only/document/reject-document-with-token.ts` |
| 撤回/删除入口 | `packages/lib/server-only/document/delete-document.ts` |
| 文档密封 | `packages/lib/jobs/definitions/internal/seal-document.handler.ts` |
| 拒签邮件 | `packages/lib/jobs/definitions/emails/send-rejection-emails.handler.ts` |
| 取消邮件 | `packages/lib/jobs/definitions/emails/send-document-cancelled-emails.handler.ts` |
| 审计日志工具 | `packages/lib/utils/document-audit-logs.ts` |
| 审计日志类型 | `packages/lib/types/document-audit-logs.ts` |
| 状态枚举 | `packages/prisma/schema.prisma:334` |
