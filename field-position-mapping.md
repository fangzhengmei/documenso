# 字段位置坐标映射流程

## 概述

本文档详细描述了从用户交互（点击/拖拽/框选）到字段坐标最终入库，再到渲染还原的完整坐标转换流程。整个流程涉及多层坐标系统之间的转换，确保字段在不同缩放级别下都能正确显示。

---

## 坐标系统层级

| 层级 | 坐标类型 | 单位 | 原点 | 说明 |
|------|---------|------|------|------|
| 1 | 屏幕坐标 (clientX/Y) | 像素 | 浏览器视口左上角 | 鼠标在视口中的位置 |
| 2 | 文档坐标 (pageX/Y) | 像素 | 文档左上角 | 鼠标相对于整个文档的位置（含滚动） |
| 3 | 页面相对像素坐标 | 像素 | PDF页面左上角 | 鼠标相对于PDF页面元素的位置 |
| 4 | Konva Stage内部坐标 | 像素 | Stage左上角（= 页面左上角） | Konva画布坐标系（未缩放） |
| 5 | 百分比坐标 | % (0-100) | PDF页面左上角 | 相对于页面宽高的百分比（最终存储格式） |

---

## 完整转换流程

### 第一阶段：交互坐标 → 页面坐标 → 百分比入库

---

#### 场景一：通过侧边栏按钮拖拽点击创建字段（Drag & Drop）

**文件位置**: `apps/remix/app/components/general/envelope-editor/envelope-editor-fields-drag-drop.tsx`

##### 1. 获取页面元素边界
```typescript
// 关键函数：getBoundingClientRect($page)
// 返回的是相对于文档的绝对坐标（已考虑滚动偏移）
const { top, left, height, width } = getBoundingClientRect($page);

// getBoundingClientRect 内部实现：
// const rect = element.getBoundingClientRect();
// const top = rect.top + window.scrollY;   // 视口坐标 + 垂直滚动 = 文档坐标
// const left = rect.left + window.scrollX;  // 视口坐标 + 水平滚动 = 文档坐标
```

##### 2. 文档坐标 → 页面相对像素坐标
```typescript
// event.pageX/Y = 鼠标相对于文档左上角的像素坐标
// left/top = 页面元素相对于文档左上角的像素坐标
// 差值 = 鼠标相对于页面元素的像素位置
const relativePixelX = event.pageX - left;
const relativePixelY = event.pageY - top;
```

##### 3. 像素坐标 → 百分比坐标
```typescript
// (像素值 / 页面像素宽度) * 100 = 百分比
let pageX = (relativePixelX / width) * 100;
let pageY = (relativePixelY / height) * 100;

// 字段尺寸也转换为百分比
const fieldPageWidth = (fieldBounds.current.width / width) * 100;
const fieldPageHeight = (fieldBounds.current.height / height) * 100;
```

##### 4. 居中调整（鼠标位置为字段中心点）
```typescript
// 将字段左上角对齐到（鼠标位置 - 字段一半尺寸）
pageX -= fieldPageWidth / 2;
pageY -= fieldPageHeight / 2;
```

##### 5. 最终字段对象（百分比坐标，准备入库）
```typescript
const field = {
  formId: nanoid(12),
  page: pageNumber,
  positionX: pageX,      // 百分比 (0-100)
  positionY: pageY,      // 百分比 (0-100)
  width: fieldPageWidth, // 百分比 (0-100)
  height: fieldPageHeight, // 百分比 (0-100)
  // ...其他字段
};

editorFields.addField(field);
```

---

#### 场景二：通过在PDF上框选区域创建字段（Selection Rectangle）

**文件位置**: `apps/remix/app/components/general/envelope-editor/envelope-editor-fields-page-renderer.tsx`

##### 1. Konva Stage 初始化（关键！理解缩放机制）
```typescript
// usePageRenderer.ts 中的 Stage 初始化
stage.current = new Konva.Stage({
  container,
  width: scaledViewport.width,   // pageWidth * scale
  height: scaledViewport.height, // pageHeight * scale
  scale: {
    x: scale,  // Stage 级别的缩放因子
    y: scale,
  },
});
```

