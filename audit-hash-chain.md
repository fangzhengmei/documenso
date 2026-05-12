# 审计日志与哈希链联动机制报告

## 1. 事件写入机制

### 1.1 审计日志数据结构

每个审计日志事件包含以下核心字段（`packages/lib/types/document-audit-logs.ts:725-734`）：

```typescript
{
  id: string;                    // 审计日志唯一ID
  createdAt: Date;               // 事件发生时间
  envelopeId: string;            // 关联的信封ID（关键关联字段）
  name: string | null;           // 操作用户姓名
  email: string | null;          // 操作用户邮箱
  userId: number | null;         // 操作用户ID
  userAgent: string | null;      // 浏览器用户代理
  ipAddress: string | null;      // 操作IP地址
  type: DOCUMENT_AUDIT_LOG_TYPE; // 事件类型
  data: object;                  // 事件特定数据
}
```

### 1.2 事件创建流程

通过 `createDocumentAuditLogData` 工具函数统一创建审计日志数据（`packages/lib/utils/document-audit-logs.ts:40-72`）：

```typescript
createDocumentAuditLogData({
  type: DOCUMENT_AUDIT_LOG_TYPE.DOCUMENT_FIELD_INSERTED,
  envelopeId: envelope.id,
  user: { email: recipient.email, name: recipient.name },
  requestMetadata,  // 包含IP和UserAgent
  data: {
    recipientEmail: recipient.email,
    recipientId: recipient.id,
    recipientName: recipient.name,
    recipientRole: recipient.role,
    fieldId: updatedField.secondaryId,
    field: { type: FieldType.SIGNATURE, data: signatureValue },
    fieldSecurity: { type: derivedRecipientActionAuth },
  },
});
```

### 1.3 关键事件类型

审计日志涵盖50+种事件类型，核心包括：

| 事件类型 | 触发时机 | 关键数据 |
|---------|---------|---------|
| `DOCUMENT_CREATED` | 文档创建 | 文档标题、来源 |
| `DOCUMENT_SENT` | 文档发送 | - |
| `DOCUMENT_OPENED` | 文档被打开 | 收件人信息、访问认证 |
| `DOCUMENT_VIEWED` | 文档被查看 | 收件人信息、访问认证 |
| `DOCUMENT_FIELD_INSERTED` | 字段被签署 | 字段ID、字段类型、字段值、签署认证 |
| `DOCUMENT_FIELD_UNINSERTED` | 字段被取消签署 | 字段ID、字段类型 |
| `DOCUMENT_RECIPIENT_COMPLETED` | 收件人完成签署 | 收件人信息、操作认证 |
| `DOCUMENT_RECIPIENT_REJECTED` | 收件人拒绝文档 | 拒绝原因 |
| `DOCUMENT_COMPLETED` | 文档最终密封 | transactionId |
| `RECIPIENT_CREATED/UPDATED/DELETED` | 收件人变更 | 变更详情 |
| `FIELD_CREATED/UPDATED/DELETED` | 字段变更 | 变更详情 |

### 1.4 字段签名时的事件写入

在 `signFieldWithToken` 函数中（`packages/lib/server-only/field/sign-field-with-token.ts:270-309`）：

```typescript
await tx.documentAuditLog.create({
  data: createDocumentAuditLogData({
    type: assistant && field.recipientId !== assistant.id
      ? DOCUMENT_AUDIT_LOG_TYPE.DOCUMENT_FIELD_PREFILLED
      : DOCUMENT_AUDIT_LOG_TYPE.DOCUMENT_FIELD_INSERTED,
    envelopeId: envelope.id,
    user: {
      email: assistant?.email ?? recipient.email,
      name: assistant?.name ?? recipient.name,
    },
    requestMetadata,
    data: {
      recipientEmail: recipient.email,
      recipientId: recipient.id,
      recipientName: recipient.name,
      recipientRole: recipient.role,
      fieldId: updatedField.secondaryId,
      field: { type: fieldType, data: fieldValue },
      fieldSecurity: derivedRecipientActionAuth
        ? { type: derivedRecipientActionAuth }
        : undefined,
    },
  }),
});
```

---

## 2. PDF 密封顺序

### 2.1 密封触发条件

当所有收件人完成签署时（`packages/lib/server-only/document/complete-document-with-token.ts:457-476`）：

