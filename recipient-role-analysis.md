# 签署流程角色分析文档

## 概述

本文档分析了 Documenso 签署流程中不同收件人角色（审批人、见证人和普通签署人）的角色判定、签署顺序以及完成条件的差异。

**重要修正**：通过深入代码分析，系统中**不存在独立的 WITNESS 枚举值**，但 **VIEWER（查看人）角色在业务场景中被明确用作见证人（Witness）**。

官方文档证据：`apps/docs/content/docs/users/documents/add-recipients.mdx:95`
> "Contract with witness: Add the main signer plus a viewer as witness."

当前系统支持的角色包括：`SIGNER`（签署人）、`APPROVER`（审批人）、`VIEWER`（查看人/见证人）、`CC`（抄送人）、`ASSISTANT`（助手）。

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

**文件位置**：`packages/lib/constants/recipient-roles.ts:5-137`

| 角色 | 动作动词 | 完成状态 | progressive | 显示类型 | 邮件类型 | 签署理由 |
|------|---------|---------|-------------|---------|---------|---------|
| SIGNER | Sign | Signed | Signing | SIGNING_REQUEST | SIGNING_REQUEST | I am a signer of this document |
| APPROVER | Approve | Approved | Approving | APPROVE_REQUEST | APPROVE_REQUEST | I am an approver of this document |
| VIEWER | View | Viewed | Viewing | VIEW_REQUEST | VIEW_REQUEST | I am a viewer of this document |
| CC | CC | CC'd | CC | - | - | I am required to receive a copy of this document |
| ASSISTANT | Assist | Assisted | Assisting | - | ASSISTING_REQUEST | I am an assistant of this document |

**关键常量定义**：
```typescript
// packages/lib/constants/recipient-roles.ts:118-137
export const RECIPIENT_ROLE_TO_DISPLAY_TYPE = {
  [RecipientRole.SIGNER]: `SIGNING_REQUEST`,
  [RecipientRole.VIEWER]: `VIEW_REQUEST`,
  [RecipientRole.APPROVER]: `APPROVE_REQUEST`,
} as const;

export const RECIPIENT_ROLE_TO_EMAIL_TYPE = {
  [RecipientRole.SIGNER]: `SIGNING_REQUEST`,
  [RecipientRole.VIEWER]: `VIEW_REQUEST`,
  [RecipientRole.APPROVER]: `APPROVE_REQUEST`,
  [RecipientRole.ASSISTANT]: `ASSISTING_REQUEST`,
} as const;

export const RECIPIENT_ROLE_SIGNING_REASONS = {
  [RecipientRole.SIGNER]: msg`I am a signer of this document`,
  [RecipientRole.APPROVER]: msg`I am an approver of this document`,
  [RecipientRole.VIEWER]: msg`I am a viewer of this document`,
  [RecipientRole.CC]: msg`I am required to receive a copy of this document`,
  [RecipientRole.ASSISTANT]: msg`I am an assistant of this document`,
}
```

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
| 角色 | 签名字段 | 其他字段 | 发送前验证 |
|------|---------|---------|-----------|
| SIGNER | ✅ 必须至少1个 | 可选 | 验证签名字段存在 |
| APPROVER | ❌ 不需要 | 可选 | 不验证 |
| VIEWER | ❌ 不需要 | 不可分配 | 不验证 |
| CC | ❌ 不需要 | 不可分配 | 不验证 |
| ASSISTANT | ❌ 不需要 | 可选（预填） | 不验证 |

**前端字段分配过滤**：
```typescript
// packages/ui/primitives/template-flow/add-template-fields.tsx:471-472
// packages/ui/primitives/recipient-selector.tsx:53-54
return (Object.entries(recipientsByRole) as [RecipientRole, TRecipientLite[]][]).filter(
  ([role]) => role !== RecipientRole.CC && role !== RecipientRole.VIEWER && role !== RecipientRole.ASSISTANT,
);
```
→ **VIEWER 在字段分配UI中被过滤，不能分配任何字段**

### 1.4 前端签署表单的角色区分

**文件位置**：`apps/remix/app/components/general/envelope-signing/envelope-signer-form.tsx:37-129`

```typescript
// VIEWER 角色：不显示任何表单字段
if (recipient.role === RecipientRole.VIEWER) {
  return null;
}

// ASSISTANT 角色：显示助手特有的字段选择界面
if (recipient.role === RecipientRole.ASSISTANT) {
  return (
    // 助手可以选择为其他收件人预填字段
    <RadioGroup>
      {assistantRecipients.filter((r) => r.fields.length > 0).map(...)}
    </RadioGroup>
  );
}

// SIGNER / APPROVER 角色：显示姓名和签名字段
return (
  <fieldset>
    <div>
      <Label>Full Name</Label>
      <Input />
    </div>
    {hasSignatureField && (
      <div>
        <Label>Signature</Label>
        <SignaturePadDialog />
      </div>
    )}
  </fieldset>
);
```

**表单显示差异**：
| 角色 | 姓名字段 | 签名字段 | 助手选择器 | 说明 |
|------|---------|---------|-----------|------|
| SIGNER | ✅ | ✅（如有） | ❌ | 需要填写姓名和签名 |
| APPROVER | ✅ | ❌（通常无） | ❌ | 只需填写姓名（通常无字段） |
| VIEWER | ❌ | ❌ | ❌ | 不显示任何表单，直接点击完成 |
| ASSISTANT | ✅ | ✅（如有） | ✅ | 可以选择为其他收件人预填 |
| CC | - | - | - | 不进入签署页面 |

### 1.5 完成页面的角色区分

**文件位置**：`apps/remix/app/routes/_recipient+/sign.$token+/complete.tsx:184-188`

```typescript
<h2 className="mt-6 max-w-[35ch] text-center font-semibold text-2xl leading-normal md:text-3xl lg:text-4xl">
  {recipient.role === RecipientRole.SIGNER && <Trans>Document Signed</Trans>}
  {recipient.role === RecipientRole.VIEWER && <Trans>Document Viewed</Trans>}
  {recipient.role === RecipientRole.APPROVER && <Trans>Document Approved</Trans>}
</h2>
```

