# Documenso 三种共享入口签署引擎实现机制

## 概述

Documenso 签署系统支持三种共享入口方式，共用同一套签署引擎、鉴权策略和数据契约：

1. **模板批量复制** - 基于模板创建多个文档
2. **直链邮件单人签** - 通过邮件直链直接签署文档
3. **嵌入外部站点就地签** - 在第三方网站嵌入签署功能

---

## 一、共用核心签署引擎

### 1.1 签名字段核心处理

**文件位置**: `packages/lib/server-only/field/sign-field-with-token.ts`

所有三种签署方式最终都通过 `signFieldWithToken` 函数处理字段签名：

```typescript
export type SignFieldWithTokenOptions = {
  token: string;              // 收件人访问令牌
  fieldId: number;            // 字段ID
  value: string;              // 签名值
  isBase64?: boolean;         // 是否为Base64图片
  userId?: number;            // 当前用户ID
  authOptions?: TRecipientActionAuth;  // 鉴权选项
  requestMetadata?: RequestMetadata;
};
```

**核心流程**:
1. 验证收件人令牌有效性
2. 检查字段状态（未签署、文档状态）
3. 验证字段类型特定规则（数字、复选框、下拉框等）
4. 执行动作鉴权验证
5. 更新字段状态并存储签名
6. 创建审计日志记录

### 1.2 字段类型处理机制

系统支持多种字段类型，共用同一验证逻辑：

- **签名类**: SIGNATURE, FREE_SIGNATURE - 支持图片Base64或文本签名
- **信息类**: NAME, EMAIL, DATE - 自动填充或用户输入
- **高级字段**: NUMBER, TEXT, CHECKBOX, RADIO, DROPDOWN - 带元数据验证

---

## 二、共用鉴权策略

### 2.1 鉴权层次结构

**文件位置**: `packages/lib/utils/document-auth.ts`

所有签署方式共用四层鉴权模型：

```typescript
export const extractDocumentAuthMethods = ({ 
  documentAuth, 
  recipientAuth 
}: ExtractDocumentAuthMethodsOptions) => {
  // 1. 文档全局访问鉴权 (globalAccessAuth)
  // 2. 文档全局动作鉴权 (globalActionAuth)
  // 3. 收件人级别访问鉴权 (accessAuth)
  // 4. 收件人级别动作鉴权 (actionAuth)
  
  // 收件人级别优先级 > 文档全局级别
  const derivedRecipientAccessAuth = 
    recipientAuthOption.accessAuth.length > 0 
      ? recipientAuthOption.accessAuth 
      : documentAuthOption.globalAccessAuth;
};
```

### 2.2 鉴权验证核心

**文件位置**: `packages/lib/server-only/document/is-recipient-authorized.ts`

统一的鉴权验证函数 `isRecipientAuthorized` 支持三种验证场景：
- `ACCESS` - 文档访问鉴权
- `ACCESS_2FA` - 双因素访问鉴权
- `ACTION` - 签署动作鉴权

**支持的鉴权方式**:
1. **ACCOUNT** - 账号验证（当前登录用户与收件人邮箱匹配）
2. **PASSKEY** - 生物特征密钥（WebAuthn标准）
3. **TWO_FACTOR_AUTH** - 双因素认证（邮件验证码或TOTP）
4. **PASSWORD** - 密码验证
5. **EXPLICIT_NONE** - 显式无鉴权

---

## 三、三种入口实现机制

### 3.1 模板批量复制

**核心文件**: 
- `packages/lib/server-only/template/createDocumentFromTemplate.ts`
- `packages/lib/server-only/template/createDocumentFromDirectTemplate.ts`

#### 实现架构

```
模板定义 (EnvelopeType.TEMPLATE)
    ↓
┌─────────────────────────────────────┐
│ 模板元数据 + 收件人配置 + 字段配置  │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│         创建文档实例                │
│  ┌───────────────────────────────┐ │
│  │ 1. 复制PDF文件资源            │ │
│  │ 2. 创建收件人并生成token      │ │
│  │ 3. 映射并创建签名字段         │ │
│  │ 4. 继承文档鉴权配置           │ │
│  │ 5. 预填充字段值 (可选)        │ │
│  │ 6. 创建审计日志               │ │
│  └───────────────────────────────┘ │
└─────────────────────────────────────┘
    ↓
文档实例 (EnvelopeType.DOCUMENT) + 触发邮件通知
```

#### 关键技术点

