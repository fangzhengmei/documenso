# 签名工作流编排

## 一、工作流概述

一份文档从准备到完成签名，经历以下阶段：

```
准备文档 → 发送文档 → 通知收件人 → 收件人签署 → 密封定稿 → 文档完成
```

---

## 二、状态定义

### 2.1 文档状态（DocumentStatus）

| 状态 | 说明 |
|------|------|
| DRAFT | 草稿：文档正在准备中，尚未发送 |
| PENDING | 待处理：文档已发送，等待收件人操作 |
| COMPLETED | 已完成：所有收件人已完成操作，文档已密封 |
| REJECTED | 已拒绝：审批人拒绝了文档，流程终止 |

定义位置：`packages/prisma/schema.prisma:334-339`

### 2.2 收件人签名状态（SigningStatus）

| 状态 | 说明 |
|------|------|
| NOT_SIGNED | 未签署：收件人尚未完成操作 |
| SIGNED | 已签署：收件人已完成签名/审批/查看/协助 |
| REJECTED | 已拒绝：审批人拒绝了文档 |

定义位置：`packages/prisma/schema.prisma:567-571`

### 2.3 签名顺序模式（DocumentSigningOrder）

| 模式 | 说明 |
|------|------|
| PARALLEL | 并行签名：所有收件人同时收到通知，可同时签署 |
| SEQUENTIAL | 顺序签名：收件人按指定顺序依次签署 |

定义位置：`packages/prisma/schema.prisma:492-495`

---

## 三、收件人角色

### 3.1 角色定义（RecipientRole）

| 角色 | 是否需要操作 | 是否可签名 | 说明 |
|------|-------------|-----------|------|
| SIGNER | 是 | 是 | 必须签署文档的核心参与方 |
| APPROVER | 是 | 可选 | 必须审批文档，签名可选 |
| VIEWER | 是 | 否 | 必须确认已查看文档 |
| ASSISTANT | 是 | 否 | 预填字段助手（仅顺序签名可用） |
| CC | 否 | 否 | 文档完成后收到副本 |

定义位置：`packages/prisma/schema.prisma:573-579`

### 3.2 各角色详细说明

#### 签名者（Signer）
- 必须完成所有分配的必填字段（至少一个签名字段）
- 签名方式：绘制签名、输入姓名选择字体、上传签名图片
- 完成后可下载包含已填写字段的文档副本

#### 审批人（Approver）
- 必须显式审批文档
- 可选择签名（如果分配了签名字段）
- 如果启用拒绝功能，拒绝后整个流程终止
- 文档无法继续推进，直到所有审批人审批通过

#### 查看者（Viewer）
- 必须确认已查看完整文档
- 无法添加签名或修改任何内容
- 适用于需要确认收到但无需签名的情况

#### 助手（Assistant）

**核心功能：**
- 为后续收件人预填字段值
- **可以提交完成**，提交后状态变为 `SIGNED`
- **仅在顺序签名模式下可用**

**可操作范围（字段权限）：**
```
┌─────────────────────────────────────────────────────────────┐
│  Assistant (Order=1) 可操作的字段：                          │
│                                                             │
│  1. 分配给自己的所有字段                                     │
│  2. 分配给后续收件人（signingOrder >= 自己的 Order）的字段   │
│     └── 且该收件人尚未 SIGNED                               │
│     └── 仅限非 SIGNATURE 类型（见下方详细对比）              │
│                                                             │
│  不可操作：                                                  │
│  ├── 前面顺序收件人的字段（Order < 自己的 Order）            │
│  ├── 已 SIGNED 收件人的字段                                  │
│  └── 后续收件人的 SIGNATURE 类型字段                         │
└─────────────────────────────────────────────────────────────┘
```

### Assistant 对三类签名字段的可操作性对比（V1 vs V2）

#### 签署页版本说明

代码位置：`apps/remix/app/routes/_recipient+\\sign.$token+\\_index.tsx:292-308`

```typescript
if (foundRecipient.envelope.internalVersion === 2) {
  // V2 版本：使用 get-envelope-for-recipient-signing + EnvelopeSigningProvider
  const payloadV2 = await handleV2Loader(loaderArgs);
} else {
  // V1 版本：使用 get-fields-for-token + DocumentSigningProvider
  const payloadV1 = await handleV1Loader(loaderArgs);
}
```

