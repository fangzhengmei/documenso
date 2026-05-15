# Konva 坐标语义一致性报告

## 核心结论

**统一坐标体系定义：**

| 术语 | 定义 | 语义 | 获取方式 |
|-----|------|------|---------|
| **逻辑坐标** (Logical Position) | 元素自身存储的 x/y 属性，不包含任何缩放变换 | 局部坐标空间 | `group.x()`, `group.y()` |
| **绝对坐标** (Absolute Position) | 相对于 Stage 原点的坐标，已包含所有父级 scale 变换 | 全局坐标空间 | `group.absolutePosition()` |
| **显示坐标** (Display Position) | 视觉上渲染到屏幕的像素坐标，与 `dragBoundFunc` 的 `pos.x/y` 同坐标系 | 视觉像素空间 | `pos.x`, `pos.y` (dragBoundFunc 参数) |
| **边界框坐标** (Client Rect) | 元素视觉边界框的像素尺寸和位置 | 视觉像素空间 | `group.getClientRect()` |

**核心变换关系：**

```
绝对坐标 = 逻辑坐标 × Stage.scale

显示坐标 = 绝对坐标 = 逻辑坐标 × Stage.scale

即：pos.x = group.x() * scale
```

---

## 一、统一坐标语义体系

### 1.1 四层坐标模型

```
第 1 层：浏览器坐标
  ├─ clientX/Y: 相对于视口左上角
  └─ pageX/Y:   相对于文档左上角（含滚动）

           ↓ DOM 元素边界计算

第 2 层：Stage 显示坐标
  ├─ Stage 容器内的像素坐标
  └─ dragBoundFunc pos.x / pos.y 所处的空间

           ↓ Konva 自动逆变换
           ↓ 返回值 / scale

第 3 层：Stage 逻辑坐标
  ├─ group.x() / group.y() 获取的值
  └─ 元素存储的原始坐标（未缩放）

           ↓ 乘百分比 → 存入 DB

第 4 层：百分比坐标
  └─ 数据库最终存储格式
```

### 1.2 各层坐标的精确关系

| 关系 | 公式 | 说明 |
|-----|------|------|
| pos → 逻辑坐标 | `group.x() = pos.x / scale` | Konva 内部自动完成 |
| 逻辑坐标 → pos | `pos.x = group.x() * scale` | 正向变换 |
| 边界值计算 | `maxXPosition = (pageWidth - fieldWidth) * scale` | 与 pos 同坐标系 |
| 逻辑坐标 → 百分比 | `positionX = (group.x() / pageWidth) * 100` | 入库前转换 |
| 百分比 → 逻辑坐标 | `group.x() = pageWidth * (positionX / 100)` | 渲染时还原 |

---

## 二、正向验证算例（拖拽 → 入库）

### 2.1 测试场景参数

| 参数 | 数值 | 说明 |
|-----|------|------|
| pageWidth | 800px | PDF 页面原始宽度（Stage 逻辑宽度） |
| pageHeight | 1200px | PDF 页面原始高度 |
| fieldWidth | 100px | 字段宽度（逻辑像素） |
| fieldHeight | 50px | 字段高度（逻辑像素） |
| scale | 0.75 | Stage 缩放因子 |
| Stage.width | 600px | 显示宽度 = 800 * 0.75 |

### 2.2 拖拽过程验证

**场景：用户将字段拖拽到 Stage 内的显示坐标 pos.x = 300**

#### 步骤 1：边界判断
```typescript
maxXPosition = (pageWidth - fieldWidth) * scale
             = (800 - 100) * 0.75
             = 700 * 0.75
             = 525px

边界判断：pos.x = 300 ≤ 525 → 允许
```

#### 步骤 2：dragBoundFunc 返回值
```typescript
returnValue.x = 300  // 与 pos.x 同坐标系
```

#### 步骤 3：Konva 内部逆变换（自动完成）
```typescript
group.x() = returnValue.x / scale
          = 300 / 0.75
          = 400px  // 逻辑坐标
```

#### 步骤 4：转换为百分比入库
```typescript
positionX = (group.x() / pageWidth) * 100
          = (400 / 800) * 100
          = 50%
```

**验证结果：** ✅ 字段被放置在页面水平 50% 的位置

---

## 三、逆向验证算例（读取 → 渲染）

### 3.1 测试场景：从数据库读取并渲染

**数据库存储值：**
- `positionX = 50%`
- `positionY = 30%`
- `width = 12.5%` (对应 100px)
- `height = 4.17%` (对应 50px)

