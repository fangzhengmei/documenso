# PDF 字段值序列化分析

本文档详细分析 Documenso 中 PDF 字段的四种状态（编辑态、存储态、文档发送阶段自动回填、签署提交阶段）的数据结构及其相互转换逻辑。所有内容已逐行核对源码，确保 100% 与实现一致。

## 状态概览

```
┌──────────────────────────────────────────────────────────────────────────┐
│                             编辑态 (Editor)                               │
│                     fieldMeta 属性配置与预填充值设置                       │
└─────────────────────────────────────┬────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                           存储态 (Database)                               │
│                  Field 表 + Signature 表（签名字段专用）                   │
└──────────────────────────┬───────────────────────────────────┬────────────┘
                           │                                   │
                           ▼                                   ▼
┌──────────────────────────────────────┐   ┌──────────────────────────────────┐
│        文档发送阶段自动回填          │   │        签署提交阶段 (Signing)    │
│    extractFieldAutoInsertValues     │   │    extractFieldInsertionValues   │
│      (sendDocument 时触发)          │   │        (签名页面提交时)          │
└──────────────────────────┬───────────┘   └───────────────┬──────────────────┘
                           │                                  │
                           └──────────────────────────────────┘
                                              ▼
                                    ┌─────────────────┐
                                    │   渲染态 (Render)
                                    │    渲染到 PDF
                                    └─────────────────┘
```

---

## 1. 编辑态 (Editing State)

### 定义
编辑态是用户在编辑器中配置字段属性时的状态。主要通过 `fieldMeta` JSON 对象来存储字段的元数据配置。

### 数据结构

#### 基础类型定义 (`packages/lib/types/field-meta.ts`)

```typescript
// 基础字段元数据
type TBaseFieldMeta = {
  label?: string;           // 字段标签
  placeholder?: string;     // 占位符
  required?: boolean;       // 是否必填
  readOnly?: boolean;       // 是否只读
  fontSize?: number;        // 字体大小 (8-96)
  overflow?: 'auto' | 'horizontal' | 'vertical' | 'crop';  // 溢出模式
};
```

#### 各字段类型的元数据结构

| 字段类型 | 类型标识 | fieldMeta 类型 | 特有字段 | 可自动回填 |
|---------|---------|----------------|---------|-----------|
| **签名** | `SIGNATURE` | `ZSignatureFieldMeta` | `overflow` | ❌ 否 |
| **自由签名** | `FREE_SIGNATURE` | `undefined` | 无 | ❌ 否 |
| **首字母** | `INITIALS` | `ZInitialsFieldMeta` | `textAlign` | ❌ 否 |
| **姓名** | `NAME` | `ZNameFieldMeta` | `textAlign` | ❌ 否 |
| **邮箱** | `EMAIL` | `ZEmailFieldMeta` | `textAlign`, `overflow` | ✅ 是 |
| **日期** | `DATE` | `ZDateFieldMeta` | `textAlign`, `overflow` | ❌ 否 |
| **文本** | `TEXT` | `ZTextFieldMeta` | `text`, `characterLimit`, `textAlign`, `lineHeight`, `letterSpacing`, `verticalAlign` | ✅ 是 |
| **数字** | `NUMBER` | `ZNumberFieldMeta` | `numberFormat`, `value`, `minValue`, `maxValue`, `textAlign`, 等 | ✅ 是 |
| **单选** | `RADIO` | `ZRadioFieldMeta` | `values: {id: number; checked: boolean; value: string}[]`, `direction` | ✅ 是 |
| **复选** | `CHECKBOX` | `ZCheckboxFieldMeta` | `values[]`, `validationRule`, `validationLength`, `direction` | ✅ 是 |
| **下拉** | `DROPDOWN` | `ZDropdownFieldMeta` | `values: {value: string}[]`, `defaultValue` | ✅ 是 |

### 编辑态预填充值设置说明

