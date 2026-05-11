# 字段位置存储与坐标系统分析

## 1. 概述

在 Documenso 中，用户可以将签名框、文本框等字段拖拽到 PDF 画布的任意位置，并在保存后下次打开时保持原位置。这一功能通过以下核心机制实现：

- **拖拽交互**：使用 `react-rnd` 库实现拖拽和调整大小
- **坐标归一化**：将像素坐标转换为百分比坐标
- **页码绑定**：将字段与特定页码绑定
- **边界校验**：坐标约束在 0-100 范围内，页码从 1 开始
- **自动保存**：防抖机制确保频繁操作不产生过多请求
- **数据库存储**：使用 Decimal 类型精确存储坐标数据

---

## 2. "拖拽后保存、重开仍在原位"完整链路

### 2.1 保存链路：从 onAutoSave 到数据库

#### 第一步：前端拖拽更新本地状态

当用户拖动字段或调整大小时，`react-rnd` 触发事件，更新表单中的字段坐标。

代码位置：`packages/ui/primitives/document-flow/field-item.tsx:265-272`

```typescript
onResizeStop={(_e, _d, ref) => {
  onFieldDeactivate?.();
  onResize?.(ref);  // 触发回调更新本地状态
}}
onDragStop={(_e, d) => {
  onFieldDeactivate?.();
  onMove?.(d.node);  // 触发回调更新本地状态
}}
```

在 `add-fields.tsx` 中，这些回调会调用 `update` 方法更新表单状态：

代码位置：`packages/ui/primitives/document-flow/add-fields.tsx:322-368`

```typescript
const onFieldResize = useCallback(
  (node: HTMLElement, index: number) => {
    const field = localFields[index];
    const $page = window.document.querySelector<HTMLElement>(
      `${PDF_VIEWER_PAGE_SELECTOR}[data-page-number="${field.pageNumber}"]`,
    );
    
    if (!$page) {
      return;
    }
    
    // 计算新的百分比坐标
    const { x: pageX, y: pageY, width: pageWidth, height: pageHeight } = getFieldPosition($page, node);
    
    update(index, {
      ...field,
      pageX,
      pageY,
      pageWidth,
      pageHeight,
    });
  },
  [getFieldPosition, localFields, update],
);
```

#### 第二步：自动保存防抖

字段状态变更后，触发自动保存机制。`useAutoSave` hook 实现了 2 秒防抖，避免频繁操作产生过多请求。

代码位置：`packages/lib/client-only/hooks/use-autosave.ts:1-61`

```typescript
export const useAutoSave = <T, R = void>(onSave: (data: T) => Promise<R>, options: { delay?: number } = {}) => {
  const { delay = 2000 } = options;  // 默认 2 秒防抖
  
  const saveTimeoutRef = useRef<NodeJS.Timeout>();
  const saveQueueRef = useRef<SaveRequest<T, R>[]>([]);
  const isProcessingRef = useRef(false);

  const scheduleSave = useCallback(
    (data: T, onResponse?: (response: R) => void) => {
      // 清除之前的定时器
      if (saveTimeoutRef.current) {
        clearTimeout(saveTimeoutRef.current);
      }
      
      // 设置新的定时器，2秒后执行保存
      saveTimeoutRef.current = setTimeout(() => void saveFormData(data, onResponse), delay);
    },
    [delay],
  );
  
  // ... 队列处理逻辑
};
```

在 `add-fields.tsx` 中，字段失焦时触发自动保存：

代码位置：`packages/ui/primitives/document-flow/add-fields.tsx:538-550`

```typescript
const { scheduleSave } = useAutoSave(onAutoSave);

const handleAutoSave = async () => {
  const isFormValid = await form.trigger();
  
  if (!isFormValid) {
    return;
  }
  
  const formData = form.getValues();
  scheduleSave(formData);  // 调度保存
};

// FieldItem 组件中触发
onBlur={() => {
  setLastActiveField(null);
  void handleAutoSave();  // 失焦时自动保存
}}
```

