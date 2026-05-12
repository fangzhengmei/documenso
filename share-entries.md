# Documenso 三种共享入口签署引擎实现机制

## 概述

Documenso 签署系统支持三种共享入口方式，**共用同一套底层数据模型和部分核心能力，但在实现路径、鉴权边界、执行流程上存在本质差异**：

1. **模板批量复制** - 基于模板创建多个待签署文档（先创建，后签署）
2. **直链邮件单人签** - 通过模板直链即时完成签署（创建与签署原子化）
3. **嵌入外部站点就地签** - 第三方系统通过预签名API创建并管理签署流程

---

## 一、核心结论与实现差异

### 1.1 关键架构差异

| 维度 | 模板批量复制 | 直链即时签署 | 嵌入预签名 |
|------|-------------|-------------|-----------|
| **签署时机** | 先创建文档，收件人后续签署 | 创建文档同时完成直链收件人签名 | 第三方控制签署时机 |
| **核心函数** | `createDocumentFromTemplate` | `createDocumentFromDirectTemplate` | `createEmbeddingPresignToken` + 嵌入API |
| **签署引擎** | 调用 `signFieldWithToken` | 内部独立签名逻辑，**不共用 signFieldWithToken** | 最终调用 `signFieldWithToken` |
| **事务边界** | 创建文档为单一事务 | 创建+签名在同一事务内完成 | 多次API调用多事务 |

### 1.2 直链即时签署的独立实现

**文件位置**: `packages/lib/server-only/template/create-document-from-direct-template.ts`

**核心设计特点**:

1. **原子化操作**: 模板克隆与直链收件人签名在**同一个数据库事务**中完成
2. **独立签名逻辑**: 内部直接处理字段签名，绕过 `signFieldWithToken` 函数
3. **前置验证**: 在创建文档前完成所有字段签名值的验证
4. **状态直接终态**: 直链收件人直接进入 `SigningStatus.SIGNED` 状态

```typescript
// 直链签署核心流程（独立实现）
const createDocumentFromDirectTemplate = async () => {
  // 1. 验证直链模板token和有效性
  // 2. 执行访问鉴权验证（有能力边界限制）
  // 3. 预验证所有签名字段值（在创建文档前）
  // 4. 并行处理所有字段：分离签名字段和非签名字段
  // 5. 复制PDF资源 + 创建Envelope
  // 6. 创建其他收件人（NOT_SIGNED状态）
  // 7. 创建直链收件人（SIGNED状态 + 字段inserted=true）
  // 8. 创建签名记录 Signature
  // 9. 生成审计日志（DOCUMENT_CREATED + DOCUMENT_FIELD_INSERTED）
  // 10. 触发邮件发送给后续收件人
}
```

---

## 二、直链访问鉴权能力边界

### 2.1 明确的鉴权支持矩阵

**文件位置**: `packages/lib/server-only/template/create-document-from-direct-template.ts:169-177`

```typescript
// 直链签署访问鉴权的硬编码能力边界
const isAccessAuthValid = match(derivedRecipientAccessAuth.at(0))
  .with(DocumentAccessAuth.ACCOUNT, () => user && user?.email === directRecipientEmail)
  .with(DocumentAccessAuth.TWO_FACTOR_AUTH, () => false) // 明确不支持！
  .with(undefined, () => true)
  .exhaustive();
```

| 鉴权方式 | 模板普通签署 | 直链即时签署 | 备注 |
|---------|-------------|-------------|------|
| **无鉴权 (undefined)** | ✓ 支持 | ✓ 支持 | 默认行为 |
| **ACCOUNT (账号验证)** | ✓ 支持 | ✓ 部分支持 | 仅验证登录用户邮箱与收件人邮箱一致 |
| **TWO_FACTOR_AUTH (双因素)** | ✓ 支持 | ✗ **不支持** | 硬编码返回 false |
| **PASSKEY (生物密钥)** | ✓ 支持 | ✗ **不支持** | 不在match分支中 |
| **PASSWORD (密码)** | ✓ 支持 | ✗ **不支持** | 不在match分支中 |