→ **完成页面根据角色显示不同的成功标题**

### 1.6 角色修改权限判定

**文件位置**：`packages/lib/utils/recipients.ts:45-87`

```typescript
export const canRecipientBeModified = (recipient, fields) => {
  if (recipient.role === RecipientRole.CC) return true; // CC始终可修改
  if (recipient.signingStatus === SigningStatus.SIGNED) return false;
  if (fields.some((field) => field.recipientId === recipient.id && field.inserted)) return false;
  return true;
};

export const canRecipientFieldsBeModified = (recipient, fields) => {
  if (!canRecipientBeModified(recipient, fields)) return false;
  return recipient.role !== RecipientRole.VIEWER && recipient.role !== RecipientRole.CC;
};
```

**修改权限矩阵**：
| 角色 | 可修改收件人信息 | 可修改字段 | 条件 |
|------|----------------|-----------|------|
| SIGNER | ✅ | ✅ | 未签署且未插入字段 |
| APPROVER | ✅ | ✅ | 未签署且未插入字段 |
| VIEWER | ✅ | ❌ | 未签署 |
| CC | ✅ 始终 | ❌ | - |
| ASSISTANT | ✅ | ✅ | 未签署且未插入字段 |

### 1.5 前端角色选择UI

**文件位置**：`packages/ui/components/recipient/recipient-role-select.tsx:30-88`

| 角色 | 图标 | 显示文本 | 工具提示 |
|------|------|---------|---------|
| SIGNER | PencilLine | Needs to sign | "The recipient is required to sign the document for it to be completed." |
| APPROVER | BadgeCheck | Needs to approve | "The recipient is required to approve the document for it to be completed." |
| VIEWER | Eye | Needs to view | "The recipient is required to view the document for it to be completed." |
| CC | Copy | Receives copy | "The recipient is not required to take any action and receives a copy of the document after it is completed." |
| ASSISTANT | User | Can prepare | "The recipient can prepare the document for later signers by pre-filling suggest values." |

---

## 2. 签署顺序与流程推进

### 2.1 签署顺序模式

**文件位置**：`packages/prisma/schema.prisma:492-495`

```prisma
enum DocumentSigningOrder {
  PARALLEL
  SEQUENTIAL
}
```

### 2.2 流程推进完整路径（三角色串联）

```
发送文档 (send-document.ts)
    ↓
┌───────────────────────────────────────────────────────────────────┐
│  阶段1：发送通知策略                                              │
├───────────────────────────────────────────────────────────────────┤
│  🔹 并行签署 (PARALLEL)                                           │
│  ├─ 收件人过滤：排除 CC                                           │
│  ├─ SIGNER、APPROVER、VIEWER、ASSISTANT 同时收到通知              │
│  └─ 邮件类型：根据角色区分                                         │
│                                                                   │
│  🔹 顺序签署 (SEQUENTIAL)                                         │
│  ├─ 按 signingOrder 排序                                          │
│  ├─ 过滤：NOT_SIGNED 且 非 CC                                     │
│  ├─ 只取第一个收件人发送通知                                      │
│  └─ 后续收件人等待前面完成                                         │
│                                                                   │
│  📌 代码分流点：send-document.ts:100-111                          │
│     if (signingOrder === SEQUENTIAL) {                            │
│       recipientsToNotify = envelope.recipients                    │
│         .filter(r => r.signingStatus === NOT_SIGNED               │
│                    && r.role !== CC)                              │
│         .slice(0, 1);  // 只取第一个                              │
│     }                                                             │
└───────────────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────────────┐
│  阶段2：轮到判定 (get-is-recipient-turn.ts)                       │
├───────────────────────────────────────────────────────────────────┤
│  🔹 并行签署：返回 true（所有人随时可签署）                         │
│                                                                   │
│  🔹 顺序签署：                                                    │
│  ├─ 找到当前收件人索引                                             │
│  ├─ 检查前面所有收件人（不区分角色）是否都为 SIGNED                │
│  ├─ 全部 SIGNED → 返回 true                                       │
│  └─ 存在 NOT_SIGNED → 返回 false                                  │
│                                                                   │
│  📌 关键逻辑：三角色在轮到判定中完全平等，都能阻塞/被阻塞          │
│     for (let i = 0; i < currentRecipientIndex; i++) {             │
│       if (recipients[i].signingStatus !== SIGNED) {               │
│         return false;  // 任何角色未完成都阻塞                    │
│       }                                                           │
│     }                                                             │
└───────────────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────────────┐
│  阶段3：邮件发送 (send-signing-email.handler.ts)                  │
├───────────────────────────────────────────────────────────────────┤
│  🔹 步骤1：排除 CC 角色 (line 74-76)                              │
│     if (recipient.role === CC) return;                            │
│                                                                   │
│  🔹 步骤2：获取邮件类型 (line 98)                                  │
│     const recipientEmailType = RECIPIENT_ROLE_TO_EMAIL_TYPE[      │
│       recipient.role                                              │
│     ];                                                            │
│                                                                   │
│  🔹 步骤3：动态生成邮件主题 (line 105-108)                         │
│     const recipientActionVerb =                                   │
│       i18n._(RECIPIENT_ROLES_DESCRIPTION[recipient.role]          │
│         .actionVerb).toLowerCase();                               │
│     emailSubject = `Please ${recipientActionVerb} this document`  │
│                                                                   │
│  📌 三角色邮件差异：                                              │
│     SIGNER   → 主题：Please sign this document                    │
│     APPROVER → 主题：Please approve this document                 │
│     VIEWER   → 主题：Please view this document                    │
└───────────────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────────────┐
│  阶段4：过期时间设置 (send-document.ts:247-267)                   │
├───────────────────────────────────────────────────────────────────┤
│  🔹 排除条件：                                                    │
│  ├─ 角色为 CC                                                     │
│  └─ 状态为 SIGNED 或 REJECTED                                     │
│                                                                   │
│  🔹 设置过期：SIGNER、APPROVER、VIEWER、ASSISTANT 都设置过期       │
│                                                                   │
│  📌 代码：                                                        │
│     where: {                                                      │
│       signingStatus: { notIn: [SIGNED, REJECTED] },               │
│       role: { not: CC },  // 只排除CC                             │
│     }                                                             │
└───────────────────────────────────────────────────────────────────┘
```

