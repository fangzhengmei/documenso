# 字段位置存储与坐标系统分析

## 1. 概述

在 Documenso 中，用户可以将签名框、文本框等字段拖拽到 PDF 画布的任意位置，并在保存后下次打开时保持原位置。这一功能通过以下核心机制实现：

- **拖拽交互**：使用 `react-rnd` 库实现拖拽和调整大小
- **坐标归一化**：将像素坐标转换为百分比坐标
- **页码绑定**：将字段与特定页码绑定
- **数据库存储**：使用 Decimal 类型精确存储坐标数据

---

## 2. 字段拖拽行为

### 2.1 技术实现

字段拖拽功能使用 `react-rnd` 库实现，相关代码位于：
- `packages/ui/primitives/document-flow/field-item.tsx`

### 2.2 拖拽流程

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

### 2.3 关键处理事件

```
onDragStart  → 激活字段
onDragStop   → 更新位置 (调用 onMove 回调)
onResizeStart → 激活字段  
onResizeStop  → 更新尺寸 (调用 onResize 回调)
```

---

## 3. 坐标归一化系统

### 3.1 为什么需要归一化？

PDF 页面在不同设备上显示的像素尺寸不同：
- 桌面端全屏：页面宽高可能是 800x1131 像素
- 移动端：页面宽高可能是 360x509 像素
- 缩放视图：页面尺寸会动态变化

如果存储像素坐标，在不同设备或缩放级别下打开时，字段位置会错位。

### 3.2 归一化方案

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

### 3.3 坐标原点

- **坐标系原点**：页面左上角 (0, 0)
- **注意**：PDF 标准坐标系原点是左下角，但系统已转换为左上原点以便网页渲染

### 3.4 实际例子

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

## 4. 与 PDF 页码绑定

### 4.1 存储结构

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

### 4.2 页码规则

- **页码从 1 开始**（不是 0 索引）
- **每个字段绑定到一个具体页码**
- **字段不会跨页**：一个字段只能在一个页面上

### 4.3 页面选择机制

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

### 4.4 跨页面复制

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

## 5. 数据库存储格式

### 5.1 字段模型核心属性

| 属性名 | 类型 | 说明 |
|--------|------|------|
| `page` | Int | 页码，从 1 开始 |
| `positionX` | Decimal | X 坐标百分比 (0-100) |
| `positionY` | Decimal | Y 坐标百分比 (0-100) |
| `width` | Decimal | 宽度百分比 (0-100) |
| `height` | Decimal | 高度百分比 (0-100) |

### 5.2 为什么使用 Decimal？

- **精度要求**：坐标需要精确到小数点后多位
- **避免浮点误差**：JavaScript 的 float 类型有精度问题
- **计算精确**：百分比计算需要高精度

### 5.3 数据流转

#### 添加字段时

1. 前端计算百分比坐标（`add-fields.tsx:282-291`）
   ```typescript
   let pageX = ((event.pageX - left) / width) * 100;
   let pageY = ((event.pageY - top) / height) * 100;
   const fieldPageWidth = (fieldBounds.current.width / width) * 100;
   const fieldPageHeight = (fieldBounds.current.height / height) * 100;
   ```

2. 保存到数据库（`create-envelope-fields.ts:244-260`）
   ```typescript
   const createdFields = await tx.field.createManyAndReturn({
     data: validatedFields.map((field) => ({
       type: field.type,
       page: field.page,
       positionX: field.positionX,
       positionY: field.positionY,
       width: field.width,
       height: field.height,
       // ...
     })),
   });
   ```

#### 加载字段时

1. 从数据库读取字段数据
2. 前端根据当前页面尺寸转换为像素坐标（`use-field-page-coords.ts`）
3. 渲染字段到正确位置

### 5.4 Placeholder 定位（特殊情况）

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

## 6. 关键文件速查

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| 拖拽渲染 | `packages/ui/primitives/document-flow/field-item.tsx` | react-rnd 封装，拖拽/调整大小 |
| 拖拽逻辑 | `packages/ui/primitives/document-flow/add-fields.tsx` | 字段添加、移动、调整的事件处理 |
| 坐标转换 | `packages/lib/client-only/hooks/use-document-element.ts` | 像素 ↔ 百分比 转换 |
| 渲染定位 | `packages/lib/client-only/hooks/use-field-page-coords.ts` | 读取时的坐标计算 |
| 存储 | `packages/lib/server-only/field/create-envelope-fields.ts` | 字段保存到数据库 |
| PDF 插入 | `packages/lib/server-only/pdf/insert-field-in-pdf-v2.ts` | 导出 PDF 时的字段渲染 |
| 数据模型 | `packages/prisma/schema.prisma` | Field 表结构定义 |

---

## 7. 总结

字段位置保存的核心设计：

1. **百分比坐标**：解决不同屏幕尺寸的适配问题
2. **页码绑定**：每个字段与特定页面关联
3. **Decimal 存储**：确保计算精度
4. **坐标系转换**：PDF 左下原点 → 网页左上原点

这套系统确保了：
- ✅ 保存后下次打开位置不变
- ✅ 在不同设备和缩放级别下位置正确
- ✅ 导出 PDF 时位置准确
- ✅ 支持跨页面复制字段