### 2.2 直链签署动作鉴权

直链签署的**动作鉴权**（ACTION）仅对**签名字段**生效：

**文件位置**: `packages/lib/server-only/document/validate-field-auth.ts:29-31`

```typescript
// 非签名字段绕过动作鉴权
if (field.type !== FieldType.SIGNATURE) {
  return undefined; // 直接跳过，不验证
}
```

**动作鉴权支持情况**:
- ✅ SIGNATURE / FREE_SIGNATURE 字段：执行完整动作鉴权
- ⚠️ NAME / EMAIL / DATE / TEXT / NUMBER 等：**跳过动作鉴权**
- ⚠️ CHECKBOX / RADIO / DROPDOWN：**跳过动作鉴权**

---

## 三、三条链路数据契约传递对照表

### 3.1 数据流转全景对比

| 数据项 | 模板克隆 (createDocumentFromTemplate) | 直链即时签署 (createDocumentFromDirectTemplate) | 嵌入预签名 (createEmbeddingDocument) |
|--------|--------------------------------------|-----------------------------------------------|--------------------------------------|
| **Envelope 创建** | ✓ 基于模板完整复制 | ✓ 基于模板复制，但source字段不同 | ✗ 不基于模板，从零创建 |
| **source 字段** | `DocumentSource.TEMPLATE` | `DocumentSource.TEMPLATE_DIRECT_LINK` | N/A (未设置特殊source) |
| **Envelope.authOptions** | ✓ 完全继承模板的全局鉴权 | ✓ 完全继承模板的全局鉴权 | ✓ 调用方自定义（默认空） |
| **Recipient 数量** | 模板所有收件人 + 默认收件人 | 模板所有收件人（直链收件人特殊处理） | 调用方传入的收件人数组 |
| **Recipient.token** | ✓ 每个收件人生成新nanoid | ✓ 每个收件人生成新nanoid | ✓ 由createEnvelope内部生成 |
| **Recipient.signingStatus** | 全部 NOT_SIGNED | 直链收件人=**SIGNED**<br>其他收件人=NOT_SIGNED | 全部 NOT_SIGNED |
| **Recipient.authOptions** | ✓ 继承模板收件人配置 | ✓ 继承模板收件人配置 | ✓ 调用方传入配置 |
| **Recipient.signedAt** | null | 设置为当前时间 | null |
| **Field 复制** | ✓ 所有字段完整映射到新收件人 | ✓ 所有字段映射，但直链字段已签名 | ✓ 调用方传入字段配置 |
| **Field.inserted** | false | 直链收件人字段=**true**<br>其他收件人字段=false | false |
| **Field.customText** | 空或预填充值 | 直链字段已填充签名值 | 空 |
| **Field.fieldMeta** | ✓ 完整继承 | ✓ 完整继承 | ✓ 调用方传入 |
| **Signature 记录** | 未创建 | ✓ 直链收件人签名字段已创建Signature | 未创建 |
| **EnvelopeItem/PDF** | ✓ 复制模板PDF文件实体 | ✓ 复制模板PDF文件实体 | ✓ 引用已上传的documentDataId |
| **DocumentMeta** | ✓ 完整继承模板配置 | ✓ 完整继承模板配置 | ✓ 调用方传入meta |
| **审计日志** | 仅 DOCUMENT_CREATED | DOCUMENT_CREATED<br>DOCUMENT_OPENED<br>**DOCUMENT_FIELD_INSERTED x N**<br>DOCUMENT_RECIPIENT_COMPLETED | 未创建（需后续调用生成） |
| **邮件触发** | 外部调用 sendDocument | ✓ 函数内部直接触发 | 未触发（需后续调用） |