---

#### V1 链路分析（internalVersion !== 2）

**V1 执行链路：**
```
handleV1Loader
  └── get-fields-for-token (后端过滤)
        └── DocumentSigningPageViewV1
              └── document-signing-form.tsx
```

**1. 字段获取阶段：get-fields-for-token（后端过滤）**

代码位置：`packages/lib/server-only/field/get-fields-for-token.ts:22-53`

```typescript
if (recipient.role === RecipientRole.ASSISTANT) {
  return await prisma.field.findMany({
    where: {
      OR: [
        {
          // ⚠️ 只排除了 SIGNATURE，没有排除 FREE_SIGNATURE 和 INITIALS
          type: { not: FieldType.SIGNATURE },
          recipient: {
            signingStatus: { not: SigningStatus.SIGNED },
            signingOrder: { gte: recipient.signingOrder ?? 0 },
            envelopeId: recipient.envelopeId,
          },
          envelope: { id: recipient.envelopeId, type: EnvelopeType.DOCUMENT },
        },
        {
          recipientId: recipient.id,  // 自己的字段
        },
      ],
    },
  });
}
```

**V1 查询结果：**
| 字段类型 | 能否从 DB 查询到 | 原因 |
|----------|-----------------|------|
| **SIGNATURE** | ❌ 被过滤 | `type: { not: FieldType.SIGNATURE }` |
| **FREE_SIGNATURE** | ✅ 可查询到 | 不在过滤条件中 |
| **INITIALS** | ✅ 可查询到 | 不在过滤条件中 |

**2. 页面渲染阶段：DocumentSigningPageViewV1**

V1 页面使用的是旧版渲染逻辑（非 envelope-signer-page-renderer），主要通过 `document-signing-form.tsx` 处理。

代码位置：`apps/remix/app/components/general/document-signing/document-signing-form.tsx`

```typescript
// 字段来源：get-fields-for-token 返回的 fields 数组
const fieldsRequiringValidation = useMemo(() => fields.filter(isFieldUnsignedAndRequired), [fields]);

// hasSignatureField 只检查 SIGNATURE 和 FREE_SIGNATURE
const hasSignatureField = fields.some((field) => isSignatureFieldType(field.type));
```

**3. 后端签署阶段：sign-field-with-token**

代码位置：`packages/lib/server-only/field/sign-field-with-token.ts:190-205`

```typescript
const isSignatureField = field.type === FieldType.SIGNATURE || field.type === FieldType.FREE_SIGNATURE;

if (isSignatureField && !signatureImageAsBase64 && !typedSignature) {
  throw new Error('Signature field must have a signature');
}
```

**V1 各字段执行结果：**

| 字段类型 | get-fields-for-token | 页面可见 | 后端校验 | 实际可操作 |
|----------|---------------------|---------|---------|-----------|
| **SIGNATURE** | ❌ 被过滤 | ❌ 不可见 | N/A | **❌ 不可操作** |
| **FREE_SIGNATURE** | ✅ 可查询 | ✅ 可见（如果渲染支持） | ❌ 需要签名数据 | **❌ 不可操作** |
| **INITIALS** | ✅ 可查询 | ✅ 可见 | ✅ 作为 customText | **✅ 可操作** |

---

#### V2 链路分析（internalVersion === 2）

**V2 执行链路：**
```
handleV2Loader
  └── get-envelope-for-recipient-signing (返回所有字段)
        └── EnvelopeSigningProvider (前端过滤)
              └── EnvelopeSignerPageRenderer
                    └── ZFullFieldSchema.parse() + match().exhaustive()
                          └── signEnvelopeField (后端校验)
```

**1. 字段获取阶段：get-envelope-for-recipient-signing（返回所有字段）**

代码位置：`packages/lib/server-only/envelope/get-envelope-for-recipient-signing.ts:167-214`

