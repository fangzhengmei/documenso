# 审计日志与哈希链联动机制报告

## 1. 审计日志生命周期机制

### 1.1 审计日志数据结构

每个审计日志事件包含以下核心字段（`packages/lib/types/document-audit-logs.ts:725-734`）：

```typescript
{
  id: string;                    // 审计日志唯一ID（数据库自增）
  createdAt: Date;               // 事件发生时间
  envelopeId: string;            // 关联的信封ID（关键关联键）
  name: string | null;           // 操作用户姓名
  email: string | null;          // 操作用户邮箱
  userId: number | null;         // 操作用户ID
  userAgent: string | null;      // 浏览器用户代理
  ipAddress: string | null;      // 操作IP地址
  type: DOCUMENT_AUDIT_LOG_TYPE; // 事件类型
  data: object;                  // 事件特定数据
}
```

### 1.2 审计日志生命周期

**重要修正**：审计日志是**持续增量写入**的，而非仅在文档完成时生成。完整生命周期：

```
┌─────────────────────────────────────────────────────────────┐
│                    审计日志生命周期                           │
├─────────────────────────────────────────────────────────────┤
│  DRAFT 阶段（文档创建→发送前）                                │
│  ├─ DOCUMENT_CREATED      - 文档创建                         │
│  ├─ RECIPIENT_*           - 收件人增删改                      │
│  ├─ FIELD_*               - 字段增删改                        │
│  └─ ENVELOPE_ITEM_*       - 信封项增删改                      │
├─────────────────────────────────────────────────────────────┤
│  PENDING 阶段（文档发送→完成前）                              │
│  ├─ DOCUMENT_SENT         - 文档发送                         │
│  ├─ EMAIL_SENT            - 邮件发送/重发                     │
│  ├─ DOCUMENT_OPENED       - 收件人打开文档                    │
│  ├─ DOCUMENT_VIEWED       - 收件人查看文档                    │
│  ├─ DOCUMENT_FIELD_INSERTED - 字段签署                       │
│  └─ DOCUMENT_RECIPIENT_COMPLETED - 收件人完成签署            │
├─────────────────────────────────────────────────────────────┤
│  COMPLETED/REJECTED 阶段（文档密封后）                        │
│  └─ DOCUMENT_COMPLETED    - 文档密封完成（含transactionId）   │
└─────────────────────────────────────────────────────────────┘
```

**关键特性**：
- 每个审计事件**独立立即写入**数据库，不等待文档完成
- 所有事件通过 `envelopeId` 关联，形成完整的操作链
- `createdAt` 时间戳提供精确的时序保证

### 1.3 事件创建流程

通过 `createDocumentAuditLogData` 工具函数统一创建审计日志数据：

```typescript
// 每次操作都会立即调用此函数生成审计数据
const auditLogData = createDocumentAuditLogData({
  type: DOCUMENT_AUDIT_LOG_TYPE.DOCUMENT_FIELD_INSERTED,
  envelopeId: envelope.id,
  user: { email: recipient.email, name: recipient.name },
  requestMetadata,  // 从当前请求提取 IP 和 UserAgent
  data: {
    recipientEmail: recipient.email,
    recipientId: recipient.id,
    fieldId: updatedField.secondaryId,
    field: { type: FieldType.SIGNATURE, data: signatureValue },
    fieldSecurity: { type: derivedRecipientActionAuth },
  },
});

// 立即写入数据库（通常在事务中）
await prisma.documentAuditLog.create({ data: auditLogData });
```

### 1.4 关键事件类型

审计日志涵盖 50+ 种事件类型，核心包括：

| 事件类型 | 触发时机 | 关键数据 |
|---------|---------|---------|
| `DOCUMENT_CREATED` | 文档创建 | 文档标题、来源 |
| `DOCUMENT_SENT` | 文档发送 | - |
| `DOCUMENT_OPENED` | 文档被打开 | 收件人信息、访问认证 |
| `DOCUMENT_VIEWED` | 文档被查看 | 收件人信息、访问认证 |
| `DOCUMENT_FIELD_INSERTED` | 字段被签署 | 字段ID、字段类型、字段值、签署认证 |
| `DOCUMENT_RECIPIENT_COMPLETED` | 单个收件人完成 | 收件人信息、操作认证 |
| `DOCUMENT_RECIPIENT_REJECTED` | 收件人拒绝文档 | 拒绝原因 |
| `DOCUMENT_COMPLETED` | **文档最终密封** | transactionId（唯一标识） |

### 1.5 字段签名时的事件写入

在 `sign-field-with-token.ts` 中，每次字段签署都会立即写入审计日志：

