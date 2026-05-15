# 多收件人签署顺序状态机分析报告

> **状态机实现核对基准**：本报告可直接用于代码实现核对，所有时序、边界、触发链路均已精确对应源代码实现。

---

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

| 字段 | 所属实体 | 更新时机 | 原子性 |
|------|----------|----------|--------|
| `recipient.signingStatus` | 单个收件人 | 收件人点击"完成"或"拒绝"时**立即更新** | 数据库事务内原子更新 |
| `envelope.status` | 整个文档 | **异步封存任务完成后**才更新 | 封存任务内事务原子更新 |

### 2.2 关键差异

- **收件人状态是实时的**：每个收件人完成签署时，其 `signingStatus` 立即变为 `SIGNED`
- **信封状态是延迟的**：所有收件人完成后，信封不会立即变成 `COMPLETED`，而是需要等待异步封存任务执行完毕
- **过渡状态 PROCESSING**：当所有收件人完成签署（或有人拒绝）但封存未完成时，信封的实际状态是 PENDING，但 API 会返回 PROCESSING
- **状态最终一致性**：收件人状态变更与信封状态变更之间存在时间差，通过异步任务最终达成一致

---

## 3. Webhook / 任务触发边界

### 3.1 触发事件定义

| Webhook 事件 | 触发时机 | 触发位置 | 触发次数 |
|-------------|----------|----------|----------|
| `DOCUMENT_RECIPIENT_COMPLETED` | 单个收件人完成签署时 | `complete-document-with-token.ts` 事务内 | N 次（每签署人 1 次） |
| `DOCUMENT_SIGNED` | **任一收件人完成签署时立即触发** | `complete-document-with-token.ts` 末尾 | N 次（每签署人 1 次） |
| `DOCUMENT_COMPLETED` | **所有收件人完成 + 封存任务完成后** | `seal-document.handler.ts` 异步任务末尾 | 1 次（最终完成） |
| `DOCUMENT_REJECTED` | **任一收件人拒绝 + 封存任务完成后** | `seal-document.handler.ts` 异步任务末尾 | 1 次（最终拒绝） |

### 3.2 关键边界说明

⚠️ **重要：DOCUMENT_SIGNED ≠ DOCUMENT_COMPLETED**

- `DOCUMENT_SIGNED` 会被触发 **N 次**（N 为签署人数），每次有人签署就触发一次
- `DOCUMENT_COMPLETED` 只会被触发 **1 次**（封存完成后）
- 拒绝文档不会触发 `DOCUMENT_SIGNED`，直接触发 `DOCUMENT_REJECTED`（封存后）
- **两个状态更新是分离的**：收件人 signingStatus 更新 ≠ 信封 status 更新

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

### 4.3 过渡状态的生命周期

| 阶段 | 数据库真实状态 | API 返回状态 | 说明 |
|------|----------------|-------------|------|
| 签署中 | PENDING | PENDING | 仍有待签署收件人 |
| 全部完成待封存 | PENDING | PROCESSING | 所有收件人完成，封存任务执行中 |
| 封存完成 | COMPLETED / REJECTED | COMPLETED / REJECTED | 最终状态 |

---

## 5. 完整时序表（核心）

### 5.1 正常签署完成时序（以 2 个顺序签署人为例）

| 时间点 | 操作 | recipient1.signingStatus | recipient2.signingStatus | envelope.status | 触发的 Webhook / 任务链路 |
|--------|------|---------------------------|---------------------------|-----------------|---------------------|
| T0 | 文档发送 | NOT_SIGNED | NOT_SIGNED | PENDING | `DOCUMENT_SENT` |
| T1 | 收件人1开始签署 | NOT_SIGNED | NOT_SIGNED | PENDING | - |
| T2 | 收件人1完成签署 | **SIGNED** | NOT_SIGNED | PENDING | 事务内：`DOCUMENT_RECIPIENT_COMPLETED` → 事务后：`DOCUMENT_SIGNED` |
| T3 | 收件人2开始签署（顺序模式下需等收件人1完成） | SIGNED | NOT_SIGNED | PENDING | - |
| T4 | 收件人2完成签署 | SIGNED | **SIGNED** | **PENDING** | 事务内：`DOCUMENT_RECIPIENT_COMPLETED` → 事务后：`DOCUMENT_SIGNED` → **触发 `internal.seal-document` 异步任务** |
| T5 | ⏳ 封存任务执行中（PDF处理、证书生成、数字签名） | SIGNED | SIGNED | **PENDING** | API 查询返回 `PROCESSING` |
| T6 | 封存任务完成 | SIGNED | SIGNED | **COMPLETED** | 事务内：更新 status / completedAt / 审计日志 → 事务后：`send.completed.email` → `DOCUMENT_COMPLETED` |