```typescript
const haveAllRecipientsSigned = await prisma.envelope.findFirst({
  where: {
    id: envelope.id,
    recipients: {
      every: {
        OR: [
          { signingStatus: SigningStatus.SIGNED },
          { role: RecipientRole.CC },
        ],
      },
    },
  },
});

if (haveAllRecipientsSigned) {
  await jobs.triggerJob({
    name: 'internal.seal-document',
    payload: {
      documentId: legacyDocumentId,
      requestMetadata,
    },
  });
}
```

### 2.2 密封作业执行流程

`seal-document` 处理器（`packages/lib/jobs/definitions/internal/seal-document.handler.ts`）按以下顺序执行：

#### 阶段一：数据准备与验证
1. 查询信封及其关联的收件人、字段、信封项目
2. 验证文档状态必须为 `PENDING`
3. 检查所有必需字段是否已签署
4. 处理被拒绝的文档

#### 阶段二：生成最终审计日志

```typescript
const envelopeCompletedAuditLog = createDocumentAuditLogData({
  type: DOCUMENT_AUDIT_LOG_TYPE.DOCUMENT_COMPLETED,
  envelopeId: envelope.id,
  requestMetadata,
  user: null,
  data: {
    transactionId: nanoid(),  // 生成唯一交易ID
    ...(isRejected ? { isRejected: true, rejectionReason } : {}),
  },
});
```

#### 阶段三：PDF 装饰与签署

调用 `decorateAndSignPdf` 函数（`seal-document.handler.ts:344-480`）按以下顺序处理：

```
1. PDF 规范化与扁平化
   ├─ pdfDoc.flattenAll() - 扁平化所有图层
   └─ pdfDoc.upgradeVersion('1.7') - 升级到 PDF 1.7

2. 添加拒绝印章（如适用）
   └─ addRejectionStampToPdf(pdfDoc, rejectionReason)

3. 附加证书页面（如启用）
   └─ 复制 certificateDoc 页面

4. 附加审计日志页面（如启用）
   └─ 复制 auditLogDoc 页面

5. 插入所有字段值
   ├─ V1: legacy_insertFieldInPDF / insertFieldInPDFV1
   └─ V2: insertFieldInPDFV2（覆盖层方式）

6. 最终扁平化处理

7. 数字签名
   └─ signPdf({ pdf: pdfDoc }) - 执行 PDF 数字签名

8. 文件存储
   └─ putPdfFileServerSide() - 保存最终签署的 PDF
```

#### 阶段四：数据库事务更新

```typescript
await prisma.$transaction(async (tx) => {
  // 1. 更新所有信封项目的文档数据引用
  for (const { oldDocumentDataId, newDocumentDataId } of newDocumentData) {
    await tx.envelopeItem.update({
      where: { envelopeId: envelope.id, documentDataId: oldDocumentDataId },
      data: { documentDataId: newDocumentDataId },
    });
  }

  // 2. 更新信封状态
  await tx.envelope.update({
    where: { id: envelope.id },
    data: {
      status: finalEnvelopeStatus,  // COMPLETED 或 REJECTED
      completedAt: new Date(),
    },
  });

  // 3. 写入最终完成审计日志
  await tx.documentAuditLog.create({
    data: envelopeCompletedAuditLog,
  });
});
```

### 2.3 证书与审计日志 PDF 生成

在密封过程中，如有设置会生成附加页面：

```typescript
const makeCertificatePdf = async () =>
  usePlaywrightPdf
    ? getCertificatePdf({ documentId, language })
    : generateCertificatePdf(certificatePayload);

const makeAuditLogPdf = async () =>
  usePlaywrightPdf
    ? getAuditLogsPdf({ documentId, language })
    : generateAuditLogPdf(certificatePayload);
```

这些 PDF 会被附加到原始文档之后。

---

## 3. 跨收件人时间线聚合

### 3.1 聚合键：envelopeId

所有审计日志事件通过 `envelopeId` 字段关联到同一文档。这是跨收件人时间线聚合的核心机制。

### 3.2 查询聚合方法

查询文档所有审计日志（按时间排序）：

```typescript
const auditLogs = await prisma.documentAuditLog.findMany({
  where: { envelopeId: envelope.id },
  orderBy: { createdAt: 'asc' },
});
```

### 3.3 时间线聚合的典型场景

#### 场景 A：完整签署流程时间线