#### 第三步：前端请求构造

`document-edit-form.tsx` 中的 `onAddFieldsFormAutoSave` 构造实际请求：

代码位置：`apps/remix/app/components/general/document/document-edit-form.tsx:284-332`

```typescript
const saveFieldsData = async (data: TAddFieldsFormSchema) => {
  return addFields({
    documentId: document.id,
    fields: data.fields.map((field) => ({
      ...field,
      id: field.nativeId,
      envelopeItemId: document.documentData.envelopeItemId,
    })),
  });
};

const onAddFieldsFormAutoSave = async (data: TAddFieldsFormSchema) => {
  try {
    await saveFieldsData(data);
  } catch (err) {
    // ... 错误处理
  }
};
```

#### 第四步：tRPC 路由层

请求通过 `fieldRouter.setFieldsForDocument` 进入后端：

代码位置：`packages/trpc/server/field-router/router.ts:281-315`

```typescript
setFieldsForDocument: authenticatedProcedure
  .input(ZSetDocumentFieldsRequestSchema)
  .output(ZSetDocumentFieldsResponseSchema)
  .mutation(async ({ input, ctx }) => {
    const { teamId } = ctx;
    const { documentId, fields } = input;

    return await setFieldsForDocument({
      userId: ctx.user.id,
      teamId,
      id: {
        type: 'documentId',
        id: documentId,
      },
      fields: fields.map((field) => ({
        id: field.id,
        recipientId: field.recipientId,
        envelopeItemId: field.envelopeItemId,
        type: field.type,
        pageNumber: field.pageNumber,
        pageX: field.pageX,
        pageY: field.pageY,
        pageWidth: field.pageWidth,
        pageHeight: field.pageHeight,
        fieldMeta: field.fieldMeta,
      })),
      requestMetadata: ctx.metadata,
    });
  }),
```

#### 第五步：后端业务逻辑 - setFieldsForDocument

核心保存逻辑在 `setFieldsForDocument` 中：

代码位置：`packages/lib/server-only/field/set-fields-for-document.ts:37-343`

```typescript
export const setFieldsForDocument = async ({
  userId,
  teamId,
  id,
  fields,
  requestMetadata,
}: SetFieldsForDocumentOptions) => {
  // 1. 获取文档和现有字段
  const envelope = await prisma.envelope.findFirst({
    where: envelopeWhereInput,
    include: {
      recipients: true,
      envelopeItems: true,
      fields: {
        include: { recipient: true },
      },
    },
  });

  // 2. 找出被删除的字段
  const removedFields = existingFields.filter(
    (existingField) => !fields.find((field) => field.id === existingField.id),
  );

  // 3. 校验和关联字段
  const linkedFields = fields.map((field) => {
    // 校验 recipient、envelopeItem 等
    return { ...field, _persisted: existing, _recipient: recipient };
  });

  // 4. 事务性 upsert 字段
  const persistedFields = await prisma.$transaction(async (tx) => {
    return await Promise.all(
      linkedFields.map(async (field) => {
        // 校验字段类型的 fieldMeta
        // ... CHECKBOX, RADIO, DROPDOWN 等校验
        
        const upsertedField = await tx.field.upsert({
          where: {
            id: field._persisted?.id ?? -1,
            envelopeId: envelope.id,
            envelopeItemId: field.envelopeItemId,
          },
          update: {
            page: field.pageNumber,
            positionX: field.pageX,  // 保存百分比坐标
            positionY: field.pageY,
            width: field.pageWidth,
            height: field.pageHeight,
            fieldMeta: parsedFieldMeta,
          },
          create: {
            type: field.type,
            page: field.pageNumber,
            positionX: field.pageX,
            positionY: field.pageY,
            width: field.pageWidth,
            height: field.pageHeight,
            // ... 其他字段
          },
        });
        
        // 5. 记录审计日志
        // ... FIELD_UPDATED / FIELD_CREATED 日志
        
        return { ...upsertedField, formId: field.formId };
      }),
    );
  });

  // 6. 删除被移除的字段
  if (removedFields.length > 0) {
    await prisma.$transaction(async (tx) => {
      await tx.field.deleteMany({ where: { id: { in: removedFields.map((f) => f.id) } } });
      // 记录 FIELD_DELETED 审计日志
    });
  }
};
```

