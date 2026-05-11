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

### Assistant 对三类签名字段的可操作性对比

#### FREE_SIGNATURE 的真实执行路径与 ZFullFieldSchema 的影响

**关键发现：FREE_SIGNATURE 是一个遗留/未完全实现的功能。** 在签署页的实际执行路径中，FREE_SIGNATURE 会在更早的阶段就失败，而不是在后端校验阶段。

**完整执行链路逐段对齐：**

```
┌─────────────────────────────────────────────────────────────────────┐
│  字段获取阶段 (get-fields-for-token)                                │
│  packages/lib/server-only/field/get-fields-for-token.ts:27-29      │
├─────────────────────────────────────────────────────────────────────┤
│  type: { not: FieldType.SIGNATURE }                                │
│                                                                     │
│  ✅ SIGNATURE: ❌ 被过滤                                           │
│  ✅ FREE_SIGNATURE: ✅ 未被过滤，可从 DB 查询到                      │
│  ✅ INITIALS: ✅ 未被过滤                                          │
└─────────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│  页面渲染阶段 (ZFullFieldSchema.parse)                              │
│  apps/remix/app/components/general/envelope-signing/...renderer.tsx:130,195 │
├─────────────────────────────────────────────────────────────────────┤
│  ZFullFieldSchema = discriminatedUnion('type', [                    │
│    ...                                                              │
│    ZFieldSignatureSchema   // type: z.literal(FieldType.SIGNATURE) │
│    ...                                                              │
│    // ❌ 缺少 ZFieldFreeSignatureSchema！                          │
│  ])                                                                 │
│                                                                     │
│  ✅ SIGNATURE: ✅ 解析成功                                         │
│  ❌ FREE_SIGNATURE: ❌ parse 抛出错误！                            │
│         └── 因为 type="FREE_SIGNATURE" 不在 discriminatedUnion 中  │
│         └── 被 unsafeRenderFieldOnLayer 捕获 → setRenderError(true) │
│  ✅ INITIALS: ✅ 解析成功                                          │
└─────────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│  交互入口阶段 (match().exhaustive())                                │
│  apps/remix/app/components/general/envelope-signing/...renderer.tsx:197-393 │
├─────────────────────────────────────────────────────────────────────┤
│  .with({ type: FieldType.SIGNATURE }, ...) → handleSignatureFieldClick │
│  .with({ type: FieldType.INITIALS }, ...)  → handleInitialsFieldClick │
│  .exhaustive()  // ❌ 没有 FREE_SIGNATURE 分支！                    │
│                                                                     │
│  ✅ SIGNATURE: ✅ 点击 → 签名弹窗 → 提交                          │
│  ❌ FREE_SIGNATURE: ❌ 无法到达此阶段（渲染已失败）                 │
│  ✅ INITIALS: ✅ 点击 → 预填姓名首字母 → 提交                      │
└─────────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│  后端校验阶段 (sign-field-with-token)                               │
│  packages/lib/server-only/field/sign-field-with-token.ts:190-205    │
├─────────────────────────────────────────────────────────────────────┤
│  const isSignatureField =                                           │
│    field.type === SIGNATURE || field.type === FREE_SIGNATURE        │
│                                                                     │
│  if (isSignatureField && !signatureImageAsBase64 && !typedSignature)│
│    throw new Error('Signature field must have a signature');       │
│                                                                     │
│  ✅ SIGNATURE: ✅ 有签名数据则通过                                  │
│  ❌ FREE_SIGNATURE: ❌ 无法到达此阶段（渲染已失败）                 │
│  ✅ INITIALS: ✅ isSignatureField=false，走 customText 路径         │
└─────────────────────────────────────────────────────────────────────┘
```

#### ZFullFieldSchema 对 FREE_SIGNATURE 的致命影响

代码位置：`packages/lib/types/field.ts:182-193`

```typescript
export const ZFullFieldSchema = z.discriminatedUnion('type', [
  ZFieldTextSchema,
  ZFieldSignatureSchema,    // type: z.literal(FieldType.SIGNATURE)
  ZFieldInitialsSchema,     // type: z.literal(FieldType.INITIALS)
  // ... 其他字段类型
  // ❌ 缺少 ZFieldFreeSignatureSchema！
]);

// 虽然定义了，但从未在 ZFullFieldSchema 中使用
export const ZFieldFreeSignatureSchema = ZFieldSignatureSchema;
```

**影响分析：**