**重要说明**：
- **Stage 的实际像素尺寸** = `pageWidth * scale`（显示在屏幕上的大小）
- **Stage 的内部逻辑坐标** = 仍然使用 `pageWidth/pageHeight`（未缩放）
- Konva 会自动将内部逻辑坐标应用 `scale` 后渲染到屏幕

##### 2. 获取框选区域的 Konva 内部坐标（未缩放）
```typescript
// Selection Rectangle 是在 Stage 内部绘制的
// 所以获取的 x/y/width/height 是相对于 unscaledViewport 的逻辑坐标
const pixelWidth = pendingFieldCreation.width();   // 逻辑像素（未缩放）
const pixelHeight = pendingFieldCreation.height();
const pixelX = pendingFieldCreation.x();
const pixelY = pendingFieldCreation.y();
```

##### 3. 使用工具函数转换：逻辑像素 → 百分比
**文件位置**: `packages/lib/universal/field-renderer/field-renderer.ts`

```typescript
// convertPixelToPercentage 内部实现
export const convertPixelToPercentage = (options) => {
  const { positionX, positionY, width, height, pageWidth, pageHeight } = options;

  // 注意：这里的 pageWidth/pageHeight 是 unscaledViewport（原始PDF尺寸）
  const fieldX = (positionX / pageWidth) * 100;
  const fieldY = (positionY / pageHeight) * 100;
  const fieldWidth = (width / pageWidth) * 100;
  const fieldHeight = (height / pageHeight) * 100;

  return { fieldX, fieldY, fieldWidth, fieldHeight };
};
```

##### 4. 调用示例（使用 unscaledViewport）
```typescript
// 关键：使用 unscaledViewport（因为 Konva 坐标是逻辑坐标）
const { fieldX, fieldY, fieldWidth, fieldHeight } = convertPixelToPercentage({
  width: pixelWidth,
  height: pixelHeight,
  positionX: pixelX,
  positionY: pixelY,
  pageWidth: unscaledViewport.width,   // = pageWidth (原始PDF宽度)
  pageHeight: unscaledViewport.height, // = pageHeight (原始PDF高度)
});
```

---

#### 场景三：字段拖拽移动或调整大小后保存（Drag & Resize）

**文件位置**: `apps/remix/app/components/general/envelope-editor/envelope-editor-fields-page-renderer.tsx`

##### 1. 获取字段当前的屏幕像素边界
```typescript
const handleResizeOrMove = (event: KonvaEventObject<Event>) => {
  const isDragEvent = event.type === 'dragend';
  const fieldGroup = event.target as Konva.Group;
  
  // 关键：getClientRect() 返回的是已应用 Stage scale 的屏幕像素坐标
  // 因为 Stage 本身有 scale 缩放，所以这里获取的值是 "显示像素"
  const {
    width: fieldPixelWidth,   // 已缩放的屏幕像素
    height: fieldPixelHeight, // 已缩放的屏幕像素
    x: fieldX,                // 已缩放的屏幕像素
    y: fieldY,                // 已缩放的屏幕像素
  } = fieldGroup.getClientRect({ skipStroke: true, skipShadow: true });
```

##### 2. 已缩放的像素坐标 → 百分比坐标
```typescript
  // 关键：使用 scaledViewport（因为 getClientRect 返回的是缩放后的坐标）
  const pageHeight = scaledViewport.height;  // pageHeight * scale
  const pageWidth = scaledViewport.width;    // pageWidth * scale

  // 计算位置百分比
  const positionPercentX = (fieldX / pageWidth) * 100;
  const positionPercentY = (fieldY / pageHeight) * 100;

  // 计算尺寸百分比
  const fieldPageWidth = (fieldPixelWidth / pageWidth) * 100;
  const fieldPageHeight = (fieldPixelHeight / pageHeight) * 100;
```

##### 3. 更新字段状态
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

