# 字段位置坐标映射流程

## 概述

本文档描述了从用户在 PDF 页面上拖动/放置字段，到最终坐标入库存储的完整坐标转换流程。整个流程涉及多层坐标系统之间的转换，确保字段在不同缩放级别和不同 PDF 页面尺寸下都能正确显示。

---

## 坐标系统层级

| 层级 | 坐标类型 | 范围 | 说明 |
|------|---------|------|------|
| 1 | 屏幕坐标 (clientX/Y) | 像素值 | 鼠标在浏览器视口中的位置 |
| 2 | 页面像素坐标 | 像素值 | 相对于 PDF 页面左上角的像素位置 |
| 3 | 百分比坐标 | 0-100 | 相对于页面宽高的百分比值（最终存储格式） |

---

## 详细转换流程

### 场景一：通过拖拽按钮创建字段（Drag & Drop）

**文件位置**: `apps/remix/app/components/general/envelope-editor/envelope-editor-fields-drag-drop.tsx`

#### 1. 鼠标点击获取页面元素
```typescript
// onMouseClick 事件处理
const $page = getPage(event, PDF_VIEWER_PAGE_SELECTOR);
const { top, left, height, width } = getBoundingClientRect($page);
const pageNumber = parseInt($page.getAttribute('data-page-number') ?? '1', 10);
```

#### 2. 屏幕坐标 -> 页面相对像素坐标
```typescript
// 计算字段相对于页面的像素位置
// event.pageX/Y = 鼠标相对于整个文档的像素位置
// left/top = 页面相对于视口的像素位置
let pageX = ((event.pageX - left) / width) * 100;
let pageY = ((event.pageY - top) / height) * 100;
```

#### 3. 计算字段尺寸的百分比
```typescript
const fieldPageWidth = (fieldBounds.current.width / width) * 100;
const fieldPageHeight = (fieldBounds.current.height / height) * 100;
```

#### 4. 居中调整（鼠标位置为字段中心）
```typescript
// 将字段中心对齐到鼠标点击位置
pageX -= fieldPageWidth / 2;
pageY -= fieldPageHeight / 2;
```

#### 5. 创建字段对象（百分比坐标）
```typescript
const field = {
  formId: nanoid(12),
  envelopeItemId: selectedEnvelopeItemId,
  type: selectedField,
  page: pageNumber,
  positionX: pageX,      // 百分比 (0-100)
  positionY: pageY,      // 百分比 (0-100)
  width: fieldPageWidth, // 百分比 (0-100)
  height: fieldPageHeight, // 百分比 (0-100)
  recipientId: selectedRecipientId,
  fieldMeta: {...},
};
```

---

### 场景二：通过框选区域创建字段（Selection Rectangle）

**文件位置**: `apps/remix/app/components/general/envelope-editor/envelope-editor-fields-page-renderer.tsx`

#### 1. 获取框选区域的像素坐标（Konva Stage）
```typescript
// Konva Stage 上的坐标已经是相对于页面的像素坐标
const pixelWidth = pendingFieldCreation.width();
const pixelHeight = pendingFieldCreation.height();
const pixelX = pendingFieldCreation.x();
const pixelY = pendingFieldCreation.y();
```

#### 2. 使用工具函数转换像素 -> 百分比
**文件位置**: `packages/lib/universal/field-renderer/field-renderer.ts`

```typescript
export const convertPixelToPercentage = (options) => {
  const { positionX, positionY, width, height, pageWidth, pageHeight } = options;

  const fieldX = (positionX / pageWidth) * 100;
  const fieldY = (positionY / pageHeight) * 100;
  const fieldWidth = (width / pageWidth) * 100;
  const fieldHeight = (height / pageHeight) * 100;

  return { fieldX, fieldY, fieldWidth, fieldHeight };
};
```

#### 3. 调用示例
```typescript
const { fieldX, fieldY, fieldWidth, fieldHeight } = convertPixelToPercentage({
  width: pixelWidth,
  height: pixelHeight,
  positionX: pixelX,
  positionY: pixelY,
  pageWidth: unscaledViewport.width,   // 页面原始宽度（像素）
  pageHeight: unscaledViewport.height, // 页面原始高度（像素）
});
```

---

### 场景三：字段拖拽移动或调整大小（Drag & Resize）

**文件位置**: `apps/remix/app/components/general/envelope-editor/envelope-editor-fields-page-renderer.tsx`

#### 1. 获取字段当前的像素边界
```typescript
const handleResizeOrMove = (event: KonvaEventObject<Event>) => {
  const isDragEvent = event.type === 'dragend';
  const fieldGroup = event.target as Konva.Group;
  
  // 获取字段的像素边界（已考虑缩放）
  const {
    width: fieldPixelWidth,
    height: fieldPixelHeight,
    x: fieldX,
    y: fieldY,
  } = fieldGroup.getClientRect({ skipStroke: true, skipShadow: true });
```