| 字段类型 | 预填充来源 | 说明 |
|---------|-----------|------|
| **EMAIL** | 无预填充 | 发送时自动用 recipient.email 回填 |
| **TEXT** | `fieldMeta.text` | 编辑器中设置的默认文本 |
| **NUMBER** | `fieldMeta.value` | 编辑器中设置的默认数值 |
| **RADIO** | `fieldMeta.values[i].checked` | 编辑器中预设的选中项 |
| **CHECKBOX** | `fieldMeta.values[i].checked` | 编辑器中预设的选中项数组 |
| **DROPDOWN** | `fieldMeta.defaultValue` | 编辑器中设置的默认选项值 |

---

## 2. 存储态 (Storage State)

### 定义
存储态是字段在数据库中持久化存储的状态。使用 Prisma 模型定义，通过 `Field` 表存储所有字段，`Signature` 表专门存储签名相关数据。

### 数据库表结构

#### Field 表 (`packages/prisma/schema.prisma`)
```prisma
model Field {
  id             Int      @id @default(autoincrement())
  secondaryId    String   @unique @default(cuid())
  envelopeId     String
  envelopeItemId String
  recipientId    Int
  type           FieldType  // 枚举: SIGNATURE, FREE_SIGNATURE, NAME, EMAIL, DATE, TEXT, NUMBER, RADIO, CHECKBOX, DROPDOWN, INITIALS
  page           Int
  
  // 位置和尺寸 (Decimal 类型)
  positionX      Decimal  @default(0)
  positionY      Decimal  @default(0)
  width          Decimal  @default(-1)
  height         Decimal  @default(-1)
  
  // 字段值存储 - 核心序列化字段
  customText     String   // 序列化后的字段值
  inserted       Boolean  // 是否已填写/签名
  
  // 元数据 (JSON)
  fieldMeta      Json?
  
  // 关联
  envelope       Envelope     @relation(fields: [envelopeId], references: [id])
  recipient      Recipient    @relation(fields: [recipientId], references: [id])
  signature      Signature?   // 仅签名字段有值
  
  @@index([envelopeId])
  @@index([envelopeItemId])
  @@index([recipientId])
}
```

#### Signature 表（仅 SIGNATURE / FREE_SIGNATURE 字段使用）
```prisma
model Signature {
  id                     Int      @id @default(autoincrement())
  created                DateTime @default(now())
  recipientId            Int
  fieldId                Int      @unique
  signatureImageAsBase64 String?  // 手写签名图片 (base64)
  typedSignature         String?  // 键入签名文本
  
  recipient Recipient @relation(fields: [recipientId], references: [id])
  field     Field     @relation(fields: [fieldId], references: [id])
}
```

### 存储态值编码规则 (customText)

| 字段类型 | 存储格式 | 说明 | 示例值 |
|---------|---------|------|--------|
| **EMAIL** | 邮箱字符串 | 直接存储 | `"user@example.com"` |
| **NAME** | 姓名字符串 | 直接存储 | `"张三"` |
| **INITIALS** | 首字母字符串 | 直接存储 | `"ZS"` |
| **DATE** | 格式化日期字符串 | 发送/签名时的当前日期 | `"2024-01-15"` |
| **NUMBER** | 数字字符串 | 直接存储 | `"123.45"` |
| **TEXT** | 文本内容 | 直接存储 | `"这是一段文本"` |
| **RADIO** | 选中索引的字符串形式 | `toRadioCustomText(index)` | `"1"` |
| **CHECKBOX** | 选中索引数组的 JSON 字符串 | `toCheckboxCustomText(indices)` | `"[0,2]"` |
| **DROPDOWN** | 选中值字符串 | 直接存储 | `"选项B"` |
| **SIGNATURE** | `""` (空字符串) | 实际值存储在 Signature 关联表 | `""` |
| **FREE_SIGNATURE** | `""` (空字符串) | 同 SIGNATURE，实际值存储在 Signature 表 | `""` |

---

## 3. 发送态 - 链路一：文档发送阶段自动回填

### 触发时机
`sendDocument` 函数执行时（仅 V2 信封，internalVersion === 2），在文档从草稿状态转为待发送状态之前。

### 核心函数
**函数名**: `extractFieldAutoInsertValues`  
**文件**: `packages/lib/server-only/document/send-document.ts` (line 359-471)