#### 步骤 1：百分比 → 逻辑坐标
```typescript
// 调用 calculateFieldPosition()
fieldX = pageWidth * (positionX / 100)
       = 800 * (50 / 100)
       = 800 * 0.5
       = 400px  // 逻辑坐标

fieldY = 1200 * (30 / 100)
       = 360px  // 逻辑坐标

fieldWidth = 800 * (12.5 / 100)
           = 100px

fieldHeight = 1200 * (4.17 / 100)
            ≈ 50px
```

#### 步骤 2：设置 Group 位置（逻辑坐标）
```typescript
group.x(fieldX)  // = 400
group.y(fieldY)  // = 360
```

#### 步骤 3：Konva 自动应用 scale 渲染
```typescript
// 渲染时的显示坐标（视觉像素）
displayX = group.x() * scale
         = 400 * 0.75
         = 300px

displayY = group.y() * scale
         = 360 * 0.75
         = 270px
```

#### 步骤 4：验证绝对坐标一致性
```typescript
absolutePosition = group.absolutePosition()
                 = { x: 300, y: 270 }

// 与 dragBoundFunc 中的 pos.x = 300 完全一致 ✓
```

**逆向验证结果：** ✅ 渲染后的显示坐标与拖拽时的目标坐标完全一致

---

## 四、关键 API 语义校正

### 4.1 dragBoundFunc 回调参数

**❌ 之前的不准确描述：**
> `pos.x/pos.y` 是屏幕像素坐标

**✅ 准确描述：**
> `pos.x/pos.y` 是 **绝对坐标**（Absolute Position），即元素在 Stage 坐标系中的视觉位置，已包含所有父级的 scale 变换。
>
> 与 `group.absolutePosition()` 的返回值处于同一坐标系。

**代码中的正确用法：**
```typescript
dragBoundFunc: (pos) => {
  // pos = 绝对坐标，已包含 Stage.scale
  // maxXPosition 也必须计算为绝对坐标空间的值
  const maxXPosition = (pageWidth - fieldWidth) * scale;
  
  const newX = Math.max(0, Math.min(maxXPosition, pos.x));
  const newY = Math.max(0, Math.min(maxYPosition, pos.y));
  
  return { x: newX, y: newY };  // Konva 自动 / scale 后赋值给 group.x()
}
```

### 4.2 absolutePosition() vs getClientRect()

| 方法 | 返回值 | 用途 |
|-----|--------|------|
| `group.absolutePosition()` | `{ x, y }` | 元素锚点的绝对坐标 |
| `group.getClientRect()` | `{ x, y, width, height }` | 元素视觉边界框的坐标和尺寸 |

**关系说明：**
- 对于简单矩形且无旋转，`getClientRect().x ≈ absolutePosition().x`
- 对于复杂 Group 或有旋转变换，两者可能不同
- **重要：** `dragBoundFunc` 的 `pos` 与 `absolutePosition` 一致，不是 `getClientRect().x`

### 4.3 group.x() vs absolutePosition().x

| 属性 | 坐标系 | 是否受 scale 影响 | 存储方式 |
|-----|--------|----------------|---------|
| `group.x()` | 局部逻辑坐标 | ❌ 不受影响 | 直接存储 |
| `group.absolutePosition().x` | 全局绝对坐标 | ✅ 受影响 | 动态计算 |

**公式：**
```typescript
group.absolutePosition().x = group.x() * group.getAbsoluteScale().x
```

在本项目中，因为 scale 只在 Stage 层设置，且 Group 自身 scale = 1，所以简化为：
```typescript
group.absolutePosition().x = group.x() * scale
```

---

## 五、现有文档术语修正清单

### 5.1 需修正的术语

| 原表述 | 修正后表述 | 影响文档 |
|-------|-----------|---------|
| "屏幕像素坐标" | "绝对坐标" 或 "显示坐标" | field-position-mapping.md |
| "Konva Stage 内部坐标" | "Konva 逻辑坐标" | field-position-mapping.md |
| "边界值用 pageWidth * scale 计算" | "边界值需转换到绝对坐标空间" | drag-bound-coordinate-check.md |

### 5.2 修正后的坐标系统层级表

