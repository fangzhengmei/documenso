# 坐标语义一致性最终报告

## 一、文档体系概览

本次坐标语义校正工作涉及三份文档，形成完整的文档体系：

| 文档 | 定位 | 核心内容 | 状态 |
|-----|-----|---------|------|
| **field-position-mapping.md** | 主文档 | 端到端完整流程说明、详细场景、术语定义 | ✅ 已更新 |
| **coordinate-semantic-consistency.md** | 术语基准 | 统一术语定义、数学证明、双向算例验证 | ✅ 已新增 |
| **drag-bound-coordinate-check.md** | 专项复核 | dragBoundFunc 语义深度解析、边界校验 | ✅ 已新增 |

---

## 二、核心术语统一

### 2.1 术语体系最终定义（三份文档完全一致）

| 术语 | 精确定义 | 相关 API | 文档引用 |
|-----|---------|---------|---------|
| **逻辑坐标** | Konva 元素存储的原始坐标值，不受 Stage.scale 影响 | `group.x()`, `group.y()` | 主文档 §2 |
| **绝对坐标**（显示坐标） | 相对于 Stage 原点的视觉坐标，已包含所有父级 scale 变换 | `pos.x/y` (dragBoundFunc), `group.absolutePosition()`, `getClientRect()` | 基准 §1.1 |
| **Stage.scale** | 全局缩放因子，仅影响渲染输出，不改变元素存储的逻辑坐标 | `Stage.scaleX`, `Stage.scaleY` | 主文档 §2 |
| **百分比坐标** | 以页面宽高百分比表示的坐标，数据库最终存储格式 | `positionX`, `positionY`, `width`, `height` | 主文档 §2 |

### 2.2 核心变换公式（三份文档完全一致）

```typescript
// 绝对坐标 = 逻辑坐标 × Stage.scale
绝对坐标 = 逻辑坐标 × Stage.scale

// dragBoundFunc 与 Konva API 的一致性
group.absolutePosition().x = group.x() × scale
pos.x = group.x() × scale

// 百分比坐标变换
百分比坐标 = (逻辑坐标 / pageWidth) × 100
逻辑坐标 = pageWidth × (百分比坐标 / 100)
```

> 数学证明：coordinate-semantic-consistency.md §6.1

---

## 三、关键术语校正清单

### 3.1 field-position-mapping.md 校正项

| 原表述（不准确） | 修正后表述（精确） | 影响行数 |
|----------------|-----------------|---------|
| "屏幕像素坐标" | "绝对坐标" | 多处 |
| "已缩放的屏幕像素" | "绝对坐标空间的宽度/高度/X/Y" | §3.3 场景三 |
| "屏幕像素尺寸" | "绝对坐标空间" | §5 scaledViewport |
| "Konva 内部逻辑坐标 → 实际渲染到屏幕" | 明确区分 `group.x()`（逻辑）vs `absolutePosition()`（绝对）vs 视觉位置 | §4.3 |
| "边界值用 pageWidth * scale 计算" | "边界值需转换到绝对坐标空间，与 pos.x 同坐标系" | FAQ §Q3 |

### 3.2 新增内容

1. **坐标术语统一定义章节**（主文档 §2）
   - 4个核心术语的精确定义
   - 4个核心变换公式
   - 交叉引用链接

2. **端到端闭环校验结论**（主文档 §6）
   - 坐标变换闭环验证（6步完整链路）
   - 边界约束闭环验证（越界拖拽场景）
   - 文档一致性交叉校验

---

## 四、数值验证体系

### 4.1 正向验证：拖拽 → 入库

**参数**：pageWidth = 800px, fieldWidth = 100px, scale = 0.75, targetX = 300

| 环节 | 计算过程 | 结果 | 一致性 |
|-----|---------|------|-------|
| 1. 拖拽目标 | pos.x = 300 | 300 | ✓ |
| 2. 边界限制 | `Math.min(525, 300)` | 300 | ✓ |
| 3. Konva 逆变换 | `group.x() = 300 / 0.75` | 400 | ✓ |
| 4. 百分比转换 | `positionX = (400 / 800) * 100` | 50% | ✓ |

> 完整算例：coordinate-semantic-consistency.md §2

### 4.2 逆向验证：读取 → 渲染

**输入**：数据库 positionX = 50%

| 环节 | 计算过程 | 结果 | 一致性 |
|-----|---------|------|-------|
| 1. 百分比 → 逻辑坐标 | `fieldX = 800 * (50 / 100)` | 400 | ✓ |
| 2. 绝对坐标计算 | `displayX = 400 * 0.75` | 300 | ✓ |

**闭环结论**：最初拖拽目标 300 = 最终渲染显示位置 300 ✓

> 完整算例：coordinate-semantic-consistency.md §3

### 4.3 越界边界验证

**场景**：pos.x = 600（超出边界 525）

| 环节 | 计算过程 | 结果 | 一致性 |
|-----|---------|------|-------|
| 1. 边界限制 | `Math.min(525, 600)` | 525 | ✓ |
| 2. 逻辑坐标 | `group.x() = 525 / 0.75` | 700 | ✓ |
| 3. 字段右下角 | `700 + 100` | 800 | 刚好不超出 ✓ |