```typescript
// ⚠️ 返回 envelope 的所有 recipient.fields，没有按字段类型过滤！
const envelope = await prisma.envelope.findFirst({
  where: { ... },
  include: {
    recipients: {
      include: {
        fields: {
          include: { signature: true },  // 所有字段都返回
        },
      },
      orderBy: { signingOrder: 'asc' },
    },
  },
});
```

**V2 查询结果：**
| 字段类型 | 能否从 DB 查询到 | 原因 |
|----------|-----------------|------|
| **SIGNATURE** | ✅ 可查询到 | 没有字段类型过滤 |
| **FREE_SIGNATURE** | ✅ 可查询到 | 没有字段类型过滤 |
| **INITIALS** | ✅ 可查询到 | 没有字段类型过滤 |

**2. 前端过滤阶段：EnvelopeSigningProvider（assistantFields）**

代码位置：`apps/remix/app/components/general/document-signing/envelope-signing-provider.tsx:241-272`

```typescript
// Assistant 可操作的收件人：signingOrder > 自己的 Order
const assistantRecipients = recipient.role === RecipientRole.ASSISTANT
  ? envelope.recipients.filter((r) => (r.signingOrder ?? 0) > (recipient.signingOrder ?? 0))
  : [];

// ⚠️ assistantFields：只排除 SIGNATURE，没有排除 FREE_SIGNATURE 和 INITIALS
const assistantFields = recipient.role === RecipientRole.ASSISTANT
  ? assistantRecipients
      .filter((r) => r.signingStatus !== SigningStatus.SIGNED)
      .flatMap((r) => r.fields.filter((field) => field.type !== FieldType.SIGNATURE))
  : [];

// selectedAssistantRecipientFields：显示在页面上的字段
const selectedAssistantRecipientFields = useMemo(() => {
  return assistantFields.filter((field) => field.recipientId === selectedAssistantRecipient?.id);
}, [recipientFields, selectedAssistantRecipient]);
```

**V2 前端过滤结果：**
| 字段类型 | assistantFields 包含 | 页面可见 | 原因 |
|----------|---------------------|---------|------|
| **SIGNATURE** | ❌ 被过滤 | ❌ 不可见 | `field.type !== FieldType.SIGNATURE` |
| **FREE_SIGNATURE** | ✅ 包含 | ✅ 可见 | 不在过滤条件中 |
| **INITIALS** | ✅ 包含 | ✅ 可见 | 不在过滤条件中 |

**3. 页面渲染阶段：EnvelopeSignerPageRenderer（ZFullFieldSchema）**

代码位置：`apps/remix/app/components/general/envelope-signing/envelope-signer-page-renderer.tsx:124-130, 195-393`

```typescript
const unsafeRenderFieldOnLayer = (unparsedField: Field & { signature?: Signature | null }) => {
  // ⚠️ ZFullFieldSchema 不包含 FREE_SIGNATURE！
  const fieldToRender = ZFullFieldSchema.parse(unparsedField);
  ...
  const parsedFoundField = ZFullFieldSchema.parse(foundField);
  // ⚠️ match().exhaustive() 也不包含 FREE_SIGNATURE！
  match(parsedFoundField)
    .with({ type: FieldType.SIGNATURE }, (field) => { ... })
    .with({ type: FieldType.INITIALS }, (field) => { ... })
    .exhaustive();
};
```

**ZFullFieldSchema 定义：** `packages/lib/types/field.ts:182-193`

```typescript
export const ZFullFieldSchema = z.discriminatedUnion('type', [
  ZFieldTextSchema,
  ZFieldSignatureSchema,    // type: z.literal(FieldType.SIGNATURE)
  ZFieldInitialsSchema,     // type: z.literal(FieldType.INITIALS)
  // ❌ 缺少 ZFieldFreeSignatureSchema！
]);

// 虽然定义了，但从未在 ZFullFieldSchema 中使用
export const ZFieldFreeSignatureSchema = ZFieldSignatureSchema;
```

**V2 渲染结果：**
| 字段类型 | ZFullFieldSchema.parse | match().exhaustive() | 渲染结果 |
|----------|----------------------|---------------------|---------|
| **SIGNATURE** | N/A（已被前端过滤） | N/A | ❌ 不可见 |
| **FREE_SIGNATURE** | ❌ parse 失败 | ❌ 无法到达 | ❌ 渲染错误 |
| **INITIALS** | ✅ 解析成功 | ✅ 匹配成功 | ✅ 正常渲染 |