```
第 1 层：浏览器屏幕坐标
  ├─ clientX/Y: 浏览器视口左上角
  └─ pageX/Y:   文档左上角（含滚动）

           ↓ 减去页面元素边界

第 2 层：页面相对 DOM 坐标
  └─ 鼠标相对于 PDF 页面 DOM 元素的位置

           ↓ Konva Stage 坐标映射

第 3 层：Konva 绝对坐标（显示坐标）
  ├─ dragBoundFunc pos.x / pos.y
  ├─ group.absolutePosition()
  └─ getClientRect() 的 x/y 值

           ↓ / scale 逆变换（Konva 内部）

第 4 层：Konva 逻辑坐标
  └─ group.x() / group.y() 存储的值

           ↓ 百分比转换

第 5 层：百分比坐标（数据库存储）
  └─ positionX, positionY, width, height
```

---

## 六、边界公式的数学证明

### 6.1 证明：maxXPosition 为什么要 * scale

**待证公式：**
```
maxXPosition = (pageWidth - fieldWidth) * scale
```

**证明过程：**

1. **逻辑坐标空间的约束：**
   ```
   group.x() + fieldWidth ≤ pageWidth
   group.x() ≤ pageWidth - fieldWidth
   ```

2. **dragBoundFunc 中 pos.x 与 group.x() 的关系：**
   ```
   group.x() = pos.x / scale  // Konva 内部转换
   ```

3. **代入约束条件：**
   ```
   pos.x / scale ≤ pageWidth - fieldWidth
   pos.x ≤ (pageWidth - fieldWidth) * scale
   ```

4. **因此边界值必须是：**
   ```
   maxXPosition = (pageWidth - fieldWidth) * scale
   ```

**证毕。** ✅

### 6.2 证明：返回值不需要 / scale

**待证：** dragBoundFunc 返回值应该直接使用绝对坐标，不需要 / scale。

**证明过程：**

1. 假设我们错误地在返回前 / scale：
   ```typescript
   dragBoundFunc: (pos) => {
     return { x: pos.x / scale };  // 错误写法
   }
   ```

2. Konva 内部还会再执行一次 / scale：
   ```typescript
   group.x() = returnValue.x / scale
             = (pos.x / scale) / scale
             = pos.x / scale²  // 缩放过度了！
   ```

3. 正确写法是直接返回绝对坐标：
   ```typescript
   return { x: pos.x };  // 正确写法
   ```

4. Konva 内部转换后得到正确的逻辑坐标：
   ```typescript
   group.x() = pos.x / scale  // 正确
   ```

**证毕。** ✅

---

## 七、关键发现总结

### 7.1 Konva 拖拽的设计意图

Konva 选择在 `dragBoundFunc` 中传递绝对坐标（已缩放）而非逻辑坐标，原因是：

1. **边界判断更直观**：直接与 Stage 的显示宽度 `stage.width()` 比较
2. **视觉一致性**：拖拽回调的值与用户实际看到的视觉位置一致
3. **嵌套缩放兼容**：支持父容器有不同 scale 的复杂场景

### 7.2 本项目坐标系统的不变量

在任何缩放比例下，以下关系始终成立：

```typescript
// 1. 拖拽目标 → 百分比
positionX = (pos.x / scale / pageWidth) * 100

// 2. 百分比 → 渲染显示位置
displayX = pageWidth * (positionX / 100) * scale

// 3. 闭环验证：displayX 应该等于最初的 pos.x
displayX = pos.x  // 总是成立 ✓
```

### 7.3 代码实现正确性确认

✅ **当前实现完全正确**，边界公式 `(pageWidth - fieldWidth) * scale` 与坐标语义一致。

---

## 八、后续建议

### 8.1 代码注释建议

在 `upsertFieldGroup` 处添加标准化注释：

```typescript
// pos.x/y 是绝对坐标（已包含 Stage.scale）
// 参考 coordinate-semantic-consistency.md
const maxXPosition = (pageWidth - fieldWidth) * scale;
const maxYPosition = (pageHeight - fieldHeight) * scale;

dragBoundFunc: (pos) => {
  // 边界值使用绝对坐标空间，与 pos 同坐标系
  const newX = Math.max(0, Math.min(maxXPosition, pos.x));
  const newY = Math.max(0, Math.min(maxYPosition, pos.y));
  return { x: newX, y: newY };  // Konva 自动逆变换为逻辑坐标
},
```

### 8.2 文档交叉引用

建议在各文档中添加交叉引用：
- `field-position-mapping.md` → 引用本报告第 3、4 节
- `drag-bound-coordinate-check.md` → 引用本报告第 6 节的数学证明

---

**报告版本：** v1.0
**最后更新：** 2026-05-15
**验证状态：** ✅ 正向 & 逆向算例验证通过