```
时间线顺序：
├─ T0: DOCUMENT_CREATED - 创建者创建文档
├─ T1: RECIPIENT_CREATED - 添加收件人A
├─ T2: RECIPIENT_CREATED - 添加收件人B
├─ T3: FIELD_CREATED - 为收件人A创建签名字段
├─ T4: FIELD_CREATED - 为收件人B创建签名字段
├─ T5: DOCUMENT_SENT - 文档发送
├─ T6: DOCUMENT_OPENED - 收件人A打开文档
├─ T7: DOCUMENT_VIEWED - 收件人A查看文档
├─ T8: DOCUMENT_FIELD_INSERTED - 收件人A签署字段1
├─ T9: DOCUMENT_FIELD_INSERTED - 收件人A签署字段2
├─ T10: DOCUMENT_RECIPIENT_COMPLETED - 收件人A完成签署
├─ T11: DOCUMENT_OPENED - 收件人B打开文档
├─ T12: DOCUMENT_VIEWED - 收件人B查看文档
├─ T13: DOCUMENT_FIELD_INSERTED - 收件人B签署字段
├─ T14: DOCUMENT_RECIPIENT_COMPLETED - 收件人B完成签署
└─ T15: DOCUMENT_COMPLETED - 文档密封（含transactionId）
```

#### 场景 B：顺序签署模式时间线

当启用顺序签署（`DocumentSigningOrder.SEQUENTIAL`）时：

```
时间线顺序：
├─ ...
├─ Tn:   DOCUMENT_RECIPIENT_COMPLETED - 收件人1完成
├─ Tn+1: RECIPIENT_UPDATED - 更新收件人2发送状态
├─ Tn+2: DOCUMENT_OPENED - 收件人2打开文档
├─ ...
```

### 3.4 审计日志格式化与展示

`formatDocumentAuditLogAction` 函数（`packages/lib/utils/document-audit-logs.ts:287-614`）将原始审计日志转换为人类可读的描述：

```typescript
const { prefix, description } = formatDocumentAuditLogAction(
  i18n,
  auditLog,
  userId
);

// 输出示例：
// "张三 signed the document"
// "You updated a field"
// "Recipient opened the document"
```

支持按当前用户视角格式化（显示"You"而非用户名）。

### 3.5 按收件人分组聚合

通过 `recipientId` 或 `recipientEmail` 在 `data` 字段中过滤，可获得单个收件人的操作时间线：

```typescript
const recipientAuditLogs = auditLogs.filter(log =>
  'recipientId' in log.data && log.data.recipientId === targetRecipientId
);
```

---

## 4. 联动机制总结

### 4.1 审计日志 → 哈希链 数据流

```
用户操作
    ↓
创建审计日志（带envelopeId）
    ↓
写入documentAuditLog表
    ↓
（重复直到所有操作完成）
    ↓
触发seal-document作业
    ↓
查询该envelopeId下所有审计日志
    ↓
生成审计日志PDF页面
    ↓
附加到原始PDF
    ↓
PDF数字签名（生成哈希）
    ↓
写入DOCUMENT_COMPLETED审计日志（含transactionId）
    ↓
文档密封完成
```

### 4.2 关键联动点

| 联动点 | 位置 | 作用 |
|-------|------|------|
| `envelopeId` | 所有审计日志 | 跨操作、跨收件人的关联键 |
| `createdAt` | 所有审计日志 | 时间排序，构建时间线 |
| `transactionId` | `DOCUMENT_COMPLETED` 事件 | 最终密封的唯一标识 |
| `requestMetadata` | 所有事件 | 捕获IP、UserAgent，用于审计追踪 |
| `fieldSecurity` | 字段插入事件 | 记录签署时的认证方式 |

### 4.3 防篡改保障层级

1. **数据库级**：审计日志为追加写入模式，不支持更新/删除
2. **PDF级**：最终PDF的数字签名确保文档完整性
3. **审计级**：`transactionId` 作为最终密封的唯一指纹，可用于第三方验证
4. **元数据级**：每个事件的IP、UserAgent、时间戳构成完整的操作上下文

### 4.4 完整验证流程

要验证一份文档的完整性：

1. 从 PDF 证书中提取 `transactionId`
2. 查询对应信封的所有审计日志
3. 找到类型为 `DOCUMENT_COMPLETED` 的审计日志
4. 比对两者的 `transactionId`
5. 按时间线重构所有操作，验证操作链的完整性
