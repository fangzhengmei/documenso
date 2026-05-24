# 签署字段校验与错误反馈流程分析

## 一、整体架构概览

```
┌───────────────────────────────────────────────────────────────────────────────────────┐
│                          签署字段提交校验与错误反馈完整流程                                  │
├───────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                    │
│  ┌───────────────┐     ┌────────────────┐     ┌─────────────────┐                      │
│  │   前端UI层    │────▶│   tRPC API层    │────▶│   业务逻辑层     │                      │
│  │  (三段校验)   │     │  (Schema校验)   │     │  (多维度校验)    │                      │
│  └───────────────┘     └────────────────┘     └─────────────────┘                      │
│          │                     │                         │                                  │
│          │                     │                         ▼                                  │
│          │                     │              ┌──────────────────────┐                   │
│          │                     │              │     数据库事务层      │                   │
│          │                     │              │      (数据持久化)      │                   │
│          │                     │              └──────────────────────┘                   │
│          │                     │                         │                                  │
│          ▼                     ▼                         ▼                                  │
│  ┌───────────────────────────────────────────────────────────────────┐                     │
│  │                     错误回传与展示层                                   │                     │
│  │  AppError → tRPC格式化 → 前端parse → 字段级错误/Toast/重抛认证        │                     │
│  └───────────────────────────────────────────────────────────────────┘                     │
│                                                                                    │
└───────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、校验分层设计

### 2.1 校验层级概览

| 层级 | 位置 | 校验时机 | 主要职责 |
|------|------|----------|----------|
| **L1 前端对话框校验** | 各 sign-field-*-dialog | 对话框输入时 | 实时验证，阻止无效对话框提交 |
| **L2 前端提交前校验** | 字段点击处理函数 | 调用API前 | 空值检查，提前终止无效调用 |
| **L3 tRPC Schema校验** | tRPC路由入口 | 请求到达时 | 输入参数格式校验 |
| **L4 业务状态校验** | sign-field-with-token / sign-envelope-field | 业务逻辑入口 | 文档/接收者/字段状态校验 |
| **L5 字段类型校验** | extractFieldInsertionValues / validate-* 系列 | 字段值处理前 | 字段值业务规则校验 |
| **L6 权限认证校验** | validateFieldAuth | 签名字段提交前 | 签署人身份认证校验 |

---

## 三、各字段类型校验规则详解

### 3.1 字段类型定义

**文件**: `packages/lib/types/field.ts`

支持的字段类型及对应校验函数：

| 字段类型 | 校验函数 | 校验规则所在文件 |
|----------|----------|------------------|
| `TEXT` | `validateTextField` | `validate-text.ts` |
| `NUMBER` | `validateNumberField` | `validate-number.ts` |
| `CHECKBOX` | `validateCheckboxField` | `validate-checkbox.ts` |
| `RADIO` | `validateRadioField` | `validate-radio.ts` |
| `DROPDOWN` | `validateDropdownField` | `validate-dropdown.ts` |
| `SIGNATURE` | 内联校验 | `sign-field-with-token.ts` |
| `DATE` | 内联校验 | 自动填入当前日期 |
| `EMAIL` | `zEmail()` + 内联校验 | `envelope-signing.ts` / `zod.ts` |
| `NAME` | `z.string().min(1)` + 内联校验 | `envelope-signing.ts` |
| `INITIALS` | `z.string().min(1)` + 内联校验 | `envelope-signing.ts` |

---

### 3.2 邮箱字段 (EMAIL) 校验规则

#### V2 路径 (Envelope V2) 完整校验链路

**校验阶段1：前端对话框校验**

**文件**: `apps/remix/app/components/dialogs/sign-field-email-dialog.tsx:20-22`

```typescript
const ZSignFieldEmailFormSchema = z.object({
  email: zEmail().min(1, { message: msg`Email is required`.id }),
});
```

校验规则：
1. **邮箱格式**：`zEmail()` - RFC 5322 合规邮箱正则 (`packages/lib/utils/zod.ts:11-38`)
2. **必填校验**：`.min(1)` - "Email is required"

**校验阶段2：字段点击处理**

**文件**: `apps/remix/app/utils/field-signing/email-field.ts:14-47`

```typescript
export const handleEmailFieldClick = async (options) => {
  // 1. 字段类型检查
  if (field.type !== FieldType.EMAIL) {
    throw new AppError(AppErrorCode.INVALID_REQUEST, { message: 'Invalid field type' });
  }

  // 2. 已插入则返回取消插入
  if (field.inserted) {
    return { type: FieldType.EMAIL, value: null };
  }

  // 3. 无预填值则弹出对话框
  if (!emailToInsert) {
    emailToInsert = await SignFieldEmailDialog.call({ placeholderEmail });
  }

  // 4. 用户取消则返回 null
  if (!emailToInsert) return null;

  return { type: FieldType.EMAIL, value: emailToInsert };
};
```

**校验阶段3：后端 extractFieldInsertionValues 校验**

**文件**: `packages/lib/utils/envelope-signing.ts:39-58`

```typescript
.with({ type: FieldType.EMAIL }, (fieldValue) => {
  const parsedEmailValue = zEmail().nullable().safeParse(fieldValue.value);

  if (!parsedEmailValue.success) {
    throw new AppError(AppErrorCode.INVALID_BODY, {
      message: 'Invalid email',
    });
  }

  if (parsedEmailValue.data === null) {
    return { customText: '', inserted: false };
  }

  return { customText: parsedEmailValue.data, inserted: true };
})
```

#### V1 路径 (Document V1) 校验逻辑

**文件**: `apps/remix/app/components/general/document-signing/document-signing-email-field.tsx:32-134`

```typescript
const onSign = async (authOptions?: TRecipientActionAuth) => {
  try {
    // 直接使用上下文提供的邮箱，无特殊校验
    const value = providedEmail ?? '';

    const payload: TSignFieldWithTokenMutationSchema = {
      token: recipient.token,
      fieldId: field.id,
      value,
      isBase64: false,
      authOptions,
    };

    await signFieldWithToken(payload);
  } catch (err) {
    // 错误处理...
  }
};
```

**关键差异**：V1 路径无邮箱格式校验，直接透传值。

---

### 3.3 姓名字段 (NAME) 校验规则

#### V2 路径 (Envelope V2) 完整校验链路

**校验阶段1：前端对话框校验**

**文件**: `apps/remix/app/components/dialogs/sign-field-name-dialog.tsx:19-21`

```typescript
const ZSignFieldNameFormSchema = z.object({
  name: z.string().min(1, { message: msg`Name is required`.id }),
});
```

校验规则：
1. **必填校验**：`.min(1)` - "Name is required"

**校验阶段2：字段点击处理**

**文件**: `apps/remix/app/utils/field-signing/name-field.ts:13-47`

```typescript
export const handleNameFieldClick = async (options) => {
  // 1. 字段类型检查
  if (field.type !== FieldType.NAME) {
    throw new AppError(AppErrorCode.INVALID_REQUEST, { message: 'Invalid field type' });
  }

  // 2. 已插入则返回取消插入
  if (field.inserted) {
    return { type: FieldType.NAME, value: null };
  }

  // 3. 无预填值则弹出对话框
  if (!nameToInsert) {
    nameToInsert = await SignFieldNameDialog.call({});
  }

  // 4. 用户取消则返回 null
  if (!nameToInsert) return null;

  return { type: FieldType.NAME, value: nameToInsert };
};
```

**校验阶段3：后端 extractFieldInsertionValues 校验**

**文件**: `packages/lib/utils/envelope-signing.ts:60-79`

```typescript
.with({ type: P.union(FieldType.NAME, FieldType.INITIALS) }, (fieldValue) => {
  const parsedGenericStringValue = z.string().min(1).nullable().safeParse(fieldValue.value);

  if (!parsedGenericStringValue.success) {
    throw new AppError(AppErrorCode.INVALID_BODY, {
      message: 'Value is required',
    });
  }

  if (parsedGenericStringValue.data === null) {
    return { customText: '', inserted: false };
  }

  return { customText: parsedGenericStringValue.data, inserted: true };
})
```

#### V1 路径 (Document V1) 校验逻辑

**文件**: `apps/remix/app/components/general/document-signing/document-signing-name-field.tsx:38-218`

```typescript
const onSign = async (authOptions?: TRecipientActionAuth, name?: string) => {
  try {
    const value = name || providedFullName || '';

    // 前端简单空值检查，不通过则重新弹出对话框
    if (!value && !isAssistantMode) {
      setShowFullNameModal(true);
      return;
    }

    const payload = {
      token: recipient.token,
      fieldId: field.id,
      value,
      isBase64: false,
      authOptions,
    };

    await signFieldWithToken(payload);
  } catch (err) {
    // 错误处理...
  }
};
```

**关键差异**：V1 路径仅有前端空值检查，无后端校验。

---

### 3.4 首字母字段 (INITIALS) 校验规则

#### V2 路径 (Envelope V2) 完整校验链路

**校验阶段1：前端对话框校验**

**文件**: `apps/remix/app/components/dialogs/sign-field-initials-dialog.tsx:19-21`

```typescript
const ZSignFieldInitialsFormSchema = z.object({
  initials: z.string().min(1, { message: msg`Initials are required`.id }),
});
```

校验规则：
1. **必填校验**：`.min(1)` - "Initials are required"

**校验阶段2：字段点击处理**

**文件**: `apps/remix/app/utils/field-signing/initial-field.ts:13-45`

```typescript
export const handleInitialsFieldClick = async (options) => {
  // 1. 字段类型检查
  if (field.type !== FieldType.INITIALS) {
    throw new AppError(AppErrorCode.INVALID_REQUEST, { message: 'Invalid field type' });
  }

  // 2. 已插入则返回取消插入
  if (field.inserted) {
    return { type: FieldType.INITIALS, value: null };
  }

  // 3. 无预填值则弹出对话框
  if (!initialsToInsert) {
    initialsToInsert = await SignFieldInitialsDialog.call({});
  }

  // 4. 用户取消则返回 null
  if (!initialsToInsert) return null;

  // Bug: 返回 initials 而非 initialsToInsert
  return { type: FieldType.INITIALS, value: initials };
};
```

**校验阶段3：后端 extractFieldInsertionValues 校验**

**文件**: `packages/lib/utils/envelope-signing.ts:60-79`

与 NAME 字段共用同一校验分支：

```typescript
.with({ type: P.union(FieldType.NAME, FieldType.INITIALS) }, (fieldValue) => {
  const parsedGenericStringValue = z.string().min(1).nullable().safeParse(fieldValue.value);

  if (!parsedGenericStringValue.success) {
    throw new AppError(AppErrorCode.INVALID_BODY, {
      message: 'Value is required',
    });
  }
  // ...
})
```

#### V1 路径 (Document V1) 校验逻辑

**文件**: `apps/remix/app/components/general/document-signing/document-signing-initials-field.tsx:33-140`

```typescript
const onSign = async (authOptions?: TRecipientActionAuth) => {
  try {
    // 从 fullName 自动提取首字母，无特殊校验
    const initials = extractInitials(fullName);
    const value = initials ?? '';

    const payload = {
      token: recipient.token,
      fieldId: field.id,
      value,
      isBase64: false,
      authOptions,
    };

    await signFieldWithToken(payload);
  } catch (err) {
    // 错误处理...
  }
};
```

**关键差异**：V1 路径自动提取首字母，无任何校验。

---

### 3.5 文本字段 (TEXT) 校验规则

**文件**: `packages/lib/advanced-fields-validation/validate-text.ts:3-33`

```typescript
validateTextField(value: string, fieldMeta: TTextFieldMeta, isSigningPage: boolean): string[]
```

校验规则：
1. **必填校验**：`required && !value && isSigningPage` → "Value is required"
2. **字符长度限制**：`characterLimit > 0 && value.length > characterLimit` → 超限提示
3. **只读字段校验**：`readOnly && value.length < 1` → "A read-only field must have text"
4. **冲突校验**：`readOnly && required` → "A field cannot be both read-only and required"
5. **字体大小校验**：`fontSize < 8 || fontSize > 96` → 字体大小范围校验

**V2 路径注意**：`envelope-signing.ts:118-139` 中 TEXT 字段校验错误消息存在 Bug：
```typescript
if (errors.length > 0) {
  throw new AppError(AppErrorCode.INVALID_BODY, {
    message: 'Invalid email',  // Bug: 应为 'Invalid text'
  });
}
```

---

### 3.6 数字字段 (NUMBER) 校验规则

**文件**: `packages/lib/advanced-fields-validation/validate-number.ts:5-65`

```typescript
validateNumberField(value: string, fieldMeta?: TNumberFieldMeta, isSigningPage: boolean): string[]
```

校验规则：
1. **格式校验**：`numberFormat` 正则匹配
2. **必填校验**：`required && !value` → "Value is required"
3. **数字格式**：`!/^[0-9,.]+$/.test(value.trim())` → "Value is not a valid number"
4. **数值范围**：
   - `minValue > 0 && numberValue < minValue` → 小于最小值
   - `maxValue > 0 && numberValue > maxValue` → 大于最大值
5. **范围冲突**：`minValue > maxValue` → 配置错误
6. **只读校验**：`readOnly && numberValue < 1` → 只读字段必须有值
7. **冲突校验**：`readOnly && required` → 不能同时为只读和必填

---

### 3.7 复选框字段 (CHECKBOX) 校验规则

**文件**: `packages/lib/advanced-fields-validation/validate-checkbox.ts:6-74`

```typescript
validateCheckboxField(values: string[], fieldMeta: TCheckboxFieldMeta, isSigningPage: boolean): string[]
```

校验规则：
1. **冲突校验**：`readOnly && required` → 不能同时为只读和必填
2. **选项存在**：`values.length === 0` → "At least one option must be added"
3. **必填校验**：`isSigningPage && required && values.length === 0` → "Selecting an option is required"
4. **验证规则完整性**：
   - `validationRule && !validationLength` → 需指定验证数量
   - `validationLength && !validationRule` → 需指定验证规则
5. **选择数量验证**：
   - `=`：`values.length !== validationLength` → 需精确选择N个
   - `>=`：`values.length < validationLength` → 至少选择N个
   - `<=`：`values.length > validationLength` → 最多选择N个

---

### 3.8 单选字段 (RADIO) 校验规则

**文件**: `packages/lib/advanced-fields-validation/validate-radio.ts:3-37`

```typescript
validateRadioField(value: string | undefined, fieldMeta: TRadioFieldMeta, isSigningPage: boolean): string[]
```

校验规则：
1. **冲突校验**：`readOnly && required` → 不能同时为只读和必填
2. **必填校验**：`isSigningPage && required && !value` → "Choosing an option is required"
3. **选项数量**：`values.length === 0` → "Radio field must have at least one option"
4. **单选约束**：`checkedRadioFieldValues.length > 1` → "There cannot be more than one checked option"

---

### 3.9 下拉字段 (DROPDOWN) 校验规则

**文件**: `packages/lib/advanced-fields-validation/validate-dropdown.ts:3-54`

```typescript
validateDropdownField(value: string | undefined, fieldMeta: TDropdownFieldMeta, isSigningPage: boolean): string[]
```

校验规则：
1. **冲突校验**：`readOnly && required` → 不能同时为只读和必填
2. **必填校验**：`isSigningPage && required && !value` → "Choosing an option is required"
3. **选项存在**：`values.length === 0` → "Select field must have at least one option"
4. **值有效性**：`value && !values.find(item => item.value === value)` → 所选值必须在选项中
5. **默认值有效性**：`defaultValue && !values.find(item => item.value === defaultValue)` → 默认值必须在选项中
6. **选项非空**：`values.some(item => item.value.length < 1)` → 选项值不能为空
7. **选项唯一性**：`new Set(values.map(item => item.value)).size !== values.length` → 不能有重复值

---

### 3.10 签名字段 (SIGNATURE) 校验规则

**文件**: `packages/lib/server-only/field/sign-field-with-token.ts:203-209`

```typescript
// 内联校验逻辑：
1. 签名值存在：`!signatureImageAsBase64 && !typedSignature` → "Signature field must have a signature"
2. 类型签名限制：`typedSignatureEnabled === false && typedSignature` → "Typed signatures are not allowed"
```

---

### 3.11 EMAIL / NAME / INITIALS V1 与 V2 校验差异对比表

| 校验维度 | V2 信封路径 | V1 文档路径 | 代码位置 |
|---------|-----------|-----------|---------|
| **EMAIL 邮箱格式校验** | ✓ `zEmail()` RFC 5322 正则 | ✗ 无 | `envelope-signing.ts:40-46` |
| **EMAIL 必填校验** | ✓ 对话框 + 后端双重校验 | ✗ 直接透传 | `sign-field-email-dialog.tsx:21` |
| **NAME 必填校验** | ✓ 对话框 + 后端双重校验 | ✓ 仅前端空值检查 | `envelope-signing.ts:61-67` |
| **INITIALS 必填校验** | ✓ 对话框 + 后端双重校验 | ✗ 自动提取无校验 | `envelope-signing.ts:61-67` |
| **字段类型检查** | ✓ 点击处理时检查 | ✗ 无 | `email-field.ts:19-23` |
| **已插入检查** | ✓ 点击处理时检查 | ✗ 无 | `email-field.ts:25-29` |
| **错误抛出类型** | AppError (带错误码) | 普通 Error (仅消息) | `envelope-signing.ts:42-46` |

---

## 四、校验生效路径详解

### 4.1 V2 信封字段签署路径 (Envelope V2)

**入口文件**: `packages/trpc/server/envelope-router/sign-envelope-field.ts:15-283`

```
用户点击字段 (Konva canvas pointerdown 事件)
    │
    ▼  envelope-signer-page-renderer.tsx:157-394