**4. 后端签署阶段：signEnvelopeField（明确校验 SIGNATURE）**

代码位置：`packages/trpc/server/envelope-router/sign-envelope-field.ts:38-91`

```typescript
const field = await prisma.field.findFirst({
  where: {
    id: fieldId,
    recipient: {
      ...(recipient.role === RecipientRole.ASSISTANT
        ? {
            signingStatus: { not: SigningStatus.SIGNED },
            signingOrder: { gte: recipient.signingOrder ?? 0 },
            envelopeId: recipient.envelopeId,
          }
        : { id: recipient.id }),
    },
  },
  ...
});

// ⚠️ V2 明确禁止 Assistant 签署他人的 SIGNATURE 字段
if (
  field.type === FieldType.SIGNATURE &&
  recipient.id !== field.recipientId &&
  recipient.role === RecipientRole.ASSISTANT
) {
  throw new AppError(AppErrorCode.INVALID_REQUEST, {
    message: `Assistant recipients cannot sign signature fields`,
  });
}
```

**注意：** V2 后端校验只检查了 `SIGNATURE`，没有检查 `FREE_SIGNATURE`。

**V2 后端校验结果：**
| 字段类型 | 后端查询 | SIGNATURE 校验 | 实际可操作 |
|----------|---------|---------------|-----------|
| **SIGNATURE** | ✅ 可查询 | ❌ 被拦截 | **❌ 不可操作** |
| **FREE_SIGNATURE** | ✅ 可查询 | ❌ 没有校验（但无法到达） | **❌ 不可操作** |
| **INITIALS** | ✅ 可查询 | N/A（非签名类型） | **✅ 可操作** |

---

#### V1 vs V2 对比总结

| 阶段 | V1 行为 | V2 行为 | 差异 |
|------|--------|--------|------|
| **字段获取** | get-fields-for-token 后端过滤（只排除 SIGNATURE） | get-envelope-for-recipient-signing 返回所有字段 | V1 更早过滤 SIGNATURE |
| **前端过滤** | 无（后端已过滤） | EnvelopeSigningProvider 过滤（只排除 SIGNATURE） | V2 多了一层前端过滤 |
| **Schema 解析** | 无 ZFullFieldSchema | ZFullFieldSchema.parse() | V2 会拦截 FREE_SIGNATURE |
| **渲染器** | 旧版渲染 | envelope-signer-page-renderer + ZFullFieldSchema | V2 更严格 |
| **后端校验** | sign-field-with-token（检查 SIGNATURE + FREE_SIGNATURE） | signEnvelopeField（只检查 SIGNATURE） | V2 后端只显式拦截 SIGNATURE |

#### 三类字段在 V1/V2 中的最终可操作边界

**统一结论（V1 和 V2 最终一致）：**

| 字段类型 | V1 可操作 | V2 可操作 | 统一结论 | 关键拦截点 |
|----------|----------|----------|---------|-----------|
| **SIGNATURE** | ❌ | ❌ | **❌ 不可操作** | V1: get-fields-for-token 过滤；V2: assistantFields 过滤 + 后端显式拦截 |
| **FREE_SIGNATURE** | ❌ | ❌ | **❌ 不可操作（遗留功能）** | V1: 后端需要签名数据；V2: ZFullFieldSchema.parse() 失败 |
| **INITIALS** | ✅ | ✅ | **✅ 可操作** | 均被当作普通文本字段处理 |

**详细分析：**

**SIGNATURE 字段：**
```
┌─────────────────────────────────────────────────────────────────────┐
│  V1:                                                                │
│  get-fields-for-token: type: { not: SIGNATURE } → ❌ 被过滤         │
│  结果：页面不可见 → 不可操作                                        │
│                                                                     │
│  V2:                                                                │
│  get-envelope-for-recipient-signing: ✅ 返回所有字段                │
│  EnvelopeSigningProvider: field.type !== SIGNATURE → ❌ 被过滤     │
│  后端校验: Assistant cannot sign signature fields → ❌ 拦截        │
│  结果：页面不可见 → 不可操作                                        │
└─────────────────────────────────────────────────────────────────────┘
```