1. **渲染阶段直接失败**：`envelope-signer-page-renderer.tsx:130` 调用 `ZFullFieldSchema.parse(unparsedField)`
   - 如果 `unparsedField.type === "FREE_SIGNATURE"`，Zod discriminated union 找不到匹配项 → 抛出错误
   - 错误被 `renderFieldOnLayer` 捕获 → `console.error(err)` + `setRenderError(true)`
   - **页面不会渲染这个字段，也无法进行任何交互**

2. **渲染器也不支持**：`packages/lib/universal/field-renderer/render-field.ts:88-90`

```typescript
.with(FieldType.FREE_SIGNATURE, () => {
  throw new Error('Free signature fields are not supported');
})
```

#### 三类字段最终可操作边界

| 字段类型 | 可从 DB 查询 | ZFullFieldSchema 解析 | 页面渲染 | 点击交互 | 后端签署 | 实际可操作 |
|----------|-------------|----------------------|---------|---------|---------|-----------|
| **SIGNATURE** | ❌ 被 `not: SIGNATURE` 过滤 | N/A | ❌ 不可见 | ❌ 无法点击 | ❌ 无法尝试 | **❌ 不可操作** |
| **FREE_SIGNATURE** | ✅ 未被过滤 | ❌ parse 失败 | ❌ 渲染错误 | ❌ 无交互入口 | ❌ 无法到达 | **❌ 不可操作（遗留功能）** |
| **INITIALS** | ✅ 未被过滤 | ✅ 解析成功 | ✅ 正常渲染 | ✅ 点击触发弹窗 | ✅ 作为 customText 提交 | **✅ 可操作** |

#### 详细代码引用

**1. get-fields-for-token（查询权限）**
- `packages/lib/server-only/field/get-fields-for-token.ts:27`：`type: { not: FieldType.SIGNATURE }`
- 只排除 SIGNATURE，FREE_SIGNATURE 和 INITIALS 均可查询到

**2. ZFullFieldSchema（前端 Schema 校验）**
- `packages/lib/types/field.ts:112-115`：`ZFieldSignatureSchema.type = z.literal(FieldType.SIGNATURE)`
- `packages/lib/types/field.ts:119`：`ZFieldFreeSignatureSchema = ZFieldSignatureSchema`（类型别名，未使用）
- `packages/lib/types/field.ts:182-193`：`ZFullFieldSchema` 的 discriminated union **不包含** FREE_SIGNATURE

**3. 页面渲染与交互**
- `apps/remix/app/components/general/envelope-signing/envelope-signer-page-renderer.tsx:124-130`：`unsafeRenderFieldOnLayer` 中调用 `ZFullFieldSchema.parse()`
- `apps/remix/app/components/general/envelope-signing/envelope-signer-page-renderer.tsx:400-407`：`renderFieldOnLayer` 捕获错误并设置 `setRenderError(true)`
- `apps/remix/app/components/general/envelope-signing/envelope-signer-page-renderer.tsx:197-393`：`match().exhaustive()` 只有 SIGNATURE 分支，无 FREE_SIGNATURE 分支

**4. 渲染器**
- `packages/lib/universal/field-renderer/render-field.ts:88-90`：`.with(FieldType.FREE_SIGNATURE, () => throw new Error('Free signature fields are not supported'))`

**5. 后端签署校验（仅为完整性，FREE_SIGNATURE 不会到达这里）**
- `packages/lib/server-only/field/sign-field-with-token.ts:190`：`isSignatureField = SIGNATURE || FREE_SIGNATURE`
- `packages/lib/server-only/field/sign-field-with-token.ts:203-205`：签名字段需要 `signatureImageAsBase64` 或 `typedSignature`

#### 最终结论

```
┌─────────────────────────────────────────────────────────────────────┐
│  Assistant 对三类签名字段的可操作边界                                │
│                                                                     │
│  SIGNATURE                                                          │
│  └── 查询阶段被过滤 → 不可见 → 不可操作                             │
│                                                                     │
│  FREE_SIGNATURE                                                     │
│  └── DB 可查询到                                                    │
│  └── ZFullFieldSchema.parse() 失败 → 页面渲染错误                   │
│  └── 无交互入口 → 不可操作                                          │
│  └── 结论：遗留/未完全实现的功能，实际上无法使用                      │
│                                                                     │
│  INITIALS                                                           │
│  └── DB 可查询到                                                    │
│  └── ZFullFieldSchema.parse() 成功                                 │
│  └── 点击 → handleInitialsFieldClick → 预填姓名首字母              │
│  └── 提交 → 后端作为 customText 处理                                │
│  └── 结论：Assistant 可以为后续收件人预填 Initials 字段             │
└─────────────────────────────────────────────────────────────────────┘
```

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