**1. 字段预填充机制**
```typescript
// 支持预设字段值，如日期、文本、选择项
const getUpdatedFieldMeta = (field: Field, prefillField?: TFieldMetaPrefillFieldsSchema) => {
  return match(prefillField)
    .with({ type: 'date' }, () => { /* 日期预填 */ })
    .with({ type: 'text' }, () => { /* 文本预填 */ })
    .with({ type: 'number' }, () => { /* 数字预填 */ })
    .with({ type: 'checkbox' }, () => { /* 复选框预填 */ })
    .with({ type: 'radio' }, () => { /* 单选框预填 */ })
    .with({ type: 'dropdown' }, () => { /* 下拉框预填 */ })
    .otherwise(() => field.fieldMeta);
};
```

**2. 批量复制优化**
- 使用 `$transaction` 确保原子性
- 并行处理文件复制和字段创建
- 保留模板的鉴权配置和元数据

### 3.2 直链邮件单人签

**核心文件**:
- `packages/lib/server-only/envelope/getEnvelopeForRecipientSigning.ts`
- `packages/lib/server-only/template/createDocumentFromDirectTemplate.ts`

#### 实现架构

```
收件人邮件
    ↓
包含签署链接: /sign/{recipientToken}
    ↓
┌─────────────────────────────────────┐
│      直链签署验证流程               │
│  ┌───────────────────────────────┐ │
│  │ 1. 验证token有效性            │ │
│  │ 2. 获取文档和收件人信息       │ │
│  │ 3. 执行访问鉴权 (ACCESS)      │ │
│  │ 4. 检查签署顺序 (串行/并行)   │ │
│  │ 5. 返回签署页面数据           │ │
│  └───────────────────────────────┘ │
└─────────────────────────────────────┘
    ↓
用户签署 → 调用 signFieldWithToken → 完成签署
```

#### 关键技术点

**1. 收件人令牌机制**
```typescript
// 每个收件人拥有唯一的访问令牌
type Recipient = {
  token: string;        // 随机生成的访问令牌
  signingStatus: SigningStatus;  // NOT_SIGNED | SIGNED | REJECTED
  expiresAt: Date;      // 链接过期时间
};
```

**2. 签署顺序控制**
```typescript
// 支持顺序签署和并行签署
if (documentMeta.signingOrder === DocumentSigningOrder.SEQUENTIAL) {
  // 检查前序收件人是否已签署
  for (let i = 0; i < currentRecipientIndex; i++) {
    if (recipients[i].signingStatus !== SigningStatus.SIGNED) {
      // 阻止当前收件人签署
    }
  }
}
```

**3. 直接模板签署**
- 用户通过模板直链直接签署时，会即时创建文档实例
- 同时完成该收件人的所有字段签名
- 触发后续收件人邮件通知

### 3.3 嵌入外部站点就地签

**核心文件**:
- `packages/lib/server-only/embedding-presign/createEmbeddingPresignToken.ts`
- `packages/lib/server-only/embedding-presign/verifyEmbeddingPresignToken.ts`
- `packages/trpc/server/embedding-router/_router.ts`

#### 实现架构

```
第三方业务系统
    ↓
1. 请求 Presign Token (携带 API Key)
    ↓
┌─────────────────────────────────────┐
│      嵌入签署预签名服务             │
│  ┌───────────────────────────────┐ │
│  │ 1. 验证API Key有效性          │ │
│  │ 2. 生成JWT格式Presign Token   │ │
│  │ 3. 设置过期时间 (默认1小时)   │ │
│  │ 4. 绑定用户/团队上下文        │ │
│  └───────────────────────────────┘ │
└─────────────────────────────────────┘
    ↓
2. 返回 Presign Token 给第三方系统
    ↓
3. 第三方系统使用 Presign Token 调用嵌入API
   - 创建文档 (createEmbeddingDocument)
   - 创建模板 (createEmbeddingTemplate)
   - 获取签署数据 (getMultiSignDocument)
    ↓
4. 嵌入式签署组件渲染 → 用户签署
```

#### 关键技术点

**1. JWT预签名令牌结构**
```typescript
// 使用 jose 库实现JWT签名验证
const token = await new SignJWT({
  aud: String(teamId ?? userId),  // 受众: 团队或用户ID
  sub: String(apiTokenId),        // 主题: API Token ID
  scope,                          // 权限范围
})
  .setProtectedHeader({ alg: 'HS256' })
  .setIssuedAt(now)
  .setExpirationTime(expiresAt)
  .sign(secret);  // 使用API Token本身作为密钥
```

**2. 令牌验证流程**
```typescript
export const verifyEmbeddingPresignToken = async ({ token, scope }) => {
  // 1. 解码JWT获取claims
  // 2. 验证Token ID和受众匹配
  // 3. 验证scope权限范围
  // 4. 使用API Token作为密钥验证签名
  // 5. 检查是否过期
};
```