**FREE_SIGNATURE 字段（遗留/未完全实现功能）：**
```
┌─────────────────────────────────────────────────────────────────────┐
│  V1:                                                                │
│  get-fields-for-token: ✅ 可查询（只排除 SIGNATURE）                │
│  渲染：旧版渲染（如果能渲染）                                        │
│  后端: isSignatureField = SIGNATURE \|\| FREE_SIGNATURE            │
│       需要 signatureImageAsBase64 或 typedSignature → ❌ 失败     │
│  结果：可见但无法提交 → 不可操作                                    │
│                                                                     │
│  V2:                                                                │
│  get-envelope-for-recipient-signing: ✅ 返回所有字段                │
│  EnvelopeSigningProvider: ✅ 包含（只排除 SIGNATURE）               │
│  ZFullFieldSchema.parse(): ❌ 失败（不包含 FREE_SIGNATURE）        │
│  renderField(): ❌ throw 'Free signature fields are not supported' │
│  结果：渲染阶段失败 → 不可操作                                      │
│                                                                     │
│  结论：FREE_SIGNATURE 是遗留功能，V2 更严格地在渲染阶段就拦截了      │
└─────────────────────────────────────────────────────────────────────┘
```

**INITIALS 字段：**
```
┌─────────────────────────────────────────────────────────────────────┐
│  V1:                                                                │
│  get-fields-for-token: ✅ 可查询                                    │
│  渲染：旧版渲染 → ✅ 可见                                           │
│  后端: isSignatureField = false → customText 路径 → ✅ 通过       │
│  结果：可操作                                                       │
│                                                                     │
│  V2:                                                                │
│  get-envelope-for-recipient-signing: ✅ 返回所有字段                │
│  EnvelopeSigningProvider: ✅ 包含（只排除 SIGNATURE）               │
│  ZFullFieldSchema.parse(): ✅ 成功                                 │
│  match().with({ type: FieldType.INITIALS }, ...): ✅ 匹配         │
│  后端: 走 customText 路径 → ✅ 通过                                │
│  结果：可操作                                                       │
│                                                                     │
│  结论：INITIALS 在 V1/V2 中均完全可用，Assistant 可以为后续收件人    │
│  预填 Initials 字段                                                 │
└─────────────────────────────────────────────────────────────────────┘
```

#### 关键代码引用

**V1 链路：**
- `apps/remix/app/routes/_recipient+\\sign.$token+\\_index.tsx:45-168`：`handleV1Loader`
- `packages/lib/server-only/field/get-fields-for-token.ts:22-53`：字段获取（后端过滤）
- `packages/lib/server-only/field/sign-field-with-token.ts:190-205`：后端校验

**V2 链路：**
- `apps/remix/app/routes/_recipient+\\sign.$token+\\_index.tsx:170-260`：`handleV2Loader`
- `packages/lib/server-only/envelope/get-envelope-for-recipient-signing.ts:167-214`：字段获取（返回所有）
- `apps/remix/app/components/general/document-signing/envelope-signing-provider.tsx:241-272`：前端过滤（assistantFields）
- `packages/lib/types/field.ts:182-193`：`ZFullFieldSchema`（不包含 FREE_SIGNATURE）
- `apps/remix/app/components/general/envelope-signing/envelope-signer-page-renderer.tsx:124-130, 195-393`：页面渲染
- `packages/trpc/server/envelope-router/sign-envelope-field.ts:38-91`：后端校验

**通用：**
- `packages/lib/universal/field-renderer/render-field.ts:88-90`：`renderField` 抛出 'Free signature fields are not supported'

**Assistant 何时记为 SIGNED：**

```
Assistant 提交流程：

1. 完成自己的必填字段
   └── 检查：uninsertedRecipientFields（分配给自己的必填未填字段）必须为空

2. 在 UI 中选择要预填的后续收件人（RadioGroup 选择）
   └── 显示所有 signingOrder >= 自己 Order 且有字段的收件人列表
   └── 代码位置：apps/remix/app/components/general/document-signing/document-signing-form.tsx:171-203

3. 点击 Continue 按钮 → 打开确认对话框

4. 确认提交 → 调用 completeDocument()
   └── 后台将 signingStatus 设为 SIGNED
   └── 记录 signedAt 时间戳
   └── 代码位置：packages/lib/server-only/document/complete-document-with-token.ts:279-290
```