1. 字段类型分发 (ts-pattern match)
    │
    ├─ EMAIL → handleEmailFieldClick()
    ├─ NAME → handleNameFieldClick()
    ├─ INITIALS → handleInitialsFieldClick()
    ├─ TEXT → handleTextFieldClick()
    ├─ NUMBER → handleNumberFieldClick()
    ├─ CHECKBOX/RADIO/DROPDOWN → 对应处理函数
    ├─ DATE → 直接构造 payload
    └─ SIGNATURE → handleSignatureFieldClick() + 认证流程
    │
    ▼
2. 字段点击处理函数 (apps/remix/app/utils/field-signing/*)
    │  ├─ 字段类型校验
    │  ├─ 已插入检查（取消插入路径）
    │  └─ 空值检查 → 弹出对话框
    │
    ▼
3. 对话框前端校验 (sign-field-*-dialog.tsx)
    │  ├─ Zod Schema 实时校验
    │  ├─ FormMessage 显示错误
    │  └─ 校验不通过禁止提交
    │
    ▼
4. 调用 signField 封装函数 (envelope-signer-page-renderer.tsx:449-475)
    │
    ▼
5. tRPC Schema 校验 (ZSignEnvelopeFieldRequestSchema)
    │  - token: string
    │  - fieldId: number
    │  - fieldValue: ZSignEnvelopeFieldValue (discriminated union by type)
    │  - authOptions?: TRecipientActionAuth
    │
    ▼
6. 基础状态校验 (sign-envelope-field.ts:28-126)
    │  ├─ 接收者存在性检查 → NOT_FOUND [L68-72]
    │  ├─ 字段存在性检查 → NOT_FOUND [L38-66]
    │  ├─ 信封版本检查 (internalVersion === 2) [L77-81]
    │  ├─ 助理角色限制 (ASSISTANT 不能签签名) [L83-91]
    │  ├─ 字段类型匹配检查 (fieldValue.type === field.type) [L93-97]
    │  ├─ 信封删除状态检查 → INVALID_REQUEST [L99-103]
    │  ├─ 信封状态检查 (status === PENDING) [L105-109]
    │  ├─ 接收者签署状态检查 (未签署) [L111-115]
    │  └─ 字段只读状态检查 (fieldMeta?.readOnly) [L117-121]
    │
    ▼
7. 字段值提取与类型校验 (extractFieldInsertionValues)
    │  │
    │  文件: packages/lib/utils/envelope-signing.ts:33-252
    │  │
    │  ├─ EMAIL: zEmail() 校验 → INVALID_BODY "Invalid email"
    │  ├─ NAME/INITIALS: z.string().min(1) → INVALID_BODY "Value is required"
    │  ├─ TEXT: validateTextField() → INVALID_BODY "Invalid email" (Bug)
    │  ├─ NUMBER: validateNumberField() → INVALID_BODY "Invalid number"
    │  ├─ RADIO: 选项索引有效性 → INVALID_BODY "Invalid radio value"
    │  ├─ CHECKBOX: 选项索引+数量验证 → INVALID_BODY "Invalid checkbox values"
    │  ├─ DROPDOWN: validateDropdownField() → INVALID_BODY "Invalid dropdown value"
    │  ├─ SIGNATURE: typedSignatureEnabled 检查 → INVALID_BODY
    │  └─ DATE: 无校验，自动填入当前日期
    │
    ├─ 取消插入路径 (!insertionValues.inserted → 清除字段值 [L131-171]
    │
    ▼
8. 认证校验 (validateFieldAuth) [L173-179]
    │  仅签名字段需要认证
    │  - 认证失败 → UNAUTHORIZED "Invalid authentication values"
    │
    ▼
9. 签名字段特殊处理 [L183-199]
    │  - Base64 图片识别
    │  - 类型签名/绘制签名分离
    │
    ▼
10. 数据库事务更新 [L201-282]
    ├─ 更新 field 表 (customText, inserted)
    ├─ 签名 upsert (SIGNATURE 字段)
    └─ 审计日志记录
```

---

### 4.2 V1 文档字段签署路径 (Document V1)

**入口文件**: `packages/trpc/server/field-router/router.ts:591-609`

```
用户点击签署
    │
    ▼
1. tRPC Schema 校验 (ZSignFieldWithTokenMutationSchema)
    │  - token: string
    │  - fieldId: number
    │  - value: string (trim().optional())
    │  - isBase64: boolean
    │  - authOptions?: TRecipientActionAuth
    │
    ▼
2. 调用 signFieldWithToken 业务函数
    │  │
    │  文件: packages/lib/server-only/field/sign-field-with-token.ts:50-313
    │
    ▼
3. 基础状态校验 (sign-field-with-token.ts:59-125)
    │  ├─ 接收者存在性检查
    │  ├─ 字段存在性检查
    │  ├─ 信封删除状态检查
    │  ├─ 信封状态检查 (PENDING)
    │  ├─ 接收者过期检查 (assertRecipientNotExpired)
    │  ├─ 接收者签署状态检查
    │  └─ 字段已插入检查
    │
    ▼
4. 字段类型特定校验 (sign-field-with-token.ts:127-172)
    │  ├─ NUMBER → validateNumberField()
    │  ├─ TEXT → validateTextField()
    │  ├─ CHECKBOX → validateCheckboxField()
    │  ├─ RADIO → validateRadioField()
    │  ├─ DROPDOWN → validateDropdownField()
    │  └─ EMAIL/NAME/INITIALS → 无校验，直接透传
    │
    ▼
5. 认证校验 (validateFieldAuth)
    │
    ▼
6. 签名字段特殊校验
    │  ├─ 签名值存在检查
    │  └─ 类型签名限制检查
    │
    ▼
7. 只读字段值校验 (sign-field-with-token.ts:211-232)
    │  校验只读字段值是否为默认值
    │
    ▼
8. 数据库事务更新
```

---

### 4.3 前端实时校验路径（V1 路径）

**以文本字段为例**: `apps/remix/app/components/general/document-signing/document-signing-text-field.tsx:49-218`

```
用户输入文本 (handleTextChange)
    │
    ▼
1. 实时调用 validateTextField(localText, parsedFieldMeta, true)
    │
    ▼
2. 错误分类存储
    │  errors.required = validationErrors.filter(e.includes('required'))
    │  errors.characterLimit = validationErrors.filter(e.includes('character limit'))
    │
    ▼
3. 错误展示
    │  - 字段下方显示红色错误文字
    │  - Sign 按钮根据 userInputHasErrors 阻止提交
```

---

## 五、V2 路径错误回传链路详解（逐行代码核对版）

### 5.1 错误来源与错误码映射表

| 错误来源 | 触发条件 | 错误码 | 错误消息 | 代码位置 |
|---------|---------|--------|---------|---------|
| **接收者不存在** | `!recipient` | `NOT_FOUND` | - | `sign-envelope-field.ts:34-36` |
| **字段不存在** | `!field` | `NOT_FOUND` | `Field ${fieldId} not found` | `sign-envelope-field.ts:68-72` |
| **信封版本不匹配** | `internalVersion !== 2` | `NOT_FOUND` | `Envelope ${envelope.id} is not a version 2 envelope` | `sign-envelope-field.ts:77-81` |
| **助理签签名** | ASSISTANT 签签名字段 | `INVALID_REQUEST` | `Assistant recipients cannot sign signature fields` | `sign-envelope-field.ts:83-91` |
| **字段类型不匹配** | `fieldValue.type !== field.type` | `NOT_FOUND` | `Selected values do not match the field values` | `sign-envelope-field.ts:93-97` |
| **信封已删除** | `envelope.deletedAt` | `INVALID_REQUEST` | `Document ${envelope.id} has been deleted` | `sign-envelope-field.ts:99-103` |
| **信封状态错误** | `status !== PENDING` | `INVALID_REQUEST` | `Document ${envelope.id} must be pending for signing` | `sign-envelope-field.ts:105-109` |
| **接收者已签署** | `signingStatus === SIGNED` | `INVALID_REQUEST` | `Recipient ${recipient.id} has already signed` | `sign-envelope-field.ts:111-115` |
| **字段只读** | `fieldMeta?.readOnly` | `INVALID_REQUEST` | `Field ${fieldId} is read only` | `sign-envelope-field.ts:117-121` |
| **EMAIL 格式错误** | `zEmail()` 校验失败 | `INVALID_BODY` | `Invalid email` | `envelope-signing.ts:42-46` |
| **NAME/INITIALS 空值** | `z.string().min(1)` 失败 | `INVALID_BODY` | `Value is required` | `envelope-signing.ts:63-67` |
| **NUMBER 验证失败** | `validateNumberField()` 有错误 | `INVALID_BODY` | `Invalid number` | `envelope-signing.ts:107-111` |
| **TEXT 验证失败** | `validateTextField()` 有错误 | `INVALID_BODY` | `Invalid email` (Bug) | `envelope-signing.ts:129-133` |
| **RADIO 值无效** | 选项索引越界 | `INVALID_BODY` | `Invalid radio value` | `envelope-signing.ts:151-155` |
| **CHECKBOX 值无效** | 选项索引越界 | `INVALID_BODY` | `Invalid checkbox values` | `envelope-signing.ts:177-181` |
| **CHECKBOX 数量验证** | 选择数量不符合规则 | `INVALID_BODY` | `Checkbox values failed length validation` | `envelope-signing.ts:191-195` |
| **DROPDOWN 验证失败** | `validateDropdownField()` 有错误 | `INVALID_BODY` | `Invalid dropdown value` | `envelope-signing.ts:218-222` |
| **类型签名被禁用** | `typedSignatureEnabled === false` | `INVALID_BODY` | `Typed signatures are not allowed...` | `envelope-signing.ts:241-245` |
| **认证失败** | 签名字段认证无效 | `UNAUTHORIZED` | `Invalid authentication values` | `validate-field-auth.ts:41-45` |

---

### 5.2 错误回传完整链路（逐行核对）

#### 5.2.1 后端到前端的错误数据转换

```
后端抛出 AppError
    │
    │  示例（认证失败）：
    │  // validate-field-auth.ts:41-45
    │  throw new AppError(AppErrorCode.UNAUTHORIZED, {
    │    message: 'Invalid authentication values',
    │  });
    │
    ▼  Step 1: tRPC 错误捕获与 cause 传递
    │  tRPC 自动将 AppError 包装为 TRPCError
    │  TRPCError {
    │    code: 'UNAUTHORIZED',           // 从 AppErrorCode 映射
    │    message: 'Invalid authentication values',
    │    cause: AppError                 // 原始 AppError 对象
    │  }
    │
    ▼  Step 2: tRPC errorFormatter 格式化
    │  文件: packages/trpc/server/trpc.ts:38-66
    │
    │  输入: { shape, error: TRPCError, ctx }
    │  处理:
    │    const originalError = error.cause;  // 提取 AppError
    │
    │    if (originalError instanceof AppError) {
    │      data = {
    │        ...shape.data,
    │        appError: AppError.toJSON(originalError),
    │                // { code: 'UNAUTHORIZED', message: 'Invalid authentication values' }
    │                // 注意: userMessage/statusCode 未设置则不包含
    │        code: 'UNAUTHORIZED',
    │        httpStatus: 401,  // 从 genericErrorCodeToTrpcErrorCodeMap 映射
    │      };
    │    }
    │
    ▼  Step 3: tRPC 错误日志处理
    │  文件: packages/trpc/utils/trpc-error-handler.ts:7-36
    │
    │  const appError = AppError.parseError(error.cause || error);
    │  - code === 'UNKNOWN_ERROR' 或 httpStatus >= 500 → error 级别日志
    │  - 其他错误 → info 级别日志
    │
    ▼  Step 4: tRPC 客户端接收
    │
    │  前端收到 TRPCClientError 对象:
    │  {
    │    name: 'TRPCClientError',
    │    message: 'UNAUTHORIZED: Invalid authentication values',
    │    data: {
    │      appError: {
    │        code: 'UNAUTHORIZED',
    │        message: 'Invalid authentication values'
    │      },
    │      code: 'UNAUTHORIZED',
    │      httpStatus: 401,
    │      path: 'envelope.field.sign',
    │      ...
    │    },
    │    cause: undefined  // 注意: cause 在客户端丢失
    │  }
```

---

#### 5.2.2 V2 路径 signField catch 分支实际执行顺序

**⚠️ 重要发现：V2 路径 signField 中 没有 AppError.parseError 调用，也没有 UNAUTHORIZED 检查**

**代码位置**: `envelope-signer-page-renderer.tsx:449-475`

```typescript
const signField = async (fieldId: number, payload: TSignEnvelopeFieldValue, authOptions?: TRecipientActionAuth) => {
  try {
    const { inserted } = await signFieldInternal(fieldId, payload, authOptions);
    
    // 嵌入上下文回调
    if (inserted && onFieldSigned) { /* ... */ }
    if (!inserted && onFieldUnsigned) { /* ... */ }
  } catch (err) {
    // ┌─────────────────────────────────────────────────────────┐
    // │  Step 1: 先打印错误日志                                 │
    // │  输入: err 是 TRPCClientError 对象（原始错误）          │
    // └─────────────────────────────────────────────────────────┘
    console.error(err);

    // ┌─────────────────────────────────────────────────────────┐
    // │  Step 2: 无条件显示 Toast（包括 UNAUTHORIZED 错误！）   │
    // │  ⚠️  Bug: 认证失败也会先显示 "Error" Toast              │
    // └─────────────────────────────────────────────────────────┘
    toast({
      title: t`Error`,
      description: t`An error occurred while signing the field.`,
      variant: 'destructive',
    });

    // ┌─────────────────────────────────────────────────────────┐
    // │  Step 3: 抛出原始错误（未解析，仍是 TRPCClientError）   │
    // │  没有检查 error.code === UNAUTHORIZED                   │
    // │  没有调用 AppError.parseError(err)                      │
    // └─────────────────────────────────────────────────────────┘
    throw err;
  }
};
```

**执行顺序总结**：
1. `console.error(err)` → **先**打印日志
2. `toast({...})` → **再**显示 Toast（所有错误，包括 UNAUTHORIZED）
3. `throw err` → **最后**抛出原始错误

---

#### 5.2.3 V1 路径 catch 分支实际执行顺序（对比）

**代码位置**: `document-signing-email-field.tsx:72-88`

```typescript
catch (err) {
  // ┌─────────────────────────────────────────────────────────┐
  // │  Step 1: 先解析错误                                     │
  // └─────────────────────────────────────────────────────────┘
  const error = AppError.parseError(err);

  // ┌─────────────────────────────────────────────────────────┐
  // │  Step 2: 检查认证错误                                   │
  // │  是 UNAUTHORIZED → 直接 throw，不 toast                 │
  // └─────────────────────────────────────────────────────────┘
  if (error.code === AppErrorCode.UNAUTHORIZED) {
    throw error;
  }

  // ┌─────────────────────────────────────────────────────────┐
  // │  Step 3: 非认证错误才打印日志和 toast                    │
  // └─────────────────────────────────────────────────────────┘
  console.error(err);

  toast({
    title: _(msg`Error`),
    description: isAssistantMode
      ? _(msg`An error occurred while signing as assistant.`)
      : _(msg`An error occurred while signing the document.`),
    variant: 'destructive',
  });
}
```

**执行顺序总结**：
1. `AppError.parseError(err)` → **先**解析
2. `if (error.code === UNAUTHORIZED) throw error` → **再**检查，认证错误直接抛出（无 Toast）
3. `console.error(err)` → 非认证错误才打印
4. `toast({...})` → 非认证错误才 Toast

---

#### 5.2.4 V2 UNAUTHORIZED 错误完整流向（签名字段场景）

```
用户点击签名字段（有认证要求）
    │
    ▼  envelope-signer-page-renderer.tsx:360-392