```typescript
return await prisma.$transaction(async (tx) => {
  // 1. 更新字段状态
  const updatedField = await tx.field.update({
    where: { id: field.id },
    data: { customText, inserted: true },
  });

  // 2. 写入审计日志（立即写入，不等待）
  await tx.documentAuditLog.create({
    data: createDocumentAuditLogData({
      type: DOCUMENT_AUDIT_LOG_TYPE.DOCUMENT_FIELD_INSERTED,
      envelopeId: envelope.id,
      user: { email: recipient.email, name: recipient.name },
      requestMetadata,  // 捕获当前请求上下文
      data: {
        recipientEmail: recipient.email,
        recipientId: recipient.id,
        fieldId: updatedField.secondaryId,
        field: { type: fieldType, data: fieldValue },
        fieldSecurity: derivedRecipientActionAuth
          ? { type: derivedRecipientActionAuth }
          : undefined,
      },
    }),
  });

  return updatedField;
});
```

---

## 2. PDF 密封与哈希链生成

### 2.1 密封触发条件

当所有收件人完成签署时触发密封作业：

```typescript
// 检查所有收件人状态
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
  // 触发异步密封作业
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

`seal-document.handler.ts` 按以下顺序执行：

#### 阶段一：数据准备与验证
1. 查询信封及其关联的收件人、字段、信封项目
2. 验证文档状态必须为 `PENDING`
3. 检查所有必需字段是否已签署
4. 处理被拒绝的文档

#### 阶段二：生成最终审计日志

```typescript
// 在密封前生成 DOCUMENT_COMPLETED 审计日志
const envelopeCompletedAuditLog = createDocumentAuditLogData({
  type: DOCUMENT_AUDIT_LOG_TYPE.DOCUMENT_COMPLETED,
  envelopeId: envelope.id,
  requestMetadata,
  user: null,
  data: {
    transactionId: nanoid(),  // 生成全局唯一交易ID
    ...(isRejected ? { isRejected: true, rejectionReason } : {}),
  },
});
```

**注意**：`transactionId` 是密封时才生成的，作为整个签署流程的最终唯一标识。

#### 阶段三：PDF 装饰与签署

调用 `decorateAndSignPdf` 函数按以下顺序处理：

```
1. PDF 规范化与扁平化
   ├─ pdfDoc.flattenAll() - 扁平化所有图层，防止篡改
   └─ pdfDoc.upgradeVersion('1.7') - 升级到 PDF 1.7 以支持高级签名

2. 添加拒绝印章（如适用）
   └─ addRejectionStampToPdf(pdfDoc, rejectionReason)

3. 附加证书页面（如启用）
   └─ 复制 generateCertificatePdf() 生成的证书页面

4. 附加审计日志页面（如启用）
   └─ 复制 generateAuditLogPdf() 生成的审计日志页面

5. 插入所有字段值
   ├─ V1: legacy_insertFieldInPDF / insertFieldInPDFV1
   └─ V2: insertFieldInPDFV2（覆盖层方式）

6. 最终扁平化处理

7. 数字签名（哈希链核心）
   └─ signPdf({ pdf: pdfDoc }) - 执行 PDF 数字签名（详情见 2.3）

8. 文件存储
   └─ putPdfFileServerSide() - 保存最终签署的 PDF