### 函数签名
```typescript
type extractFieldAutoInsertValues = (
  unknownField: Field,
  recipient: Pick<Recipient, 'email'>
) => { fieldId: number; customText: string } | null;
```

### 自动回填规则

| 字段类型 | 回填条件 | 回填值来源 | 说明 |
|---------|---------|-----------|------|
| **EMAIL** | `isRecipientEmailValidForSending(recipient)` 且<br>`recipient.email !== DIRECT_TEMPLATE_RECIPIENT_EMAIL` | `recipient.email` | 自动填充接收者邮箱 |
| **TEXT** | `fieldMeta.text` 存在且非空 | `fieldMeta.text` | 使用编辑器预设的文本 |
| **NUMBER** | `fieldMeta.value` 存在且非空 | `fieldMeta.value` | 使用编辑器预设的数值 |
| **RADIO** | `fieldMeta.values` 中存在 `checked=true` 的项 | 第一个 checked 项的索引 `→ toRadioCustomText(index)` | 使用编辑器预设的选中项 |
| **DROPDOWN** | `fieldMeta.defaultValue` 存在<br>且该值在 `fieldMeta.values` 中 | `fieldMeta.defaultValue` | 使用编辑器预设的默认选项 |
| **CHECKBOX** | `fieldMeta.values` 中存在 `checked=true` 的项<br>且通过验证规则检查 | 所有 checked 项的索引数组 `→ toCheckboxCustomText(indices)` | 使用编辑器预设的选中项 |
| **SIGNATURE** | - | 不自动回填 | 需要用户手动签名 |
| **FREE_SIGNATURE** | - | 不自动回填 | fieldMeta 为 undefined，需要手动签名 |
| **NAME** | - | 不自动回填 | 需要用户填写 |
| **INITIALS** | - | 不自动回填 | 需要用户填写 |
| **DATE** | - | 不自动回填 | 签署时自动填充当前日期 |

### 关键限制条件
1. **仅 V2 信封生效**: `envelope.internalVersion === 2` (line 184)
2. **仅未发送的接收者**: `recipient.sendStatus !== SendStatus.SENT` (line 197)
3. **自动设置 inserted=true**: 回填后字段自动标记为已填写
4. **审计日志**: 记录 `DOCUMENT_FIELDS_AUTO_INSERTED` 事件

---

## 4. 发送态 - 链路二：签署提交阶段

### 触发时机
用户在签名页面点击"完成"或"签署"按钮时，通过 tRPC API `envelope.signField` 提交。

### 核心函数
**函数名**: `extractFieldInsertionValues`  
**文件**: `packages/lib/utils/envelope-signing.ts` (line 33-253)

### 函数签名
```typescript
type extractFieldInsertionValues = (options: {
  fieldValue: TSignEnvelopeFieldValue;
  field: Field;
  documentMeta: Pick<TDocumentMeta, 'timezone' | 'dateFormat' | 'typedSignatureEnabled'>;
}) => { customText: string; inserted: boolean };
```

### 请求数据结构 (TSignEnvelopeFieldValue)
**文件**: `packages/trpc/server/envelope-router/sign-envelope-field.types.ts` (line 7-48)

```typescript
type TSignEnvelopeFieldValue = 
  | { type: 'CHECKBOX'; value: number[] }       // 选中项的索引数组
  | { type: 'RADIO'; value: number | null }     // 选中项的索引
  | { type: 'NUMBER'; value: string | null }    // 数字字符串
  | { type: 'EMAIL'; value: string | null }     // 邮箱字符串
  | { type: 'NAME'; value: string | null }      // 姓名字符串
  | { type: 'INITIALS'; value: string | null }  // 首字母字符串
  | { type: 'TEXT'; value: string | null }      // 文本字符串
  | { type: 'DROPDOWN'; value: string | null }  // 下拉选中值
  | { type: 'DATE'; value: boolean }            // true = 使用当前日期
  | { type: 'SIGNATURE'; value: string | null } // 签名值 (base64 图片或文本)
```

> **重要说明**: `FREE_SIGNATURE` 类型未在 tRPC 签署请求 schema 中定义，仅在 `signFieldWithToken` 旧函数中支持。

### 签署提交转换规则