handleSignatureFieldClick() 返回 payload { type: SIGNATURE, value: 'xxx' }
    │
    ▼  payload.value 存在 → 走认证流程
void executeActionAuthProcedure({
  onReauthFormSubmit: async (authOptions) => {
    await signField(field.id, payload, authOptions);  // 🔴 这里抛出的错误由谁捕获？
    loadingSpinnerGroup.destroy();
  },
  actionTarget: field.type,
});
    │
    ▼  onReauthFormSubmit 在认证对话框中被调用
    │  document-signing-auth-password.tsx:50-69
    │
    ▼  Step 1: signField 内部调用
    │  signFieldInternal → 后端 API → validateFieldAuth
    │
    ▼  Step 2: 后端认证失败
    │  validate-field-auth.ts:41-45
    │  throw new AppError(UNAUTHORIZED, { message: 'Invalid authentication values' })
    │
    ▼  Step 3: 错误经过 tRPC 链路到达前端
    │  TRPCClientError { data: { code: 'UNAUTHORIZED', ... } }
    │
    ▼  Step 4: signField catch 捕获（不检查错误码）
    │  envelope-signer-page-renderer.tsx:464-474
    │  1. console.error(err)
    │  2. toast({ title: 'Error', description: 'An error occurred...' })  ← ⚠️ 先显示通用 Toast
    │  3. throw err  ← 🔴 重新抛出原始错误
    │
    ▼  Step 5: 认证表单 catch 捕获
    │  document-signing-auth-password.tsx:62-96
    │
    │  try {
    │    await onReauthFormSubmit({ type: PASSWORD, password });
    │  } catch (err) {
    │    setIsCurrentlyAuthenticating(false);
    │
    │    const error = AppError.parseError(err);  // ✅ 解析错误
    │    setFormErrorCode(error.code);            // 'UNAUTHORIZED'
    │    // 注释 "// Todo: Alert." 是过时的，Alert 已实现
    │  }
    │
    ▼  Step 6: 表单渲染时根据 formErrorCode 显示 Alert
    │  document-signing-auth-password.tsx:87-96
    │
    │  {formErrorCode && (
    │    <Alert variant="destructive">
    │      <AlertTitle><Trans>Unauthorized</Trans></AlertTitle>
    │      <AlertDescription>
    │        <Trans>We were unable to verify your details. Please try again or contact support</Trans>
    │      </AlertDescription>
    │    </Alert>
    │  )}
    │
    └─ 最终效果（用户看到两个错误提示）：
       1. 🔴 Toast 先显示："Error: An error occurred while signing the field."
          （由 signField catch 无条件显示，包括 UNAUTHORIZED）
       2. 🟡 Alert 后显示："Unauthorized - We were unable to verify your details..."
          （由认证表单内根据 formErrorCode 渲染）
       3. 认证对话框保持打开，用户可重试