### 第二阶段：数据库存储

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
  positionX: z.number().min(0),  // 百分比（核心存储格式）
  positionY: z.number().min(0),  // 百分比
  width: z.number().min(0),      // 百分比
  height: z.number().min(0),     // 百分比
  fieldMeta: ZFieldMetaSchema,
});
```

#### 2. 边界限制函数
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

#### 3. 服务端写入数据库
**文件位置**: `packages/lib/server-only/field/create-envelope-fields.ts`

```typescript
// 百分比坐标直接存储到数据库
const newlyCreatedFields = await tx.field.createManyAndReturn({
  data: validatedFields.map((field) => ({
    type: field.type,
    page: field.page,
    positionX: field.positionX,  // DECIMAL 类型，存储百分比
    positionY: field.positionY,  // DECIMAL 类型
    width: field.width,          // DECIMAL 类型
    height: field.height,        // DECIMAL 类型
    // ...其他字段
  })),
});
```

---

### 第三阶段：渲染还原（百分比 → 像素 → 屏幕显示）

#### 1. 百分比坐标 → Konva 逻辑像素坐标
**文件位置**: `packages/lib/universal/field-renderer/field-renderer.ts`

```typescript
export const calculateFieldPosition = (field, pageWidth, pageHeight) => {
  // pageWidth/pageHeight = unscaledViewport（原始PDF尺寸）
  // 百分比 → 逻辑像素
  const fieldWidth = pageWidth * (Number(field.width) / 100);
  const fieldHeight = pageHeight * (Number(field.height) / 100);
  const fieldX = pageWidth * (Number(field.positionX) / 100);
  const fieldY = pageHeight * (Number(field.positionY) / 100);

  return { fieldX, fieldY, fieldWidth, fieldHeight };
};
```

#### 2. 渲染时设置字段 Group 位置
**文件位置**: `packages/lib/universal/field-renderer/field-generic-items.ts`

```typescript
export const upsertFieldGroup = (field: FieldToRender, options: RenderFieldElementOptions): Konva.Group => {
  const { pageWidth, pageHeight, pageLayer, editable, scale } = options;

  // 1. 百分比 → 逻辑像素（未缩放）
  const { fieldX, fieldY, fieldWidth, fieldHeight } = calculateFieldPosition(field, pageWidth, pageHeight);

  const fieldGroup: Konva.Group =
    pageLayer.findOne(`#${field.renderId}`) || new Konva.Group({ ... });

  // 2. 拖拽边界考虑 scale（因为拖拽回调 pos.x/y 是显示坐标）
  // 重要：dragBoundFunc 的入参 pos.x/y 是"显示坐标"
  // 即：逻辑坐标 × Stage.scale，与 getClientRect() 返回值同坐标系
  const maxXPosition = (pageWidth - fieldWidth) * scale;
  const maxYPosition = (pageHeight - fieldHeight) * scale;

  // 3. 设置 Group 位置（注意：x/y 使用的是未缩放的逻辑像素）
  fieldGroup.setAttrs({
    scaleX: 1,       // Group 自身不缩放
    scaleY: 1,
    x: fieldX,       // 逻辑像素坐标（未缩放）
    y: fieldY,       // Konva Stage 的 scale 会自动应用到渲染
    draggable: editable,
    dragBoundFunc: (pos) => {
      // pos.x/y 是显示坐标（已应用 Stage.scale）
      // 边界值 maxXPosition/maxYPosition 也必须用显示坐标计算
      const newX = Math.max(0, Math.min(maxXPosition, pos.x));
      const newY = Math.max(0, Math.min(maxYPosition, pos.y));
      return { x: newX, y: newY };  // Konva 会自动转换回逻辑坐标
    },
  });

  return fieldGroup;
};
```

#### 3. Konva 自动缩放机制
```
数据库百分比坐标
    ↓ (calculateFieldPosition)
Konva 逻辑像素坐标 (x, y, width, height)  ← group.x()/group.y() 获取的值
    ↓ (Stage.scale 自动应用于渲染)