### 2.3 三角色通知顺序对比

| 签署模式 | SIGNER | APPROVER | VIEWER (见证人) |
|---------|--------|----------|----------------|
| **并行签署** | 同时收到通知 | 同时收到通知 | 同时收到通知 |
| **顺序签署** | 按 signingOrder 依次收到 | 按 signingOrder 依次收到 | 按 signingOrder 依次收到 |

**关键结论**：
- 三种角色在通知顺序上完全平等，只由 `signingOrder` 决定
- 三种角色在轮到判定上完全平等，都能阻塞后续流程
- 只有 CC 角色被排除在通知和流程之外

### 2.4 轮到签署判定逻辑

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
  // 注意：这里不区分角色，只要是在前面的收件人（包括 SIGNER、APPROVER、VIEWER、ASSISTANT）
  // 都必须为 SIGNED 状态
  const currentRecipientIndex = recipients.findIndex((r) => r.token === token);
  for (let i = 0; i < currentRecipientIndex; i++) {
    if (recipients[i].signingStatus !== SigningStatus.SIGNED) {
      return false;
    }
  }

  return true;
}
```

**关键点**：顺序签署中，VIEWER（见证人）的完成状态会**阻塞**后续收件人。

### 2.4 发送时的快速完成判定

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

**逻辑**：如果所有非CC收件人都已 SIGNED，发送时直接完成文档。

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

### 3.2 单个收件人完成路径（三角色串联）

```
点击完成按钮 (前端)
    ↓
┌───────────────────────────────────────────────────────────────────┐
│  步骤1：基础校验 (complete-document-with-token.ts:84-115)        │
├───────────────────────────────────────────────────────────────────┤
│  ✅ 文档状态为 PENDING                                            │
│  ✅ 收件人存在                                                    │
│  ✅ 未过期                                                        │
│  ✅ 未签署、未拒绝                                                │
│  ✅ 是当前收件人轮到（顺序签署时）                                 │
│                                                                   │
│  📌 三角色无差异，统一校验                                        │
└───────────────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────────────┐
│  步骤2：字段校验 (complete-document-with-token.ts:275)           │
├───────────────────────────────────────────────────────────────────┤
│  调用 fieldsContainUnsignedRequiredField(fields)                      │
│                                                                   │
│  🔹 SIGNER：                                                    │
│  ├─ 签名字段（SIGNATURE、FREE_SIGNATURE）→ 始终必填                │
│  ├─ 其他高级字段（TEXT、NUMBER等）→ 根据 fieldMeta.required 判断  │
│  └─ 有未签署必填字段 → 抛出错误                                    │
│                                                                   │
│  🔹 APPROVER：                                                  │
│  ├─ 无签名字段（因为不能分配）                                   │
│  ├─ 如有其他高级字段，按 required 判断                           │
│  └─ 通常无字段 → 校验通过                                        │
│                                                                   │
│  🔹 VIEWER（见证人）：                                            │
│  ├─ 无任何字段（UI中被过滤，见 1.4 节）                             │
│  ├─ fields 数组为空 → fieldsContainUnsignedRequiredField 返回 true       │
│  └─ 直接通过校验                                                │
│                                                                   │
│  📌 代码验证：envelope-signer-form.tsx:37-39                        │
│     if (recipient.role === RecipientRole.VIEWER) {                 │
│       return null;  // VIEWER 不显示任何表单字段                    │
│     }                                                             │
└───────────────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────────────┐
│  步骤3：更新状态 (complete-document-with-token.ts:279-290)          │
├───────────────────────────────────────────────────────────────────┤
│  📌 所有角色统一更新为：                                        │
│     signingStatus: SigningStatus.SIGNED                             │
│     signedAt: new Date()                                          │
│                                                                   │
│  🔹 三角色无差异，统一使用 SIGNED 状态                              │
│  🔹 审计日志记录 recipientRole 用于后续审计追踪                          │
└───────────────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────────────┐
│  步骤4：流程推进 (complete-document-with-token.ts:369-455)         │
├───────────────────────────────────────────────────────────────────┤
│  🔹 查找待处理收件人：                                           │
│  ├─ 排除：SIGNED 状态                                           │
│  ├─ 排除：CC 角色（不排除 VIEWER）                                 │
│  ├─ 排序：signingOrder ASC, id ASC                                │
│                                                                   │
│  🔹 顺序签署时：                                                 │
│  ├─ 取下一个待处理收件人（pendingRecipients[0]）                   │
│  ├─ 更新其 sendStatus 为 SENT                                     │
│  └─ 发送签署请求邮件（根据角色决定邮件类型）                       │
│                                                                   │
│  🔹 并行签署时：                                                 │
│  └─ 仅发送"等待其他人完成"邮件给当前收件人                         │
│                                                                   │
│  📌 代码分流点：complete-document-with-token.ts:382-384              │
│     role: {                                                      │
│       not: RecipientRole.CC,  // 只排除CC，包含VIEWER            │
│     },                                                             │
└───────────────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────────────┐
│  步骤5：文档完成判定 (complete-document-with-token.ts:457-476)     │
├───────────────────────────────────────────────────────────────────┤
│  检查所有收件人：                                               │
│  every( recipient =>                                            │
│    recipient.signingStatus === SIGNED ||                          │
│    recipient.role === CC                                        │
│  )                                                               │
│                                                                   │
│  📌 三角色都需要 SIGNED，只有 CC 自动满足                          │
│  📌 VIEWER（见证人）必须 SIGNED 才能完成文档                      │
│                                                                   │
│  条件满足 → 触发 internal.seal-document 任务                    │
└───────────────────────────────────────────────────────────────────┘
```

### 3.2.1 三角色完成动作对比

| 步骤 | SIGNER | APPROVER | VIEWER (见证人) |
|-----|--------|----------|----------------|
| **基础校验** | 统一校验 | 统一校验 | 统一校验 |
| **字段校验** | 需检查签名字段+必填字段 | 需检查必填字段（通常无） | 无字段，直接通过 |
| **状态更新** | signingStatus = SIGNED | signingStatus = SIGNED | signingStatus = SIGNED |
| **signedAt** | 记录时间 | 记录时间 | 记录时间 |
| **流程推进** | 触发下一个收件人 | 触发下一个收件人 | 触发下一个收件人 |
| **文档完成条件** | 需 SIGNED | 需 SIGNED | 需 SIGNED |
| **完成页面标题** | Document Signed | Document Approved | Document Viewed |

**关键差异点**：
1. **字段校验**：只有 SIGNER 必须有签名字段，VIEWER 无任何字段
2. **前端表单**：VIEWER 不显示任何表单，直接点击完成
3. **完成页面**：显示不同的成功标题

### 3.2.2 前端完成按钮的角色感知

**文件位置**：`apps/remix/app/routes/_recipient+/sign.$token+/complete.tsx:184-188`

```typescript
// 根据角色显示不同的完成标题
{recipient.role === RecipientRole.SIGNER && <Trans>Document Signed</Trans>}
{recipient.role === RecipientRole.VIEWER && <Trans>Document Viewed</Trans>}
{recipient.role === RecipientRole.APPROVER && <Trans>Document Approved</Trans>}
```

→ **前端通过 `recipient.role` 字段实现差异化展示

### 3.3 必填字段判定逻辑

**文件位置**：`packages/lib/utils/advanced-fields-helpers.ts:18-39`

```typescript
export const ADVANCED_FIELD_TYPES_WITH_OPTIONAL_SETTING: FieldType[] = [
  FieldType.NUMBER,
  FieldType.TEXT,
  FieldType.DROPDOWN,
  FieldType.RADIO,
  FieldType.CHECKBOX,
];