```

##### 四种认证方式的 Alert 展示代码位置：

| 认证方式 | Alert 展示代码位置 | 已实现 |
|---------|-------------------|--------|
| **密码 (PASSWORD)** | `document-signing-auth-password.tsx:87-96` | ✅ |
| **2FA** | `document-signing-auth-2fa.tsx:177-186` | ✅ |
| **Passkey** | `document-signing-auth-passkey.tsx:284-293` | ✅ |
| **Account** | 无 formErrorCode，直接显示登录提示 | ✅ |

> **重要发现**：代码中的 `// Todo: Alert.` 注释（password.tsx:68、2fa.tsx:73、passkey.tsx:98）是过时的，实际上所有认证方式的 Alert 展示都已实现。

---

### 5.3 AppError.parseError 解析逻辑详解

**文件**: `packages/lib/errors/app-error.ts:130-168`

```typescript
static parseError(error: any): AppError {
  // 分支1: 已是 AppError → 直接返回
  if (error instanceof AppError) {
    return error;
  }

  // 分支2: TRPCClientError → 从 data.appError 解析
  if (error?.name === 'TRPCClientError') {
    const parsedJsonError = AppError.parseFromJSON(error.data?.appError);
    // parsedJsonError = AppError {
    //   code: 'UNAUTHORIZED',
    //   message: 'Invalid authentication values',
    //   userMessage: undefined,
    //   statusCode: undefined
    // }

    const fallbackError = new AppError(AppErrorCode.UNKNOWN_ERROR, {
      message: error?.message,
    });

    return parsedJsonError || fallbackError;
  }

  // 分支3: 未知错误 → 转换为 UNKNOWN_ERROR
  const { code, message, userMessage, statusCode } = error as { ... };
  const validCode = typeof code === 'string' ? code : AppErrorCode.UNKNOWN_ERROR;
  
  return new AppError(validCode, { message, userMessage, statusCode });
}
```

