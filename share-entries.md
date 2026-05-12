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
| **入口层代码复用** | 独立实现（不调用createEnvelope） | 独立实现（不调用createEnvelope） | 调用公共 `createEnvelope` |

### 1.2 直链即时签署的独立实现

**文件位置**: `packages/lib/server-only/template/create-document-from-direct-template.ts`

**核心设计特点**:

1. **原子化操作**: 模板克隆与直链收件人签名在**同一个数据库事务**中完成
2. **独立签名逻辑**: 内部直接处理字段签名，绕过 `signFieldWithToken` 函数
3. **前置验证**: 在创建文档前完成所有字段签名值的验证
4. **状态直接终态**: 直链收件人直接进入 `SigningStatus.SIGNED` 状态
5. **不使用公共 createEnvelope**: 完全独立的 Envelope 创建逻辑

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

## 二、访问鉴权与动作鉴权能力边界

### 2.1 访问鉴权支持矩阵（Recipient级别）

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

### 2.2 动作鉴权支持矩阵（Field级别）

**文件位置**: `packages/lib/server-only/document/validate-field-auth.ts:28-31`

```typescript
// 仅SIGNATURE字段执行动作鉴权，包括FREE_SIGNATURE在内的其他字段全部跳过
if (field.type !== FieldType.SIGNATURE) {
  return undefined; // 直接跳过，不验证
}
```

| 字段类型 | 是否执行动作鉴权 | 备注 |
|---------|----------------|------|
| **SIGNATURE** | ✅ 执行 | 唯一受动作鉴权保护的字段类型 |
| **FREE_SIGNATURE** | ❌ 跳过 | 与普通文本字段一样，不做鉴权 |
| NAME | ❌ 跳过 | |
| EMAIL | ❌ 跳过 | |
| DATE | ❌ 跳过 | |
| TEXT | ❌ 跳过 | |
| NUMBER | ❌ 跳过 | |
| CHECKBOX | ❌ 跳过 | |
| RADIO | ❌ 跳过 | |
| DROPDOWN | ❌ 跳过 | |

**关键结论**: **动作鉴权仅对 `SIGNATURE` 类型字段生效**，包括 `FREE_SIGNATURE` 在内的所有其他字段类型都会直接绕过鉴权检查。

---

## 三、三条链路数据契约传递对照表

### 3.1 数据流转全景对比（逐行代码核对版）

