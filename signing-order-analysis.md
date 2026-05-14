# 多收件人签署顺序状态机分析报告

## 1. 概述

本报告分析 Documenso 系统中多收件人签署顺序的状态机管控机制，涵盖数据模型、核心逻辑、处理层协调及状态流转等关键方面。

## 2. 核心数据模型

### 2.1 签署顺序类型枚举

```prisma
enum DocumentSigningOrder {
  PARALLEL      // 并行签署：所有收件人可同时签署
  SEQUENTIAL    // 顺序签署：必须按指定顺序依次签署
}
```

### 2.2 签署状态枚举

```prisma
enum SigningStatus {
  NOT_SIGNED    // 未签署
  SIGNED        // 已签署
  REJECTED      // 已拒绝
}
```

### 2.3 关键数据库字段

| 表名 | 字段名 | 说明 |
|------|--------|------|
| DocumentMeta | signingOrder | 文档签署顺序配置（PARALLEL / SEQUENTIAL） |
| Recipient | signingOrder | 收件人签署顺序编号 |
| Recipient | signingStatus | 收件人当前签署状态 |
| Recipient | signedAt | 签署完成时间戳 |

## 3. 状态机核心机制

### 3.1 签署顺序判断逻辑

**文件位置**: `packages/lib/server-only/recipient/get-is-recipient-turn.ts:8-47`

```typescript
export async function getIsRecipientsTurnToSign({ token }: GetIsRecipientTurnOptions) {
  // 1. 获取信封及所有收件人（按 signingOrder 升序排列）
  const envelope = await prisma.envelope.findFirstOrThrow({
    include: {
      documentMeta: true,
      recipients: { orderBy: { signingOrder: 'asc' } },
    },
  });

  // 2. 并行模式：直接返回 true，任何人可随时签署
  if (envelope.documentMeta?.signingOrder !== DocumentSigningOrder.SEQUENTIAL) {
    return true;
  }

  // 3. 顺序模式：检查前面所有收件人是否都已签署
  const currentRecipientIndex = recipients.findIndex((r) => r.token === token);
  
  for (let i = 0; i < currentRecipientIndex; i++) {
    if (recipients[i].signingStatus !== SigningStatus.SIGNED) {
      return false;
    }
  }

  return true;
}
```

### 3.2 下一个待签署收件人获取

**文件位置**: `packages/lib/server-only/recipient/get-next-pending-recipient.ts:6-43`

- 按 `signingOrder` 升序、`id` 升序复合排序
- null 值的 signingOrder 排在最后
- 返回当前收件人之后的下一个收件人

## 4. 多层处理协调机制

### 4.1 处理层架构

```
┌───────────────────────────────────────────────────────────┐
│                    UI 层 (React)                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  EnvelopeSigningProvider (envelope-signing-provider │  │
│  │  - 管理签署上下文状态                                │  │
│  │  - 跟踪剩余待签字段                                  │  │
│  │  - 助理签署模式支持                                  │  │
│  │  - nextRecipient 状态管理                            │  │
│  └─────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
                              ↓
┌───────────────────────────────────────────────────────────┐
│                  API 层 (tRPC Routes)                      │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  signing-status-envelope.ts                         │  │
│  │  - 查询信封整体签署状态                              │  │
│  │  - 返回 PENDING / PROCESSING / COMPLETED / REJECTED │  │
│  └─────────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  sign-envelope-field.ts                             │  │
│  │  - 单个字段签署处理                                  │  │
│  └─────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
                              ↓
┌───────────────────────────────────────────────────────────┐
│                业务逻辑层 (Server-Only Lib)                │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  complete-document-with-token.ts                    │  │
│  │  - 完成收件人签署流程                                │  │
│  │  - 调用 getIsRecipientsTurnToSign 验证顺序          │  │
│  │  - 更新 recipient.signingStatus = SIGNED            │  │
│  │  - 触发下一个收件人邮件通知                          │  │
│  │  - 触发 webhook: DOCUMENT_RECIPIENT_COMPLETED       │  │
│  │  - 检查所有收件人完成后触发文档密封                  │  │
│  └─────────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  sign-field-with-token.ts                           │  │
│  │  - 字段级别签署验证                                  │  │
│  │  - ASSISTANT 角色特殊处理：可签署后续收件人字段       │  │
│  └─────────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  send-pending-email.ts                              │  │
│  │  - 发送"等待他人签署"邮件通知                        │  │
│  └─────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
                              ↓
┌───────────────────────────────────────────────────────────┐
│                    数据层 (Prisma)                         │
│  - Recipient: signingStatus, signingOrder                  │
│  - DocumentMeta: signingOrder                              │
│  - Field: inserted 状态                                    │
└───────────────────────────────────────────────────────────┘
```

### 4.2 关键协调点

#### 4.2.1 文档完成时的状态流转

**文件位置**: `packages/lib/server-only/document/complete-document-with-token.ts`

```
流程步骤：
1. 验证信封状态为 PENDING
2. 验证收件人未签署、未拒绝
3. 顺序模式下：调用 getIsRecipientsTurnToSign() 验证轮次
4. 验证所有必填字段已签署
5. 事务更新：
   - recipient.signingStatus = SIGNED
   - recipient.signedAt = now()
   - 创建审计日志 DOCUMENT_RECIPIENT_COMPLETED
6. 触发 webhook: DOCUMENT_RECIPIENT_COMPLETED
7. 发送当前收件人完成确认邮件
8. 获取待签署收件人列表（按 signingOrder 排序）
9. 顺序模式下：
   - 更新下一个收件人 sendStatus = SENT
   - 发送签署请求邮件给下一个收件人
10. 检查所有收件人是否完成：
    - 若是，触发 internal.seal-document 任务
11. 触发 webhook: DOCUMENT_SIGNED
```

