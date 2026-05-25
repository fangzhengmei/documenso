# 签署流程角色分析文档

## 概述

本文档分析了 Documenso 签署流程中不同收件人角色（审批人、见证人和普通签署人）的角色判定、签署顺序以及完成条件的差异。

**重要说明**：通过代码分析发现，系统中**不存在 WITNESS（见证人）** 角色。当前系统支持的角色包括：`SIGNER`（签署人）、`APPROVER`（审批人）、`VIEWER`（查看人）、`CC`（抄送人）、`ASSISTANT`（助手）。下文将详细分析这些角色的差异。

---

## 1. 角色定义与判定

### 1.1 角色枚举定义

**文件位置**：`packages/prisma/schema.prisma:573-579`

```prisma
enum RecipientRole {
  CC
  SIGNER
  VIEWER
  APPROVER
  ASSISTANT
}
```

### 1.2 角色描述与行为动词

**文件位置**：`packages/lib/constants/recipient-roles.ts:5-116`

| 角色 | 动作动词 | 完成状态 |  progressive | 显示类型 | 邮件类型 |
|------|---------|---------|-------------|---------|---------|
| SIGNER | Sign | Signed | Signing | SIGNING_REQUEST | SIGNING_REQUEST |
| APPROVER | Approve | Approved | Approving | APPROVE_REQUEST | APPROVE_REQUEST |
| VIEWER | View | Viewed | Viewing | VIEW_REQUEST | - |
| CC | CC | CC'd | CC | - | - |
| ASSISTANT | Assist | Assisted | Assisting | - | ASSISTING_REQUEST |

### 1.3 角色字段需求判定

**文件位置**：`packages/lib/utils/recipients.ts:16-38`

```typescript
export const RECIPIENT_ROLES_THAT_REQUIRE_FIELDS = [RecipientRole.SIGNER] as const;

export const getRecipientsWithMissingFields = (recipients, fields) => {
  return recipients.filter((recipient) => {
    if (recipient.role === RecipientRole.SIGNER) {
      const hasSignatureField = fields.some(
        (field) => field.recipientId === recipient.id && isSignatureFieldType(field.type),
      );
      return !hasSignatureField;
    }
    return false;
  });
};
```

**关键差异**：
- **SIGNER**：必须至少有一个签名字段，否则无法发送文档
- **APPROVER/VIEWER/ASSISTANT**：不需要签名字段
- **CC**：不需要任何字段

### 1.4 角色修改权限判定

**文件位置**：`packages/lib/utils/recipients.ts:45-87`

```typescript
// 收件人是否可修改
export const canRecipientBeModified = (recipient, fields) => {
  if (recipient.role === RecipientRole.CC) return true; // CC始终可修改
  if (recipient.signingStatus === SigningStatus.SIGNED) return false;
  if (fields.some((field) => field.recipientId === recipient.id && field.inserted)) return false;
  return true;
};

// 收件人字段是否可修改
export const canRecipientFieldsBeModified = (recipient, fields) => {
  if (!canRecipientBeModified(recipient, fields)) return false;
  return recipient.role !== RecipientRole.VIEWER && recipient.role !== RecipientRole.CC;
};
```

**关键差异**：
- **CC**：始终可修改（除非文档已完成）
- **VIEWER/CC**：不能修改字段
- **其他角色**：未签署且未插入字段时可修改

---

## 2. 签署顺序逻辑

### 2.1 签署顺序模式

**文件位置**：`packages/prisma/schema.prisma:492-495`

```prisma
enum DocumentSigningOrder {
  PARALLEL
  SEQUENTIAL
}
```

### 2.2 顺序签署 - 轮到判定

**文件位置**：`packages/lib/server-only/recipient/get-is-recipient-turn.ts:8-47`