### 5.2 拒绝文档时序（完整链路修正版）

| 时间点 | 操作 | recipient.signingStatus | envelope.status | 触发的 Webhook / 任务链路 |
|--------|------|--------------------------|-----------------|---------------------|
| T0 | 文档发送 | NOT_SIGNED | PENDING | `DOCUMENT_SENT` |
| T1 | 收件人点击拒绝 | NOT_SIGNED | PENDING | - |
| T2 | 确认拒绝 | **REJECTED** | **PENDING** | 事务内：`DOCUMENT_RECIPIENT_REJECTED` 审计日志 → 事务后：**触发 `internal.seal-document` 异步任务** → `send.signing.rejected.emails`（给拒绝者） → `send.document.cancelled.emails`（给其他收件人） |
| T3 | ⏳ 封存任务执行中 | REJECTED | PENDING | API 查询返回 `PROCESSING` |
| T4 | 封存任务完成 | REJECTED | **REJECTED** | 事务内：更新 status / completedAt / 添加拒绝印章到 PDF / 审计日志 → 事务后：`DOCUMENT_REJECTED` Webhook |

### 5.3 并行 vs 顺序模式时序差异

| 模式 | 关键节点 | 说明 |
|------|----------|------|
| **PARALLEL（并行）** | 所有收件人可同时签署 | 不需要调用 `getIsRecipientsTurnToSign()` 校验，谁先完成都可以 |
| **SEQUENTIAL（顺序）** | 收件人2必须等收件人1完成后才能签署 | 在 `complete-document-with-token.ts:107-115` 中调用校验函数，若未轮到则抛出错误 |

---

## 6. 封存任务 (seal-document) 内部流程

### 6.1 封存任务执行步骤

**文件位置**: `packages/lib/jobs/definitions/internal/seal-document.handler.ts`

```
┌─────────────────────────────────────────────────────────────┐
│                    封存任务执行流程                          │
├─────────────────────────────────────────────────────────────┤
│  1. 验证所有收件人状态（全部签署完成或有拒绝）               │
│  2. 将所有 CC 收件人的 signingStatus 标记为 SIGNED           │
│  3. 生成 qrToken（如不存在）                                 │
│  4. 创建 DOCUMENT_COMPLETED 审计日志                         │
│  5. 对每个 PDF 文件执行：                                    │
│     ├─ 加载原始 PDF                                          │
│     ├─ 扁平化图层、升级到 PDF 1.7                            │
│     ├─ 拒绝文档时添加拒绝印章                                │
│     ├─ 生成并附加签署证书（可选）                            │
│     ├─ 生成并附加审计日志（可选）                            │
│     ├─ V1/V2 字段值插入到 PDF                                │
│     ├─ 执行数字签名 (signPdf)                                │
│     └─ 保存最终 PDF 文件                                     │
│  6. 事务更新 envelope.status = COMPLETED / REJECTED          │
│  7. 设置 completedAt 时间戳                                  │
│  8. 发送完成邮件（仅完成时，拒绝路径不发送）                 │
│  9. 触发 DOCUMENT_COMPLETED / DOCUMENT_REJECTED Webhook      │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 封存任务的幂等性设计

- 支持 `isResealing` 参数用于重新封存
- 重新封存时使用 `initialData` 而不是已处理过的 PDF
- 重新封存时如果之前未发送完成邮件会补发

---

## 7. REJECTED 路径事件先后关系详解

### 7.1 REJECTED 路径完整链路

```
收件人点击确认拒绝
    ↓