#### 保存链路完整流程图

```
用户拖拽/调整大小
       ↓
react-rnd onDragStop/onResizeStop
       ↓
add-fields.tsx onFieldMove/onFieldResize
       ↓
react-hook-form update (更新本地 state)
       ↓
field blur → handleAutoSave()
       ↓
useAutoSave.scheduleSave() (2秒防抖)
       ↓
document-edit-form.tsx onAddFieldsFormAutoSave
       ↓
trpc.field.setFieldsForDocument (tRPC 路由)
       ↓
setFieldsForDocument (业务逻辑)
       ↓
prisma.field.upsert (数据库写入)
```

---

### 2.2 读取链路：从数据库到画布渲染

#### 第一步：加载文档数据

用户打开文档编辑页面时，通过 `trpc.document.get` 获取文档数据（包含字段）。

代码位置：`apps/remix/app/components/general/document/document-edit-form.tsx:54-88`

```typescript
const { data: document, refetch: refetchDocument } = trpc.document.get.useQuery(
  {
    documentId: initialDocument.id,
  },
  {
    initialData: initialDocument,
    ...SKIP_QUERY_BATCH_META,
  },
);

const { recipients, fields } = document;  // 解构出 fields
```

后端 `getDocumentRoute` 调用 `getDocumentWithDetailsById`：

代码位置：`packages/trpc/server/document-router/get-document.ts:1-28`

```typescript
export const getDocumentRoute = authenticatedProcedure
  .input(ZGetDocumentRequestSchema)
  .output(ZGetDocumentResponseSchema)
  .query(async ({ input, ctx }) => {
    return await getDocumentWithDetailsById({
      userId: user.id,
      teamId,
      id: {
        type: 'documentId',
        id: documentId,
      },
    });
  });
```

`getEnvelopeById` 查询时包含 `fields`：

代码位置：`packages/lib/server-only/envelope/get-envelope-by-id.ts:39-80`

```typescript
const envelope = await prisma.envelope.findFirst({
  where: envelopeWhereInput,
  include: {
    envelopeItems: { include: { documentData: true } },
    recipients: true,
    fields: true,  // 包含字段数据
    // ...
  },
});
```

#### 第二步：表单初始化

字段数据传入 `AddFieldsFormPartial`，通过 `react-hook-form` 的 `defaultValues` 初始化：

代码位置：`packages/ui/primitives/document-flow/add-fields.tsx:104-121`

```typescript
const form = useForm<TAddFieldsFormSchema>({
  defaultValues: {
    fields: fields.map((field) => ({
      nativeId: field.id,
      formId: `${field.id}-${field.envelopeItemId}`,
      pageNumber: field.page,           // 页码
      type: field.type,
      pageX: Number(field.positionX),   // 百分比 X 坐标
      pageY: Number(field.positionY),   // 百分比 Y 坐标
      pageWidth: Number(field.width),   // 百分比宽度
      pageHeight: Number(field.height), // 百分比高度
      signerEmail: recipients.find((r) => r.id === field.recipientId)?.email ?? '',
      recipientId: field.recipientId,
      fieldMeta: field.fieldMeta ? ZFieldMetaSchema.parse(field.fieldMeta) : undefined,
    })),
  },
  resolver: zodResolver(ZAddFieldsFormSchema),
});
```

#### 第三步：字段渲染 - FieldItem

`FieldItemInner` 组件根据百分比坐标计算实际像素位置：

代码位置：`packages/ui/primitives/document-flow/field-item.tsx:115-142`