export const isRequiredField = (field: Field) => {
  // 所有不在"可选设置列表"中的字段都默认为必填
  // 包括：SIGNATURE、FREE_SIGNATURE、DATE、EMAIL、NAME、INITIALS
  if (!ADVANCED_FIELD_TYPES_WITH_OPTIONAL_SETTING.includes(field.type)) {
    return true;
  }

  // 高级字段根据 fieldMeta.required 判断
  if (!field.fieldMeta) return false;
  const parsedData = ZFieldMetaSchema.safeParse(field.fieldMeta);
  if (!parsedData.success) return false;
  return parsedData.data?.required === true;
};
```

**各角色实际必填字段**：
| 角色 | 可能拥有的字段 | 必填字段 | 完成条件 |
|------|-------------|---------|---------|
| SIGNER | SIGNATURE, FREE_SIGNATURE, DATE, EMAIL, NAME, INITIALS, 高级字段 | SIGNATURE/FREE_SIGNATURE（始终必填）+ 标记为required的高级字段 | 所有必填字段已插入 + 状态SIGNED |
| APPROVER | DATE, EMAIL, NAME, INITIALS, 高级字段 | 同左（但通常不分配这些字段） | 所有必填字段已插入 + 状态SIGNED |
| VIEWER | 无（UI中被过滤） | 无 | 仅需状态SIGNED |
| ASSISTANT | 同APPROVER | 同APPROVER | 所有必填字段已插入 + 状态SIGNED |
| CC | 无 | 无 | 无需操作，自动满足 |

### 3.4 待处理收件人判定

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

### 3.5 文档整体完成条件

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

**核心公式**：
```
文档完成 = (SIGNER₁.status = SIGNED) AND 
          (SIGNER₂.status = SIGNED) AND 
          ...
          (APPROVER₁.status = SIGNED) AND 
          ...
          (VIEWER₁.status = SIGNED) AND  // 见证人必须确认查看
          ...
          (ASSISTANT₁.status = SIGNED) AND
          ...
          // CC角色自动满足，无需检查
```

---

## 4. 典型场景流程示例

### 4.1 场景1：带见证人的合同签署（并行）

**配置**：
- 收件人A：SIGNER（order 1）
- 收件人B：VIEWER（见证人，order 1）
- 收件人C：CC

**流程**：
```
发送文档
    ↓
A 和 B 同时收到通知
    ↓
A 签署合同（需要签名字段）→ 状态 SIGNED
B 查看文档（只需点击完成）→ 状态 SIGNED
    ↓
检查：A.SIGNED AND B.SIGNED → true
    ↓
文档完成，C 收到副本
```

### 4.2 场景2：带见证人的合同签署（顺序）

**配置**：
- 收件人A：APPROVER（order 1）
- 收件人B：SIGNER（order 2）
- 收件人C：VIEWER（见证人，order 3）

**流程**：
```
发送文档
    ↓
只有 A 收到通知（顺序签署，第一个）
    ↓
A 审批通过 → 状态 SIGNED
    ↓
检查轮到 B：前面只有 A，已 SIGNED → 是
B 收到通知，签署合同 → 状态 SIGNED
    ↓
检查轮到 C：前面 A、B 都已 SIGNED → 是
C 收到通知，查看文档 → 状态 SIGNED
    ↓
检查：A.SIGNED AND B.SIGNED AND C.SIGNED → true
    ↓
文档完成
```

**关键**：VIEWER（见证人）在顺序签署中会阻塞流程，必须等前面的人完成，也会阻塞后面的人。

### 4.3 场景3：审批工作流

**配置**（来自官方文档示例）：
- Admin Assistant：ASSISTANT（order 1）
- Department Head：APPROVER（order 2）
- Contract Party A：SIGNER（order 3）
- Contract Party B：SIGNER（order 3）
- Legal Team：CC

**流程**：
```
发送文档 → Assistant 预填字段 → SIGNED
    ↓
Department Head 审批 → SIGNED
    ↓
Party A 和 Party B 同时签署 → SIGNED
    ↓
