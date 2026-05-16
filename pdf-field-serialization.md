# PDF 字段值序列化分析

本文档详细分析 Documenso 中 PDF 字段的三种状态（编辑态、存储态、发送态）的数据结构及其相互转换逻辑。

## 状态概览

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│                 │      │                 │      │                 │
│    编辑态       │ ───▶ │    存储态       │ ───▶ │    发送态       │
│   (Editor)      │      │  (Database)     │      │   (Signing)     │
│                 │      │                 │      │                 │
└─────────────────┘      └─────────────────┘      └─────────────────┘
         ▲                        ▲                        ▲
         │                        │                        │
         └────────────────────────┴────────────────────────┘
                              渲染
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

| 字段类型 | 类型标识 | 特有字段 |
|---------|---------|---------|
| **签名** | `signature` | `overflow` |
| **首字母** | `initials` | `textAlign: 'left' \| 'center' \| 'right'` |
| **姓名** | `name` | `textAlign` |
| **邮箱** | `email` | `textAlign`, `overflow` |
| **日期** | `date` | `textAlign`, `overflow` |
| **文本** | `text` | `text`, `characterLimit`, `textAlign`, `lineHeight`, `letterSpacing`, `verticalAlign` |
| **数字** | `number` | `numberFormat`, `value`, `minValue`, `maxValue`, `textAlign`, `lineHeight`, `letterSpacing`, `verticalAlign` |
| **单选** | `radio` | `values: {id: number; checked: boolean; value: string}[]`, `direction: 'vertical' \| 'horizontal'` |
| **复选** | `checkbox` | `values[]`, `validationRule`, `validationLength`, `direction` |
| **下拉** | `dropdown` | `values: {value: string}[]`, `defaultValue` |

### 编辑态数据示例

```typescript
// 单选字段编辑态配置
const radioFieldMeta: TRadioFieldMeta = {
  type: 'radio',
  required: true,
  fontSize: 12,
  direction: 'vertical',
  values: [
    { id: 0, checked: false, value: '选项 A' },
    { id: 1, checked: false, value: '选项 B' },
    { id: 2, checked: false, value: '选项 C' }
  ]
};

// 复选字段编辑态配置
const checkboxFieldMeta: TCheckboxFieldMeta = {
  type: 'checkbox',
  required: true,
  fontSize: 12,
  direction: 'horizontal',
  validationRule: '>=',
  validationLength: 1,
  values: [
    { id: 0, checked: false, value: '同意条款' },
    { id: 1, checked: false, value: '接收通知' }
  ]
};
```

---

## 2. 存储态 (Storage State)

### 定义
存储态是字段在数据库中持久化存储的状态。使用 Prisma 模型定义，通过 `Field` 和 `Signature` 表存储。

### 数据库表结构 (`packages/prisma/schema.prisma`)

#### Field 表
```prisma
model Field {
  id             Int      @id @default(autoincrement())
  secondaryId    String   @unique @default(cuid())
  envelopeId     String
  envelopeItemId String
  recipientId    Int
  type           FieldType  // 枚举: SIGNATURE, NAME, EMAIL, DATE, TEXT, NUMBER, RADIO, CHECKBOX, DROPDOWN, INITIALS
  page           Int
  
  // 位置和尺寸 (Decimal 类型)
  positionX      Decimal  @default(0)
  positionY      Decimal  @default(0)
  width          Decimal  @default(-1)
  height         Decimal  @default(-1)
  
  // 字段值存储
  customText     String   // 序列化后的字段值
  inserted       Boolean  // 是否已填写
  
  // 元数据 (JSON)
  fieldMeta      Json?
  
  // 关联
  envelope       Envelope     @relation(fields: [envelopeId], references: [id])
  recipient      Recipient    @relation(fields: [recipientId], references: [id])
  signature      Signature?
  
  @@index([envelopeId])
  @@index([envelopeItemId])
  @@index([recipientId])
}
```