**V2 路径中的调用位置**：
- ❌ `envelope-signer-page-renderer.tsx:449-475`（signField）→ **不调用**
- ✅ `document-signing-auth-password.tsx:65`（认证表单）→ **调用**
- ✅ `envelope-signing-complete-dialog.tsx:125`（完成签署）→ **调用**

---

### 5.4 前端错误展示触发条件与代码对应（修正版）

| 展示方式 | 触发条件 | 代码位置 | 展示效果 |
|---------|---------|---------|---------|
| **对话框表单错误** | 对话框内 Zod 校验失败 | `sign-field-email-dialog.tsx:60-63` | 输入框下方红色文字 |
| **Toast 全局提示** | 后端返回**任何**错误（含 UNAUTHORIZED） | `envelope-signer-page-renderer.tsx:464-474` | 右上角红色 Toast |
| **认证表单 Alert** | 签名字段认证失败，`formErrorCode` 被设置后重新渲染 | `auth-password.tsx:87-96` | 红色 Alert 显示 "Unauthorized - We were unable to verify your details..." |
| **字段 Tooltip** | 点击 Complete 时存在未插入字段 | `document-signing-form.tsx:116-120` | 字段旁黄色 Tooltip + 滚动定位 |
| **字段红色边框** | 提交验证时未插入字段 | `fields.ts:38-40` | 字段边框变红 |