> 完整验证：drag-bound-coordinate-check.md §2.2

---

## 五、关键 API 语义澄清

### 5.1 dragBoundFunc 参数语义

**❌ 误解**：pos.x/y 是屏幕像素坐标

**✅ 正确语义**：pos.x/y 是**绝对坐标**（Absolute Position），即：
- 已包含所有父级 scale 变换
- 与 `group.absolutePosition()` 返回值同坐标系
- 与 `getClientRect()` 返回值同坐标系

> 详细分析：coordinate-semantic-consistency.md §4.1

### 5.2 边界公式推导

**证明**：为什么 maxXPosition 必须 `* scale`

```
1. 逻辑坐标约束：group.x() + fieldWidth ≤ pageWidth
                group.x() ≤ pageWidth - fieldWidth

2. pos.x 与 group.x() 的关系：group.x() = pos.x / scale

3. 代入约束：pos.x / scale ≤ pageWidth - fieldWidth
              pos.x ≤ (pageWidth - fieldWidth) * scale

4. 结论：maxXPosition = (pageWidth - fieldWidth) * scale
```

> 完整证明：coordinate-semantic-consistency.md §6.1

### 5.3 返回值语义

**❌ 误解**：返回值需要 `/ scale` 转换为逻辑坐标

**✅ 正确语义**：dragBoundFunc 返回值使用绝对坐标，Konva 内部自动执行逆变换

```typescript
// ✅ 正确写法
return { x: pos.x };

// Konva 内部隐式执行（开发者不需要关心）
group.x() = returnValue.x / scale;
```

> 数学证明：coordinate-semantic-consistency.md §6.2

---

## 六、文档交叉引用体系

| 源文档 | 引用目标 | 引用章节 | 引用内容 |
|-------|---------|---------|---------|
| field-position-mapping.md | coordinate-semantic-consistency.md | §2, §Q3 | 术语定义、边界公式证明 |
| field-position-mapping.md | drag-bound-coordinate-check.md | §4.4 | 边界详细验证 |
| coordinate-semantic-consistency.md | field-position-mapping.md | §8 | 关键文件索引 |
| coordinate-semantic-consistency.md | drag-bound-coordinate-check.md | §7.3 | 边界实现验证 |
| drag-bound-coordinate-check.md | coordinate-semantic-consistency.md | §5.3 | 数学证明 |

---

## 七、代码实现正确性确认

### 7.1 upsertFieldGroup 边界代码

```typescript
const maxXPosition = (pageWidth - fieldWidth) * scale;
const maxYPosition = (pageHeight - fieldHeight) * scale;

dragBoundFunc: (pos) => {
  const newX = Math.max(0, Math.min(maxXPosition, pos.x));
  const newY = Math.max(0, Math.min(maxYPosition, pos.y));
  return { x: newX, y: newY };
}
```

**验证结论**：✅ 代码实现完全正确，与文档推导一致

### 7.2 坐标转换函数

```typescript
// 像素 → 百分比
export const convertPixelToPercentage = (...) => {
  const fieldX = (positionX / pageWidth) * 100;
  // ...
};

// 百分比 → 像素
export const calculateFieldPosition = (...) => {
  const fieldX = pageWidth * (Number(field.positionX) / 100);
  // ...
};
```

**验证结论**：✅ 双向转换互为逆运算，闭环一致

---

## 八、最终结论

### 8.1 一致性状态

| 校验维度 | 状态 | 说明 |
|---------|------|------|
| 术语一致性 | ✅ 完全一致 | 三份文档使用相同的4个核心术语 |
| 公式一致性 | ✅ 完全一致 | 核心变换公式在所有文档中完全相同 |
| 算例一致性 | ✅ 完全一致 | 800/100/0.75 参数集的计算结果一致 |
| 代码一致性 | ✅ 完全一致 | 文档推导与实际代码实现匹配 |
| 闭环验证 | ✅ 完全通过 | 拖拽→入库→渲染的端到端坐标一致 |

### 8.2 核心成果

1. **消除了关键歧义**：明确了 pos.x/y 的"绝对坐标"语义，纠正了"屏幕像素"的不准确表述
2. **建立了统一术语体系**：4个核心术语的精确定义，为后续开发提供基准
3. **构建了完整验证体系**：正向算例、逆向算例、边界算例三重验证
4. **数学证明支撑**：边界公式的严谨推导，消除经验性判断
5. **文档交叉引用**：三份文档相互印证，形成完整知识体系

### 8.3 后续建议

1. **代码注释更新**：建议在 `upsertFieldGroup` 处添加标准化注释，引用 coordinate-semantic-consistency.md §4.1
2. **新开发参考**：涉及坐标变换的新功能开发，以 coordinate-semantic-consistency.md 为术语基准
3. **文档维护**：坐标相关变更应同步更新三份文档，保持一致性

---

**报告版本**：v1.0 最终版
**编制日期**：2026-05-15
**校验人**：AI Assistant
**最终状态**：✅ 所有文档一致，所有验证通过