| 字段类型 | 输入格式 | 验证规则 | 转换到 customText |
|---------|---------|---------|------------------|
| **CHECKBOX** | `number[]` (索引数组) | 1. 所有索引在 values 范围内<br>2. 通过 `validationRule + validationLength` 检查 | `JSON.stringify(value)` |
| **RADIO** | `number \| null` (索引) | 索引在 values 范围内 | `value.toString()` |
| **NUMBER** | `string \| null` | 通过数字字段格式验证 | 直接存储 |
| **EMAIL** | `string \| null` | 邮箱格式验证 | 直接存储 |
| **NAME** | `string \| null` | 非空验证 | 直接存储 |
| **INITIALS** | `string \| null` | 非空验证 | 直接存储 |
| **TEXT** | `string \| null` | 通过文本长度/格式验证 | 直接存储 |
| **DROPDOWN** | `string \| null` | 值在 values 列表中存在 | 直接存储 |
| **DATE** | `boolean` | `true` 触发 | 格式化当前日期字符串 |
| **SIGNATURE** | `string \| null` (base64 或文本) | 1. 非空<br>2. 若 typedSignatureEnabled=false 则必须是 base64 图片 | `""` (空字符串，值存储在 Signature 表) |

---

## 5. 渲染态 (Rendering State)

### 核心函数
**函数名**: `renderField`  
**文件**: `packages/lib/universal/field-renderer/render-field.ts` (line 57-92)

### 渲染模式
```typescript
type FieldRenderMode = 
  | 'edit'    // 编辑器页面渲染
  | 'sign'    // 签名页面渲染
  | 'export'; // 导出到 PDF 时渲染
```

### 反序列化与渲染规则

| 字段类型 | customText 格式 | 反序列化函数 | 渲染方式 |
|---------|----------------|------------|---------|
| **通用文本类**<br>(EMAIL, NAME, INITIALS, DATE, TEXT, NUMBER) | 直接字符串 | 无（直接使用） | `renderGenericTextFieldElement` |
| **CHECKBOX** | JSON 数组字符串 `"[0,2]"` | `parseCheckboxCustomText` → `number[]` | `renderCheckboxFieldElement` |
| **RADIO** | 数字字符串 `"1"` | `parseRadioCustomText` → `number` | `renderRadioFieldElement` |
| **DROPDOWN** | 直接字符串 | 无（直接使用） | `renderDropdownFieldElement` |
| **SIGNATURE** | `""` (空字符串) | 从 `field.signature` 关联读取 | `renderSignatureFieldElement` |
| **FREE_SIGNATURE** | `""` (空字符串) | 从 `field.signature` 关联读取 | ❌ 抛出错误 "Free signature fields are not supported" (line 88) |

---

## 关键序列化工具函数

### 1. parseCheckboxCustomText / toCheckboxCustomText (主流程序列化)
**文件**: `packages/lib/utils/fields.ts` (line 100-110)

> **⚠️ 主流程使用：签署提交、自动回填、渲染均使用此函数**

```typescript
/**
 * 存储态 customText → 渲染态索引数组
 * 输入: JSON 字符串，如 "[0,2]"
 * 输出: number[] 索引数组，如 [0, 2]
 */
function parseCheckboxCustomText(customText: string): number[] {
  if (!customText) {
    return [];
  }
  return JSON.parse(customText);
}

/**
 * 发送态/自动回填 → 存储态 customText
 * 输入: number[] 索引数组，如 [0, 2]
 * 输出: JSON 字符串，如 "[0,2]"
 */
function toCheckboxCustomText(checkedValues: number[]): string {
  return JSON.stringify(checkedValues);
}
```

### 2. parseRadioCustomText / toRadioCustomText (单选按钮序列化)
**文件**: `packages/lib/utils/fields.ts` (line 112-118)

```typescript
/**
 * 存储态 customText → 渲染态索引值
 * 输入: 数字字符串，如 "1"
 * 输出: number 数值，如 1
 */
function parseRadioCustomText(customText: string): number {
  return Number(customText);
}

/**
 * 发送态/自动回填 → 存储态 customText
 * 输入: number 索引值，如 1
 * 输出: 字符串，如 "1"
 */
function toRadioCustomText(value: number): string {
  return value.toString();
}
```