> **修正说明**：之前的结论 "认证错误码未展示给用户" 是错误的。实际上所有认证方式（密码、2FA、Passkey）都已实现了 `{formErrorCode && (<Alert>...</Alert>)}` 的条件渲染，`// Todo: Alert.` 注释是过时的。

---

### 5.5 错误码到 HTTP 状态码映射

**文件**: `packages/lib/errors/app-error.ts:32-53`

| AppErrorCode | tRPC code | HTTP Status |
|-------------|-----------|-------------|
| `NOT_FOUND` | `NOT_FOUND` | 404 |
| `UNAUTHORIZED` | `UNAUTHORIZED` | 401 |
| `FORBIDDEN` | `FORBIDDEN` | 403 |
| `INVALID_REQUEST` | `BAD_REQUEST` | 400 |
| `INVALID_BODY` | `BAD_REQUEST` | 400 |
| `TOO_MANY_REQUESTS` | `TOO_MANY_REQUESTS` | 429 |
| `UNKNOWN_ERROR` | `INTERNAL_SERVER_ERROR` | 500 |
| 默认 | `BAD_REQUEST` | 400 |

---

### 5.6 V1 与 V2 错误处理边界差异（逐行对比）

| 对比项 | V1 路径 (Document) | V2 路径 (Envelope) |
|--------|-------------------|-------------------|
| **catch 位置** | `document-signing-*-field.tsx` | `envelope-signer-page-renderer.tsx` |
| **AppError.parseError 调用** | ✅ catch 开头第一行 | ❌ signField 中不调用<br>✅ 仅认证对话框中调用 |
| **UNAUTHORIZED 检查** | ✅ 检查后直接 throw，不 toast | ❌ 不检查，所有错误都 toast |
| **执行顺序** | `parse → check UNAUTHORIZED → console.error → toast` | `console.error → toast → throw` |
| **UNAUTHORIZED 时是否 toast** | ❌ 不 toast | ✅ 先 toast 再 throw |
| **错误消息内容** | 区分助理模式：<br>- "An error occurred while signing as assistant."<br>- "An error occurred while signing the document." | 固定消息：<br>"An error occurred while signing the field." |
| **throw 的错误类型** | `AppError`（已解析） | `TRPCClientError`（原始） |
| **throw 后的处理** | 由上层认证上下文捕获 | 由认证对话框表单 catch |
| **用户体验（认证失败）** | 无干扰 Toast，直接在认证表单内反馈 | 先弹出通用错误 Toast，再显示认证失败 |

---

### 5.7 V2 错误处理现存问题（基于代码核对）

| 问题描述 | 代码位置 | 影响 |
|---------|---------|------|
| **认证失败时不必要的 Toast** | `envelope-signer-page-renderer.tsx:467-471` | 用户看到两个错误提示：通用 Toast + 认证 Alert，造成困惑 |
| **缺少 AppError.parseError 调用** | `envelope-signer-page-renderer.tsx:464-474` | 无法区分错误类型进行差异化处理 |
| **缺少 UNAUTHORIZED 分支处理** | `envelope-signer-page-renderer.tsx:464-474` | 认证错误与其他错误同样处理 |
| **执行顺序不合理** | `envelope-signer-page-renderer.tsx:464-474` | `console.error → toast → throw`，应先解析错误再决定处理方式 |
| **过时的 TODO 注释** | `auth-password.tsx:68`<br>`auth-2fa.tsx:73`<br>`auth-passkey.tsx:98` | `// Todo: Alert.` 注释已过时，可能误导后续开发 |
| **Passkey 未处理 UNAUTHORIZED** | `auth-passkey.tsx:91-93` | `if (err.name === 'NotAllowedError') return` 只处理了 NotAllowedError，其他错误继续执行 |

> **修正说明**：之前的结论 "认证错误码未展示给用户" 已修正。实际上 Alert 显示功能已经实现，问题在于 signField catch 中提前显示了多余的 Toast。

---

### 5.8 formErrorCode 与 signField catch toast 的关系详解

#### 5.8.1 时序关系图

```
时间轴 →
┌────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                         │
│  用户输入密码并点击 "Sign"                                                              │
│         │                                                                                │
│         ▼                                                                                │
│  onFormSubmit 执行                                                                       │
│   ├─ setIsCurrentlyAuthenticating(true)                                                  │
│   └─ await onReauthFormSubmit({ type: PASSWORD, password })                              │
│         │                                                                                │
│         ├─ 内部调用 signField(field.id, payload, authOptions)                           │
│         │    │                                                                           │
│         │    ▼ 后端 API 返回 UNAUTHORIZED                                              │
│         │  signField catch 捕获                                                         │
│         │   ├─ console.error(err)                 [Step 1, 同步]                        │
│         │   ├─ toast({ title: 'Error', ... })     [Step 2, 同步]  ← ⚠️ 多余的 Toast    │
│         │   └─ throw err                         [Step 3, 同步]                        │
│         │         │                                                                      │
│         │         ▼                                                                      │
│         └─ onFormSubmit catch 捕获                                                      │
│            ├─ setIsCurrentlyAuthenticating(false)                                        │
│            ├─ const error = AppError.parseError(err)                                    │
│            └─ setFormErrorCode(error.code)      [Step 4, 同步]                          │
│                                                                                         │
│  React 状态更新触发重新渲染                                                                 │
│   └─ {formErrorCode && (<Alert>...</Alert>)}    [Step 5, 异步渲染]  ← ✅ 正确的 Alert  │
│                                                                                         │
└────────────────────────────────────────────────────────────────────────────────────────────┘
                                        ▲
                                        │
                                    最终效果：
                                    1. 先看到 Toast（立即）
                                    2. 后看到 Alert（React 重渲染后）
```