```typescript
const calculateCoords = useCallback(() => {
  const $page = document.querySelector<HTMLElement>(
    `${PDF_VIEWER_PAGE_SELECTOR}[data-page-number="${field.pageNumber}"]`,
  );

  if (!$page) {
    return;
  }

  const { height, width } = $page.getBoundingClientRect();
  const top = $page.getBoundingClientRect().top + window.scrollY;
  const left = $page.getBoundingClientRect().left + window.scrollX;

  // 百分比 → 像素
  const pageX = (field.pageX / 100) * width + left;
  const pageY = (field.pageY / 100) * height + top;
  const pageHeight = (field.pageHeight / 100) * height;
  const pageWidth = (field.pageWidth / 100) * width;

  setCoords({
    pageX: pageX,
    pageY: pageY,
    pageHeight: pageHeight,
    pageWidth: pageWidth,
  });
}, [field.pageHeight, field.pageNumber, field.pageWidth, field.pageX, field.pageY]);
```

然后通过 `react-rnd` 渲染到正确位置：

代码位置：`packages/ui/primitives/document-flow/field-item.tsx:234-272`

```typescript
return createPortal(
  <Rnd
    default={{
      x: coords.pageX,      // 计算后的像素 X
      y: coords.pageY,      // 计算后的像素 Y
      height: fixedSize ? 'auto' : coords.pageHeight,
      width: fixedSize ? 'auto' : coords.pageWidth,
    }}
    bounds={`${PDF_VIEWER_PAGE_SELECTOR}[data-page-number="${field.pageNumber}"]`}
    // ...
  >
    {/* 字段内容 */}
  </Rnd>,
  document.body,
);
```

#### 第四步：动态响应尺寸变化

页面缩放或窗口大小变化时，`useFieldPageCoords` hook 会自动重新计算：

代码位置：`packages/lib/client-only/hooks/use-field-page-coords.ts:38-53`

```typescript
useEffect(() => {
  const onResize = () => {
    calculateCoords();  // 窗口 resize 时重新计算
  };

  window.addEventListener('resize', onResize);
  return () => window.removeEventListener('resize', onResize);
}, [calculateCoords]);
```

#### 读取链路完整流程图

```
用户打开文档编辑页面
       ↓
trpc.document.get.useQuery()
       ↓
getDocumentWithDetailsById → getEnvelopeById
       ↓
prisma.envelope.findFirst(include: { fields: true })
       ↓
AddFieldsFormPartial (fields prop)
       ↓
react-hook-form defaultValues (Number(field.positionX))
       ↓
FieldItemInner.calculateCoords() 百分比→像素
       ↓
react-rnd 渲染到正确位置
       ↓
ResizeObserver / window resize 监听 → 重新计算
```

---

## 3. 坐标边界校验与页码约束

### 3.1 Zod 校验定义

所有坐标和页码都通过 Zod schema 进行严格校验：

代码位置：`packages/lib/types/field.ts:57-89`

```typescript
// 页码：最小值为 1
export const ZFieldPageNumberSchema = z.number().min(1).describe('The page number the field will be on.');

// X 坐标：0-100 百分比范围
export const ZClampedFieldPositionXSchema = z
  .number()
  .min(0)
  .max(100)
  .describe('The percentage based X coordinate where the field will be placed.');

// Y 坐标：0-100 百分比范围
export const ZClampedFieldPositionYSchema = z
  .number()
  .min(0)
  .max(100)
  .describe('The percentage based Y coordinate where the field will be placed.');

// 宽度：0-100 百分比范围
export const ZClampedFieldWidthSchema = z
  .number()
  .min(0)
  .max(100)
  .describe('The percentage based width of the field on the page.');

// 高度：0-100 百分比范围
export const ZClampedFieldHeightSchema = z
  .number()
  .min(0)
  .max(100)
  .describe('The percentage based height of the field on the page.');
```

### 3.2 校验应用位置

#### tRPC 路由层校验

`set-envelope-fields.types.ts` 中应用这些约束：