**Assistant 提交条件（前端校验）：**

```typescript
// 必须先完成自己的必填字段才能点击 Continue
const onAssistantFormSubmit = () => {
  if (uninsertedRecipientFields.length > 0) {
    return;  // 自己的必填字段未完成，不能提交
  }
  setIsConfirmationDialogOpen(true);
};
```

代码位置：`apps/remix/app/components/general/document-signing/document-signing-form.tsx:89-95`

**对流程推进的影响：**

```
┌─────────────────────────────────────────────────────────────┐
│  Assistant 提交后的流程推进：                                │
│                                                             │
│  1. Assistant 状态变为 SIGNED                               │
│                                                             │
│  2. 由于是顺序签名模式，触发下一阶段通知逻辑                  │
│     └── 代码：complete-document-with-token.ts:369-455       │
│                                                             │
│  3. 查找下一个待处理收件人（排除 CC）                        │
│     └── signingOrder 最低的 NOT_SIGNED 收件人               │
│                                                             │
│  4. 可选：如果启用 allowDictateNextSigner                    │
│     └── Assistant 可以指定下一个收件人的姓名/邮箱            │
│     └── 代码：complete-document-with-token.ts:398-441       │
│                                                             │
│  5. 发送签名请求邮件给下一阶段收件人                          │
│     └── 触发 send.signing.requested.email 任务              │
└─────────────────────────────────────────────────────────────┘
```

**适用场景：**
- 行政人员为高管准备文档（预填信息）
- 一个人收集信息，另一个人最终签名
- 减轻最终签署人的填写负担

#### 抄送（CC）
- 不参与签名流程
- 仅在文档完成后收到完整副本
- 不影响工作流推进条件

---

## 四、工作流编排

### 4.1 并行签名（默认模式）

**特点：** 所有收件人同时收到通知，可任意顺序完成操作。

```
        ┌─────────────────────────────────────────┐
        │            发送文档 (PENDING)           │
        └─────────────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Signer A    │  │  Signer B    │  │  Viewer C    │
│  可立即签名  │  │  可立即签名  │  │  可立即查看  │
└──────────────┘  └──────────────┘  └──────────────┘
        │                 │                 │
        └─────────────────┼─────────────────┘
                          ▼
        ┌─────────────────────────────────────────┐
        │         全部完成 → 密封文档             │
        └─────────────────────────────────────────┘
```

**适用场景：**
- 签署顺序无关紧要
- 追求最快完成时间
- 收件人之间相互独立

### 4.2 顺序签名

**特点：** 收件人按指定顺序依次签署，前一阶段全部完成后才通知下一阶段。

```
┌──────────────────────────────────────────────────┐
│              发送文档 (PENDING)                  │
│              仅通知 Order=1 的收件人              │
└──────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────┐
│  阶段 1: Order=1  (Assistant / Approver)        │
│  所有 Order=1 的收件人完成后，通知 Order=2       │
└──────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────┐
│  阶段 2: Order=2  (Approver / Signer)           │
│  所有 Order=2 的收件人完成后，通知 Order=3       │
└──────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────┐
│  阶段 3: Order=3  (Signer 可并行)                │
│  相同 Order 的收件人可同时操作                    │
└──────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────┐
│         全部阶段完成 → 密封文档                  │
└──────────────────────────────────────────────────┘
```

**关键规则（`packages/lib/server-only/recipient/get-is-recipient-turn.ts`）：**
- 按 `signingOrder` 字段排序收件人
- 收件人只能在轮到自己时才能签署
- 判断逻辑：检查该收件人之前的所有收件人是否都已 `SIGNED`
- 相同 `signingOrder` 的收件人在同一阶段并行操作

**适用场景：**
- 后续签署者需要看到前面的填写内容
- 审批必须在最终签名之前进行
- 公司政策要求特定的签署顺序

