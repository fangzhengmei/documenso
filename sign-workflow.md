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
| SIGNED | 已签署：收件人已完成签名/审批/查看 |
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
| ASSISTANT | 是 | 否 | 为其他收件人预填字段（仅顺序签名可用） |
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
- 可以为后续签名者预填字段值
- 不能代签，不能提交完成
- **仅在顺序签名模式下可用**
- 适用于行政人员为高管准备文档的场景

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
| Assistant | 必须完成预填 → SIGNED |
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

### 示例 2：并行签署

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
| 收件人轮次判断 | `packages/lib/server-only/recipient/get-is-recipient-turn.ts` |
| 完成文档逻辑 | `packages/lib/server-only/document/complete-document-with-token.ts` |
| 文档工具函数 | `packages/lib/utils/document.ts` |