### 3. fromCheckboxValue / toCheckboxValue (通用复选框转换)
**文件**: `packages/lib/universal/field-checkbox.ts` (line 1-21)

> **⚠️ 重要说明：与签署提交流程和渲染流程无关，仅用于 V1 遗留 API 和特定验证场景**
> 主流程序列化使用 `parseCheckboxCustomText` / `toCheckboxCustomText`，切勿混淆！

```typescript
/**
 * 从 customText 反序列化为字符串数组
 * 
 * 【返回类型】: string[]，不是 number[]
 * 【fallback 行为】: JSON 解析失败时，直接按逗号分割字符串，不做 Number 转换
 */
function fromCheckboxValue(customText: string): string[] {
  if (!customText) {
    return [];
  }

  try {
    const parsed = JSON.parse(customText);
    
    if (!Array.isArray(parsed)) {
      throw new Error('Parsed checkbox values are not an array');
    }
    
    return parsed;  // 返回 string[]，取决于存入时的格式
  } catch {
    // 兼容旧格式：逗号分隔字符串 → ["0","2"]（字符串数组，不是数字）
    return customText.split(',').filter(Boolean);
  }
}

/**
 * 将字符串数组序列化为 JSON 字符串
 * 
 * 【输入类型】: string[]，不是 number[]
 */
function toCheckboxValue(values: string[]): string {
  return JSON.stringify(values);
}
```

---

## 状态转换完整对照表

| 字段类型 | 编辑态 (fieldMeta) | 文档发送自动回填<br>extractFieldAutoInsertValues | 签署提交输入<br>TSignEnvelopeFieldValue | 存储态<br>Field.customText | 渲染态 |
|---------|-------------------|-----------------------------------------------|----------------------------------------|---------------------------|--------|
| **EMAIL** | `{type: 'email'}` | ✅ `recipient.email` | `{type:'EMAIL', value:string}` | 邮箱字符串 | 直接渲染文本 |
| **NAME** | `{type: 'name'}` | ❌ 不回填 | `{type:'NAME', value:string}` | 姓名字符串 | 直接渲染文本 |
| **INITIALS** | `{type: 'initials'}` | ❌ 不回填 | `{type:'INITIALS', value:string}` | 首字母字符串 | 直接渲染文本 |
| **DATE** | `{type: 'date'}` | ❌ 不回填 | `{type:'DATE', value:boolean}` | 格式化日期 | 直接渲染文本 |
| **NUMBER** | `{type: 'number', value}` | ✅ `fieldMeta.value` | `{type:'NUMBER', value:string}` | 数字字符串 | 直接渲染文本 |
| **TEXT** | `{type: 'text', text}` | ✅ `fieldMeta.text` | `{type:'TEXT', value:string}` | 文本字符串 | 直接渲染文本 |
| **RADIO** | `{type: 'radio', values}` | ✅ `checked 项索引` | `{type:'RADIO', value:number}` | 索引字符串 `"1"` | 映射到 fieldMeta.values[index].value |
| **CHECKBOX** | `{type: 'checkbox', values}` | ✅ `checked 项索引数组` | `{type:'CHECKBOX', value:number[]}` | JSON 数组字符串 `"[0,2]"` | 映射到 fieldMeta.values[indices] |
| **DROPDOWN** | `{type: 'dropdown', defaultValue}` | ✅ `defaultValue` | `{type:'DROPDOWN', value:string}` | 选项值字符串 | 直接渲染选中值 |
| **SIGNATURE** | `{type: 'signature'}` | ❌ 不回填 | `{type:'SIGNATURE', value:string}` | `""` (空) | 从 Signature 表读取渲染 |
| **FREE_SIGNATURE** | `undefined` | ❌ 不回填 | ❌ 不在 tRPC schema 中 | `""` (空) | ❌ 不支持，抛出错误 |

> **脚注**: 
> - CHECKBOX 主流程序列化使用: `parseCheckboxCustomText` (存储→渲染) 和 `toCheckboxCustomText` (发送/回填→存储)，均使用 `number[]` 类型
> - `fromCheckboxValue`/`toCheckboxValue` 是另一套独立的通用转换函数，使用 `string[]` 类型，**不在**主序列化流程中使用