### 4.3 见证人（通过 Viewer 角色实现）

Documenso 没有专门的 Witness 角色，见证人功能通过 **Viewer 角色** 实现。

**使用方式：**
```
┌──────────────────────────────────────────────────┐
│  场景：合同需要主签方 + 见证人                    │
└──────────────────────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Signer A    │  │  Signer B    │  │  Viewer C    │
│  (主签方 1)  │  │  (主签方 2)  │  │  (见证人)    │
└──────────────┘  └──────────────┘  └──────────────┘
```

**见证人配置：**
- 角色设为 **Viewer**
- 必须确认已查看文档
- 不参与签名，但参与工作流完成条件
- 文档必须等见证人确认查看后才算完整完成

**在顺序签名中的见证人：**
```
Order | 收件人        | 角色      | 说明
------|---------------|-----------|-------------------
1     | 行政助理      | Assistant | 预填合同信息
2     | 部门主管      | Approver  | 审批合同
3     | 合同方A       | Signer    | 签署合同
3     | 合同方B       | Signer    | 签署合同
4     | 法务          | Viewer    | 见证（查看确认）
-     | 法务存档      | CC        | 存档副本
```

---

## 五、工作流推进条件

### 5.1 发送条件

文档从 DRAFT → PENDING 前必须满足：
- 所有 Signer 角色的收件人至少分配了一个签名字段
- 文档已上传

### 5.2 收件人操作权限（顺序签名）

收件人能否操作由 `getIsRecipientsTurnToSign` 函数判断（`packages/lib/server-only/recipient/get-is-recipient-turn.ts:40-46`）：

```typescript
for (let i = 0; i < currentRecipientIndex; i++) {
  if (recipients[i].signingStatus !== SigningStatus.SIGNED) {
    return false; // 前面有人没完成，不能操作
  }
}
return true;
```

**规则：**
1. 并行签名模式：所有人都可以操作
2. 顺序签名模式：只有当该收件人**之前的所有收件人**都已 `SIGNED` 时才能操作

### 5.3 阶段推进（顺序签名）

当前收件人完成后，系统检查：
1. 是否还有待处理的收件人（排除 CC）
2. 如果是顺序签名模式，自动通知下一个收件人

代码位置：`packages/lib/server-only/document/complete-document-with-token.ts:369-455`

```typescript
const pendingRecipients = await prisma.recipient.findMany({
  where: {
    envelopeId: envelope.id,
    signingStatus: { not: SigningStatus.SIGNED },
    role: { not: RecipientRole.CC },
  },
});

if (pendingRecipients.length > 0 && signingOrder === SEQUENTIAL) {
  // 通知下一个收件人
  await jobs.triggerJob({ name: 'send.signing.requested.email', ... });
}
```

### 5.4 文档完成条件

文档从 PENDING → COMPLETED 的条件：
- 所有非 CC 角色的收件人状态为 `SIGNED`

代码位置：`packages/lib/server-only/document/complete-document-with-token.ts:457-476`

```typescript
const haveAllRecipientsSigned = await prisma.envelope.findFirst({
  where: {
    id: envelope.id,
    recipients: {
      every: {
        OR: [{ signingStatus: SigningStatus.SIGNED }, { role: RecipientRole.CC }],
      },
    },
  },
});

if (haveAllRecipientsSigned) {
  await jobs.triggerJob({ name: 'internal.seal-document', ... });
}
```

**完成条件总结：**
| 条件 | 说明 |
|------|------|
| Signer | 必须完成所有签名字段 → SIGNED |
| Approver | 必须审批通过 → SIGNED |
| Viewer | 必须确认查看 → SIGNED |
| Assistant | 完成自己的必填字段后可提交 → SIGNED |
| CC | 无需操作，不影响完成条件 |

### 5.5 异常终止条件

文档从 PENDING → REJECTED 的条件：
- 任一 Approver 角色的收件人选择拒绝 → 整个流程立即终止
- 后续收件人不会收到通知

---

## 六、完整工作流示例

### 示例 1：合同审批 + 签署 + 见证

