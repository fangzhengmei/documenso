# 签署字段校验与错误反馈流程分析

## 一、整体架构概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                     签署字段提交校验与错误反馈完整流程                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌──────────────┐    ┌────────────────┐            │
│  │  前端UI层  │───▶│  tRPC API层  │───▶│  业务逻辑层    │            │
│  │  (双校验)   │    │  (Schema校验) │    │  (多维度校验)  │            │
│  └─────────────┘    └──────────────┘    └────────────────┘            │
│         │                   │                   │                                 │
│         │                   │                   ▼                                 │
│         │                   │            ┌──────────────────────┐                          │
│         │                   │            │  数据库事务层  │                          │
│         │                   │            │  (数据持久化)   │                          │
│         │                   │            └──────────────────────┘                          │
│         │                   │                   │                                 │
│         ▼                   ▼                   ▼                                 │
│  ┌──────────────────────────────────────────────────────┐                          │
│  │                  错误回传与展示层                        │                          │
│  │  AppError → tRPC格式化 → 前端解析 → UI展示               │                          │
│  └──────────────────────────────────────────────────────┘                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、前端校验分层设计

### 2.1 校验层级概览

| 层级 | 位置 | 校验时机 | 主要职责 |
|------|------|----------|----------|
| **L1 前端实时校验 | 各字段组件 | 用户输入时 | 实时验证，即时反馈 |
| **L2 前端提交前校验 | 对话框Sign按钮 | 点击提交前 | 最终校验，阻止无效提交 |
| **L3 tRPC Schema校验 | tRPC路由入口 | 请求到达时 | 输入参数格式校验 |
| **L4 业务状态校验 | sign-field-with-token / sign-envelope-field | 业务逻辑入口 | 文档/接收者/字段状态校验 |
| **L5 字段类型校验 | validate-* 系列函数 | 字段值处理前 | 字段值业务规则校验 |
| **L6 权限认证校验 | validateFieldAuth | 签名字段提交前 | 签署人身份认证校验 |

---

## 三、各字段类型校验规则详解

### 3.1 字段类型定义

**文件**: `packages/lib/types/field.ts`

支持的字段类型及对应校验函数：

| 字段类型 | 校验函数 | 校验规则所在文件 |
|----------|----------|------------------|
| `TEXT` | `validateTextField` | `validate-text.ts |
| `NUMBER` | `validateNumberField` | `validate-number.ts |
| `CHECKBOX` | `validateCheckboxField` | `validate-checkbox.ts |
| `RADIO` | `validateRadioField` | `validate-radio.ts |
| `DROPDOWN` | `validateDropdownField` | `validate-dropdown.ts |
| `SIGNATURE` | 内联校验 | `sign-field-with-token.ts` |
| `DATE` | 内联校验 | 自动填入当前日期 |
| `EMAIL` / `NAME` / `INITIALS` | 无特殊校验 | - |

### 3.2 文本字段 (TEXT) 校验规则

**文件**: `packages/lib/advanced-fields-validation/validate-text.ts:3-33

```typescript
validateTextField(value: string, fieldMeta: TTextFieldMeta, isSigningPage: boolean): string[]
```

校验规则：
1. **必填校验**：`required && !value && isSigningPage → "Value is required"
2. **字符长度限制**：`characterLimit > 0 && value.length > characterLimit` → 超限提示
3. **只读字段校验**：`readOnly && value.length < 1` → "A read-only field must have text"
4. **冲突校验**：`readOnly && required` → "A field cannot be both read-only and required"
5. **字体大小校验**：`fontSize < 8 || fontSize > 96` → 字体大小范围校验

### 3.3 数字字段 (NUMBER) 校验规则

**文件**: `packages/lib/advanced-fields-validation/validate-number.ts:5-65

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
5. **范围冲突：`minValue > maxValue` → 配置错误
6. **只读校验**：`readOnly && numberValue < 1` → 只读字段必须有值
7. **冲突校验**：`readOnly && required` → 不能同时为只读和必填

### 3.4 复选框字段 (CHECKBOX) 校验规则