#### Signature 表（仅签名字段使用）
```prisma
model Signature {
  id                     Int      @id @default(autoincrement())
  created                DateTime @default(now())
  recipientId            Int
  fieldId                Int      @unique
  signatureImageAsBase64 String?  // 手写签名图片
  typedSignature         String?  // 键入签名
  
  recipient Recipient @relation(fields: [recipientId], references: [id])
  field     Field     @relation(fields: [fieldId], references: [id])
}
```

### 存储态值编码规则

| 字段类型 | `customText` 存储格式 | 说明 |
|---------|---------------------|------|
| **EMAIL** | 邮箱字符串 | 直接存储 |
| **NAME** | 姓名字符串 | 直接存储 |
| **INITIALS** | 首字母字符串 | 直接存储 |
| **DATE** | 格式化日期字符串 | e.g. "2024-01-15" |
| **NUMBER** | 数字字符串 | e.g. "123.45" |
| **TEXT** | 文本内容 | 直接存储 |
| **RADIO** | 选中索引的字符串形式 | e.g. `"1"` (通过 `toRadioCustomText`) |
| **CHECKBOX** | 选中索引数组的 JSON 字符串 | e.g. `"[0,2]"` (通过 `toCheckboxCustomText`) |
| **DROPDOWN** | 选中值字符串 | 直接存储 |
| **SIGNATURE** | `""` (空字符串) | 实际值存储在 Signature 关联表中 |

### 存储态数据示例

```typescript
// 选中第 0 和第 2 个选项的复选框
const checkboxField: Field = {
  id: 123,
  secondaryId: "cuid-xxx",
  envelopeId: "env-xxx",
  envelopeItemId: "item-xxx",
  recipientId: 456,
  type: "CHECKBOX",
  page: 1,
  positionX: new Decimal(100),
  positionY: new Decimal(200),
  width: new Decimal(300),
  height: new Decimal(50),
  customText: "[0,2]",  // 选中索引 0 和 2
  inserted: true,
  fieldMeta: {
    type: "checkbox",
    values: [
      { id: 0, checked: false, value: "选项 A" },
      { id: 1, checked: false, value: "选项 B" },
      { id: 2, checked: false, value: "选项 C" }
    ],
    required: true
  }
};

// 签名字段 - 值存储在关联表中
const signatureField: Field = {
  id: 124,
  type: "SIGNATURE",
  customText: "",  // 空字符串
  inserted: true,
  // ... 其他字段
};

// 关联的 Signature 记录
const signature: Signature = {
  id: 789,
  fieldId: 124,
  recipientId: 456,
  signatureImageAsBase64: "data:image/png;base64,...",  // 手写签名
  typedSignature: null  // 或 "张三" (键入签名)
};
```

---

## 3. 发送态/签名态 (Signing State)

### 定义
发送态是前端签名页面提交字段值时使用的状态，通过 tRPC API 提交。

### 请求数据结构 (`packages/trpc/server/envelope-router/sign-envelope-field.types.ts`)

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

### API 请求示例

```typescript
// 签署复选框字段
const signCheckboxRequest: TSignEnvelopeFieldRequest = {
  token: "recipient-token-xxx",
  fieldId: 123,
  fieldValue: {
    type: "CHECKBOX",
    value: [0, 2]  // 选中第 0 和第 2 个选项
  },
  authOptions: {}
};

// 签署签名字段 (手写签名)
const signSignatureRequest: TSignEnvelopeFieldRequest = {
  token: "recipient-token-xxx",
  fieldId: 124,
  fieldValue: {
    type: "SIGNATURE",
    value: "data:image/png;base64,iVBORw0KGgo..."  // base64 图片
  }
};

// 签署签名字段 (键入签名)
const signTypedSignatureRequest: TSignEnvelopeFieldRequest = {
  token: "recipient-token-xxx",
  fieldId: 124,
  fieldValue: {
    type: "SIGNATURE",
    value: "张三"  // 文本签名
  }
};
```

---

## 状态转换逻辑

### 转换核心函数

#### 1. `extractFieldInsertionValues` (发送态 → 存储态)
**文件**: `packages/lib/utils/envelope-signing.ts`