| 数据项 | 模板克隆 (createDocumentFromTemplate) | 直链即时签署 (createDocumentFromDirectTemplate) | 嵌入预签名 (createEmbeddingDocument) | 代码验证 |
|--------|--------------------------------------|-----------------------------------------------|--------------------------------------|---------|
| **Envelope 创建方式** | 独立实现，不调用 createEnvelope | 独立实现，不调用 createEnvelope | 调用公共 `createEnvelope` 函数 | ✓ line 33 |
| **source 字段值** | `DocumentSource.TEMPLATE` | `DocumentSource.TEMPLATE_DIRECT_LINK` | `DocumentSource.DOCUMENT` | ✓ line 343<br>✓ line 535<br>✓ line 319 |
| **Envelope.authOptions** | ✓ 完全继承模板的 globalAccessAuth/globalActionAuth | ✓ 完全继承模板的 globalAccessAuth/globalActionAuth | ❌ **空数组（路由不支持传参）** | ✓ line 533-536<br>✓ line 338<br>✓ 路由input无authOptions |
| **默认收件人** | ✅ 添加 (模板收件人 + 团队默认收件人) | ❌ 不添加 (仅复制模板收件人) | ✅ 添加 (调用方传入 + 团队默认收件人) | ✓ line 408-430<br>✓ line 358-369 |
| **Recipient.token 生成** | ✓ `nanoid()` 函数生成 | ✓ `nanoid()` 函数生成 | ✓ `nanoid()` 函数生成 | ✓ line 404<br>✓ line 417 |
| **Recipient.signingStatus** | 全部 NOT_SIGNED (CC除外) | 直链收件人=**SIGNED**<br>其他收件人=NOT_SIGNED | 全部 NOT_SIGNED (CC除外) | ✓ line 569<br>✓ line 419 |
| **Recipient.authOptions** | ✓ 继承模板收件人 authOptions | ✓ 继承模板收件人 authOptions | ❌ 路由不支持传 authOptions，故为空 | ✓ line 418<br>✓ types.ts line 34-53 |
| **Recipient.signedAt** | null | 设置为当前时间 (仅直链收件人) | null | ✓ line 419<br>✓ 无显式设置 |
| **Field 复制策略** | 所有模板收件人字段完整映射 | 所有模板字段映射，但直链字段已签名 | 仅调用方传入的字段配置 | ✓ line 632-689 |
| **Field.inserted 初始值** | false | 直链收件人字段=**true**<br>其他收件人字段=false | false | ✓ line 644<br>✓ line 405 |
| **Field.customText** | 空或预填充值 | 直链字段已填充签名值 | 空字符串 | ✓ line 643<br>✓ line 404 |
| **Field.fieldMeta** | ✓ 完整继承模板字段元数据 | ✓ 完整继承模板字段元数据 | ✓ 调用方传入 | ✓ line 645<br>✓ line 406 |
| **动作鉴权保护字段** | 仅 SIGNATURE | 仅 SIGNATURE | 仅 SIGNATURE | ✓ validate-field-auth.ts line 29 |
| **Signature 记录创建** | 未创建，待后续签署时生成 | ✓ 直链收件人签名字段已创建Signature | 未创建，待后续签署时生成 | ✓ line 459-497<br>✓ 签名字段单独处理 |
| **PDF复制策略** | 使用 `putNormalizedPdfFileServerSide` 复制 | 使用 `putPdfFileServerSide` 复制 | 不复制，直接引用已有 documentDataId | ✓ line 479<br>✓ line 285<br>✓ line 51 |
| **DocumentMeta 继承** | ✓ 完整继承模板配置 | ✓ 完整继承模板配置 | ✓ 调用方传入 meta 参数 | ✓ line 507-525<br>✓ line 304-306 |
| **审计日志类型与数量** | 仅 1 条：DOCUMENT_CREATED | 至少 4 条：<br>DOCUMENT_CREATED +<br>DOCUMENT_OPENED +<br>DOCUMENT_FIELD_INSERTED × N +<br>DOCUMENT_RECIPIENT_COMPLETED | 至少 1 条：DOCUMENT_CREATED<br>(createEnvelope内部创建) | ✓ line 698-711<br>✓ line 513-611<br>✓ line 565-581 |
| **邮件触发时机** | ❌ 函数内部不触发，需外部调用 sendDocument | ✅ 函数内部直接调用 sendDocument | ❌ 不触发 | ✓ line 727<br>✓ 无调用 |
| **附件复制** | ✅ 模板附件 + 调用方传入附件 | ✅ 仅复制模板附件 | ❌ **路由不支持传参** | ✓ line 713-738<br>✓ types.ts line 29-71 |

### 3.2 核心字段传递路径详解

#### 模板克隆链路
```
模板 Envelope (TEMPLATE)
    ├─> authOptions ────────────────────> 新文档 Envelope.authOptions (完整继承 globalAccessAuth/globalActionAuth)
    ├─> documentMeta ───────────────────> 新文档 DocumentMeta (完整继承)
    ├─> source = DocumentSource.TEMPLATE
    ├─> recipients[]
    │     ├─> 模板收件人映射
    │     ├─> + 团队默认收件人 (defaultRecipientsFinal)
    │     ├─> token = nanoid()
    │     ├─> authOptions 继承模板收件人配置
    │     ├─> signingStatus = NOT_SIGNED (CC=SIGNED)
    │     └─> fields[]
    │           ├─> inserted = false
    │           ├─> customText = '' 或预填充值
    │           ├─> fieldMeta 完整继承
    │           └─> 动作鉴权：仅 SIGNATURE 字段受保护
    ├─> envelopeItems[]
    │     └─> PDF二进制复制：putNormalizedPdfFileServerSide
    ├─> 附件：模板附件 + 调用方传入附件
    └─> 审计日志: DOCUMENT_CREATED × 1
```

