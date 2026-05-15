# 拖拽边界回调坐标语义复核报告

## 核心结论

**dragBoundFunc 回调参数 `pos.x / pos.y` 的坐标语义**：
- ✅ 不是 Konva 内部逻辑坐标
- ✅ 不是屏幕像素坐标
- ✅ 是 **Stage 内部坐标 × Stage.scale 后的结果**（即"显示坐标"）
- ✅ 与 `getClientRect()` 返回值同坐标系

这就是为什么边界公式需要 `* scale`。

---

## 一、坐标语义详解

### 1.1 Konva Stage 缩放模型

```
┌───────────────────────────────────────────────────────┐
│  Stage 配置                                           │
├───────────────────────────────────────────────────────┤
│                                                       │
│  Stage.width  = pageWidth * scale     ← 显示宽度    │
│  Stage.height = pageHeight * scale    ← 显示高度    │
│  Stage.scale  = { x: scale, y: scale } ← 缩放因子  │
│                                                       │
└───────────────────────────────────────────────────────┘
```

### 1.2 坐标系统对比

| 坐标类型 | 获取方式 | 值的含义 | 与 scale 的关系 |
|---------|---------|---------|----------------|
| 元素逻辑坐标 | `group.x()` | 元素在 Stage 内部的逻辑位置 | **不包含** scale |
| dragBoundFunc 入参 | `pos.x / pos.y` | 拖拽时的目标位置 | **已包含** scale |
| getClientRect 返回 | `rect.x / rect.y` | 元素边界框 | **已包含** scale |

### 1.3 关键公式推导

```
pos.x (拖拽回调坐标) = group.x() (逻辑坐标) × Stage.scale.x

因此：
group.x() = pos.x / Stage.scale.x
```

---

## 二、数值验证示例

### 2.1 测试场景设定

| 参数 | 数值 | 说明 |
|-----|------|------|
| pageWidth | 800px | PDF 页面宽度（逻辑像素） |
| pageHeight | 1200px | PDF 页面高度（逻辑像素） |
| scale | 0.75 | 显示缩放因子（页面缩放到 75%） |
| fieldWidth | 100px | 字段宽度（逻辑像素） |
| fieldHeight | 50px | 字段高度（逻辑像素） |

### 2.2 计算过程

#### 步骤1：Stage 显示尺寸
```typescript
Stage.width  = pageWidth  * scale = 800 * 0.75 = 600px
Stage.height = pageHeight * scale = 1200 * 0.75 = 900px
```

#### 步骤2：字段最大逻辑位置（不超出右边界）
```typescript
// 字段右下角不能超过页面右边界
// 逻辑坐标下的最大 x 值
maxLogicalX = pageWidth - fieldWidth = 800 - 100 = 700px
```

#### 步骤3：拖拽边界需要的最大值（已缩放）
```typescript
// 因为 dragBoundFunc 的 pos.x 是已缩放的值
// 所以边界也需要缩放
maxXPosition = maxLogicalX * scale = 700 * 0.75 = 525px

// 代码中的写法：
maxXPosition = (pageWidth - fieldWidth) * scale
```

### 2.3 拖拽边界校验

| 拖拽目标位置 pos.x | 边界判断 | 最终返回值 | 对应逻辑坐标 |
|-------------------|---------|----------|------------|
| -50 | < 0 | 0 | 0 / 0.75 = 0px |
| 100 | 在范围内 | 100 | 100 / 0.75 ≈ 133.33px |
| 525 | 在范围内 | 525 | 525 / 0.75 = 700px |
| 600 | > 525 | 525 | 525 / 0.75 = 700px |

### 2.4 边界验证

**当 pos.x = 525（最大值）时：**
```
group.x() = 525 / 0.75 = 700px （逻辑坐标）
字段右下角 = group.x() + fieldWidth = 700 + 100 = 800px 
刚好等于 pageWidth，不超出边界 ✓
```

**当 pos.x = 600（超过边界）时：**
```
如果不做限制：group.x() = 600 / 0.75 = 800px
字段右下角 = 800 + 100 = 900px > pageWidth(800px)，超出页面 ❌

经过 Math.min(525, 600) 限制后：
group.x() = 525 / 0.75 = 700px
字段右下角 = 700 + 100 = 800px，刚好在边界 ✓
```

---

## 三、代码实现分析

### 3.1 源码位置

**文件**: `packages/lib/universal/field-renderer/field-generic-items.ts`
**函数**: `upsertFieldGroup`