将签名页面提交的字段值转换为数据库存储格式。

```typescript
function extractFieldInsertionValues(options: {
  fieldValue: TSignEnvelopeFieldValue;
  field: Field;
  documentMeta: DocumentMeta;
}): { customText: string; inserted: boolean };
```

**转换规则**:

| 字段类型 | 转换逻辑 |
|---------|---------|
| **EMAIL** | 验证邮箱格式 → 直接存储 |
| **NAME/INITIALS** | 验证非空 → 直接存储 |
| **DATE** | `true` → 使用当前日期格式化 |
| **NUMBER** | 验证数字格式 → 直接存储字符串 |
| **TEXT** | 验证文本规则 → 直接存储 |
| **RADIO** | `value: number` → `toString()` 存储 |
| **CHECKBOX** | `value: number[]` → `JSON.stringify()` 存储 |
| **DROPDOWN** | 验证选项有效性 → 直接存储值 |
| **SIGNATURE** | `customText = ""`，值存储在 Signature 表 |

#### 2. `toCheckboxCustomText` / `parseCheckboxCustomText`
**文件**: `packages/lib/utils/fields.ts`

```typescript
// 发送态 → 存储态
function toCheckboxCustomText(checkedValues: number[]): string {
  return JSON.stringify(checkedValues);  // [0, 2] → "[0,2]"
}

// 存储态 → 渲染态
function parseCheckboxCustomText(customText: string): number[] {
  return JSON.parse(customText);  // "[0,2]" → [0, 2]
}
```

#### 3. `toRadioCustomText` / `parseRadioCustomText`
**文件**: `packages/lib/utils/fields.ts`

```typescript
// 发送态 → 存储态
function toRadioCustomText(value: number): string {
  return value.toString();  // 1 → "1"
}

// 存储态 → 渲染态
function parseRadioCustomText(customText: string): number {
  return Number(customText);  // "1" → 1
}
```

#### 4. `fromCheckboxValue` / `toCheckboxValue`
**文件**: `packages/lib/universal/field-checkbox.ts`

用于在存储值（索引数组）和显示值（文本数组）之间转换：

```typescript
// 存储态 customText → 显示态值数组
function fromCheckboxValue(customText: string): string[] {
  try {
    const indices: number[] = JSON.parse(customText);
    // 需要结合 fieldMeta.values 映射到实际文本
    return indices.map(i => fieldMeta.values[i].value);
  } catch {
    return customText.split(',').filter(Boolean);
  }
}

// 显示态值数组 → 存储态 customText
function toCheckboxValue(values: string[]): string {
  return JSON.stringify(values);
}
```

---

### 完整转换流程示例

#### 场景：签署单选字段

```
用户操作: 在签名页面选中第 1 个单选选项

     发送态 (提交)
        ↓
{ type: "RADIO", value: 1 }
        ↓
     extractFieldInsertionValues()
        ↓
     toRadioCustomText(1)
        ↓
     customText = "1"
        ↓
     存储态 (数据库)
        ↓
Field { type: "RADIO", customText: "1", inserted: true }
        ↓
     渲染态 (PDF 渲染)
        ↓
parseRadioCustomText("1") → 1
→ 映射到 fieldMeta.values[1].value → "选项 B"
```

#### 场景：签署复选字段

```
用户操作: 选中第 0 和第 2 个复选框

     发送态 (提交)
        ↓
{ type: "CHECKBOX", value: [0, 2] }
        ↓
     extractFieldInsertionValues()
        ↓
     toCheckboxCustomText([0, 2])
        ↓
     customText = "[0,2]"
        ↓
     存储态 (数据库)
        ↓
Field { type: "CHECKBOX", customText: "[0,2]", inserted: true }
        ↓
     渲染态 (PDF 渲染)
        ↓
parseCheckboxCustomText("[0,2]") → [0, 2]
→ 映射到 fieldMeta.values → ["选项 A", "选项 C"]
```

#### 场景：签署签名字段