代码位置：`packages/trpc/server/envelope-router/set-envelope-fields.types.ts:12-31`

```typescript
export const ZSetEnvelopeFieldsRequestSchema = z.object({
  envelopeId: z.string(),
  envelopeType: z.nativeEnum(EnvelopeType),
  fields: z.array(
    z.object({
      id: z.number().optional(),
      formId: z.string().optional(),
      envelopeItemId: z.string(),
      recipientId: z.number(),
      type: z.nativeEnum(FieldType),
      page: z.number().min(1),           // 页码 ≥ 1
      positionX: ZClampedFieldPositionXSchema,  // 0-100
      positionY: ZClampedFieldPositionYSchema,  // 0-100
      width: ZClampedFieldWidthSchema,         // 0-100
      height: ZClampedFieldHeightSchema,       // 0-100
      fieldMeta: ZFieldMetaSchema,
    }),
  ),
});
```

#### 前端表单层校验

`add-fields.types.ts` 中也有基础校验：

代码位置：`packages/ui/primitives/document-flow/add-fields.types.ts:5-21`

```typescript
export const ZAddFieldsFormSchema = z.object({
  fields: z.array(
    z.object({
      formId: z.string().min(1),
      nativeId: z.number().optional(),
      type: z.nativeEnum(FieldType),
      signerEmail: z.string().min(1),
      recipientId: z.number().min(1),
      pageNumber: z.number().min(1),   // 页码 ≥ 1
      pageX: z.number().min(0),        // X ≥ 0
      pageY: z.number().min(0),        // Y ≥ 0
      pageWidth: z.number().min(0),    // 宽度 ≥ 0
      pageHeight: z.number().min(0),   // 高度 ≥ 0
      fieldMeta: ZFieldMetaSchema,
    }),
  ),
});
```

### 3.3 UI 层约束 - 拖拽边界

除了数据校验，UI 层也通过 `bounds` 属性限制拖拽范围：

代码位置：`packages/ui/primitives/document-flow/field-item.tsx:253`

```typescript
<Rnd
  bounds={`${PDF_VIEWER_PAGE_SELECTOR}[data-page-number="${field.pageNumber}"]`}
  // ...
/>
```

以及 `isWithinPageBounds` 在添加字段时的校验：

代码位置：`packages/lib/client-only/hooks/use-document-element.ts:50-71`

```typescript
const isWithinPageBounds = useCallback((event: MouseEvent, pageSelector: string, mouseWidth = 0, mouseHeight = 0) => {
  const $page = getPage(event, pageSelector);
  
  if (!$page) {
    return false;
  }
  
  const { top, left, height, width } = $page.getBoundingClientRect();
  
  const halfMouseWidth = mouseWidth / 2;
  const halfMouseHeight = mouseHeight / 2;
  
  // Y 边界检查
  if (event.clientY > top + height - halfMouseHeight || event.clientY < top + halfMouseHeight) {
    return false;
  }
  
  // X 边界检查
  if (event.clientX > left + width - halfMouseWidth || event.clientX < left + halfMouseWidth) {
    return false;
  }
  
  return true;
}, []);
```

### 3.4 后端业务约束

`setFieldsForDocument` 中还有额外的业务约束：

代码位置：`packages/lib/server-only/field/set-fields-for-document.ts:107-119`

```typescript
// 已签名的收件人字段不能修改
if (existing && hasFieldBeenChanged(existing, field) && !canRecipientFieldsBeModified(recipient, existingFields)) {
  throw new AppError(AppErrorCode.INVALID_REQUEST, {
    message: 'Cannot modify a field where the recipient has already interacted with the document',
  });
}

// 已签名的收件人不能新增字段
if (!existing && !canRecipientFieldsBeModified(recipient, existingFields)) {
  throw new AppError(AppErrorCode.INVALID_REQUEST, {
    message: 'Cannot modify a field where the recipient has already interacted with the document',
  });
}
```

### 3.5 校验层级汇总