所有人 SIGNED → 文档完成 → Legal Team 收到副本
```

---

## 5. 特殊角色处理

### 5.1 ASSISTANT（助手）角色

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
- 只能替 `signingOrder >= 自己signingOrder` 的收件人填写
- 只能替未签署的收件人填写
- 自己也需要完成签署（标记为SIGNED）

### 5.2 签署时的审计日志区分

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

## 6. 角色行为对比表

| 维度 | SIGNER | APPROVER | VIEWER (见证人) | CC | ASSISTANT |
|------|--------|----------|----------------|----|-----------|
| **需要签名字段** | ✅ 是 | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 |
| **可分配字段** | ✅ 所有类型 | ✅ 除签名外 | ❌ 不可分配 | ❌ 不可分配 | ✅ 除签名外 |
| **需要标记SIGNED** | ✅ 是 | ✅ 是 | ✅ 是 | ❌ 否 | ✅ 是 |
| **参与顺序签署** | ✅ 是 | ✅ 是 | ✅ 是 | ❌ 否 | ✅ 是 |
| **接收签署通知** | ✅ 是 | ✅ 是 | ✅ 是 | ❌ 否 | ✅ 是 |
| **设置过期时间** | ✅ 是 | ✅ 是 | ✅ 是 | ❌ 否 | ✅ 是 |
| **可修改字段** | ✅ 未签署时 | ✅ 未签署时 | ❌ 否 | ❌ 否 | ✅ 未签署时 |
| **可修改收件人信息** | ✅ 未签署时 | ✅ 未签署时 | ✅ 未签署时 | ✅ 始终 | ✅ 未签署时 |
| **可替他人签署** | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | ✅ 是（有条件） |
| **文档完成条件** | 需SIGNED | 需SIGNED | 需SIGNED | 自动满足 | 需SIGNED |
| **邮件类型** | SIGNING_REQUEST | APPROVE_REQUEST | VIEW_REQUEST | - | ASSISTING_REQUEST |
| **显示类型** | SIGNING_REQUEST | APPROVE_REQUEST | VIEW_REQUEST | - | - |
| **图标** | PencilLine | BadgeCheck | Eye | Copy | User |
| **动作动词** | Sign | Approve | View | CC | Assist |
| **完成状态词** | Signed | Approved | Viewed | CC'd | Assisted |

---

## 7. 关键代码位置汇总

| 功能 | 文件路径 |
|------|---------|
| 角色枚举定义 | `packages/prisma/schema.prisma:573-579` |
| 角色描述配置 | `packages/lib/constants/recipient-roles.ts` |
| 字段需求判定 | `packages/lib/utils/recipients.ts:16-38` |
| 修改权限判定 | `packages/lib/utils/recipients.ts:45-87` |
| 必填字段判定 | `packages/lib/utils/advanced-fields-helpers.ts:18-39` |
| 轮到签署判定 | `packages/lib/server-only/recipient/get-is-recipient-turn.ts` |
| 下一个收件人 | `packages/lib/server-only/recipient/get-next-pending-recipient.ts` |
| 完成签署逻辑 | `packages/lib/server-only/document/complete-document-with-token.ts` |
| 发送文档逻辑 | `packages/lib/server-only/document/send-document.ts` |
| 签署字段逻辑 | `packages/lib/server-only/field/sign-field-with-token.ts` |
| 获取签署信封 | `packages/lib/server-only/envelope/get-envelope-for-recipient-signing.ts` |
| 发送邮件逻辑 | `packages/lib/jobs/definitions/emails/send-signing-email.handler.ts` |
| 角色选择UI | `packages/ui/components/recipient/recipient-role-select.tsx` |
| 角色图标定义 | `packages/ui/primitives/recipient-role-icons.tsx` |
| 字段分配UI过滤 | `packages/ui/primitives/recipient-selector.tsx:53-54` |
| 官方文档说明 | `apps/docs/content/docs/users/documents/add-recipients.mdx:95` |

---

## 8. 容易混淆的点

### 8.1 审批人 (APPROVER) vs 签署人 (SIGNER)

**共同点**：
- 都需要标记为 SIGNED
- 都参与签署顺序（顺序签署时会阻塞/被阻塞）
- 都设置过期时间
- 都接收签署通知邮件

**不同点**：
| 对比项 | SIGNER | APPROVER |
|--------|--------|----------|
| 签名字段 | ✅ 必须有 | ❌ 不需要 |
| 字段分配 | 可分配所有类型 | 不能分配签名字段 |
| 邮件类型 | SIGNING_REQUEST | APPROVE_REQUEST |
| 显示类型 | SIGNING_REQUEST | APPROVE_REQUEST |
| 动作动词 | Sign | Approve |
| 完成状态 | Signed | Approved |
| UI图标 | PencilLine | BadgeCheck |
| 签署理由 | "I am a signer..." | "I am an approver..." |

**代码中的分流点**：
```typescript
// 1. 发送前校验：只有 SIGNER 检查签名字段
// packages/lib/utils/recipients.ts:28-33
if (recipient.role === RecipientRole.SIGNER) {
  const hasSignatureField = fields.some(...);
  return !hasSignatureField;
}

// 2. 邮件类型区分
// packages/lib/constants/recipient-roles.ts:124-128
[RecipientRole.SIGNER]: `SIGNING_REQUEST`,
[RecipientRole.APPROVER]: `APPROVE_REQUEST`,
```

### 8.2 查看人/见证人 (VIEWER) vs 抄送人 (CC)

**共同点**：
- 都不需要签名字段
- 都不能分配字段（UI中被过滤）

**不同点**：
| 对比项 | VIEWER (见证人) | CC |
|--------|----------------|----|
| 需要SIGNED | ✅ 是 | ❌ 否 |
| 参与顺序签署 | ✅ 是 | ❌ 否 |
| 接收签署通知 | ✅ 是 | ❌ 否 |
| 设置过期时间 | ✅ 是 | ❌ 否 |
| 阻塞流程 | ✅ 是 | ❌ 否 |
| 可修改字段 | ❌ 否 | ❌ 否 |
| 可修改收件人信息 | ✅ 未签署时 | ✅ 始终 |
| 邮件类型 | VIEW_REQUEST | - |
| 显示类型 | VIEW_REQUEST | - |
| 动作动词 | View | CC |
| 完成状态 | Viewed | CC'd |
| UI图标 | Eye | Copy |
| 签署理由 | "I am a viewer..." | "I am required to receive a copy..." |

**代码中的分流点**：
```typescript
// 1. 待处理收件人：排除 CC，但包含 VIEWER
// packages/lib/server-only/document/complete-document-with-token.ts:382-384
role: {
  not: RecipientRole.CC, // 只排除CC，不排除VIEWER
},