---

## 关键函数映射表

| 函数名 | 文件位置 | 作用链路 | 输入 → 输出 |
|-------|---------|---------|------------|
| `extractFieldAutoInsertValues` | `packages/lib/server-only/document/send-document.ts:359` | 文档发送 → 存储态 | `Field` → `{fieldId, customText}` \| `null` |
| `extractFieldInsertionValues` | `packages/lib/utils/envelope-signing.ts:33` | 签署提交 → 存储态 | `TSignEnvelopeFieldValue` → `{customText, inserted}` |
| `toCheckboxCustomText` | `packages/lib/utils/fields.ts:108` | 发送/自动回填 → 存储态 | `number[]` → `string` (JSON) |
| `parseCheckboxCustomText` | `packages/lib/utils/fields.ts:100` | 存储态 → 渲染态 | `string` → `number[]` |
| `toRadioCustomText` | `packages/lib/utils/fields.ts:116` | 发送/自动回填 → 存储态 | `number` → `string` |
| `parseRadioCustomText` | `packages/lib/utils/fields.ts:112` | 存储态 → 渲染态 | `string` → `number` |
| `fromCheckboxValue` | `packages/lib/universal/field-checkbox.ts:1` | 存储态 → 通用转换 | `string` → `string[]` (无 Number 转换) |
| `toCheckboxValue` | `packages/lib/universal/field-checkbox.ts:19` | 通用转换 → 存储态 | `string[]` → `string` (JSON) |
| `renderField` | `packages/lib/universal/field-renderer/render-field.ts:57` | 存储态 → 渲染到 PDF | `FieldToRender` → `Konva.Node` |

---

## 关键设计决策与注意事项

### 1. 使用索引数组而非实际值
- **原因**: 选项文本可能在签署前被修改，使用索引可以确保选项顺序不变时值正确映射
- **实现**: `CHECKBOX` 使用 `number[]`，`RADIO` 使用 `number`
- **注意**: 必须保证 `fieldMeta.values` 数组的顺序在签署过程中不被修改
- **映射关系**: 渲染时需手动 `fieldMeta.values[index].value` 才能得到显示文本

### 2. 签名字段值分离存储
- **原因**: 签名图片可能很大（base64 编码），分离存储避免 Field 表过大
- **实现**: `Field.customText = ""`，实际值存储在 `Signature` 关联表的 `signatureImageAsBase64` 或 `typedSignature` 字段

### 3. FREE_SIGNATURE 是遗留类型
- **现状**: V2 渲染器抛出错误不支持，仅在 V1 遗留 API 和 `signFieldWithToken` 中处理
- **区别**: `FREE_SIGNATURE` 的 `fieldMeta` 类型是 `undefined`，无配置项
- **建议**: 新代码使用 `SIGNATURE` 类型

### 4. 两套复选框转换函数并存
- **主流程**: `parseCheckboxCustomText` / `toCheckboxCustomText` → 使用 `number[]`
- **通用/遗留**: `fromCheckboxValue` / `toCheckboxValue` → 使用 `string[]`，无数值转换
- **切勿混淆**: 两者类型不兼容，混用可能导致运行时错误

### 5. 自动回填的条件限制
- 仅 `internalVersion === 2` 的 V2 信封支持自动回填
- 接收者必须是未发送状态 (`sendStatus !== SendStatus.SENT`)
- `DOCUMENT_FIELDS_AUTO_INSERTED` 审计日志记录自动回填事件

---

## ✅ 源码一致性核对清单（终稿确认）

以下 16 项已逐条核对源码，确认与文档描述 100% 一致：