#### 直链即时签署链路
```
模板 Envelope (TEMPLATE)
    ├─> authOptions ────────────────────> 新文档 Envelope.authOptions (完整继承)
    ├─> documentMeta ───────────────────> 新文档 DocumentMeta (完整继承)
    ├─> source = DocumentSource.TEMPLATE_DIRECT_LINK
    ├─> recipients[]
    │     ├─> [直链收件人]
    │     │     ├─> token = nanoid()
    │     │     ├─> signingStatus = SIGNED
    │     │     ├─> signedAt = 当前时间
    │     │     └─> fields[]
    │     │           ├─> inserted = true
    │     │           ├─> customText = 用户签名值
    │     │           ├─> Signature 记录已创建 ✓
    │     │           ├─> fieldMeta 完整继承
    │     │           └─> 动作鉴权：仅 SIGNATURE 字段验证
    │     └─> [其他收件人]
    │           ├─> token = nanoid()
    │           ├─> signingStatus = NOT_SIGNED
    │           ├─> authOptions 继承模板收件人配置
    │           └─> fields[].inserted = false
    ├─> envelopeItems[]
    │     └─> PDF二进制复制：putPdfFileServerSide
    ├─> 附件：仅复制模板附件
    ├─> 审计日志: DOCUMENT_CREATED + DOCUMENT_OPENED + DOCUMENT_FIELD_INSERTED × N + DOCUMENT_RECIPIENT_COMPLETED
    └─> 内部触发 sendDocument 给后续收件人
```

#### 嵌入预签名链路
```
第三方系统调用
    ├─> Presign Token (JWT) 验证
    └─> 调用 createEnvelope
          ├─> type = DOCUMENT
          ├─> source = DocumentSource.DOCUMENT
          ├─> title, externalId 由调用方传入
          ├─> authOptions = 空数组（路由不支持传参）
          ├─> recipients[] (调用方传入)
          │     ├─> email, name, role, signingOrder
          │     ├─> ❌ 路由不支持传 authOptions/accessAuth/actionAuth
          │     ├─> token = nanoid()
          │     ├─> signingStatus = NOT_SIGNED
          │     └─> fields[] (位置+类型配置，无预填充值)
          │           ├─> inserted = false
          │           ├─> customText = ''
          │           ├─> fieldMeta 调用方传入
          │           └─> 动作鉴权：仅 SIGNATURE 字段受保护
          ├─> envelopeItems[]
          │     └─> documentDataId = 直接引用已上传资源，不复制
          ├─> + 团队默认收件人
          ├─> ❌ 路由不支持传 attachments 参数
          └─> 审计日志: DOCUMENT_CREATED × 1 (createEnvelope内部创建)
```

---

## 四、共用核心签署引擎与差异点

### 4.1 签名字段处理的共用与差异

**文件位置**: `packages/lib/server-only/field/sign-field-with-token.ts`

| 场景 | 使用 signFieldWithToken | 动作鉴权生效范围 |
|------|------------------------|----------------|
| 模板克隆后收件人签署 | ✓ | 仅 SIGNATURE |
| 直链即时签署 | ✗ 内部独立实现 | 仅 SIGNATURE |
| 普通文档邮件签署 | ✓ | 仅 SIGNATURE |
| 嵌入签署最终签名动作 | ✓ | 仅 SIGNATURE |

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
  // 8. 调用 validateFieldAuth 执行动作鉴权（仅SIGNATURE类型）
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
    // 3. 独立调用 validateFieldAuth（仅SIGNATURE类型）
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
| 创建时访问鉴权 | - (登录态创建) | ✓ (ACCOUNT或无) | ✓ (Presign Token验证) |
| 签署时访问鉴权 | ✓ (完整支持5种) | - (已在创建时完成) | ✓ (完整支持5种) |
| 字段动作鉴权 | ✓ (仅 SIGNATURE) | ✓ (仅 SIGNATURE) | ✓ (仅 SIGNATURE) |
| FREE_SIGNATURE鉴权 | ❌ 跳过 | ❌ 跳过 | ❌ 跳过 |
| 签署顺序控制 | ✓ | ✓ | ✓ |
| 创建时配置鉴权 | ✓ 继承模板 | ✓ 继承模板 | ❌ 路由不支持传参，创建后需单独更新 |

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
5. **动作鉴权粒度**: 仅 `SIGNATURE` 字段受动作鉴权保护，`FREE_SIGNATURE` 与普通文本字段一致

---

## 七、统一数据模型

### 7.1 Envelope (核心载体) - 100% 共用