#### 5.8.2 两个错误提示的对比

| 维度 | signField catch 中的 Toast | 认证表单中的 Alert |
|------|---------------------------|-------------------|
| **触发时机** | 错误抛出后**立即**（同步） | 状态更新后 React**重渲染**（异步） |
| **显示位置** | 右上角全局 Toast | 认证对话框内表单上方 |
| **错误消息** | 通用模糊："An error occurred while signing the field." | 精确："Unauthorized - We were unable to verify your details..." |
| **错误特定性** | 所有错误都显示相同消息 | 仅认证错误显示，且已国际化 |
| **关闭方式** | 自动超时或手动关闭 | 对话框关闭时自动消失，或下次提交前重置 |
| **是否必要** | ❌ 对于 UNAUTHORIZED 是多余的 | ✅ 必要，提供精确反馈 |

#### 5.8.3 对用户体验的影响

**当前行为（认证失败时）**：
1. 用户输入错误密码 → 点击 "Sign"
2. 立即看到 Toast 弹出："Error - An error occurred while signing the field."
3. 约几毫秒后，对话框内出现红色 Alert："Unauthorized - We were unable to verify your details..."
4. 用户看到两个错误提示，可能困惑：到底是字段签名出错了，还是密码不对？

**理想行为（修复后）**：
1. 用户输入错误密码 → 点击 "Sign"
2. 对话框内出现红色 Alert："Unauthorized - We were unable to verify your details..."
3. 没有多余的 Toast，用户明确知道是认证失败

**代码优化建议（V2 signField catch）**：

**优化前** (`envelope-signer-page-renderer.tsx:464-474`):
```typescript
catch (err) {
  console.error(err);

  toast({
    title: t`Error`,
    description: t`An error occurred while signing the field.`,
    variant: 'destructive',
  });

  throw err;
}
```

**优化后**:
```typescript
catch (err) {
  const error = AppError.parseError(err);

  // UNAUTHORIZED 错误由认证表单处理，不显示通用 Toast
  if (error.code === AppErrorCode.UNAUTHORIZED) {
    throw error;
  }

  console.error(err);

  toast({
    title: t`Error`,
    description: t`An error occurred while signing the field.`,
    variant: 'destructive',
  });
}
```

---

## 六、错误对象体系设计

### 6.1 AppErrorCode 枚举

**文件**: `packages/lib/errors/app-error.ts:7-30`

```typescript
enum AppErrorCode {
  NOT_FOUND = 'NOT_FOUND',
  INVALID_REQUEST = 'INVALID_REQUEST',
  INVALID_BODY = 'INVALID_BODY',
  UNAUTHORIZED = 'UNAUTHORIZED',
  FORBIDDEN = 'FORBIDDEN',
  UNKNOWN_ERROR = 'UNKNOWN_ERROR',
  // ... 其他错误码
}
```

### 6.2 AppError 类结构

**文件**: `packages/lib/errors/app-error.ts:88-257`

```typescript
class AppError extends Error {
  code: string;              // 错误码
  message?: string;          // 内部日志消息
  userMessage?: string;      // 可展示给用户的消息
  statusCode?: number;       // HTTP 状态码
  headers?: Record<string, string>;
  name = 'AppError';

  constructor(errorCode: string, options?: AppErrorOptions);

  static parseError(error: any): AppError;      // 解析任意错误
  static toJSON(appError: AppError): TAppErrorJsonSchema;
  static parseFromJSON(value: unknown): AppError | null;
}
```

### 6.3 后端错误抛出方式

#### 方式一：AppError 抛出（V2 路径标准方式）

**文件**: `packages/lib/utils/envelope-signing.ts:42-46`

```typescript
if (!parsedEmailValue.success) {
  throw new AppError(AppErrorCode.INVALID_BODY, {
    message: 'Invalid email',
  });
}
```

#### 方式二：普通 Error 抛出（V1 路径遗留方式）

**文件**: `packages/lib/server-only/field/sign-field-with-token.ts:131-133`

```typescript
if (errors.length > 0) {
  throw new Error(errors.join(', '));
}
```

---

## 七、两条签署路径对比

| 维度 | V2 信封路径 (sign-envelope-field) | V1 文档路径 (sign-field-with-token) |
|------|---------------------------------|-----------------------------------|
| **入口** | `trpc.envelope.field.sign` | `trpc.field.signFieldWithToken` |
| **Schema** | `ZSignEnvelopeFieldRequestSchema` | `ZSignFieldWithTokenMutationSchema` |
| **字段值格式** | discriminated union by type | 统一 string |
| **EMAIL/NAME/INITIALS 校验** | 前端对话框 + 后端 Zod 双重校验 | 无校验或仅前端空值检查 |
| **字段类型校验位置** | `extractFieldInsertionValues` | `sign-field-with-token.ts:127-172` |
| **错误抛出** | AppError（带错误码） | 普通 Error（仅消息字符串） |
| **适用场景** | Envelope V2 文档 | 传统 Document V1 文档 |
| **字段值类型** | CHECKBOX: number[]<br>RADIO: number \| null<br>DATE: boolean | 统一 string 编码 |
| **前端交互** | Konva canvas 点击 + 对话框 | DOM 组件点击 + 对话框 |
| **错误展示** | Toast 全局提示 | Toast + 字段级错误 |

---

## 八、关键代码位置索引

### 8.1 校验函数位置

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| EMAIL 邮箱校验 | `packages/lib/utils/zod.ts` | 34-38 |
| EMAIL V2 后端校验 | `packages/lib/utils/envelope-signing.ts` | 39-58 |
| NAME/INITIALS V2 后端校验 | `packages/lib/utils/envelope-signing.ts` | 60-79 |
| 文本字段校验 | `packages/lib/advanced-fields-validation/validate-text.ts` | 3-33 |
| 数字字段校验 | `packages/lib/advanced-fields-validation/validate-number.ts` | 5-65 |
| 复选框字段校验 | `packages/lib/advanced-fields-validation/validate-checkbox.ts` | 6-74 |
| 单选字段校验 | `packages/lib/advanced-fields-validation/validate-radio.ts` | 3-37 |
| 下拉字段校验 | `packages/lib/advanced-fields-validation/validate-dropdown.ts` | 3-54 |
| 字段认证校验 | `packages/lib/server-only/document/validate-field-auth.ts` | 21-48 |

### 8.2 路由入口位置

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| V2 信封签署路由 | `packages/trpc/server/envelope-router/sign-envelope-field.ts` | 15-283 |
| V2 字段值提取 | `packages/lib/utils/envelope-signing.ts` | 33-252 |
| V1 字段签署路由 | `packages/trpc/server/field-router/router.ts` | 591-609 |
| V1 签署业务逻辑 | `packages/lib/server-only/field/sign-field-with-token.ts` | 50-313 |

### 8.3 字段点击处理函数位置

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| EMAIL 字段点击处理 | `apps/remix/app/utils/field-signing/email-field.ts` | 14-47 |
| NAME 字段点击处理 | `apps/remix/app/utils/field-signing/name-field.ts` | 13-47 |
| INITIALS 字段点击处理 | `apps/remix/app/utils/field-signing/initial-field.ts` | 13-45 |
| V2 页面渲染与事件分发 | `apps/remix/app/components/general/envelope-signing/envelope-signer-page-renderer.tsx` | 46-546 |

### 8.4 对话框组件位置

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| EMAIL 输入对话框 | `apps/remix/app/components/dialogs/sign-field-email-dialog.tsx` | 30-84 |
| NAME 输入对话框 | `apps/remix/app/components/dialogs/sign-field-name-dialog.tsx` | 29-81 |
| INITIALS 输入对话框 | `apps/remix/app/components/dialogs/sign-field-initials-dialog.tsx` | 29-84 |