| 核对项 | 函数名 | 核对结论 | 源码路径与行号 |
|-------|-------|---------|----------------|
| **1** | `parseCheckboxCustomText` | ✅ 一致<br>输入: `string`，输出: `number[]`<br>实现: `JSON.parse(customText)` | `packages/lib/utils/fields.ts:100` |
| **2** | `toCheckboxCustomText` | ✅ 一致<br>输入: `number[]`，输出: `string`<br>实现: `JSON.stringify(checkedValues)` | `packages/lib/utils/fields.ts:108` |
| **3** | `parseRadioCustomText` | ✅ 一致<br>输入: `string`，输出: `number`<br>实现: `Number(customText)` | `packages/lib/utils/fields.ts:112` |
| **4** | `toRadioCustomText` | ✅ 一致<br>输入: `number`，输出: `string`<br>实现: `value.toString()` | `packages/lib/utils/fields.ts:116` |
| **5** | `fromCheckboxValue` 返回类型 | ✅ 一致<br>❌ 原错误: `number[]`<br>✅ 实际: `string[]` 字符串数组 | `packages/lib/universal/field-checkbox.ts:1` |
| **6** | `fromCheckboxValue` fallback 行为 | ✅ 一致<br>❌ 原错误: 包含 `map(Number)`<br>✅ 实际: `split(',').filter(Boolean)`，无数值转换 | `packages/lib/universal/field-checkbox.ts:15` |
| **7** | `toCheckboxValue` 输入类型 | ✅ 一致<br>❌ 原错误: `number[]`<br>✅ 实际: `string[]` 字符串数组 | `packages/lib/universal/field-checkbox.ts:19` |
| **8** | FREE_SIGNATURE fieldMeta 类型 | ✅ 一致<br>类型为 `undefined`，无配置项 | `packages/lib/types/field-meta.ts:264,409` |
| **9** | FREE_SIGNATURE 渲染行为 | ✅ 一致<br>抛出错误: "Free signature fields are not supported" | `packages/lib/universal/field-renderer/render-field.ts:88` |
| **10** | `extractFieldAutoInsertValues` 存在性 | ✅ 一致<br>仅 V2 信封支持，函数行号 359-471 | `packages/lib/server-only/document/send-document.ts:359` |
| **11** | `extractFieldInsertionValues` 函数 | ✅ 一致<br>行号 33-253，处理 10 种字段类型 | `packages/lib/utils/envelope-signing.ts:33` |
| **12** | SIGNATURE customText 存储值 | ✅ 一致<br>返回空字符串 `""`，实际值存 Signature 表 | `packages/lib/utils/envelope-signing.ts:248` |
| **13** | `TSignEnvelopeFieldValue` 类型定义 | ✅ 一致<br>无 FREE_SIGNATURE 分支定义 | `packages/trpc/server/envelope-router/sign-envelope-field.types.ts:7` |
| **14** | 自动回填 V2 版本限制 | ✅ 一致<br>条件判断: `envelope.internalVersion === 2` | `packages/lib/server-only/document/send-document.ts:184` |
| **15** | 自动回填 sendStatus 限制 | ✅ 一致<br>条件判断: `recipient.sendStatus !== SendStatus.SENT` | `packages/lib/server-only/document/send-document.ts:197` |
| **16** | 两套复选框转换函数区分 | ✅ 一致<br>主流程用 `*CustomText` (number[])<br>通用/遗留下用 `*CheckboxValue` (string[]) | 多处核对 |

---

## 代码位置参考

| 功能 | 文件路径 |
|------|---------|
| 字段类型定义 | `packages/lib/types/field-meta.ts` |
| 签署提交值转换 | `packages/lib/utils/envelope-signing.ts` |
| 文档发送自动回填 | `packages/lib/server-only/document/send-document.ts` |
| 复选/单选序列化工具 | `packages/lib/utils/fields.ts` |
| 复选框通用转换函数 | `packages/lib/universal/field-checkbox.ts` |
| 签名 API tRPC 路由 | `packages/trpc/server/envelope-router/sign-envelope-field.ts` |
| 签名 API tRPC 类型 | `packages/trpc/server/envelope-router/sign-envelope-field.types.ts` |
| 旧版签名函数 | `packages/lib/server-only/field/sign-field-with-token.ts` |
| 数据库模型 | `packages/prisma/schema.prisma` |
| 字段渲染器 | `packages/lib/universal/field-renderer/` |

---

**文档版本**: 终稿 v1.0  
**核对完成日期**: 2026-05-17  
**状态**: ✅ 所有内容已与源码核对一致