```typescript
export async function getIsRecipientsTurnToSign({ token }) {
  const envelope = await prisma.envelope.findFirstOrThrow({
    where: { type: EnvelopeType.DOCUMENT, recipients: { some: { token } } },
    include: {
      documentMeta: true,
      recipients: { orderBy: { signingOrder: 'asc' } },
    },
  });

  // 并行签署：所有人都可以随时签署
  if (envelope.documentMeta?.signingOrder !== DocumentSigningOrder.SEQUENTIAL) {
    return true;
  }

  // 顺序签署：检查前面所有收件人是否已签署
  const currentRecipientIndex = recipients.findIndex((r) => r.token === token);
  for (let i = 0; i < currentRecipientIndex; i++) {
    if (recipients[i].signingStatus !== SigningStatus.SIGNED) {
      return false;
    }
  }

  return true;
}
```

### 2.3 发送文档时的通知策略

**文件位置**：`packages/lib/server-only/document/send-document.ts:98-111`

```typescript
const signingOrder = envelope.documentMeta?.signingOrder || DocumentSigningOrder.PARALLEL;

let recipientsToNotify = envelope.recipients;

if (signingOrder === DocumentSigningOrder.SEQUENTIAL) {
  // 顺序签署：只通知第一个未签署且非CC的收件人
  recipientsToNotify = envelope.recipients
    .filter((r) => r.signingStatus === SigningStatus.NOT_SIGNED && r.role !== RecipientRole.CC)
    .slice(0, 1);
}

// 发送邮件时再次过滤CC
if (sendEmail || (isRecipientSigningRequestEmailEnabled && sendEmail === undefined)) {
  await Promise.all(
    recipientsToNotify.map(async (recipient) => {
      if (recipient.sendStatus === SendStatus.SENT || recipient.role === RecipientRole.CC) {
        return;
      }
      // 发送签署请求邮件...
    }),
  );
}
```

**关键差异**：
- **并行签署**：所有非CC角色同时收到通知，可同时签署
- **顺序签署**：按`signingOrder`排序，只有第一个未签署的非CC收件人收到通知
- **CC角色**：始终被排除在签署通知之外，只在文档完成时收到副本

### 2.4 下一个待处理收件人

**文件位置**：`packages/lib/server-only/recipient/get-next-pending-recipient.ts:6-42`

```typescript
export const getNextPendingRecipient = async ({ documentId, currentRecipientId }) => {
  const recipients = await prisma.recipient.findMany({
    where: { envelope: { type: EnvelopeType.DOCUMENT, secondaryId: mapDocumentIdToSecondaryId(documentId) } },
    orderBy: [
      { signingOrder: { sort: 'asc', nulls: 'last' } },
      { id: 'asc' },
    ],
  });

  const currentIndex = recipients.findIndex((r) => r.id === currentRecipientId);
  if (currentIndex === -1 || currentIndex === recipients.length - 1) {
    return null;
  }

  return recipients[currentIndex + 1];
};
```

---

## 3. 完成条件差异

### 3.1 签署状态枚举

**文件位置**：`packages/prisma/schema.prisma:567-571`

```prisma
enum SigningStatus {
  NOT_SIGNED
  SIGNED
  REJECTED
}
```

### 3.2 单个收件人完成条件

**文件位置**：`packages/lib/server-only/document/complete-document-with-token.ts:275-277`

```typescript
// 检查是否有未签署的必填字段
if (fieldsContainUnsignedRequiredField(fields)) {
  throw new Error(`Recipient ${recipient.id} has unsigned fields`);
}

// 更新状态为已签署
await tx.recipient.update({
  where: { id: recipient.id },
  data: {
    signingStatus: SigningStatus.SIGNED,
    signedAt: new Date(),
    name: recipientName,
    email: recipientEmail,
  },
});
```

**关键差异**：
- **SIGNER**：必须完成所有必填字段（包括签名字段）
- **APPROVER/VIEWER**：只需标记为SIGNED，无需签名字段
- **CC**：无需任何操作，不参与签署流程

### 3.3 待处理收件人判定

**文件位置**：`packages/lib/server-only/document/complete-document-with-token.ts:369-389`