```prisma
model Envelope {
  id          String          @id @unique
  type        EnvelopeType    // DOCUMENT | TEMPLATE
  status      DocumentStatus  // DRAFT | PENDING | COMPLETED
  source      DocumentSource? // DOCUMENT | TEMPLATE | TEMPLATE_DIRECT_LINK
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
          │ 独立创建Envelope      │ 独立创建Envelope      │ 调用公共createEnvelope
          │ +团队默认收件人        │ 无默认收件人          │ +团队默认收件人
          │ 创建Recipients        │ 创建Recipients        │
          │ 创建Fields            │ 创建Fields            │
          │ signingStatus=NOT_SIGNED│ 直链收件人=SIGNED    │
          │ inserted=false        │ 直链字段inserted=true │
          │ source=TEMPLATE       │ source=TEMPLATE_DIRECT_LINK│ source=DOCUMENT
          │ authOptions继承模板   │ authOptions继承模板   │ authOptions空数组
          │ 支持传附件            │ 复制模板附件          │ 不支持传附件
          │ 动作鉴权=仅SIGNATURE  │ 动作鉴权=仅SIGNATURE  │ 动作鉴权=仅SIGNATURE
          │                       │ 创建Signature记录 ✓   │
          │                       │ 写入多类审计日志 ✓     │
          │                       │ 内部触发sendDocument  │
          └───────────────────────┼───────────────────────┘
                                  │
                                  ▼
                        ┌─────────────────────┐
                        │   后续签署动作      │
                        │ signFieldWithToken │
                        │ (直链跳过此步)       │
                        │ 动作鉴权=仅SIGNATURE │
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
2. **仅 SIGNATURE 字段验证动作鉴权**: 包括 `FREE_SIGNATURE` 在内的所有其他字段类型都会绕过鉴权检查
3. **无法中途变更**: 直链签署是原子操作，创建即完成，无法撤销或修改
4. **版本差异处理**: V2 版本处理只读字段和预填充字段的逻辑不同
5. **无默认收件人**: 直链签署仅复制模板配置的收件人，不添加团队默认收件人

### 9.2 动作鉴权关键注意事项

1. **仅 SIGNATURE 受保护**: `validateFieldAuth` 函数只对 `FieldType.SIGNATURE` 执行鉴权检查
2. **FREE_SIGNATURE 无保护**: 自由签名字段与普通文本字段一样，不执行任何动作鉴权
3. **所有入口一致**: 模板克隆、直链签署、嵌入签署三种入口都遵循同样的动作鉴权规则
4. **文档全局配置同样受限**: 即使 `globalActionAuth` 配置了鉴权方式，对非 SIGNATURE 字段也不生效

### 9.3 嵌入预签名关键限制

1. **不支持 authOptions**: `createEmbeddingDocument` 路由入参不包含全局鉴权和收件人鉴权配置，创建后需通过其他API更新
2. **不支持附件**: 路由入参不包含 attachments 字段，无法在创建文档时上传附件
3. **内部版本固定**: 强制使用 internalVersion = 1，不支持 V2 版本特性
4. **单文档数据**: 仅支持单个 envelopeItem（documentDataId），不支持多文档
5. **动作鉴权规则一致**: 嵌入签署同样仅 SIGNATURE 字段受动作鉴权保护

### 9.4 代码复用率统计（代码核对版）

| 模块 | 模板克隆 | 直链签署 | 嵌入预签名 |
|------|---------|---------|-----------|
| Envelope数据模型 | 100% | 100% | 100% |
| Recipient数据模型 | 100% | 100% | 100% |
| Field数据模型 | 100% | 100% | 100% |
| extractDocumentAuthMethods | 100% | 100% | 100% |
| isRecipientAuthorized | 100% | ~80% (访问鉴权分支受限) | 100% |
| validateFieldAuth | 100% (仅SIGNATURE) | 100% (仅SIGNATURE) | 100% (仅SIGNATURE) |
| signFieldWithToken | 100% | 0% (内部独立实现) | 100% |
| createEnvelope 公共函数 | 0% (独立实现) | 0% (独立实现) | 100% |
| PDF复制函数 | putNormalizedPdfFileServerSide | putPdfFileServerSide | N/A (直接引用) |
| 团队默认收件人 | ✅ 添加 | ❌ 不添加 | ✅ 添加 |
| 支持鉴权配置 | ✅ 继承模板 | ✅ 继承模板 | ❌ 创建时不支持 |
| 支持附件 | ✅ | ✅ 仅模板附件 | ❌ |
| FREE_SIGNATURE鉴权 | ❌ 跳过 | ❌ 跳过 | ❌ 跳过 |

---

**文档版本**: 2.3  
**最后更新**: 2024  
**核对状态**: ✅ 所有字段已与源代码逐行核对  
**关键修正**: 修正动作鉴权描述（仅SIGNATURE字段执行鉴权，FREE_SIGNATURE直接跳过）、补充动作鉴权矩阵、所有对照表和总结文字完全对齐、全文档口径统一