### 3.2 核心字段传递路径详解

#### 模板克隆链路
```
模板 Envelope (TEMPLATE)
    ├─> authOptions ────────────────────> 新文档 Envelope.authOptions (完整继承)
    ├─> documentMeta ───────────────────> 新文档 DocumentMeta (完整继承)
    ├─> recipients[]
    │     ├─> token 生成 (nanoid)
    │     ├─> authOptions ─────────────> 新收件人.authOptions (继承)
    │     ├─> signingStatus = NOT_SIGNED
    │     └─> fields[]
    │           ├─> inserted = false
    │           ├─> customText = '' 或预填充值
    │           └─> fieldMeta (完整继承)
    └─> envelopeItems[]
          └─> PDF二进制复制 + 创建新 DocumentData
```

#### 直链即时签署链路
```
模板 Envelope (TEMPLATE)
    ├─> authOptions ────────────────────> 新文档 Envelope.authOptions (完整继承)
    ├─> documentMeta ───────────────────> 新文档 DocumentMeta (完整继承)
    ├─> source = TEMPLATE_DIRECT_LINK
    ├─> recipients[]
    │     ├─> [直链收件人]
    │     │     ├─> token 生成 (nanoid)
    │     │     ├─> signingStatus = SIGNED
    │     │     ├─> signedAt = 当前时间
    │     │     └─> fields[]
    │     │           ├─> inserted = true
    │     │           ├─> customText = 用户签名值
    │     │           ├─> Signature 记录已创建 ✓
    │     │           └─> fieldMeta (完整继承)
    │     └─> [其他收件人]
    │           ├─> token 生成 (nanoid)
    │           ├─> signingStatus = NOT_SIGNED
    │           └─> fields[].inserted = false
    └─> envelopeItems[]
          └─> PDF二进制复制 + 创建新 DocumentData
```

#### 嵌入预签名链路
```
第三方系统调用
    ├─> Presign Token (JWT) 验证
    └─> createEnvelope 参数
          ├─> type = DOCUMENT
          ├─> title, externalId
          ├─> recipients[] (调用方传入)
          │     ├─> email, name, role
          │     ├─> authOptions (调用方传入)
          │     └─> fields[] (位置+类型配置)
          ├─> envelopeItems[]
          │     └─> documentDataId (引用已上传)
          └─> meta (签署配置)
```

---

## 四、共用核心签署引擎与差异点

### 4.1 签名字段处理的共用与差异

**文件位置**: `packages/lib/server-only/field/sign-field-with-token.ts`

| 场景 | 使用 signFieldWithToken | 备注 |
|------|------------------------|------|
| 模板克隆后收件人签署 | ✓ | 标准签署流程 |
| 直链即时签署 | ✗ | 内部独立实现字段验证和Signature创建 |
| 普通文档邮件签署 | ✓ | 标准签署流程 |
| 嵌入签署最终签名动作 | ✓ | 嵌入渲染后调用标准签名 |

### 4.2 signFieldWithToken 标准流程

```typescript
export const signFieldWithToken = async () => {
  // 1. 查询 recipient (by token)
  // 2. 查询 field (by fieldId)
  // 3. 验证 envelope 状态 (必须 PENDING)
  // 4. 验证 recipient 未过期
  // 5. 验证 recipient 未签署
  // 6. 验证 field 未 inserted
  // 7. 按字段类型验证值格式 (NUMBER/TEXT/CHECKBOX等)
  // 8. 调用 validateFieldAuth 执行动作鉴权
  // 9. 事务更新 field.inserted = true, customText = 值
  // 10. 签名字段创建/更新 Signature 记录
  // 11. 写入 DOCUMENT_FIELD_INSERTED 审计日志
};
```

### 4.3 直链签署的内置签名逻辑（不共用）