```typescript
const pendingRecipients = await prisma.recipient.findMany({
  select: { id: true, signingOrder: true, name: true, email: true, role: true },
  where: {
    envelopeId: envelope.id,
    signingStatus: { not: SigningStatus.SIGNED },
    role: { not: RecipientRole.CC }, // 排除CC角色
  },
  orderBy: [{ signingOrder: { sort: 'asc', nulls: 'last' } }, { id: 'asc' }],
});
```

### 3.4 文档整体完成条件

**文件位置**：`packages/lib/server-only/document/complete-document-with-token.ts:457-476`

```typescript
const haveAllRecipientsSigned = await prisma.envelope.findFirst({
  where: {
    id: envelope.id,
    recipients: {
      every: {
        OR: [
          { signingStatus: SigningStatus.SIGNED },
          { role: RecipientRole.CC }, // CC角色自动满足
        ],
      },
    },
  },
});

if (haveAllRecipientsSigned) {
  await jobs.triggerJob({
    name: 'internal.seal-document',
    payload: { documentId: legacyDocumentId, requestMetadata },
  });
}
```

**核心逻辑**：文档完成 = 所有非CC收件人状态为SIGNED

### 3.5 发送时的快速完成判定

**文件位置**：`packages/lib/server-only/document/send-document.ts:156-179`

```typescript
const allRecipientsHaveNoActionToTake = envelope.recipients.every(
  (recipient) => recipient.role === RecipientRole.CC || recipient.signingStatus === SigningStatus.SIGNED,
);

if (allRecipientsHaveNoActionToTake) {
  // 直接封存文档，无需发送签署请求
  await jobs.triggerJob({
    name: 'internal.seal-document',
    payload: { documentId: legacyDocumentId, requestMetadata: requestMetadata?.requestMetadata },
  });
}
```

### 3.6 过期时间设置

**文件位置**：`packages/lib/server-only/document/send-document.ts:247-267`

```typescript
const expiresAt = resolveExpiresAt(envelope.documentMeta?.envelopeExpirationPeriod ?? null);

if (expiresAt) {
  await tx.recipient.updateMany({
    where: {
      envelopeId: envelope.id,
      signingStatus: { notIn: [SigningStatus.SIGNED, SigningStatus.REJECTED] },
      role: { not: RecipientRole.CC }, // CC不设置过期时间
    },
    data: {
      expiresAt,
      expirationNotifiedAt: null,
    },
  });
}
```

**关键差异**：
- **非CC角色**：设置过期时间，超时无法签署
- **CC角色**：不设置过期时间

---

## 4. 特殊角色处理

### 4.1 ASSISTANT（助手）角色

**文件位置**：`packages/lib/server-only/field/sign-field-with-token.ts:69-81`

```typescript
const field = await prisma.field.findFirstOrThrow({
  where: {
    id: fieldId,
    recipient: {
      // 非助手：只能签署自己的字段
      ...(recipient.role !== RecipientRole.ASSISTANT
        ? { id: recipient.id }
        : {
            // 助手：可以签署 signingOrder >= 自己signingOrder 的其他收件人的字段
            signingStatus: { not: SigningStatus.SIGNED },
            signingOrder: { gte: recipient.signingOrder ?? 0 },
            envelopeId: recipient.envelopeId,
          }),
    },
  },
  include: { envelope: { include: { recipients: true } }, recipient: true },
});
```

**助手特性**：
- 可以替其他收件人预填字段
- 只能替`signingOrder >= 自己signingOrder`的收件人填写
- 只能替未签署的收件人填写

### 4.2 签署时的审计日志区分

**文件位置**：`packages/lib/server-only/field/sign-field-with-token.ts:270-308`

```typescript
const assistant = recipient.role === RecipientRole.ASSISTANT ? recipient : undefined;

await tx.documentAuditLog.create({
  data: createDocumentAuditLogData({
    type:
      assistant && field.recipientId !== assistant.id
        ? DOCUMENT_AUDIT_LOG_TYPE.DOCUMENT_FIELD_PREFILLED // 助手预填
        : DOCUMENT_AUDIT_LOG_TYPE.DOCUMENT_FIELD_INSERTED, // 本人签署
    // ...
  }),
});
```

---

## 5. 角色行为对比表