| 层级 | 校验内容 | 代码位置 |
|------|----------|----------|
| **UI 拖拽层** | 限制在页面范围内 | `field-item.tsx` bounds 属性 |
| **添加字段时** | 鼠标位置在页面边界内 | `use-document-element.ts` isWithinPageBounds |
| **前端表单** | pageNumber ≥ 1, pageX ≥ 0 等 | `add-fields.types.ts` |
| **tRPC 路由** | page ≥ 1, positionX/Y 0-100, width/height 0-100 | `set-envelope-fields.types.ts` |
| **后端业务** | 已签名收件人字段不能修改 | `set-fields-for-document.ts` |

---

## 4. 字段拖拽行为

### 4.1 技术实现

字段拖拽功能使用 `react-rnd` 库实现，相关代码位于：
- `packages/ui/primitives/document-flow/field-item.tsx`

### 4.2 拖拽流程

1. **点击添加字段**
   - 选择字段类型（签名、文本、日期等）
   - 选择签单人
   - 点击 PDF 页面添加字段

2. **初始坐标计算** (`add-fields.tsx:261-320`)
   - 获取鼠标点击位置
   - 检测点击落在哪个 PDF 页面上
   - 计算字段的初始位置和大小

3. **拖拽移动**
   - 用户可以自由拖动字段在页面内移动
   - 字段被限制在页面边界内（`bounds` 属性）
   - 拖拽结束后更新坐标

4. **调整大小**
   - 用户可以调整字段的宽高
   - 调整结束后更新尺寸坐标

### 4.3 关键处理事件

```
onDragStart  → 激活字段
onDragStop   → 更新位置 (调用 onMove 回调)
onResizeStart → 激活字段  
onResizeStop  → 更新尺寸 (调用 onResize 回调)
```

---

## 5. 坐标归一化系统

### 5.1 为什么需要归一化？

PDF 页面在不同设备上显示的像素尺寸不同：
- 桌面端全屏：页面宽高可能是 800x1131 像素
- 移动端：页面宽高可能是 360x509 像素
- 缩放视图：页面尺寸会动态变化

如果存储像素坐标，在不同设备或缩放级别下打开时，字段位置会错位。

### 5.2 归一化方案

**采用百分比坐标系统**（相对于页面的宽度和高度）

#### 像素转百分比（添加/拖拽时）

代码位置：`packages/lib/client-only/hooks/use-document-element.ts:30-41`

```typescript
const getFieldPosition = (page: HTMLElement, field: HTMLElement) => {
  const { top: pageTop, left: pageLeft, height: pageHeight, width: pageWidth } = 
    getBoundingClientRect(page);
  
  const { top: fieldTop, left: fieldLeft, height: fieldHeight, width: fieldWidth } = 
    getBoundingClientRect(field);

  return {
    x: ((fieldLeft - pageLeft) / pageWidth) * 100,      // 转百分比
    y: ((fieldTop - pageTop) / pageHeight) * 100,       // 转百分比
    width: (fieldWidth / pageWidth) * 100,              // 转百分比
    height: (fieldHeight / pageHeight) * 100,           // 转百分比
  };
};
```

**计算公式：**
```
百分比坐标 = (像素坐标 / 页面尺寸) × 100
```

#### 百分比转像素（渲染时）

代码位置：`packages/lib/client-only/hooks/use-field-page-coords.ts:23-35`

```typescript
// X 和 Y 是页面高度和宽度的百分比
const fieldX = (Number(field.positionX) / 100) * width + left;
const fieldY = (Number(field.positionY) / 100) * height + top;

const fieldHeight = (Number(field.height) / 100) * height;
const fieldWidth = (Number(field.width) / 100) * width;
```

**计算公式：**
```
像素坐标 = (百分比坐标 / 100) × 当前页面尺寸
```

### 5.3 坐标原点

- **坐标系原点**：页面左上角 (0, 0)
- **注意**：PDF 标准坐标系原点是左下角，但系统已转换为左上原点以便网页渲染