#### 2. 像素坐标 -> 百分比坐标
```typescript
  const pageHeight = scaledViewport.height;
  const pageWidth = scaledViewport.width;

  // 计算位置百分比
  const positionPercentX = (fieldX / pageWidth) * 100;
  const positionPercentY = (fieldY / pageHeight) * 100;

  // 计算尺寸百分比
  const fieldPageWidth = (fieldPixelWidth / pageWidth) * 100;
  const fieldPageHeight = (fieldPixelHeight / pageHeight) * 100;
```

#### 3. 更新字段状态
```typescript
  const fieldUpdates: Partial<TLocalField> = {
    positionX: positionPercentX,
    positionY: positionPercentY,
  };

  // 仅在调整大小时更新宽高（拖拽移动时不更新避免精度问题）
  if (!isDragEvent) {
    fieldUpdates.width = fieldPageWidth;
    fieldUpdates.height = fieldPageHeight;
  }

  editorFields.updateFieldByFormId(fieldFormId, fieldUpdates);
};
```

---

### 场景四：字段渲染（百分比 -> 像素）

**文件位置**: `packages/lib/universal/field-renderer/field-renderer.ts`

#### 1. 百分比坐标 -> 像素坐标
```typescript
export const calculateFieldPosition = (field, pageWidth, pageHeight) => {
  // 百分比 -> 像素
  const fieldWidth = pageWidth * (Number(field.width) / 100);
  const fieldHeight = pageHeight * (Number(field.height) / 100);
  const fieldX = pageWidth * (Number(field.positionX) / 100);
  const fieldY = pageHeight * (Number(field.positionY) / 100);

  return { fieldX, fieldY, fieldWidth, fieldHeight };
};
```

#### 2. 渲染时调用（Konva Canvas）
**文件位置**: `packages/lib/universal/field-renderer/render-field.ts`

```typescript
// 在 renderField 函数中
const { fieldX, fieldY, fieldWidth, fieldHeight } = calculateFieldPosition(
  field,
  pageWidth,
  pageHeight
);

// 应用缩放因子
const scaledX = fieldX * scale;
const scaledY = fieldY * scale;
const scaledWidth = fieldWidth * scale;
const scaledHeight = fieldHeight * scale;
```

---

### 场景五：数据库存储与读取

#### 1. 字段状态管理（本地）
**文件位置**: `packages/lib/client-only/hooks/use-editor-fields.ts`

```typescript
// TLocalField 类型定义
export const ZLocalFieldSchema = z.object({
  id: z.number().optional(),
  formId: z.string().min(1),
  envelopeItemId: z.string(),
  type: z.nativeEnum(FieldType),
  recipientId: z.number(),
  page: z.number().min(1),
  positionX: z.number().min(0),  // 百分比
  positionY: z.number().min(0),  // 百分比
  width: z.number().min(0),      // 百分比
  height: z.number().min(0),     // 百分比
  fieldMeta: ZFieldMetaSchema,
});
```

#### 2. 边界限制
```typescript
const restrictFieldPosValues = (field) => {
  return {
    positionX: Math.max(0, Math.min(100, field.positionX)),
    positionY: Math.max(0, Math.min(100, field.positionY)),
    width: Math.max(0, Math.min(100, field.width)),
    height: Math.max(0, Math.min(100, field.height)),
  };
};
```

#### 3. 服务端创建字段
**文件位置**: `packages/lib/server-only/field/create-envelope-fields.ts`

```typescript
// 百分比坐标直接存储到数据库
const newlyCreatedFields = await tx.field.createManyAndReturn({
  data: validatedFields.map((field) => ({
    type: field.type,
    page: field.page,
    positionX: field.positionX,  // 百分比
    positionY: field.positionY,  // 百分比
    width: field.width,          // 百分比
    height: field.height,        // 百分比
    customText: '',
    inserted: false,
    fieldMeta: field.fieldMeta,
    envelopeId: envelope.id,
    envelopeItemId: field.envelopeItemId,
    recipientId: field.recipientId,
  })),
});
```

---

### 场景六：AI 检测字段的坐标转换

**文件位置**: `packages/lib/server-only/field/create-envelope-fields.ts`

#### 1. PDF 原始坐标（左下角原点）
```typescript
// PDF 内部坐标系统：原点在左下角，单位是 point
// match.bbox = { x, y, width, height } (points)
// page.height = 页面高度 (points)
```

#### 2. 转换为左上角原点的百分比坐标
```typescript
// 1. Y 轴翻转（左下角 -> 左上角）
const topLeftY = page.height - match.bbox.y - match.bbox.height;

// 2. 像素/点 -> 百分比
const widthPercent = (match.bbox.width / page.width) * 100;
const heightPercent = (match.bbox.height / page.height) * 100;

// 3. 最终字段坐标
return {
  positionX: (match.bbox.x / page.width) * 100,
  positionY: (topLeftY / page.height) * 100,
  width: widthPercent,
  height: heightPercent,
  page: match.pageIndex + 1,
  // ...其他字段
};
```