| 维度 | SIGNER | APPROVER | VIEWER | CC | ASSISTANT |
|------|--------|----------|--------|----|-----------|
| **需要签名字段** | ✅ 是 | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 |
| **需要标记SIGNED** | ✅ 是 | ✅ 是 | ✅ 是 | ❌ 否 | ✅ 是 |
| **参与顺序签署** | ✅ 是 | ✅ 是 | ✅ 是 | ❌ 否 | ✅ 是 |
| **接收签署通知** | ✅ 是 | ✅ 是 | ✅ 是 | ❌ 否 | ✅ 是 |
| **设置过期时间** | ✅ 是 | ✅ 是 | ✅ 是 | ❌ 否 | ✅ 是 |
| **可修改字段** | ✅ 未签署时 | ✅ 未签署时 | ❌ 否 | ❌ 否 | ✅ 未签署时 |
| **可修改收件人信息** | ✅ 未签署时 | ✅ 未签署时 | ✅ 未签署时 | ✅ 始终 | ✅ 未签署时 |
| **可替他人签署** | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | ✅ 是（有条件） |
| **文档完成条件** | 需SIGNED | 需SIGNED | 需SIGNED | 自动满足 | 需SIGNED |
| **邮件类型** | SIGNING_REQUEST | APPROVE_REQUEST | - | - | ASSISTING_REQUEST |
| **显示类型** | SIGNING_REQUEST | APPROVE_REQUEST | VIEW_REQUEST | - | - |

---

## 6. 关键代码位置汇总

| 功能 | 文件路径 |
|------|---------|
| 角色枚举定义 | `packages/prisma/schema.prisma:573-579` |
| 角色描述配置 | `packages/lib/constants/recipient-roles.ts` |
| 字段需求判定 | `packages/lib/utils/recipients.ts:16-38` |
| 修改权限判定 | `packages/lib/utils/recipients.ts:45-87` |
| 轮到签署判定 | `packages/lib/server-only/recipient/get-is-recipient-turn.ts` |
| 下一个收件人 | `packages/lib/server-only/recipient/get-next-pending-recipient.ts` |
| 完成签署逻辑 | `packages/lib/server-only/document/complete-document-with-token.ts` |
| 发送文档逻辑 | `packages/lib/server-only/document/send-document.ts` |
| 签署字段逻辑 | `packages/lib/server-only/field/sign-field-with-token.ts` |
| 获取签署信封 | `packages/lib/server-only/envelope/get-envelope-for-recipient-signing.ts` |

---

## 7. 容易混淆的点

### 7.1 审批人 vs 签署人
- **共同点**：都需要标记为SIGNED，都参与签署顺序，都设置过期时间
- **不同点**：签署人必须有签名字段，审批人不需要；邮件类型和显示类型不同

### 7.2 查看人 vs 抄送人
- **共同点**：都不需要签名字段
- **不同点**：查看人需要标记为SIGNED才算完成，抄送人自动满足；查看人参与签署顺序，抄送人不参与

### 7.3 助手角色的特殊性
- 助手可以替其他人签署字段，但有`signingOrder`限制
- 助手自己也需要完成签署（标记为SIGNED）

### 7.4 CC角色的"隐形"特性
- CC角色在很多逻辑中被显式排除（`role: { not: RecipientRole.CC }`）
- CC角色不参与签署流程，但会收到最终文档
- CC角色的修改权限最宽松

---

## 8. 关于"见证人"角色的说明

通过全面代码搜索，**当前代码库中不存在 WITNESS（见证人）角色**。用户提到的"见证人"可能是：
1. 未来计划添加的角色
2. 业务上对"审批人"或"查看人"的另一种称呼
3. 其他文档系统中的概念映射

如果需要添加见证人角色，需要：
1. 在`RecipientRole`枚举中添加`WITNESS`
2. 在`RECIPIENT_ROLES_DESCRIPTION`中添加对应的描述
3. 在各处角色过滤逻辑中考虑是否需要排除/包含WITNESS
4. 确定见证人的字段需求、完成条件和签署顺序行为