```
准备阶段 (DRAFT)
├── 上传合同 PDF
├── 设置收件人：
│   ├── Order=1: 财务部小王 (Approver)
│   ├── Order=2: 总经理 (Approver)
│   ├── Order=3: 供应商 (Signer)
│   ├── Order=3: 我方代表 (Signer)
│   ├── Order=4: 法务顾问 (Viewer - 见证人)
│   └── Order=-: 法务存档 (CC)
└── 发送文档 → 状态变为 PENDING

执行阶段 (PENDING)
├── 阶段 1：财务部小王审批通过 → SIGNED
├── 阶段 2：总经理审批通过 → SIGNED
├── 阶段 3：供应商 + 我方代表 可同时签署
│   ├── 供应商签署 → SIGNED
│   └── 我方代表签署 → SIGNED
└── 阶段 4：法务顾问查看确认 → SIGNED

完成阶段 (COMPLETED)
├── 所有非 CC 收件人已 SIGNED
├── 触发 internal.seal-document 任务
├── 文档密封，生成数字证书
└── 所有收件人（含 CC）收到完成文档
```

### 示例 2：Assistant 预填 + 顺序签

```
准备阶段 (DRAFT)
├── 上传合同 PDF
├── 启用顺序签名
├── 设置收件人：
│   ├── Order=1: 行政助理 (Assistant)
│   │   └── 分配字段：合同编号、日期、甲方信息
│   ├── Order=2: 部门主管 (Approver)
│   ├── Order=3: 总经理 (Signer)
│   │   └── 签名字段（Assistant 不可操作）
│   └── Order=-: 档案部 (CC)
└── 发送文档 → 状态变为 PENDING

执行阶段 (PENDING)
├── 阶段 1：行政助理操作
│   ├── 可操作字段：
│   │   ├── 自己的字段：合同编号、日期
│   │   ├── 部门主管的非签名字段（如果有）
│   │   └── 总经理的非签名字段（如甲方信息文本字段）
│   ├── 不可操作：总经理的签名字段
│   ├── 完成自己的必填字段
│   ├── 在 UI 中选择预填的收件人
│   ├── 点击 Continue → 确认 → 提交
│   └── 状态变为 SIGNED → 通知部门主管
│
├── 阶段 2：部门主管审批
│   ├── 看到助理预填的内容
│   ├── 审批通过 → SIGNED → 通知总经理
│
└── 阶段 3：总经理签署
    ├── 看到前面所有内容
    ├── 签署自己的签名字段
    └── 提交 → SIGNED

完成阶段 (COMPLETED)
├── 所有非 CC 收件人已 SIGNED
└── 文档密封完成
```

### 示例 3：并行签署

```
准备阶段 (DRAFT)
├── 上传协议 PDF
├── 设置收件人（并行签名）：
│   ├── 甲公司代表 (Signer)
│   ├── 乙公司代表 (Signer)
│   ├── 见证律师 (Viewer - 见证人)
│   └── 双方行政 (CC)
└── 发送文档 → 状态变为 PENDING

执行阶段 (PENDING)
├── 甲、乙、律师 同时收到通知
├── 甲签署 → SIGNED
├── 乙签署 → SIGNED
└── 律师确认查看 → SIGNED

完成阶段 (COMPLETED)
└── 全部完成，文档密封
```

---

## 七、相关代码位置

| 功能 | 文件位置 |
|------|----------|
| 数据库枚举定义 | `packages/prisma/schema.prisma` |
| 文档状态常量 | `packages/lib/constants/document.ts` |
| 收件人角色常量 | `packages/lib/constants/recipient-roles.ts` |
| 收件人轮次判断 | `packages/lib/server-only/recipient/get-is-recipient-turn.ts` |
| Assistant 可操作字段查询 | `packages/lib/server-only/field/get-fields-for-token.ts` |
| Assistant 签署字段逻辑 | `packages/lib/server-only/field/sign-field-with-token.ts` |
| Assistant 收件人列表查询 | `packages/lib/server-only/recipient/get-recipients-for-assistant.ts` |
| 完成文档逻辑（含阶段推进） | `packages/lib/server-only/document/complete-document-with-token.ts` |
| Assistant 前端提交流程 | `apps/remix/app/components/general/document-signing/document-signing-form.tsx` |