```typescript
const maxXPosition = (pageWidth - fieldWidth) * scale;
const maxYPosition = (pageHeight - fieldHeight) * scale;

dragBoundFunc: (pos) => {
  const newX = Math.max(0, Math.min(maxXPosition, pos.x));
  const newY = Math.max(0, Math.min(maxYPosition, pos.y));
  return { x: newX, y: newY };
},
```

### 3.2 正确性验证

| 步骤 | 代码行为 | 正确性 |
|-----|---------|-------|
| 1 | `pageWidth - fieldWidth` 计算最大逻辑位置 | ✅ 正确 |
| 2 | `* scale` 将最大逻辑位置转换为拖拽回调的坐标空间 | ✅ 正确 |
| 3 | `Math.max(0, Math.min(maxXPosition, pos.x))` 限制范围 | ✅ 正确 |

### 3.3 常见误区对比

❌ **错误理解一**：认为 `pos.x` 是逻辑坐标
```typescript
// 错误写法 - 边界会过大
const maxXPosition = pageWidth - fieldWidth;  // ❌ 少了 * scale
dragBoundFunc: (pos) => {
  return { x: Math.min(maxXPosition, pos.x) };  // ❌ 边界大了 scale 倍
}
```

❌ **错误理解二**：认为返回值需要除以 scale
```typescript
// 错误写法 - 返回值会过小
dragBoundFunc: (pos) => {
  return { x: Math.min(maxXPosition, pos.x) / scale };  // ❌ 不需要 / scale
}
```

✅ **正确理解**：Konva 会自动处理返回值到逻辑坐标的转换
```typescript
// Konva 内部隐式转换（开发者无需关心）
group.x() = dragBoundFuncReturnValue.x / Stage.scale.x;
```

---

## 四、端到端坐标流动验证

### 4.1 完整链路

```
1. 用户鼠标拖动到屏幕位置
        ↓
2. Konva 计算目标位置 pos = { x: 525, y: ... } （已缩放）
        ↓
3. dragBoundFunc 限制边界
   newX = Math.min(525, 525) = 525
        ↓
4. Konva 内部转换为逻辑坐标
   group.x() = 525 / 0.75 = 700 （逻辑像素）
        ↓
5. 保存时 convertPixelToPercentage 转换为百分比
   positionX = (700 / 800) * 100 = 87.5%
        ↓
6. 数据库存储 87.5%
        ↓
7. 下次渲染 calculateFieldPosition 还原
   fieldX = 800 * (87.5 / 100) = 700px （逻辑像素）
        ↓
8. Konva 渲染时自动应用 scale
   屏幕位置 = 700 * 0.75 = 525px
        ↓
9. 字段回到正确位置 ✓
```

### 4.2 闭环验证

| 环节 | 计算 | 结果 |
|-----|------|------|
| 拖拽目标 | pos.x = 525 | 525px（显示坐标） |
| 边界限制 | Math.min(525, 525) | 525px |
| Konva 内部转换 | 525 / 0.75 | 700px（逻辑） |
| 存储转换 | (700 / 800) * 100 | 87.5% |
| 渲染还原 | 800 * (87.5 / 100) | 700px（逻辑） |
| 最终显示 | 700 * 0.75 | 525px ✓ |

---

## 五、结论与建议

### 5.1 结论

1. **dragBoundFunc 回调的 `pos.x / pos.y` 是已应用 Stage.scale 的显示坐标**，不是逻辑坐标
2. **边界值 `maxXPosition / maxYPosition` 必须 `* scale`**，才能与 `pos` 处于同一坐标系
3. **返回值不需要 `/ scale`**，Konva 会自动完成显示坐标到逻辑坐标的逆转换

### 5.2 当前代码正确性

✅ **当前实现是正确的**，边界公式 `(pageWidth - fieldWidth) * scale` 是准确的。

### 5.3 术语澄清建议

建议在代码注释中使用以下术语，避免混淆：

| 术语 | 含义 |
|-----|------|
| 逻辑坐标 / 未缩放坐标 | Konva 元素的 `x/y` 属性值（未应用 scale） |
| 显示坐标 / 缩放坐标 | `dragBoundFunc` 入参、`getClientRect()` 返回值 |

```typescript
// 建议的代码注释
const maxXPosition = (pageWidth - fieldWidth) * scale;  // 显示坐标下的最大边界
const maxYPosition = (pageHeight - fieldHeight) * scale;

dragBoundFunc: (pos) => {
  // pos.x/y 是显示坐标（已应用 Stage.scale）
  // 所以边界值也需要用显示坐标进行比较
  const newX = Math.max(0, Math.min(maxXPosition, pos.x));
  const newY = Math.max(0, Math.min(maxYPosition, pos.y));
  return { x: newX, y: newY };  // Konva 会自动转换回逻辑坐标
},
```