---

## 完整流程示意图

```
┌─────────────────────────────────────────────────────────────────┐
│                    用户交互阶段                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐     鼠标点击/拖拽     ┌─────────────────┐   │
│  │  鼠标指针    │ ───────────────────> │  clientX/Y     │   │
│  │  (屏幕)      │                        │  (屏幕像素)    │   │
│  └──────────────┘                        └────────┬────────┘   │
│                                                    │            │
│                                                    ▼            │
│                                          ┌─────────────────┐   │
│                                          │ getBoundingCli │   │
│                                          │ entRect($page)  │   │
│                                          │ 获取页面边界    │   │
│                                          └────────┬────────┘   │
│                                                    │            │
│                                                    ▼            │
│                                          ┌─────────────────┐   │
│                                          │ 像素相对位置    │   │
│                                          │ (pageX - left) │   │
│                                          └────────┬────────┘   │
│                                                    │            │
│                                                    ▼            │
│                                          ┌─────────────────┐   │
│                                          │  百分比计算     │   │
│                                          │  (px / width)  │   │
│                                          │      * 100     │   │
│                                          └────────┬────────┘   │
│                                                    │            │
└────────────────────────────────────────────────────┼────────────┘
                                                     │
                                                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    数据存储阶段                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐                                          │
│  │  本地状态    │  useEditorFields                        │
│  │  (React)     │  TLocalField { positionX, Y, W, H }    │
│  └──────┬───────┘                                          │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────┐     TRPC API调用     ┌─────────────────┐   │
│  │  自动保存    │ ───────────────────> │  createEnvelope │   │
│  │  (AutoSave)  │                        │    Fields      │   │
│  └──────────────┘                        └────────┬────────┘   │
│                                                    │            │
│                                                    ▼            │
│                                          ┌─────────────────┐   │
│                                          │   Prisma DB     │   │
│                                          │   Field 表      │   │
│                                          │  positionX (DECIMAL)│
│                                          └─────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                                                     │
                                                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    字段渲染阶段                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐                                          │
│  │  数据库读取   │  SELECT positionX, Y, W, H            │
│  └──────┬───────┘                                          │
│         │                                                    │
│         ▼                                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  calculateFieldPosition(field, pageWidth, pageHeight)│   │
│  │  fieldX = pageWidth * (positionX / 100)             │   │
│  │  fieldY = pageHeight * (positionY / 100)            │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                       │
│                         ▼                                       │
│            ┌──────────────────────────────┐                │
│            │  Konva Group (x, y, width, height)          │   │
│            │    renderField() 渲染到 Canvas               │   │
│            └──────────────────────────────┘                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 关键文件索引

| 功能模块 | 文件路径 | 核心函数 |
|---------|---------|---------|
| 拖拽创建字段 | `envelope-editor-fields-drag-drop.tsx` | `onMouseClick` |
| 框选创建字段 | `envelope-editor-fields-page-renderer.tsx` | `createFieldFromPendingTemplate` |
| 拖拽/调整大小 | `envelope-editor-fields-page-renderer.tsx` | `handleResizeOrMove` |
| 像素转百分比 | `field-renderer.ts` | `convertPixelToPercentage` |
| 百分比转像素 | `field-renderer.ts` | `calculateFieldPosition` |
| 字段状态管理 | `use-editor-fields.ts` | `addField`, `updateFieldByFormId` |
| 服务端创建 | `create-envelope-fields.ts` | `createEnvelopeFields` |
| 字段渲染 | `render-field.ts` | `renderField` |

---

## 注意事项

### 1. 坐标原点
- **屏幕坐标**: 左上角为原点 (0,0)
- **PDF 页面渲染**: 左上角为原点 (0,0)
- **PDF 原始内部坐标**: 左下角为原点 (0,0) ⚠️

### 2. 精度问题
- 百分比坐标使用小数存储（例如 23.4567）
- 像素与百分比之间的来回转换可能有微小误差
- 拖拽移动时不更新宽高以避免累积误差

### 3. 缩放因子 (Scale)
- Konva Stage 应用了缩放因子
- 在 `handleResizeOrMove` 中使用 `scaledViewport`（已缩放）
- 在 `convertPixelToPercentage` 中使用 `unscaledViewport`（未缩放）

### 4. 边界限制
- 所有坐标值限制在 [0, 100] 范围内
- 通过 `restrictFieldPosValues` 函数强制约束

---

## 相关常量定义

**文件位置**: `packages/lib/universal/field-renderer/field-renderer.ts`

```typescript
export const MIN_FIELD_WIDTH_PX = 36;   // 字段最小宽度（像素）
export const MIN_FIELD_HEIGHT_PX = 12;  // 字段最小高度（像素）
```