```
用户操作: 绘制手写签名或键入姓名

     发送态 (提交)
        ↓
{ type: "SIGNATURE", value: "data:image/png;base64,..." }
        ↓
     extractFieldInsertionValues()
        ↓
     customText = "" (空字符串)
     inserted = true
        ↓
     存储态 (数据库)
        ↓
Field { type: "SIGNATURE", customText: "", inserted: true }
Signature {
  fieldId: xxx,
  signatureImageAsBase64: "data:image/png;base64,...",
  typedSignature: null
}
        ↓
     渲染态 (PDF 渲染)
        ↓
从 Signature 关联表读取
→ 渲染图片到 PDF
```

---

## 状态转换关系图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         编辑态 (Editor)                               │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  fieldMeta: {                                                 │  │
│  │    type: 'checkbox',                                          │  │
│  │    values: [{id:0, value:'A'}, {id:1, value:'B'}],            │  │
│  │    required: true,                                            │  │
│  │    direction: 'vertical'                                      │  │
│  │  }                                                           │  │
│  └───────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────┬────────────────────────────┘
                                         │ 保存
                                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        存储态 (Database)                              │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  Field {                                                      │  │
│  │    type: 'CHECKBOX',                                          │  │
│  │    customText: '[0,1]',  ←── 序列化后的值                    │  │
│  │    inserted: true,                                            │  │
│  │    fieldMeta: { /* 编辑态元数据 */ }                          │  │
│  │  }                                                           │  │
│  │                                                               │  │
│  │  Signature { (仅签名类型)                                     │  │
│  │    signatureImageAsBase64: '...',                             │  │
│  │    typedSignature: null                                       │  │
│  │  }                                                           │  │
│  └───────────────────────────────────────────────────────────────┘  │
└───────────────────┬─────────────────────────────┬────────────────────┘
                    │ 读取                         │ 签署提交
                    ▼                             ▼
┌─────────────────────────────┐   ┌─────────────────────────────────────┐
│      渲染态 (Renderer)       │   │        发送态 (Signing)             │
│  ┌─────────────────────────┐  │   │  ┌───────────────────────────────┐│
│  │  customText → 反序列化  │  │   │  │ { type: 'CHECKBOX',           ││
│  │  渲染到 PDF Canvas      │  │   │  │   value: [0, 1] }             ││
│  └─────────────────────────┘  │   │  └───────────────────────────────┘│
└───────────────────────────────┘   └─────────────────────────────────────┘
```

---

## 关键设计决策

### 1. 使用索引数组而非实际值
- **原因**: 选项文本可能在签署前被修改，使用索引可以确保选项顺序不变时值正确映射
- **实现**: `CHECKBOX` 使用 `number[]`，`RADIO` 使用 `number`
- **注意**: 必须保证 `fieldMeta.values` 数组的顺序和 ID 在签署过程中不被修改

### 2. 签名字段值分离存储
- **原因**: 签名图片可能很大（base64 编码），分离存储避免 Field 表过大
- **实现**: Field.customText 为空，实际值存储在 Signature 关联表中

### 3. 统一序列化入口
- **原因**: 确保所有字段类型的转换逻辑集中管理，避免重复代码
- **实现**: `extractFieldInsertionValues()` 函数处理所有类型

### 4. JSON 序列化用于复杂类型
- **原因**: 复选框可能有多个值，数组形式最直观
- **实现**: `JSON.stringify(number[])` → `customText`

---

## 代码位置参考

| 功能 | 文件路径 |
|------|---------|
| 字段类型定义 | `packages/lib/types/field-meta.ts` |
| 字段值转换核心函数 | `packages/lib/utils/envelope-signing.ts` |
| 复选/单选序列化工具 | `packages/lib/utils/fields.ts` |
| 复选框值转换 | `packages/lib/universal/field-checkbox.ts` |
| 签名 API 路由 | `packages/trpc/server/envelope-router/sign-envelope-field.ts` |
| 数据库模型 | `packages/prisma/schema.prisma` |
| 字段渲染器 | `packages/lib/universal/field-renderer/` |