### 5.4 实际例子

假设 PDF 页面尺寸为：
- 页面宽度：612 pt（8.5 英寸）
- 页面高度：792 pt（11 英寸）

在浏览器中渲染时实际像素为：
- 页面宽度：800 px
- 页面高度：1035 px

**字段位置：**
- 距离页面左边：120 px
- 距离页面上边：207 px
- 宽度：100 px
- 高度：40 px

**归一化后存储：**
```
positionX = (120 / 800) × 100 = 15.00
positionY = (207 / 1035) × 100 = 20.00
width = (100 / 800) × 100 = 12.50
height = (40 / 1035) × 100 = 3.86
```

**在另一个设备上渲染（页面宽 400 px，高 518 px）：**
```
左边距 = (15.00 / 100) × 400 = 60 px
上边距 = (20.00 / 100) × 518 = 103.6 px
宽度 = (12.50 / 100) × 400 = 50 px
高度 = (3.86 / 100) × 518 ≈ 20 px
```

字段保持相对位置不变！

---

## 6. 与 PDF 页码绑定

### 6.1 存储结构

每个字段通过 `page` 字段与 PDF 页码绑定：

代码位置：`packages/prisma/schema.prisma:634-658`

```prisma
model Field {
  id             Int          @id @default(autoincrement())
  secondaryId    String       @unique @default(cuid())
  envelopeId     String
  envelopeItemId String
  recipientId    Int
  type           FieldType
  
  // 页码绑定
  page           Int          /// @zod.number.describe("The page number of the field on the document. Starts from 1.")
  
  // 坐标和尺寸（百分比）
  positionX      Decimal      @default(0)
  positionY      Decimal      @default(0)
  width          Decimal      @default(-1)
  height         Decimal      @default(-1)
  
  // ... 其他字段
}
```

### 6.2 页码规则

- **页码从 1 开始**（不是 0 索引）
- **每个字段绑定到一个具体页码**
- **字段不会跨页**：一个字段只能在一个页面上

### 6.3 页面选择机制

当用户点击添加字段时，系统会检测鼠标位置落在哪个页面上：

代码位置：`packages/lib/client-only/hooks/use-document-element.ts:8-24`

```typescript
const getPage = (event: MouseEvent, pageSelector: string) => {
  if (!(event.target instanceof HTMLElement)) {
    return null;
  }

  const target = event.target;

  // 尝试从事件目标向上查找最近的页面元素
  const $page =
    target.closest<HTMLElement>(pageSelector) ??
    // 如果找不到，使用坐标点检测
    document.elementsFromPoint(event.clientX, event.clientY)
      .find((el) => el.matches(pageSelector));

  if (!$page) {
    return null;
  }

  return $page;
};
```

页面元素通过 `data-page-number` 属性标识页码：

```typescript
// add-fields.tsx:279
const pageNumber = parseInt($page.getAttribute('data-page-number') ?? '1', 10);
```

### 6.4 跨页面复制

系统支持将字段复制到所有页面：

代码位置：`packages/ui/primitives/document-flow/add-fields.tsx:393-418`

```typescript
if (duplicateAll) {
  const totalPages = getPdfPagesCount();
  
  for (let pageNumber = 1; pageNumber <= totalPages; pageNumber += 1) {
    if (pageNumber === lastActiveField.pageNumber) {
      continue;  // 跳过当前页
    }
    
    const newField = {
      ...structuredClone(lastActiveField),
      nativeId: undefined,
      formId: nanoid(12),
      pageNumber,  // 改变页码
    };
    
    append(newField);
  }
}
```

---

## 7. 数据库存储格式

### 7.1 字段模型核心属性

| 属性名 | 类型 | 说明 |
|--------|------|------|
| `page` | Int | 页码，从 1 开始 |
| `positionX` | Decimal | X 坐标百分比 (0-100) |
| `positionY` | Decimal | Y 坐标百分比 (0-100) |
| `width` | Decimal | 宽度百分比 (0-100) |
| `height` | Decimal | 高度百分比 (0-100) |