**文件**: `packages/lib/advanced-fields-validation/validate-checkbox.ts:6-74

```typescript
validateCheckboxField(values: string[], fieldMeta: TCheckboxFieldMeta, isSigningPage: boolean): string[]
```

校验规则：
1. **冲突校验：`readOnly && required` → 不能同时为只读和必填
2. **选项存在**：`values.length === 0` → "At least one option must be added"
3. **必填校验**：`isSigningPage && required && values.length === 0` → "Selecting an option is required"
4. **验证规则完整性**：
   - `validationRule && !validationLength` → 需指定验证数量
   - `validationLength && !validationRule` → 需指定验证规则
5. **选择数量验证**：
   - `=`：`values.length !== validationLength` → 需精确选择N个
   - `>=`：`values.length < validationLength` → 至少选择N个
   - `<=`：`values.length > validationLength` → 最多选择N个

### 3.5 单选字段 (RADIO) 校验规则

**文件**: `packages/lib/advanced-fields-validation/validate-radio.ts:3-37

```typescript
validateRadioField(value: string | undefined, fieldMeta: TRadioFieldMeta, isSigningPage: boolean): string[]
```

校验规则：
1. **冲突校验**：`readOnly && required` → 不能同时为只读和必填
2. **必填校验**：`isSigningPage && required && !value` → "Choosing an option is required"
3. **选项数量**：`values.length === 0` → "Radio field must have at least one option"
4. **单选约束**：`checkedRadioFieldValues.length > 1` → "There cannot be more than one checked option"

### 3.6 下拉字段 (DROPDOWN) 校验规则

**文件**: `packages/lib/advanced-fields-validation/validate-dropdown.ts:3-54

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

### 3.7 签名字段 (SIGNATURE) 校验规则

**文件**: `packages/lib/server-only/field/sign-field-with-token.ts:203-209

```typescript
// 内联校验逻辑：
1. 签名值存在：`!signatureImageAsBase64 && !typedSignature` → "Signature field must have a signature"
2. 类型签名限制：`typedSignatureEnabled === false && typedSignature` → "Typed signatures are not allowed"
```

---

## 四、校验生效路径详解

### 4.1 V2 信封字段签署路径 (Envelope V2)

**入口文件**: `packages/trpc/server/envelope-router/sign-envelope-field.ts:15-283

```
用户点击签署
    │
    ▼
1. tRPC Schema 校验 (ZSignEnvelopeFieldRequestSchema)
    │  - token: string
    │  - fieldId: number
    │  - fieldValue: ZSignEnvelopeFieldValue (discriminated union by type)
    │  - authOptions?: TRecipientActionAuth
    │
    ▼
2. 基础状态校验 (sign-envelope-field.ts:28-126)
    │  ├─ 接收者存在性检查 → NOT_FOUND
    │  ├─ 字段存在性检查 → NOT_FOUND
    │  ├─ 信封版本检查 (internalVersion === 2)
    │  ├─ 助理角色限制 (ASSISTANT 不能签签名)
    │  ├─ 字段类型匹配检查 (fieldValue.type === field.type)
    │  ├─ 信封删除状态检查 → INVALID_REQUEST
    │  ├─ 信封状态检查 (status === PENDING)
    │  ├─ 接收者签署状态检查 (未签署)
    │  └─ 字段只读状态检查 (fieldMeta?.readOnly)
    │
    ▼