```

#### 阶段四：数据库事务更新

```typescript
await prisma.$transaction(async (tx) => {
  // 1. 更新所有信封项的文档数据引用（指向新的已签名PDF）
  for (const { oldDocumentDataId, newDocumentDataId } of newDocumentData) {
    await tx.envelopeItem.update({
      where: { envelopeId: envelope.id, documentDataId: oldDocumentDataId },
      data: { documentDataId: newDocumentDataId },
    });
  }

  // 2. 更新信封状态（PENDING → COMPLETED/REJECTED）
  await tx.envelope.update({
    where: { id: envelope.id },
    data: {
      status: finalEnvelopeStatus,
      completedAt: new Date(),
    },
  });

  // 3. 写入 DOCUMENT_COMPLETED 审计日志（含 transactionId）
  await tx.documentAuditLog.create({
    data: envelopeCompletedAuditLog,
  });
});
```

### 2.3 数字签名与哈希链生成

**核心实现位置**：`packages/signing/`

#### 签名架构

```typescript
export const signPdf = async ({ pdf }: SignOptions) => {
  const signer = await getSigner();       // 获取签名器（本地/HSM）
  const tsa = getTimestampAuthority();    // 获取时间戳权威（可选）

  const { bytes } = await pdf.sign({
    signer,
    reason: 'Signed by Documenso',
    location: NEXT_PUBLIC_WEBAPP_URL(),
    contactInfo: NEXT_PUBLIC_SIGNING_CONTACT_INFO(),
    // 签名子过滤器：传统 PKCS7 或现代 ETSI CAdES
    subFilter: NEXT_PRIVATE_USE_LEGACY_SIGNING_SUBFILTER()
      ? 'adbe.pkcs7.detached'
      : 'ETSI.CAdES.detached',
    timestampAuthority: tsa ?? undefined,
    longTermValidation: !!tsa,       // 启用长期验证
    archivalTimestamp: !!tsa,         // 启用归档时间戳
  });

  return bytes;
};
```

#### 本地签名器实现

使用 P12（PKCS#12）证书文件：

```typescript
export const createLocalSigner = async () => {
  const p12 = loadP12();  // 从文件或环境变量加载 P12 证书

  return await P12Signer.create(
    p12,
    env('NEXT_PRIVATE_SIGNING_PASSPHRASE') || '',  // 证书密码
    { buildChain: true }  // 构建证书链
  );
};
```

#### 时间戳权威（TSA）

```typescript
const setupTimestampAuthorities = once(() => {
  const timestampAuthority = NEXT_PRIVATE_SIGNING_TIMESTAMP_AUTHORITY();

  if (!timestampAuthority) {
    return null;
  }

  // 支持多个 TSA 服务器，随机选择
  const timestampAuthorities = timestampAuthority
    .trim()
    .split(',')
    .filter(Boolean)
    .map((url) => new HttpTimestampAuthority(url));

  return timestampAuthorities;
});
```

#### 哈希链校验层级

| 层级 | 技术 | 作用 |
|-----|------|------|
| **L1 - PDF 数字签名** | PKCS#7 / CAdES | 确保 PDF 文件内容未被篡改 |
| **L2 - 证书链验证** | X.509 证书链 | 验证签名者身份的合法性 |
| **L3 - 时间戳** | RFC 3161 TSA | 证明签名发生的准确时间 |
| **L4 - 长期验证（LTV）** | CRL/OCSP 嵌入 | 确保证书在未来仍然有效 |
| **L5 - 审计日志链** | transactionId + 时序 | 完整操作链的可追溯性 |

### 2.4 证书页面生成

证书页面包含所有签署者的详细信息和哈希验证数据：

```typescript
export type CertificateRecipient = {
  id: number;
  name: string;
  email: string;
  role: RecipientRole;
  signingStatus: SigningStatus;
  signatureField?: {
    id: number;
    secondaryId: string;  // 字段外部唯一标识
    recipientId: number;
    signature?: {
      signatureImageAsBase64: string | null;
      typedSignature: string | null;
    } | null;
  };
  authLevel: string;  // 认证级别
  logs: {
    emailed: BaseAuditLog | null;    // 邮件发送记录
    sent: BaseAuditLog | null;       // 发送记录
    opened: BaseAuditLog | null;     // 打开记录
    completed: BaseAuditLog | null;  // 完成记录
    rejected: BaseAuditLog | null;   // 拒绝记录
  };
};
```

证书页面还包含：
- 每个签署者的签名图像或手写签名
- 签署时的 IP 地址和设备信息（UserAgent 解析）
- 签署时间戳（带时区）
- QR 码指向公开验证页面
- `envelopeId` 作为页脚标识

---

## 3. 跨收件人时间线聚合

### 3.1 聚合键：envelopeId

所有审计日志事件通过 `envelopeId` 字段关联到同一文档。这是跨收件人时间线聚合的核心机制。

### 3.2 查询聚合方法

查询文档所有审计日志（按时间排序）：

```typescript
const auditLogs = await prisma.documentAuditLog.findMany({
  where: { envelopeId: envelope.id },
  orderBy: { createdAt: 'asc' },  // 严格按时间排序
});
```

### 3.3 时间线聚合的典型场景

#### 场景 A：完整签署流程时间线

```
时间线顺序（按 createdAt 排序）：
├─ T0: DOCUMENT_CREATED - 创建者创建文档
├─ T1: RECIPIENT_CREATED - 添加收件人A
├─ T2: RECIPIENT_CREATED - 添加收件人B
├─ T3: FIELD_CREATED - 为收件人A创建签名字段
├─ T4: FIELD_CREATED - 为收件人B创建签名字段
├─ T5: DOCUMENT_SENT - 文档发送（状态 DRAFT → PENDING）
├─ T6: DOCUMENT_OPENED - 收件人A打开文档
├─ T7: DOCUMENT_VIEWED - 收件人A查看文档
├─ T8: DOCUMENT_FIELD_INSERTED - 收件人A签署字段1
├─ T9: DOCUMENT_FIELD_INSERTED - 收件人A签署字段2
├─ T10: DOCUMENT_RECIPIENT_COMPLETED - 收件人A完成签署
├─ T11: DOCUMENT_OPENED - 收件人B打开文档
├─ T12: DOCUMENT_VIEWED - 收件人B查看文档
├─ T13: DOCUMENT_FIELD_INSERTED - 收件人B签署字段
├─ T14: DOCUMENT_RECIPIENT_COMPLETED - 收件人B完成签署
└─ T15: DOCUMENT_COMPLETED - 文档密封（含 transactionId，状态 PENDING → COMPLETED）
```

#### 场景 B：顺序签署模式时间线

当启用顺序签署（`DocumentSigningOrder.SEQUENTIAL`）时，时间线会按收件人顺序分段：

```
时间线顺序：
├─ ...
├─ Tn:   DOCUMENT_RECIPIENT_COMPLETED - 收件人1完成
├─ Tn+1: RECIPIENT_UPDATED - 更新收件人2发送状态
├─ Tn+2: EMAIL_SENT - 向收件人2发送签署邮件
├─ Tn+3: DOCUMENT_OPENED - 收件人2打开文档
├─ ...
```

### 3.4 审计日志格式化与展示

`formatDocumentAuditLogAction` 函数将原始审计日志转换为人类可读的描述：

```typescript
const { prefix, description } = formatDocumentAuditLogAction(
  i18n,
  auditLog,
  currentUserId
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
用户操作（任意时间点）
    ↓
创建审计日志（带 envelopeId、createdAt、请求元数据）
    ↓
立即写入 documentAuditLog 表（原子操作）
    ↓
（重复：每个操作都独立写入审计日志）
    ↓
所有收件人完成签署
    ↓
触发 seal-document 作业
    ↓
├─ 读取该 envelopeId 下的所有审计日志
├─ 生成审计日志 PDF 页面（时间线聚合）
├─ 生成证书 PDF 页面（含签署者详情、QR 码）
├─ 将所有页面合并到原始 PDF
├─ 执行 PDF 数字签名（生成哈希链）
│   ├─ 计算文档内容哈希
│   ├─ 使用私钥签名
│   ├─ 可选：添加 TSA 时间戳
│   └─ 嵌入证书链和 LTV 信息
└─ 写入 DOCUMENT_COMPLETED 审计日志（含 transactionId）
    ↓
更新信封状态：PENDING → COMPLETED
    ↓
文档密封完成
```

### 4.2 关键联动点

| 联动点 | 位置 | 作用 |
|-------|------|------|
| `envelopeId` | 所有审计日志 | 跨操作、跨收件人的关联键 |
| `createdAt` | 所有审计日志 | 时间排序，构建不可篡改的时间线 |
| `transactionId` | `DOCUMENT_COMPLETED` 事件 | 整个签署流程的最终唯一标识 |
| `requestMetadata` | 所有事件 | 捕获 IP、UserAgent，用于审计追踪 |
| `fieldSecurity` | 字段插入事件 | 记录签署时的认证方式（2FA 等） |
| `secondaryId` | 签名字段 | 字段级别的哈希标识，显示在证书上 |

### 4.3 防篡改保障层级

1. **数据库级**：审计日志为追加写入模式，不支持更新/删除
2. **PDF 数字签名级**：PKCS#7/CAdES 签名确保 PDF 文件内容未被篡改
3. **证书链级**：X.509 证书链验证签名者身份的合法性
4. **时间戳级**：RFC 3161 TSA 证明签名发生的准确时间
5. **长期验证级**：LTV 嵌入 CRL/OCSP 信息，确保证书在未来仍然有效
6. **审计日志链级**：`transactionId` + 时序形成完整操作链的可追溯性

### 4.4 完整验证流程

要验证一份文档的完整性：

1. **从 PDF 提取数字签名**
   - 使用 PDF 阅读器（如 Adobe Acrobat）验证签名有效性
   - 检查签名证书的信任链
   - 验证时间戳（如果有）

2. **从证书页面提取标识**
   - 提取 `envelopeId`（页脚）
   - 提取字段 `secondaryId`（签署者详情）
   - 扫描 QR 码访问公开验证页面

3. **查询审计日志链**
   - 使用 `envelopeId` 查询所有审计日志
   - 找到类型为 `DOCUMENT_COMPLETED` 的审计日志
   - 提取 `transactionId` 作为最终标识

4. **交叉验证**
   - 验证审计日志时间线的完整性（无缺失、无时间倒流）
   - 验证每个签署操作的认证级别（`fieldSecurity`）
   - 比对证书上的签署时间与审计日志中的 `createdAt`

5. **完整链验证**
   ```
   PDF 签名哈希 → 证书 transactionId → 审计日志链 → 每个操作的元数据
   ```