### 7.2 为什么使用 Decimal？

- **精度要求**：坐标需要精确到小数点后多位
- **避免浮点误差**：JavaScript 的 float 类型有精度问题
- **计算精确**：百分比计算需要高精度

### 7.3 数据流转汇总

| 阶段 | 坐标格式 | 转换公式 | 代码位置 |
|------|----------|----------|----------|
| **用户操作** | 像素坐标 (px) | - | react-rnd |
| **保存时** | 百分比 (0-100) | `(像素 / 页面尺寸) × 100` | `use-document-element.ts` |
| **数据库** | Decimal | - | Prisma Field 表 |
| **读取时** | 百分比 → 像素 | `(百分比 / 100) × 当前页面尺寸` | `use-field-page-coords.ts` |

### 7.4 Placeholder 定位（特殊情况）

除了拖拽定位，系统还支持通过文本占位符自动定位：

代码位置：`packages/lib/server-only/field/create-envelope-fields.ts:170-227`

```typescript
if (isPlaceholderPosition(field)) {
  const matches = pdfDoc.findText(field.placeholder);
  
  return matchesToProcess.map((match) => {
    // PDF 坐标系是左下角为原点
    // 需要转换为系统使用的左上角原点
    const topLeftY = page.height - match.bbox.y - match.bbox.height;
    
    return {
      page: match.pageIndex + 1,  // 转成从 1 开始的页码
      positionX: (match.bbox.x / page.width) * 100,
      positionY: (topLeftY / page.height) * 100,
      width: widthPercent,
      height: heightPercent,
    };
  });
}
```

---

## 8. 关键文件速查

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| 拖拽渲染 | `packages/ui/primitives/document-flow/field-item.tsx` | react-rnd 封装，拖拽/调整大小 |
| 拖拽逻辑 | `packages/ui/primitives/document-flow/add-fields.tsx` | 字段添加、移动、调整的事件处理 |
| 自动保存 | `packages/lib/client-only/hooks/use-autosave.ts` | 2秒防抖的保存队列 |
| 保存入口 | `apps/remix/app/components/general/document/document-edit-form.tsx` | onAddFieldsFormAutoSave |
| tRPC 路由 | `packages/trpc/server/field-router/router.ts` | setFieldsForDocument 路由 |
| 保存业务 | `packages/lib/server-only/field/set-fields-for-document.ts` | upsert 字段、审计日志 |
| 读取入口 | `packages/trpc/server/document-router/get-document.ts` | document.get 路由 |
| 读取查询 | `packages/lib/server-only/envelope/get-envelope-by-id.ts` | include fields |
| 坐标转换 | `packages/lib/client-only/hooks/use-document-element.ts` | 像素 → 百分比 |
| 渲染定位 | `packages/lib/client-only/hooks/use-field-page-coords.ts` | 百分比 → 像素 |
| 约束定义 | `packages/lib/types/field.ts` | Zod schema 定义 |
| 约束应用 | `packages/trpc/server/envelope-router/set-envelope-fields.types.ts` | tRPC 请求校验 |
| 数据模型 | `packages/prisma/schema.prisma` | Field 表结构定义 |

---

## 9. 总结

字段位置保存的核心设计：

1. **百分比坐标**：解决不同屏幕尺寸的适配问题
2. **页码绑定**：每个字段与特定页面关联
3. **Decimal 存储**：确保计算精度
4. **多层校验**：UI → 表单 → tRPC → 业务逻辑
5. **自动保存**：2秒防抖 + 队列处理
6. **坐标系转换**：PDF 左下原点 → 网页左上原点

这套系统确保了：
- ✅ 保存后下次打开位置不变
- ✅ 在不同设备和缩放级别下位置正确
- ✅ 导出 PDF 时位置准确
- ✅ 支持跨页面复制字段
- ✅ 防止无效坐标写入数据库
- ✅ 已签名字段不可修改