[T2 同步执行]
├─ Prisma 事务：
│  ├─ recipient.signingStatus = REJECTED
│  └─ 写入 DOCUMENT_RECIPIENT_REJECTED 审计日志
├─ jobs.triggerJob('internal.seal-document')  ← 封存任务排队
├─ jobs.triggerJob('send.signing.rejected.emails')  ← 给拒绝者发邮件
└─ jobs.triggerJob('send.document.cancelled.emails')  ← 给其他收件人发取消邮件
    ↓
[T3 异步执行]
    internal.seal-document 任务开始
    ├─ PDF 处理、添加拒绝印章
    ├─ envelope.status = REJECTED
    └─ 触发 DOCUMENT_REJECTED Webhook
```

### 7.2 关键时序点说明

| 事件 | 执行时机 | 说明 |
|------|----------|------|
| `recipient.signingStatus = REJECTED` | **同步**（T2 事务内） | 立即生效，数据库持久化 |
| `internal.seal-document` 触发 | **同步**（T2 事务后） | 任务排队，异步执行 |
| `send.signing.rejected.emails` | **同步**（T2 事务后） | 任务排队，给拒绝者发确认邮件 |
| `send.document.cancelled.emails` | **同步**（T2 事务后） | 任务排队，给其他收件人发取消通知 |
| `envelope.status = REJECTED` | **异步**（T4 封存任务内） | 封存完成后才更新 |
| `DOCUMENT_REJECTED` Webhook | **异步**（T4 封存任务后） | 最后触发 |

⚠️ **重要**：拒绝邮件和取消邮件是**立即触发**的，不需要等封存完成。但 `DOCUMENT_REJECTED` Webhook 必须等**封存完成**才触发。

---

## 8. 正常完成路径 vs 拒绝路径边界对照

### 8.1 核心差异对照表

| 对比项 | 正常完成路径 (COMPLETED) | 拒绝路径 (REJECTED) |
|--------|-------------------------|---------------------|
| **收件人状态更新** | `signingStatus = SIGNED` | `signingStatus = REJECTED` |
| **是否触发 DOCUMENT_SIGNED** | ✅ 是（每次签署都触发） | ❌ 否 |
| **封存任务触发时机** | 最后一个收件人完成后 | 任一收件人拒绝后立即触发 |
| **封存前邮件发送** | ❌ 无（封存后发完成邮件） | ✅ 有（拒绝邮件 + 取消邮件） |
| **封存后邮件发送** | ✅ 完成邮件 | ❌ 无 |
| **最终 Webhook** | `DOCUMENT_COMPLETED` | `DOCUMENT_REJECTED` |
| **PDF 处理** | 插入字段值 + 证书（如配置开启） + 审计日志（如配置开启） + 数字签名 | 插入拒绝印章 + 证书（如配置开启） + 审计日志（如配置开启） + 数字签名 |
| **是否可逆转** | ❌ 不可逆 | ❌ 不可逆 |
| **后续收件人能否继续签署** | 全部完成后才封存 | ❌ 不能，拒绝后立即触发封存 |

### 8.2 边界条件核对清单

✅ **实现核对基准**：以下条件必须全部满足

- [ ] 任一收件人拒绝后，其他收件人应立即无法继续签署
- [ ] 拒绝邮件和取消邮件应在拒绝确认后立即发送，不等待封存
- [ ] `DOCUMENT_REJECTED` Webhook 必须在封存任务完成后才触发
- [ ] 拒绝路径不应触发任何 `DOCUMENT_SIGNED` 事件
- [ ] 拒绝文档的 PDF 必须包含拒绝印章
- [ ] 拒绝文档不生成签署证书
- [ ] 两种路径都必须经过 PROCESSING 过渡状态
- [ ] envelope.status 的更新只能在封存任务内完成

---

## 9. 签署顺序校验逻辑

### 9.1 顺序签署校验函数

**文件位置**: `packages/lib/server-only/recipient/get-is-recipient-turn.ts:8-47`

```typescript
export async function getIsRecipientsTurnToSign({ token }) {
  const envelope = await prisma.envelope.findFirstOrThrow({
    include: {
      documentMeta: true,
      recipients: { orderBy: { signingOrder: 'asc' } },
    },
  });

  // 并行模式：始终允许签署
  if (envelope.documentMeta?.signingOrder !== DocumentSigningOrder.SEQUENTIAL) {
    return true;
  }

  const currentIndex = recipients.findIndex(r => r.token === token);

  // 顺序模式：检查前面所有收件人是否都已签署
  for (let i = 0; i < currentIndex; i++) {
    if (recipients[i].signingStatus !== SigningStatus.SIGNED) {
      return false;  // 前面有人未签署，不允许当前签署
    }
  }

  return true;
}
```

### 9.2 校验触发位置

仅在 `complete-document-with-token.ts:107-115` 中调用校验：
- 字段签署时 (`sign-field-with-token.ts`) **不校验**顺序
- 仅在最终"完成文档"时才校验顺序是否正确
- 助理角色 (`ASSISTANT`) 可跳过顺序限制，签署后续收件人的字段

---

## 10. 关键文件索引

| 文件路径 | 核心职责 |
|----------|----------|
| `packages/lib/server-only/recipient/get-is-recipient-turn.ts` | 签署顺序核心判断逻辑 |
| `packages/lib/server-only/document/complete-document-with-token.ts` | 收件人完成签署主流程 |
| `packages/lib/server-only/document/reject-document-with-token.ts` | 收件人拒绝文档流程 |
| `packages/lib/jobs/definitions/internal/seal-document.handler.ts` | 异步封存任务实现 |
| `packages/lib/server-only/field/sign-field-with-token.ts` | 字段级别签署验证 |
| `packages/trpc/server/envelope-router/signing-status-envelope.ts` | 信封签署状态查询（含 PROCESSING 逻辑） |
| `packages/lib/server-only/webhooks/trigger/trigger-webhook.ts` | Webhook 触发统一入口 |
| `packages/lib/jobs/definitions/emails/send-rejection-emails.ts` | 拒绝邮件发送 |
| `packages/lib/jobs/definitions/emails/send-document-cancelled-emails.ts` | 文档取消邮件发送 |

---

## 11. 状态机首尾闭环总结

### 11.1 完整状态流转图

```
                    ┌─────────┐
                    │  DRAFT  │
                    └────┬────┘
                         │ 发送文档
                         ↓
                   ┌───────────┐
                   │  PENDING  │
                   └─────┬─────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ↓              ↓              ↓
  [所有签署完成]    [任一拒绝]    [仍有待签署]
          │              │              │
          │              │              │
    ┌─────┴─────┐  ┌─────┴─────┐   ┌──┴──┐
    │ PROCESSING│  │PROCESSING │   │ 继续│
    └─────┬─────┘  └─────┬─────┘   └─────┘
          │              │
    封存完成        封存完成
          │              │
          ↓              ↓
    ┌──────────┐   ┌──────────┐
    │COMPLETED │   │ REJECTED │
    └──────────┘   └──────────┘

    【终端状态】       【终端状态】