### 8.5 错误处理位置

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| AppError 定义 | `packages/lib/errors/app-error.ts` | 88-257 |
| AppError 解析 | `packages/lib/errors/app-error.ts` | 130-168 |
| tRPC 错误格式化 | `packages/trpc/server/trpc.ts` | 38-66 |
| tRPC 错误日志 | `packages/trpc/utils/trpc-error-handler.ts` | 7-36 |
| V2 前端错误处理 | `apps/remix/app/components/general/envelope-signing/envelope-signer-page-renderer.tsx` | 449-475 |
| V1 前端错误处理 | `apps/remix/app/components/general/document-signing/document-signing-email-field.tsx` | 72-89 |

---

## 九、设计特点与注意事项

### 9.1 设计优点

1. **三段校验**：前端对话框校验 + 前端点击处理 + 后端校验，确保数据安全
2. **统一错误体系**：AppError 统一错误码 + 友好用户消息分离，支持国际化
3. **字段元数据驱动**：通过 fieldMeta 配置灵活配置各字段校验规则
4. **错误分类展示**：字段级错误即时反馈，系统级错误 Toast 提示，认证错误单独处理
5. **V2 路径 EMAIL/NAME/INITIALS 完整校验**：填补了 V1 路径的校验空白

### 9.2 现存问题与代码缺陷

| 问题描述 | 位置 | 影响 |
|---------|------|------|
| **V2 signField 缺少 AppError.parseError** | `envelope-signer-page-renderer.tsx:464-474` | 无法区分错误类型进行差异化处理 |
| **V2 signField 缺少 UNAUTHORIZED 检查** | `envelope-signer-page-renderer.tsx:464-474` | 认证失败也会显示通用 Error Toast，与 Alert 重复提示 |
| **V2 signField 执行顺序不合理** | `envelope-signer-page-renderer.tsx:464-474` | `console.error → toast → throw`，应先解析错误再处理 |
| **V1 路径错误处理不一致** | `sign-field-with-token.ts:131-133` | 使用普通 `Error` 而非 `AppError`，丢失错误码 |
| **TEXT 校验错误消息错误** | `envelope-signing.ts:131` | 错误消息为 "Invalid email"，应为 "Invalid text" |
| **INITIALS 返回值 Bug** | `initial-field.ts:43` | 返回 `initials` 而非 `initialsToInsert`，可能导致值不正确 |
| **过时的 TODO 注释** | `auth-password.tsx:68`<br>`auth-2fa.tsx:73`<br>`auth-passkey.tsx:98` | `// Todo: Alert.` 注释已过时，Alert 已实现，可能误导开发 |
| **Passkey 未完整处理错误** | `auth-passkey.tsx:91-93` | 仅处理 `NotAllowedError`，其他错误继续执行可能显示双重提示 |
| **校验逻辑重复** | V1 与 V2 路径 | 状态校验逻辑重复，维护成本高 |
| **错误消息硬编码** | 所有 validate-*.ts | 校验函数中错误消息为英文硬编码，未使用 i18n |
| **V1 EMAIL/NAME/INITIALS 无校验** | V1 路径 | 存在数据安全隐患 |

> **修正说明**：已移除 "认证错误码未展示给用户" 的问题，因为代码中所有认证方式（密码、2FA、Passkey）都已实现了 `{formErrorCode && (<Alert>...</Alert>)}` 的条件渲染。问题是 signField catch 中提前显示了多余的 Toast，导致双重提示。

### 9.3 代码优化建议

#### 优化点1：V1 路径统一使用 AppError

**问题**：V1 路径使用普通 Error 而非 AppError

**优化前** (`sign-field-with-token.ts:131-133`):
```typescript
if (errors.length > 0) {
  throw new Error(errors.join(', '));
}
```

**优化后**:
```typescript
if (errors.length > 0) {
  throw new AppError(AppErrorCode.INVALID_REQUEST, {
    message: errors.join(', '),
    userMessage: errors.join(', '),
  });
}
```

#### 优化点2：修复 TEXT 字段错误消息

**问题**：TEXT 字段校验失败返回 "Invalid email"

**优化前** (`envelope-signing.ts:129-133`):
```typescript
if (errors.length > 0) {
  throw new AppError(AppErrorCode.INVALID_BODY, {
    message: 'Invalid email',  // Bug
  });
}
```

**优化后**:
```typescript
if (errors.length > 0) {
  throw new AppError(AppErrorCode.INVALID_BODY, {
    message: 'Invalid text',
  });
}
```

#### 优化点3：修复 INITIALS 字段返回值 Bug

**问题**：返回 `initials` 而非 `initialsToInsert`

**优化前** (`initial-field.ts:41-44`):
```typescript
return {
  type: FieldType.INITIALS,
  value: initials,  // Bug
};
```

**优化后**:
```typescript
return {
  type: FieldType.INITIALS,
  value: initialsToInsert,
};
```

#### 优化点4：V2 signField 错误处理逻辑修正

**问题**：缺少错误解析和 UNAUTHORIZED 检查，所有错误都 toast

**优化前** (`envelope-signer-page-renderer.tsx:464-474`):
```typescript
catch (err) {
  console.error(err);

  toast({
    title: t`Error`,
    description: t`An error occurred while signing the field.`,
    variant: 'destructive',
  });

  throw err;
}
```

**优化后**:
```typescript
catch (err) {
  const error = AppError.parseError(err);

  if (error.code === AppErrorCode.UNAUTHORIZED) {
    throw error;
  }

  console.error(err);

  toast({
    title: t`Error`,
    description: t`An error occurred while signing the field.`,
    variant: 'destructive',
  });
}
```

#### 优化点5：错误消息国际化

**问题**：校验函数返回硬编码英文消息

**优化建议**：校验函数返回错误码而非直接消息，前端根据错误码进行 i18n 翻译。

```typescript
// 优化前
return ['Value is required'];

// 优化后
return [
  { code: 'FIELD_REQUIRED', params: { fieldType: 'TEXT' } }
];
```

#### 优化点6：移除过时的 TODO 注释

**问题**：`// Todo: Alert.` 注释已过时，Alert 显示功能已实现

**优化建议**：删除三个认证组件中的过时注释，避免误导后续开发。

**优化前** (`auth-password.tsx:68`):
```typescript
// Todo: Alert.
```

**优化后**:
```typescript
// 删除过时注释，Alert 显示功能已在第 87-96 行实现
```

**需修改文件**：
- `auth-password.tsx:68`
- `auth-2fa.tsx:73`
- `auth-passkey.tsx:98`

---

#### 优化点7：Passkey 错误处理完善

**问题**：仅处理 `NotAllowedError`，其他错误继续执行可能显示双重提示

**优化建议**：在 Passkey 认证组件中完整处理 UNAUTHORIZED 错误。

**优化前** (`auth-passkey.tsx:88-99`):
```typescript
catch (err) {
  setIsCurrentlyAuthenticating(false);

  if (err.name === 'NotAllowedError') {
    return;
  }

  const error = AppError.parseError(err);
  setFormErrorCode(error.code);

  // Todo: Alert.
}
```

**优化后**:
```typescript
catch (err) {
  setIsCurrentlyAuthenticating(false);

  if (err.name === 'NotAllowedError') {
    return;
  }

  const error = AppError.parseError(err);
  
  // UNAUTHORIZED 错误由认证表单 Alert 处理，不重复处理
  if (error.code !== AppErrorCode.UNAUTHORIZED) {
    setFormErrorCode(error.code);
  }
}
```