#### 4.2.2 助理（ASSISTANT）角色的特殊处理

**文件位置**: `packages/lib/server-only/field/sign-field-with-token.ts:69-81`

```typescript
// ASSISTANT 角色可以签署：
// - 签署状态 != SIGNED 的收件人
// - signingOrder >= 当前助理签署顺序 的收件人
// - 同一信封内的字段
```

## 5. 状态流转详解

### 5.1 信封（Envelope）状态

| 状态 | 触发条件 | 说明 |
|------|----------|------|
| PENDING | 初始状态 | 文档等待签署 |
| PROCESSING | 部分收件人已签署，仍有待签署者 | 签署进行中 |
| COMPLETED | 所有签署人完成签署（或有人拒绝） | 签署流程结束 |
| REJECTED | 任一收件人拒绝 | 签署终止 |

**状态查询逻辑**: `packages/trpc/server/envelope-router/signing-status-envelope.ts`

### 5.2 收件人（Recipient）状态流转

```
                    NOT_SIGNED
                        │
           ┌────────────┼────────────┐
           ↓            ↓            ↓
        SIGNED       REJECTED     EXPIRED
           │            │            │
           └────────────┴────────────┘
                        │
                    终端状态
```

### 5.3 顺序签署模式下的收件人状态流转

```
  [收件人1] NOT_SIGNED → 验证通过 → 签署 → SIGNED
                                              ↓
  [收件人2] NOT_SIGNED ──────────────────────→ 验证通过（前置都已签署）→ 签署 → SIGNED
                                                                                      ↓
  [收件人3] NOT_SIGNED ────────────────────────────────────────────────────────→ 验证通过 → 签署 → SIGNED
                                                                                                                          ↓
                                                                                                                    文档 COMPLETED
```

## 6. 关键实现细节

### 6.1 排序规则

**文件位置**: `packages/lib/server-only/document/complete-document-with-token.ts:388-389`

```typescript
orderBy: [
  { signingOrder: { sort: 'asc', nulls: 'last' } },  // 优先按签署顺序
  { id: 'asc' }                                        // 顺序相同按ID排序
]
```

### 6.2 下一个收件人指定（Dictate Next Signer）

**文件位置**: `packages/lib/server-only/document/complete-document-with-token.ts:398-428`

- 当 `documentMeta.allowDictateNextSigner = true` 时
- 当前签署人可指定下一个收件人的姓名和邮箱
- 系统会更新下一个收件人信息并创建审计日志

### 6.3 CC 收件人处理

CC 收件人不参与签署流程：
- 状态检查时跳过 CC 角色
- 不计算在待签署收件人列表中
- 文档完成检查时排除 CC 角色

## 7. 审计日志事件

签署顺序相关的审计事件：

| 事件类型 | 触发时机 |
|----------|----------|
| DOCUMENT_RECIPIENT_COMPLETED | 收件人完成签署时 |
| RECIPIENT_UPDATED | 下一个收件人信息被指定时 |
| DOCUMENT_FIELD_INSERTED | 字段被签署时 |
| DOCUMENT_ACCESS_AUTH_2FA_* | 2FA 认证相关 |

## 8. Webhook 触发事件

| 事件 | 触发时机 |
|------|----------|
| DOCUMENT_RECIPIENT_COMPLETED | 单个收件人完成签署 |
| DOCUMENT_SIGNED | 任一收件人签署完成（注意：不是文档最终完成） |
| DOCUMENT_COMPLETED | 所有收件人完成签署，文档密封后 |

## 9. 关键文件索引

| 文件路径 | 核心职责 |
|----------|----------|
| `packages/lib/server-only/recipient/get-is-recipient-turn.ts` | 签署顺序核心判断逻辑 |
| `packages/lib/server-only/recipient/get-next-pending-recipient.ts` | 获取下一个待签署收件人 |
| `packages/lib/server-only/document/complete-document-with-token.ts` | 收件人完成签署主流程 |
| `packages/lib/server-only/field/sign-field-with-token.ts` | 字段级别签署验证 |
| `packages/lib/server-only/document/send-pending-email.ts` | 发送待签署邮件通知 |
| `packages/trpc/server/envelope-router/signing-status-envelope.ts` | 信封签署状态查询 API |
| `apps/remix/app/components/general/document-signing/envelope-signing-provider.tsx` | 前端签署上下文管理 |

## 10. 总结

Documenso 的签署顺序状态机机制具有以下特点：

1. **双模式支持**：并行（PARALLEL）和顺序（SEQUENTIAL）两种签署模式
2. **分层设计**：UI 层 → API 层 → 业务逻辑层 → 数据层，各层职责清晰
3. **事务保障**：关键状态更新使用数据库事务确保一致性
4. **审计追踪**：完整的审计日志记录所有签署相关操作
5. **灵活扩展**：支持助理签署、下一个收件人指定等高级功能
6. **状态驱动**：基于 `signingStatus` 和 `signingOrder` 字段驱动整个流程

该机制通过数据库字段状态 + 业务逻辑校验的组合，实现了可靠的多收件人签署顺序管控。