屏幕显示像素坐标 = 逻辑像素 × scale         ← pos.x / getClientRect() 获取的值
```

**关键点**：
- 字段元素本身从不设置 `scaleX/Y`，缩放完全由 Stage 统一管理
- `group.x()` / `group.y()` 获取的是**逻辑坐标**（未缩放）
- `dragBoundFunc` 的 `pos.x / pos.y` 是**显示坐标**（已缩放）
- 两者关系：`pos.x = group.x() * scale`

#### 4. 数值验证示例（pageWidth=800, fieldWidth=100, scale=0.75）

| 场景 | 计算过程 | 结果 |
|-----|---------|------|
| 最大逻辑X | `800 - 100` | 700px |
| 边界值（显示坐标） | `700 * 0.75` | 525px |
| 拖拽 pos.x=525 时 | `group.x() = 525 / 0.75` | 700px ✓ |
| 字段右下角 | `700 + 100` | 800px（刚好不超出边界） |

> 详细验证请参考：`drag-bound-coordinate-check.md`

---

## 坐标转换总结表

| 操作场景 | 输入坐标 | 使用的 Viewport | 转换方向 | 核心公式 |
|---------|---------|----------------|---------|---------|
| 侧边按钮点击创建 | 鼠标文档坐标 (pageX/Y) | 页面DOM元素的 bounding rect | 像素 → 百分比 | `((event.pageX - left) / width) * 100` |
| 框选区域创建 | Konva 逻辑坐标 | unscaledViewport | 像素 → 百分比 | `(pixelX / pageWidth) * 100` |
| 拖拽/调整大小后 | getClientRect() 已缩放坐标 | scaledViewport | 像素 → 百分比 | `(fieldX / (pageWidth * scale)) * 100` |
| 字段渲染 | 数据库百分比 | unscaledViewport | 百分比 → 像素 | `pageWidth * (positionX / 100)` |

---

## Viewport 详解

### unscaledViewport（未缩放视口）
**定义**：PDF 文档的原始像素尺寸
```typescript
const unscaledViewport = {
  scale: 1,
  width: pageWidth,   // PDF 原始宽度（像素）
  height: pageHeight, // PDF 原始高度（像素）
};
```

**使用场景**：
- 框选区域创建字段时的坐标转换
- 字段渲染时 `calculateFieldPosition` 计算逻辑像素
- `renderField` 调用时传入 `pageWidth/pageHeight`

### scaledViewport（缩放后视口）
**定义**：考虑显示缩放因子后的屏幕像素尺寸
```typescript
const scaledViewport = {
  scale: scale,
  width: pageWidth * scale,   // 屏幕上显示的宽度
  height: pageHeight * scale, // 屏幕上显示的高度
};
```

**使用场景**：
- Konva Stage 的 `width/height` 设置
- 字段拖拽/调整大小后，`getClientRect()` 返回值的转换
- 拖拽边界 `maxXPosition/maxYPosition` 的计算

---

## 常见问题与注意事项

### Q1: 为什么拖拽移动时不更新 width/height？
A: `getClientRect()` 在字段仅移动不缩放时，返回的宽高可能因浮点精度问题产生微小变化。为避免这种"无意义"的更新导致字段轻微"抖动"，仅在 `transformend`（调整大小）时才更新宽高。

### Q2: Konva Stage.scale 是如何工作的？
A: Stage.scale 是一个矩阵变换，会应用到所有子元素。设置子元素的 `x=100` 时：
- Konva 内部逻辑坐标：x=100
- 实际渲染到屏幕：x = 100 * scale

### Q3: dragBoundFunc 为什么要乘 scale？
A: 拖拽事件回调中的 `pos.x/pos.y` 是**屏幕像素坐标**（已应用 scale），所以边界限制也需要用 `pageWidth * scale` 来计算。

### Q4: 为什么有两套坐标转换函数？
A: 不是两套，而是同一转换的正反方向：
- `convertPixelToPercentage`: 像素 → 百分比（写入）
- `calculateFieldPosition`: 百分比 → 像素（读取）

---

## 关键文件索引

| 功能模块 | 文件路径 | 核心函数/变量 |
|---------|---------|-------------|
| DOM 边界获取 | `packages/lib/client-only/get-bounding-client-rect.ts` | `getBoundingClientRect` |
| 视口定义 | `packages/lib/client-only/hooks/use-page-renderer.ts` | `unscaledViewport`, `scaledViewport` |
| 像素百分比互转 | `packages/lib/universal/field-renderer/field-renderer.ts` | `convertPixelToPercentage`, `calculateFieldPosition` |
| 侧边栏拖拽创建 | `envelope-editor-fields-drag-drop.tsx` | `onMouseClick` |
| 框选创建/拖拽移动 | `envelope-editor-fields-page-renderer.tsx` | `createFieldFromPendingTemplate`, `handleResizeOrMove` |
| 字段Group渲染 | `field-generic-items.ts` | `upsertFieldGroup` |
| 字段状态管理 | `use-editor-fields.ts` | `addField`, `updateFieldByFormId` |
| 服务端创建 | `create-envelope-fields.ts` | `createEnvelopeFields` |