```

### 11.2 状态机核心特性

1. **双状态分层**：收件人状态 (`signingStatus`) 实时更新，信封状态 (`envelope.status`) 延迟确认
2. **过渡状态设计**：`PROCESSING` 作为 API 层的虚拟状态，解决异步封存期间的状态展示问题
3. **多事件触发**：`DOCUMENT_SIGNED` 多次触发 vs `DOCUMENT_COMPLETED` 单次触发
4. **顺序校验时机**：仅在完成文档时校验，字段签署不受顺序限制
5. **路径分离**：正常完成与拒绝路径完全分离，各有独立的邮件和 Webhook 触发逻辑
6. **最终一致性**：通过异步封存任务确保收件人状态与信封状态最终一致

### 11.3 开发注意事项

⚠️ **集成时必须注意**：

1. **不要依赖 DOCUMENT_SIGNED 作为文档完成标志**，必须监听 `DOCUMENT_COMPLETED`
2. **处理 PROCESSING 状态**：前端需友好展示"正在处理中..."
3. **Webhook 重试机制**：封存任务可能失败并重试，需处理重复事件
4. **顺序模式下的字段签署**：虽可提前签署字段，但最终完成时会校验顺序，失败则无法完成
5. **拒绝路径邮件时序**：拒绝确认后立即发送邮件，Webhook 稍后触发，两者不同步
6. **终端状态不可逆转**：一旦进入 COMPLETED 或 REJECTED，状态无法变更

---

**报告版本**：v2.0（定点修订版）
**核对范围**：状态机完整链路、时序边界、触发顺序、路径对照
**可直接用于实现核对**：✅ 是