3. 字段值提取 (extractFieldInsertionValues)
    │
    ├─ 取消插入路径 (!insertionValues.inserted → 清除字段值
    │
    ▼
4. 认证校验 (validateFieldAuth)
    │  仅签名字段需要认证
    │  - document/auth → UNAUTHORIZED
    │
    ▼
5. 签名字段特殊处理
    │  - Base64 图片识别
    │  - 类型签名/绘制签名分离
    │
    ▼
6. 数据库事务更新
    ├─ 更新 field 表 (customText, inserted)
    ├─ 签名 upsert (SIGNATURE)
    └─ 审计日志记录
```

### 4.2 V1 文档字段签署路径 (Document V1)

**入口文件**: `packages/trpc/server/field-router/router.ts:591-609

```
用户点击签署
    │
    ▼
1. tRPC Schema 校验 (ZSignFieldWithTokenMutationSchema)
    │  - token: string
    │  - fieldId: number
    │  - value: string (trim().optional()
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
    │  └─ DROPDOWN → validateDropdownField()
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

### 4.3 前端实时校验路径

**以文本字段为例**: `apps/remix/app/components/general/document-signing/document-signing-text-field.tsx:49-218

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

## 五、错误信息回传路径

### 5.1 错误对象体系设计

**核心文件**: `packages/lib/errors/app-error.ts:88-257

#### AppErrorCode 枚举：
```typescript
enum AppErrorCode {
  NOT_FOUND = 'NOT_FOUND',
  INVALID_REQUEST = 'INVALID_REQUEST',
  UNAUTHORIZED = 'UNAUTHORIZED',
  FORBIDDEN = 'FORBIDDEN',
  // ... 其他错误码
}
```

#### AppError 类结构：
```typescript
class AppError extends Error {
  code: string;           // 错误码
  message?: string;           // 内部日志消息
  userMessage?: string;   // 可展示给用户的消息
  statusCode?: number;    // HTTP 状态码
  headers?: Record<string, string>;
}
```

### 5.2 后端错误抛出与格式化

#### 方式一：AppError 抛出（推荐）
**文件**: `packages/trpc/server/envelope-router/sign-envelope-field.ts:34-36

```typescript
// 抛出 AppError
throw new AppError(AppErrorCode.NOT_FOUND);

// 带自定义消息
throw new AppError(AppErrorCode.INVALID_REQUEST, {
  message: `Document ${envelope.id} has been deleted`
});
```

#### 方式二：普通 Error 抛出（V1 路径）
**文件**: `packages/lib/server-only/field/sign-field-with-token.ts:131-133

```typescript
if (errors.length > 0) {
  throw new Error(errors.join(', '));
}
```

### 5.3 tRPC 错误格式化中间件

**文件**: `packages/trpc/server/trpc.ts:38-66

```typescript
errorFormatter(opts) {
  const { shape, error, ctx } = opts;
  const originalError = error.cause;

  if (originalError instanceof AppError) {
    data = {
      ...data,
      appError: AppError.toJSON(originalError),
      code: originalError.code,
      httpStatus: originalError.statusCode ?? genericErrorCodeToTrpcErrorCodeMap[originalError.code]?.status ?? 400,
    };
  }

  return { ...shape, data };
}
```

### 5.4 tRPC 错误日志处理

**文件**: `packages/trpc/utils/trpc-error-handler.ts:7-36

```typescript
handleTrpcRouterError({ error, ctx, path }, source) {
  const appError = AppError.parseError(error.cause || error);
  // 根据错误类型决定日志级别
  // 500 错误或特定错误码记录 error 级别
  // 其他错误记录 info 级别
}
```

### 5.5 前端错误解析

**文件**: `packages/lib/errors/app-error.ts:130-168

```typescript
static parseError(error: any): AppError {
  // 1. 已是 AppError 直接返回
  if (error instanceof AppError) return error;

  // 2. 处理 TRPCClientError
  if (error?.name === 'TRPCClientError') {
    const parsedJsonError = AppError.parseFromJSON(error.data?.appError);
    return parsedJsonError || fallbackError;
  }

  // 3. 未知错误转换为 UNKNOWN_ERROR
  return new AppError(validCode, options);
}
```

### 5.6 前端错误展示

#### 方式一：字段级错误展示（实时校验）
**文件**: `apps/remix/app/components/general/document-signing/document-signing-text-field.tsx:89-100

```typescript
const handleTextChange = (e: React.ChangeEvent<HTMLTextAreaElement>) => {
  const text = e.target.value;
  setLocalCustomText(text);

  if (parsedFieldMeta) {
    const validationErrors = validateTextField(text, parsedFieldMeta, true);
    setErrors({
      required: validationErrors.filter((error) => error.includes('required')),
      characterLimit: validationErrors.filter((error) => error.includes('character limit')),
    });
  }
};
```

#### 方式二：Toast 全局提示（后端错误）
**文件**: `apps/remix/app/components/general/document-signing/document-signing-text-field.tsx:164-179

```typescript
catch (err) {
  const error = AppError.parseError(err);

  if (error.code === AppErrorCode.UNAUTHORIZED) {
    throw error;  // 重新抛出由认证上下文处理
  }

  toast({
    title: _(msg`Error`),
    description: _(msg`An error occurred while signing the document.`),
    variant: 'destructive',
  });
}
```

#### 方式三：未插入字段提示
**文件**: `packages/lib/utils/fields.ts:25-65

```typescript
export const validateFieldsInserted = (fields: Field[]): boolean => {
  // 设置 data-validate 属性触发字段高亮
  // 滚动到第一个未插入字段
  // 返回是否所有字段已插入
}
```

---

## 六、两条签署路径对比

| 维度 | V2 信封路径 (sign-envelope-field) | V1 文档路径 (sign-field-with-token) |
|------|---------------------------------|-----------------------------------|
| **入口** | `trpc.envelope.signEnvelopeField | `trpc.field.signFieldWithToken` |
| **Schema** | `ZSignEnvelopeFieldRequestSchema | `ZSignFieldWithTokenMutationSchema` |
| **字段值格式** | discriminated union by type | 统一 string |
| **字段类型校验** | 在 `extractFieldInsertionValues 中处理 | 调用 validate-* 系列函数 |
| **错误抛出** | AppError（带错误码） | 普通 Error（消息字符串） |
| **适用场景** | Envelope V2 文档 | 传统 Document V1 文档 |
| **字段值类型** | CHECKBOX: number[]<br>RADIO: number \| null<br>DATE: boolean | 统一 string 编码 |

---

## 七、关键代码位置索引

### 校验函数位置
| 功能 | 文件路径 | 行号 |
|------|---------|------|
| 文本字段校验 | `packages/lib/advanced-fields-validation/validate-text.ts` | 3-33 |
| 数字字段校验 | `packages/lib/advanced-fields-validation/validate-number.ts` | 5-65 |
| 复选框字段校验 | `packages/lib/advanced-fields-validation/validate-checkbox.ts` | 6-74 |
| 单选字段校验 | `packages/lib/advanced-fields-validation/validate-radio.ts` | 3-37 |
| 下拉字段校验 | `packages/lib/advanced-fields-validation/validate-dropdown.ts` | 3-54 |
| 字段认证校验 | `packages/lib/server-only/document/validate-field-auth.ts` | 21-48 |

### 路由入口位置
| 功能 | 文件路径 | 行号 |
|------|---------|------|
| V2 信封签署路由 | `packages/trpc/server/envelope-router/sign-envelope-field.ts` | 15-283 |
| V1 字段签署路由 | `packages/trpc/server/field-router/router.ts` | 591-609 |
| V1 签署业务逻辑 | `packages/lib/server-only/field/sign-field-with-token.ts` | 50-313 |

### 错误处理位置
| 功能 | 文件路径 | 行号 |
|------|---------|------|
| AppError 定义 | `packages/lib/errors/app-error.ts` | 88-257 |
| tRPC 错误格式化 | `packages/trpc/server/trpc.ts` | 38-66 |
| tRPC 错误日志 | `packages/trpc/utils/trpc-error-handler.ts` | 7-36 |
| 前端错误解析 | `packages/lib/errors/app-error.ts` | 130-168 |

### 前端组件位置
| 功能 | 文件路径 | 行号 |
|------|---------|------|
| 文本字段组件 | `apps/remix/app/components/general/document-signing/document-signing-text-field.tsx` | 49-218 |
| 数字字段组件 | `apps/remix/app/components/general/document-signing/document-signing-number-field.tsx` | 46-120 |
| 签署表单组件 | `apps/remix/app/components/general/document-signing/document-signing-form.tsx` | 42-300 |
| 字段插入校验 | `packages/lib/utils/fields.ts` | 25-65 |

---

## 八、设计特点与注意事项

### 8.1 设计优点
1. **双层校验**：前端实时校验提升用户体验，后端校验确保数据安全
2. **统一错误体系**：AppError 统一错误码 + 友好用户消息分离
3. **字段元数据驱动**：通过 fieldMeta 配置灵活配置各字段校验规则
4. **错误分类展示**：字段级错误即时反馈，系统级错误 Toast 提示

### 8.2 现存问题
1. **V1 路径错误处理不一致**：使用普通 `throw new Error(errors.join(', ')) 而非 AppError
2. **校验逻辑重复**：sign-field-with-token.ts 与 sign-envelope-field.ts 存在重复逻辑
3. **错误消息硬编码**：校验函数中错误消息为英文硬编码，未使用 i18n
4. **字段类型差异**：V1 与 V2 路径字段值格式不统一

### 8.3 代码优化建议

**问题**：V1 路径使用普通 Error 而非 AppError

**优化前** (`sign-field-with-token.ts:131-133):
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

**问题**：前端错误消息未国际化

**优化建议**：校验函数返回错误码而非直接消息，前端根据错误码进行 i18n 翻译。