```typescript
// createDocumentFromDirectTemplate 内部签名处理
// 注意：这是在 Envelope 创建前的内存中处理！

const fieldsToProcess = directTemplateRecipient.fields.filter(...);

const createDirectRecipientFieldArgs = await Promise.all(
  fieldsToProcess.map(async (templateField) => {
    // 1. 验证必填字段必须有值
    // 2. NAME字段特殊处理：值作为recipient name
    // 3. 独立调用 validateFieldAuth (仅签名字段)
    // 4. 分离签名字段和非签名字段
    // 5. DATE字段自动填充当前时间
    
    return {
      templateField,
      customText,
      derivedRecipientActionAuth,
      signature: isSignatureField ? { ... } : null
    };
  })
);

// 后续在事务中：
// - 非签名字段：批量 createMany (inserted=true/false 依版本)
// - 签名字段：循环逐个创建 field + 关联 signature
```

---

## 五、共用鉴权策略与边界

### 5.1 统一的鉴权层次结构（所有入口共用）

**文件位置**: `packages/lib/utils/document-auth.ts`

```typescript
export const extractDocumentAuthMethods = ({ documentAuth, recipientAuth }) => {
  // 优先级：收件人级别 > 文档全局级别
  // 1. 文档全局访问鉴权 globalAccessAuth
  // 2. 文档全局动作鉴权 globalActionAuth
  // 3. 收件人访问鉴权 accessAuth (覆盖全局)
  // 4. 收件人动作鉴权 actionAuth (覆盖全局)
};
```

### 5.2 各入口的鉴权验证点

| 验证点 | 模板克隆 | 直链签署 | 嵌入预签名 |
|--------|---------|---------|-----------|
| 创建时访问鉴权 | - (登录态创建) | ✓ (ACCOUNT或无) | ✓ (Presign Token) |
| 签署时访问鉴权 | ✓ (完整支持) | - (已在创建时完成) | ✓ (完整支持) |
| 字段动作鉴权 | ✓ (签名字段) | ✓ (仅签名字段) | ✓ (仅签名字段) |
| 签署顺序控制 | ✓ | ✓ | ✓ |

---

## 六、嵌入预签名的安全机制

### 6.1 令牌生命周期

**文件位置**: `packages/lib/server-only/embedding-presign/create-embedding-presign-token.ts`

```
第三方系统
    ├─> 1. 使用 API Key 请求 Presign Token
    └─> 2. 服务端签发 JWT
          ├─> aud = teamId 或 userId
          ├─> sub = apiTokenId
          ├─> scope (可选)
          ├─> iat = 签发时间
          ├─> exp = 默认1小时后
          └─> 签名密钥 = API Key 本身

    ├─> 3. 第三方使用 JWT 调用嵌入API (Authorization: Bearer <JWT>)
    └─> 4. 服务端验证：解码JWT -> 查询API Token -> 验证签名 -> 检查过期
```

### 6.2 关键安全设计

1. **自包含签名**: 使用 API Key 本身作为 JWT 签名密钥，无需额外存储
2. **短时有效**: 默认1小时过期，开发环境可设置为0（立即过期用于测试）
3. **范围限制**: 支持 `scope` 参数限制令牌权限范围
4. **无状态验证**: JWT 自包含所有验证信息，验证时不依赖额外数据库查询（除获取API Key）

---

## 七、统一数据模型

### 7.1 Envelope (核心载体) - 100% 共用

```prisma
model Envelope {
  id          String          @id @unique
  type        EnvelopeType    // DOCUMENT | TEMPLATE
  status      DocumentStatus  // DRAFT | PENDING | COMPLETED
  source      DocumentSource? // TEMPLATE | TEMPLATE_DIRECT_LINK
  authOptions Json            // 全局鉴权配置
  
  recipients  Recipient[]     // 一对多
  envelopeItems EnvelopeItem[] // PDF文件
  documentMeta DocumentMeta?  // 签署配置
  directLink  DirectLink?     // 直链签署专用配置
}
```