// 2. 文档完成条件：CC自动满足，VIEWER需要SIGNED
// packages/lib/server-only/document/complete-document-with-token.ts:462
OR: [
  { signingStatus: SigningStatus.SIGNED },
  { role: RecipientRole.CC }, // 只有CC自动满足
],

// 3. 顺序签署轮到判定：VIEWER参与顺序检查，CC不参与
// （因为CC不在recipientsToNotify列表中）
```

### 8.3 VIEWER 为什么是见证人

**证据链**：
1. **官方文档明确说明**：`apps/docs/content/docs/users/documents/add-recipients.mdx:95`
   > "Contract with witness: Add the main signer plus a viewer as witness."

2. **角色描述匹配**：
   - VIEWER 的 tooltip："The recipient is required to view the document for it to be completed."
   - 见证人的核心职责就是确认已查看文档

3. **行为逻辑匹配**：
   - 不需要签署（无签名字段）
   - 需要确认查看（标记为SIGNED）
   - 可以在流程中作为必要环节（顺序签署时会阻塞）
   - 接收查看通知邮件（VIEW_REQUEST）

4. **完成条件正确**：
   - 文档完成需要 VIEWER 标记为 SIGNED
   - 确保见证人确实确认过文档

### 8.4 CC角色的"隐形"特性

CC角色在很多逻辑中被显式排除（`role: { not: RecipientRole.CC }`）：

| 排除位置 | 代码位置 | 影响 |
|---------|---------|------|
| 待处理收件人 | `complete-document-with-token.ts:382-384` | 不参与流程推进 |
| 文档完成条件 | `complete-document-with-token.ts:462-463` | 自动满足完成条件 |
| 顺序签署通知 | `send-document.ts:145-148` | 不接收签署通知 |
| 邮件发送 | `send-signing-email.handler.ts:74-76` | 不发送签署邮件 |
| 过期时间设置 | `send-document.ts:258-260` | 不设置过期时间 |
| 快速完成判定 | `send-document.ts:285-287` | 自动算作"无需操作" |
| 字段分配UI | `recipient-selector.tsx:53-54` | 不能分配字段 |

### 8.5 为什么 VIEWER 也需要标记为 SIGNED

这是系统设计的关键洞察：

1. **状态一致性**：所有参与流程的角色（SIGNER、APPROVER、VIEWER、ASSISTANT）都使用相同的状态机（NOT_SIGNED → SIGNED），简化了逻辑判断
2. **明确的确认点**：VIEWER 必须主动点击"完成"按钮，而不是打开邮件就算完成，确保"见证"是主动行为
3. **审计追踪**：`signedAt` 字段记录了见证人确认的时间点，可用于审计
4. **流程控制**：在顺序签署中，VIEWER 的 SIGNED 状态可以正确阻塞/解锁后续流程

---

## 9. 流程验证点（可核实）

要验证各角色的行为差异，可以检查以下代码点：

### 9.1 验证 VIEWER = 见证人
- ✅ 检查 `apps/docs/content/docs/users/documents/add-recipients.mdx:95` 的文档说明
- ✅ 检查 VIEWER 的签署理由："I am a viewer of this document"
- ✅ 检查 VIEWER 不能分配字段：`recipient-selector.tsx:53-54`

### 9.2 验证角色分流逻辑
- ✅ 发送前只验证 SIGNER 的签名字段：`recipients.ts:28-33`
- ✅ 轮到判定不区分角色，只看状态：`get-is-recipient-turn.ts:124-129`
- ✅ 完成条件只排除 CC，不排除 VIEWER：`complete-document-with-token.ts:462-463`

### 9.3 验证完成条件差异
- ✅ SIGNER 必须有签名字段：`recipients.ts:28-33`
- ✅ VIEWER 无字段检查即可完成：`advanced-fields-helpers.ts:18-39`（无字段即返回true）
- ✅ 文档完成需要 VIEWER 为 SIGNED：`complete-document-with-token.ts:461-463`

### 9.4 验证邮件类型区分
- ✅ 检查 `RECIPIENT_ROLE_TO_EMAIL_TYPE` 映射：`recipient-roles.ts:124-129`
- ✅ 检查邮件发送时的角色过滤：`send-signing-email.handler.ts:74-76`
- ✅ 检查邮件主题动态生成：`send-signing-email.handler.ts:105-108`

---

## 10. 关于"WITNESS"角色的最终结论

### 10.1 现状总结

1. **没有独立的 WITNESS 枚举值**：`RecipientRole` 枚举中不存在 WITNESS
2. **VIEWER 就是业务上的见证人**：官方文档明确说明在"Contract with witness"场景下使用 VIEWER 作为见证人
3. **VIEWER 的行为完全符合见证人角色**：
   - 无需签署（无签名字段）
   - 需要确认查看（标记为SIGNED）
   - 接收查看通知（VIEW_REQUEST邮件）
   - 参与签署顺序（可阻塞流程）
   - 可作为合同签署的必要环节

### 10.2 与普通签署人的核心差异

| 对比维度 | SIGNER (签署人) | APPROVER (审批人) | VIEWER (见证人) |
|---------|----------------|------------------|----------------|
| **核心动作** | 签署文档 | 审批通过 | 见证查看 |
| **签名字段** | ✅ 必须有 | ❌ 不需要 | ❌ 不需要 |
| **字段分配** | 所有类型 | 除签名外 | ❌ 不可分配 |
| **完成条件** | 所有必填字段已插入 + SIGNED | 所有必填字段已插入 + SIGNED | 只需 SIGNED |
| **邮件类型** | SIGNING_REQUEST | APPROVE_REQUEST | VIEW_REQUEST |
| **显示类型** | SIGNING_REQUEST | APPROVE_REQUEST | VIEW_REQUEST |
| **签署理由** | "I am a signer" | "I am an approver" | "I am a viewer" |
| **阻塞流程** | ✅ 是 | ✅ 是 | ✅ 是 |

### 10.3 如需要独立 WITNESS 角色的改造点

如果未来需要添加独立的 WITNESS 枚举值，需要修改以下位置：

1. **枚举定义**：`packages/prisma/schema.prisma:573-579`
   ```prisma
   enum RecipientRole {
     CC
     SIGNER
     VIEWER
     WITNESS  // 新增
     APPROVER
     ASSISTANT
   }
   ```

2. **角色描述**：`packages/lib/constants/recipient-roles.ts`
   - 添加 `RECIPIENT_ROLES_DESCRIPTION[RecipientRole.WITNESS]`
   - 添加 `RECIPIENT_ROLE_TO_DISPLAY_TYPE` 映射
   - 添加 `RECIPIENT_ROLE_TO_EMAIL_TYPE` 映射
   - 添加 `RECIPIENT_ROLE_SIGNING_REASONS` 条目

3. **字段需求判定**：`packages/lib/utils/recipients.ts:16-38`
   - 确认 WITNESS 是否需要字段

4. **修改权限判定**：`packages/lib/utils/recipients.ts:45-87`
   - 确认 WITNESS 的修改权限

5. **字段分配UI过滤**：`packages/ui/primitives/recipient-selector.tsx:53-54`
   - 确认是否要在字段分配中过滤 WITNESS

6. **各处角色过滤**：搜索所有 `role !== RecipientRole.CC` 或 `role: { not: RecipientRole.CC }` 的位置
   - 确认 WITNESS 是否应该被包含/排除

7. **前端角色选择**：`packages/ui/components/recipient/recipient-role-select.tsx`
   - 添加 WITNESS 选项
   - 添加图标和描述

8. **角色图标**：`packages/ui/primitives/recipient-role-icons.tsx`
   - 添加 WITNESS 对应的图标

9. **数据库迁移**：创建 Prisma migration 添加枚举值

---

## 11. 完整可核实流程总结

### 11.1 三角色端到端流程对比表

| 流程阶段 | 判定点 | SIGNER | APPROVER | VIEWER (见证人) | 代码验证位置 |
|---------|-------|--------|----------|----------------|-------------|
| **1. 角色定义** | 枚举值 | `SIGNER` | `APPROVER` | `VIEWER` | schema.prisma:573-579 |
| | 动作动词 | Sign | Approve | View | recipient-roles.ts:5-137 |
| | 邮件类型 | `SIGNING_REQUEST` | `APPROVE_REQUEST` | `VIEW_REQUEST` | recipient-roles.ts:124-129 |
| | 签署理由 | "I am a signer" | "I am an approver" | "I am a viewer" | recipient-roles.ts:131-137 |
| **2. 发送前校验** | 签名字段要求 | ✅ 必须有 | ❌ 不需要 | ❌ 不需要 | recipients.ts:28-33 |
| | 字段分配UI | ✅ 可分配所有 | ✅ 除签名外 | ❌ 被过滤 | recipient-selector.tsx:53-54 |
| **3. 发送通知** | 并行签署 | 同时收到 | 同时收到 | 同时收到 | send-document.ts:100-111 |
| | 顺序签署 | 按 order 依次 | 按 order 依次 | 按 order 依次 | send-document.ts:100-111 |
| | 邮件主题 | "Please sign..." | "Please approve..." | "Please view..." | send-signing-email.handler.ts:105-108 |
| | 设置过期 | ✅ | ✅ | ✅ | send-document.ts:258-260 |
| **4. 轮到判定** | 顺序签署检查 | 检查前面所有 | 检查前面所有 | 检查前面所有 | get-is-recipient-turn.ts:40-44 |
| | 阻塞能力 | ✅ 可阻塞 | ✅ 可阻塞 | ✅ 可阻塞 | get-is-recipient-turn.ts:40-44 |
| **5. 签署页面** | 表单显示 | 姓名+签名 | 姓名（通常无字段） | ❌ 无表单 | envelope-signer-form.tsx:37-129 |
| | 完成按钮 | "Sign" | "Approve" | "View" | 前端动态按钮 |
| **6. 完成动作** | 字段校验 | 签名字段+必填 | 必填字段（通常无） | 无字段 | complete-document-with-token.ts:275 |
| | 状态更新 | `SIGNED` | `SIGNED` | `SIGNED` | complete-document-with-token.ts:285 |
| | 审计日志 | 记录 role | 记录 role | 记录 role | complete-document-with-token.ts:342 |
| **7. 流程推进** | 待处理收件人 | 包含 | 包含 | 包含 | complete-document-with-token.ts:382-384 |
| | 触发下一个 | ✅ | ✅ | ✅ | complete-document-with-token.ts:394-454 |
| **8. 文档完成** | 个人完成条件 | 字段完成+SIGNED | 字段完成+SIGNED | 只需 SIGNED | 见 3.2.1 节 |
| | 文档完成条件 | 需 SIGNED | 需 SIGNED | 需 SIGNED | complete-document-with-token.ts:462-463 |
| | 完成页面标题 | Document Signed | Document Approved | Document Viewed | complete.tsx:184-188 |

### 11.2 可核实的代码检查清单

要验证三角色流程的正确性，可以按以下清单检查代码：

#### ✅ 验证 VIEWER = 见证人
1. [ ] 检查 `apps/docs/content/docs/users/documents/add-recipients.mdx:95` 的官方文档说明
2. [ ] 检查 `recipient-roles.ts:72-93` 中 VIEWER 的动作动词为 "View"
3. [ ] 检查 `recipient-selector.tsx:53-54` 中 VIEWER 在字段分配UI中被过滤
4. [ ] 检查 `envelope-signer-form.tsx:37-39` 中 VIEWER 不显示任何表单

#### ✅ 验证角色分流逻辑
1. [ ] 发送前只验证 SIGNER 的签名字段：`recipients.ts:28-33`
2. [ ] 轮到判定不区分角色，只看状态：`get-is-recipient-turn.ts:40-44`
3. [ ] 待处理收件人只排除 CC，不排除 VIEWER：`complete-document-with-token.ts:382-384`
4. [ ] 邮件类型根据角色区分：`send-signing-email.handler.ts:98`

#### ✅ 验证完成条件差异
1. [ ] SIGNER 必须有签名字段：`recipients.ts:28-33`
2. [ ] VIEWER 无字段检查即可完成：`complete-document-with-token.ts:275`（fields 为空）
3. [ ] 文档完成需要 VIEWER 为 SIGNED：`complete-document-with-token.ts:462-463`
4. [ ] 完成页面根据角色显示不同标题：`complete.tsx:184-188`

#### ✅ 验证流程推进逻辑
1. [ ] 并行签署：所有非 CC 同时收到通知：`send-document.ts:100-111`
2. [ ] 顺序签署：只通知第一个非 CC 非 SIGNED：`send-document.ts:104-106`
3. [ ] 完成后触发下一个收件人：`complete-document-with-token.ts:394-454`
4. [ ] 过期时间排除 CC，但包含 VIEWER：`send-document.ts:258-260`

### 11.3 典型场景的流程验证

#### 场景：带见证人的合同签署（顺序签署）
**配置**：APPROVER (order 1) → SIGNER (order 2) → VIEWER (order 3)

**验证步骤**：
1. [ ] 发送文档时，只有 APPROVER 收到通知（`send-document.ts:104-106`）
2. [ ] APPROVER 完成后，SIGNER 收到通知（`complete-document-with-token.ts:394-454`）
3. [ ] SIGNER 完成后，VIEWER 收到通知（`complete-document-with-token.ts:394-454`）
4. [ ] VIEWER 点击完成，无表单显示（`envelope-signer-form.tsx:37-39`）
5. [ ] VIEWER 完成后，文档状态变为 COMPLETED（`complete-document-with-token.ts:457-476`）
6. [ ] VIEWER 完成页面显示 "Document Viewed"（`complete.tsx:186`）

---

## 12. 关于"WITNESS"角色的最终结论（补充）

### 12.1 代码中 WITNESS 的搜索结果

通过全代码库搜索 `witness` / `WITNESS`（不区分大小写），只找到 2 个文件：
1. `recipient-role-analysis.md` - 本文档
2. `apps/docs/content/docs/users/documents/add-recipients.mdx:95` - 官方文档

**结论**：
- ❌ **代码中不存在独立的 WITNESS 枚举值**
- ❌ **代码中没有任何条件判断使用 witness 作为角色**
- ✅ **VIEWER 角色在业务场景中被明确用作见证人**

### 12.2 VIEWER 作为见证人的完整证据链

| 证据类型 | 证据位置 | 证据内容 |
|---------|---------|---------|
| 官方文档 | `add-recipients.mdx:95` | "Contract with witness: Add the main signer plus a viewer as witness." |
| 角色定义 | `recipient-roles.ts:72-93` | 动作动词 "View"，完成状态 "Viewed" |
| 签署理由 | `recipient-roles.ts:135` | "I am a viewer of this document" |
| 字段限制 | `recipient-selector.tsx:53-54` | VIEWER 在字段分配UI中被过滤 |
| 表单显示 | `envelope-signer-form.tsx:37-39` | VIEWER 不显示任何表单字段 |
| 流程参与 | `get-is-recipient-turn.ts:40-44` | VIEWER 参与顺序签署，可阻塞流程 |
| 完成条件 | `complete-document-with-token.ts:462-463` | VIEWER 必须 SIGNED 才能完成文档 |
| 邮件通知 | `send-signing-email.handler.ts:98` | VIEWER 接收 VIEW_REQUEST 邮件 |
| 完成页面 | `complete.tsx:186` | VIEWER 完成页面显示 "Document Viewed" |

### 12.3 三角色核心差异总览

| 维度 | SIGNER (签署人) | APPROVER (审批人) | VIEWER (见证人) |
|-----|----------------|------------------|----------------|
| **核心职责** | 签署文档，承担法律责任 | 审批通过，确认内容合规 | 见证签署过程，确认属实 |
| **核心动作** | 签名 + 填写字段 | 点击批准 | 点击确认查看 |
| **签名字段** | ✅ 必须有 | ❌ 不需要 | ❌ 不需要 |
| **字段分配** | 所有类型 | 除签名外 | ❌ 不可分配 |
| **前端表单** | 姓名 + 签名 | 姓名（通常无字段） | ❌ 无表单 |
| **完成条件** | 所有必填字段已插入 + SIGNED | 所有必填字段已插入 + SIGNED | 只需 SIGNED |
| **邮件类型** | SIGNING_REQUEST | APPROVE_REQUEST | VIEW_REQUEST |
| **邮件主题** | "Please sign this document" | "Please approve this document" | "Please view this document" |
| **完成标题** | Document Signed | Document Approved | Document Viewed |
| **阻塞流程** | ✅ 是 | ✅ 是 | ✅ 是 |
| **审计追踪** | 记录 signingStatus + signedAt | 记录 signingStatus + signedAt | 记录 signingStatus + signedAt |

### 12.4 关键设计洞察

1. **状态机复用**：所有参与流程的角色（SIGNER/APPROVER/VIEWER/ASSISTANT）都使用相同的 `SigningStatus` 状态机（NOT_SIGNED → SIGNED），大大简化了逻辑判断。

2. **角色分流点**：代码中主要通过三处实现角色分流：
   - **字段分配**：`recipient-selector.tsx` 过滤 VIEWER/CC/ASSISTANT
   - **邮件类型**：`RECIPIENT_ROLE_TO_EMAIL_TYPE` 映射决定邮件内容
   - **前端展示**：`envelope-signer-form.tsx` 和 `complete.tsx` 根据角色显示不同UI

3. **见证人设计**：VIEWER 作为见证人的设计非常巧妙——不需要签名字段，但必须主动点击完成，确保"见证"是一个主动确认行为，而不是被动接收。

4. **CC 的"隐形"特性**：CC 角色在所有关键逻辑点（`role !== RecipientRole.CC`）被显式排除，形成了"参与但不影响流程"的独特定位。