**3. 嵌入API路由**
- `createEmbeddingPresignToken` - 创建预签名令牌 (公开)
- `verifyEmbeddingPresignToken` - 验证预签名令牌 (公开)
- `createEmbeddingDocument` - 创建嵌入文档 (需预签名)
- `createEmbeddingTemplate` - 创建嵌入模板 (需预签名)
- `updateEmbeddingDocument` - 更新嵌入文档 (需预签名)
- `getMultiSignDocument` - 获取多文档签署数据 (需预签名)

---

## 四、统一数据契约

### 4.1 Envelope 统一数据模型

所有三种入口都使用相同的 Envelope 数据结构：

```prisma
model Envelope {
  id          String          @id @unique
  type        EnvelopeType    // DOCUMENT | TEMPLATE
  status      DocumentStatus  // DRAFT | PENDING | COMPLETED
  title       String
  
  // 鉴权配置 (三种入口共用)
  authOptions Json            // TDocumentAuthOptions
  
  // 关联关系
  recipients  Recipient[]
  envelopeItems EnvelopeItem[]
  documentMeta DocumentMeta?
  
  // 嵌入签署专用
  directLink  DirectLink?     // 直链配置
}
```

### 4.2 Recipient 统一收件人模型

```prisma
model Recipient {
  id            Int            @id @default(autoincrement())
  envelopeId    String
  email         String
  name          String?
  role          RecipientRole  // SIGNER | CC | ASSISTANT
  
  // 访问令牌 (所有签署方式共用)
  token         String         @unique
  
  // 签署状态
  signingStatus SigningStatus  // NOT_SIGNED | SIGNED | REJECTED
  signedAt      DateTime?
  
  // 收件人级别鉴权 (覆盖文档级别)
  authOptions   Json           // TRecipientAuthOptions
  
  // 签名字段
  fields        Field[]
}
```

### 4.3 Field 统一字段模型

```prisma
model Field {
  id             Int           @id @default(autoincrement())
  envelopeId     String
  recipientId    Int
  type           FieldType     // SIGNATURE | NAME | EMAIL | DATE | TEXT 等
  
  // 位置信息
  page           Int
  positionX      Float
  positionY      Float
  width          Float
  height         Float
  
  // 签名状态
  inserted       Boolean       @default(false)
  customText     String        @default("")
  
  // 高级字段元数据
  fieldMeta      Json?         // 验证规则、选项、默认值等
  
  // 签名数据
  signature      Signature?
}
```

---

## 五、架构设计优势

### 5.1 代码复用
- **签署引擎**: 100% 共用 `signFieldWithToken`
- **鉴权逻辑**: 100% 共用 `isRecipientAuthorized` 和 `extractDocumentAuthMethods`
- **数据模型**: 100% 共用 Envelope/Recipient/Field 模型

### 5.2 扩展能力
- 新签署方式只需实现入口层，核心逻辑无需修改
- 新鉴权方式只需在 `isRecipientAuthorized` 中添加分支
- 新字段类型只需在 `signFieldWithToken` 中添加验证

### 5.3 安全保障
- 统一的令牌过期机制
- 统一的审计日志记录
- 统一的权限验证流程
- JWT 嵌入签署防篡改

---

## 六、调用关系总览

```
                        ┌─────────────────────┐
                        │   三种签署入口      │
                        └─────────┬───────────┘
                                  │
          ┌───────────────────────┼───────────────────────┐
          │                       │                       │
          ▼                       ▼                       ▼
┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐
│  模板批量复制     │  │  直链邮件单人签   │  │  嵌入外部站点签   │
│  (Template)       │  │  (Direct Link)    │  │  (Embedding)      │
└─────────┬─────────┘  └─────────┬─────────┘  └─────────┬─────────┘
          │                       │                       │
          └───────────────────────┼───────────────────────┘
                                  │
                                  ▼
                        ┌─────────────────────┐
                        │   共用签署引擎      │
                        │  signFieldWithToken │
                        └─────────┬───────────┘
                                  │
                                  ▼
                        ┌─────────────────────┐
                        │   共用鉴权策略      │
                        │ isRecipientAuthorized│
                        └─────────┬───────────┘
                                  │
                                  ▼
                        ┌─────────────────────┐
                        │   统一数据契约      │
                        │ Envelope/Recipient/Field│
                        └─────────────────────┘
```

---

**文档版本**: 1.0  
**最后更新**: 2024  
**适用范围**: Documenso 签署系统三种共享入口