### 7.2 Recipient (收件人) - 100% 共用

```prisma
model Recipient {
  id            Int            @id @default(autoincrement())
  envelopeId    String
  token         String         @unique  // 访问令牌，所有方式共用
  signingStatus SigningStatus  // NOT_SIGNED | SIGNED | REJECTED
  signedAt      DateTime?      // 直链签署立即设置
  authOptions   Json           // 收件人级别鉴权
  fields        Field[]        // 一对多
}
```

### 7.3 Field / Signature - 100% 共用

字段和签名模型三种方式完全一致，区别仅在于**创建时机**和**inserted初始值**。

---

## 八、调用关系总览

```
                        ┌─────────────────────┐
                        │   三种签署入口      │
                        └─────────┬───────────┘
                                  │
          ┌───────────────────────┼───────────────────────┐
          │                       │                       │
          ▼                       ▼                       ▼
┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐
│  模板批量复制     │  │  直链即时签署     │  │  嵌入预签名创建   │
│ createDocument-  │  │ createDocument-   │  │ createEmbedding-  │
│ FromTemplate      │  │ FromDirectTemplate│  │ Document          │
└─────────┬─────────┘  └─────────┬─────────┘  └─────────┬─────────┘
          │                       │                       │
          │ 创建Envelope          │ 创建Envelope          │ 验证Presign Token
          │ 创建Recipients        │ 创建Recipients        │ ────────────────
          │ 创建Fields            │ 创建Fields            │ createEnvelope
          │ signingStatus=NOT_SIGNED │ 直链收件人=SIGNED   │
          │ inserted=false        │ 直链字段inserted=true │
          │                       │ 创建Signature记录 ✓   │
          │                       │ 写入多类审计日志 ✓     │
          └───────────────────────┼───────────────────────┘
                                  │
                                  ▼
                        ┌─────────────────────┐
                        │   后续签署动作      │
                        │ signFieldWithToken │
                        │ (直链跳过此步)       │
                        └─────────┬───────────┘
                                  │
                                  ▼
                        ┌─────────────────────┐
                        │   共用鉴权策略      │
                        │ isRecipient-        │
                        │ Authorized          │
                        └─────────┬───────────┘
                                  │
                                  ▼
                        ┌─────────────────────┐
                        │   统一数据模型      │
                        │ Envelope/Recipient/│
                        │ Field/Signature     │
                        └─────────────────────┘
```

---

## 九、关键注意事项

### 9.1 直链签署使用限制

1. **不支持双因素认证**: `TWO_FACTOR_AUTH` 硬编码返回 false，配置了该鉴权的模板无法通过直链签署
2. **仅签名字段验证动作鉴权**: 其他字段类型绕过动作鉴权
3. **无法中途变更**: 直链签署是原子操作，创建即完成，无法撤销或修改
4. **版本差异处理**: V2 版本处理只读字段和预填充字段的逻辑不同

### 9.2 代码复用率统计

| 模块 | 模板克隆 | 直链签署 | 嵌入预签名 |
|------|---------|---------|-----------|
| Envelope数据模型 | 100% | 100% | 100% |
| Recipient数据模型 | 100% | 100% | 100% |
| Field数据模型 | 100% | 100% | 100% |
| extractDocumentAuthMethods | 100% | 100% | 100% |
| isRecipientAuthorized | 100% | ~80% (访问鉴权分支受限) | 100% |
| validateFieldAuth | 100% | 100% | 100% |
| signFieldWithToken | 100% | 0% (内部独立实现) | 100% |
| PDF复制逻辑 | 100% | ~90% (使用 putPdfFileServerSide vs putNormalizedPdfFileServerSide) | N/A (引用已有) |

---

**文档版本**: 2.0  
**最后更新**: 2024  
**关键修正**: 明确直链签署独立实现路径、补充鉴权能力边界、新增三条链路数据契约传递对照表
