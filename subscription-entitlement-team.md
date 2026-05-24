# 订阅信息驱动团队功能开关与额度判断协作面分析

## 一、核心数据模型与概念

### 1.1 三层权限模型

系统采用**三层模型**来管理订阅与权限的关系：

| 模型 | 定义 | 存储位置 | 作用 |
|------|------|----------|------|
| `SubscriptionClaim` | 订阅档位模板 | 数据库表 | 定义各档位的标准权限和额度，如Free/Team/Enterprise等 |
| `OrganisationClaim` | 组织实际权限声明 | 数据库表 | 每个组织实际拥有的权限，从SubscriptionClaim派生，可独立调整 |
| `Subscription` | 订阅状态记录 | 数据库表 | 记录Stripe订阅状态（ACTIVE/INACTIVE/PAST_DUE） |

#### 数据结构定义

**SubscriptionClaim 表** (`packages/prisma/schema.prisma:260-273`)
```prisma
model SubscriptionClaim {
  id        String   @id
  name      String           // 档位名称：Free/Individual/Team/Enterprise等
  locked    Boolean          // 是否锁定（系统内置档位不可修改）
  teamCount         Int      // 团队数量上限（0表示无限制）
  memberCount       Int      // 成员数量上限（0表示无限制）
  envelopeItemCount Int      // 单个信封可包含的文档数量上限
  flags             Json     // 功能开关集合
}
```

**OrganisationClaim 表** (`packages/prisma/schema.prisma:276-289`)
```prisma
model OrganisationClaim {
  id        String   @id
  originalSubscriptionClaimId String?  // 来源档位ID
  teamCount         Int
  memberCount       Int
  envelopeItemCount Int
  flags             Json
  organisation      Organisation?
}
```

### 1.2 功能开关（Flags）定义与真实数量

在 `packages/lib/types/subscription.ts:9-38` 中定义了**13个**功能开关：

```typescript
export const ZClaimFlagsSchema = z.object({
  allowCustomBranding: z.boolean().optional(),      // [1] 自定义品牌（元标记）
  hidePoweredBy: z.boolean().optional(),            // [2] 隐藏"Powered by"标识
  unlimitedDocuments: z.boolean().optional(),       // [3] 无限文档
  emailDomains: z.boolean().optional(),             // [4] 邮箱域名管理（企业版）
  embedAuthoring: z.boolean().optional(),           // [5] 嵌入编辑功能访问
  embedAuthoringWhiteLabel: z.boolean().optional(), // [6] 嵌入编辑白标
  embedSigning: z.boolean().optional(),             // [7] 嵌入签署功能访问
  embedSigningWhiteLabel: z.boolean().optional(),   // [8] 嵌入签署白标
  cfr21: z.boolean().optional(),                    // [9] 21 CFR合规
  hipaa: z.boolean().optional(),                    // [10] HIPAA合规（预留）
  authenticationPortal: z.boolean().optional(),     // [11] 认证门户（企业版）
  allowLegacyEnvelopes: z.boolean().optional(),     // [12] 允许旧版信封
  signingReminders: z.boolean().optional(),         // [13] 签署提醒
});
```

> **重要纠正**：
> 1. 共 **13个** 字段，而非12个（之前计数遗漏了 `allowLegacyEnvelopes`）
> 2. `allowCustomBranding` 是一个元标记flag，实际品牌控制由 `embedSigningWhiteLabel` 和 `embedAuthoringWhiteLabel` 执行

#### 1.2.1 字段映射一致性说明

`SUBSCRIPTION_CLAIM_FEATURE_FLAGS` 映射表同样包含完整的13个字段（`packages/lib/types/subscription.ts:51-109`）。

⚠️ **已知不一致**：`backport-subscription-claims` Job 的schema仅定义了10个字段，缺少以下3个（代码注释已标记为TODO）：
- `emailDomains`
- `authenticationPortal`
- `allowLegacyEnvelopes`

这意味着通过Job回溯更新档位模板时，上述3个flag不会被批量同步到组织。

### 1.3 内置档位定义与真实flags配置

系统在 `packages/lib/types/subscription.ts:126-218` 中预定义了6种内置档位。**以下为真实配置，纠正之前的偏差**：

```typescript
export enum INTERNAL_CLAIM_ID {
  FREE = 'free',
  INDIVIDUAL = 'individual',
  TEAM = 'team',
  EARLY_ADOPTER = 'earlyAdopter',
  PLATFORM = 'platform',
  ENTERPRISE = 'enterprise',
}
```

**各档位配额与功能对比（真实配置）**：

| 档位 | teamCount | memberCount | envelopeItemCount | 真实flags配置 |
|------|-----------|-------------|-------------------|--------------|
| **FREE** | 1 | 1 | 5 | `{}` |
| **INDIVIDUAL** | 1 | 1 | 5 | `{ unlimitedDocuments: true, signingReminders: true }` |
| **TEAM** | 1 | 5 | 5 | `{ unlimitedDocuments: true, allowCustomBranding: true, embedSigning: true, signingReminders: true }` |
| **PLATFORM** | 1 | 0（无限） | 10 | `{ unlimitedDocuments: true, allowCustomBranding: true, hidePoweredBy: true, emailDomains: false, embedAuthoring: false, embedAuthoringWhiteLabel: true, embedSigning: false, embedSigningWhiteLabel: true, signingReminders: true }` |
| **ENTERPRISE** | 0（无限） | 0（无限） | 10 | `{ unlimitedDocuments: true, allowCustomBranding: true, hidePoweredBy: true, emailDomains: true, embedAuthoring: true, embedAuthoringWhiteLabel: true, embedSigning: true, embedSigningWhiteLabel: true, cfr21: true, authenticationPortal: true, signingReminders: true }` |
| **EARLY_ADOPTER** | 0（无限） | 0（无限） | 5 | `{ unlimitedDocuments: true, allowCustomBranding: true, hidePoweredBy: true, embedSigning: true, embedSigningWhiteLabel: true, signingReminders: true }` |

> **关键纠正**：
> 1. PLATFORM档位显式设置了 `emailDomains: false`, `embedAuthoring: false`, `embedSigning: false`，但白标功能为true
> 2. ENTERPRISE档位没有 `hipaa: true`（仅在schema中定义，未实际启用）
> 3. PLATFORM档位虽然 `embedAuthoring: false`，但有 `embedAuthoringWhiteLabel: true`，这是一个特殊配置

---

## 二、订阅变更驱动权限更新的完整数据流

### 2.1 完整数据流全景图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Stripe Webhook 入口                            │
└─────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│  on-subscription-created.ts                                          │
│  on-subscription-updated.ts                                          │
│  on-subscription-deleted.ts                                          │
└─────────────────────────────────────────────────────────────────────┘
                               ↓ 提取 price.metadata.claimId
┌─────────────────────────────────────────────────────────────────────┐
│  extractStripeClaimId(price) → 查找 SubscriptionClaim               │
│  优先级：Price.metadata → Product.metadata                            │
└─────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│  createOrganisationClaimUpsertData() 转换格式                        │
└─────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│  OrganisationClaim 操作（按场景分支）                                 │
│  ├─ 新建：createOrganisation() 创建新的 OrganisationClaim           │
│  ├─ 更新：organisationClaim.update() 覆盖现有                       │
│  ├─ 删除：INDIVIDUAL → 重置为FREE；其他 → 标记INACTIVE              │
│  ├─ 回溯：backport-subscription-claims Job 批量更新                  │
│  └─ 转移：swap-organisation-subscription 组织间转移                  │
└─────────────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│  后续所有功能检查点读取 organisationClaim.flags / 配额字段判断        │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 场景1：新订阅创建 (on-subscription-created.ts)

**处理流程** (`packages/ee/server-only/stripe/webhook/on-subscription-created.ts`):

1. 提取 `customerId` 和 `subscriptionItem`
2. 调用 `extractStripeClaim(price)` 从数据库查找对应的 `SubscriptionClaim`
3. 根据 `subscription.metadata.organisationCreateData` 分两种情况：
   - **包含createData**：调用 `createOrganisation()` 创建新组织及其claim
   - **不包含createData**：调用 `handleOrganisationUpdate()` 更新现有组织的claim
4. Upsert `Subscription` 记录，设置状态（ACTIVE/INACTIVE/PAST_DUE）

**组织创建链路** (`packages/lib/server-only/organisation/create-organisation.ts`):

```typescript
const organisation = await prisma.$transaction(async (tx) => {
  // 1. 创建 OrganisationClaim
  const organisationClaim = await tx.organisationClaim.create({
    data: {
      id: generateDatabaseId('org_claim'),
      originalSubscriptionClaimId: claim.id,
      ...createOrganisationClaimUpsertData(claim),
    },
  });

  // 2. 创建 Organisation 并关联 claim
  const organisation = await tx.organisation.create({
    data: {
      // ...
      organisationClaimId: organisationClaim.id,
      // ...
    },
  });

  // 3. 自动创建第一个团队
  await createTeam({
    userId,
    teamName: 'Personal Team',
    teamUrl: prefixedId('personal'),
    organisationId: organisation.id,
    inheritMembers: true,
  });
});
```

### 2.3 场景2：订阅更新 (on-subscription-updated.ts)

**处理流程** (`packages/ee/server-only/stripe/webhook/on-subscription-updated.ts`):

1. 通过 `customerId` 查找对应组织
2. 对比 `previousAttributes` 和当前subscription的price变化
3. 若 `claimId` 发生变化（升级/降级）：
   ```typescript
   if (newClaimFound) {
     await tx.organisationClaim.update({
       where: { id: organisation.organisationClaim.id },
       data: {
         originalSubscriptionClaimId: updatedSubscriptionClaim.id,
         ...createOrganisationClaimUpsertData(updatedSubscriptionClaim),
       },
     });
   }
   ```
4. 更新 `Subscription` 状态和 `periodEnd` 时间
5. 若组织类型需要迁移（如从PERSONAL转为ORGANISATION），同步更新

### 2.4 场景3：订阅删除 (on-subscription-deleted.ts)

**处理流程** (`packages/ee/server-only/stripe/webhook/on-subscription-deleted.ts`):

1. 通过 `planId` 查找现有subscription记录
2. 提取被删除订阅的 `claimId`
3. **分档位处理**：
   - **INDIVIDUAL档位**：完整重置为FREE档位
     ```typescript
     if (subscriptionClaimId === INTERNAL_CLAIM_ID.INDIVIDUAL) {
       await prisma.$transaction(async (tx) => {
         await tx.subscription.delete({ where: { id: existingSubscription.id } });
         await tx.organisationClaim.update({
           where: { id: existingSubscription.organisation.organisationClaim.id },
           data: {
             originalSubscriptionClaimId: INTERNAL_CLAIM_ID.FREE,
             ...createOrganisationClaimUpsertData(internalClaims[INTERNAL_CLAIM_ID.FREE]),
           },
         });
       });
     }
     ```
   - **其他档位**：仅标记为INACTIVE，保留claim配置
     ```typescript
     await prisma.subscription.update({
       where: { id: existingSubscription.id },
       data: { status: SubscriptionStatus.INACTIVE },
     });
     ```

### 2.5 场景4：档位回溯更新 (backport-subscription-claims Job)

当管理员修改了 `SubscriptionClaim` 模板的flags后，需要批量更新所有使用该档位的组织：

**Job定义** (`packages/lib/jobs/definitions/internal/backport-subscription-claims.ts`):

```typescript
export const run = async ({ payload, io }) => {
  const { subscriptionClaimId, flags } = payload;

  await io.runTask('backport-claims', async () => {
    const newFlagsJson = JSON.stringify(flags);

    // 使用原生SQL批量合并flags（jsonb || 操作）
    await prisma.$executeRaw`
      UPDATE "OrganisationClaim"
      SET "flags" = "flags" || ${newFlagsJson}::jsonb
      WHERE "originalSubscriptionClaimId" = ${subscriptionClaimId}
    `;
  });
};
```

> **注意**：此Job仅合并指定的true flags，不会移除已有的flag。

### 2.6 场景5：组织间订阅转移 (swap-organisation-subscription)

管理员可将一个组织的订阅转移到另一个组织（必须同一用户所有）：

**处理流程** (`packages/trpc/server/admin-router/swap-organisation-subscription.ts`):

```typescript
await prisma.$transaction(async (tx) => {
  // 1. 删除目标组织的过期INACTIVE订阅
  if (targetOrg.subscription) {
    await tx.subscription.delete({ where: { id: targetOrg.subscription.id } });
  }

  // 2. 转移customerId
  await tx.organisation.update({
    where: { id: sourceOrganisationId },
    data: { customerId: null },
  });
  await tx.organisation.update({
    where: { id: targetOrganisationId },
    data: { customerId },
  });

  // 3. 转移subscription记录
  await tx.subscription.update({
    where: { id: sourceOrg.subscription!.id },
    data: { organisationId: targetOrganisationId },
  });

  // 4. 复制claim权限到目标组织
  await tx.organisationClaim.update({
    where: { id: targetOrg.organisationClaim.id },
    data: {
      originalSubscriptionClaimId: sourceOrg.organisationClaim.originalSubscriptionClaimId,
      teamCount: sourceOrg.organisationClaim.teamCount,
      memberCount: sourceOrg.organisationClaim.memberCount,
      envelopeItemCount: sourceOrg.organisationClaim.envelopeItemCount,
      flags: sourceOrg.organisationClaim.flags,
    },
  });

  // 5. 源组织重置为FREE
  await tx.organisationClaim.update({
    where: { id: sourceOrg.organisationClaim.id },
    data: {
      originalSubscriptionClaimId: INTERNAL_CLAIM_ID.FREE,
      ...createOrganisationClaimUpsertData(internalClaims[INTERNAL_CLAIM_ID.FREE]),
    },
  });
});
```

### 2.7 关键转换函数

`createOrganisationClaimUpsertData()` (`packages/lib/server-only/organisation/create-organisation.ts:187-201`):

```typescript
export const createOrganisationClaimUpsertData = (subscriptionClaim: InternalClaim) => {
  const data: Omit<Prisma.SubscriptionClaimCreateInput, 'id' | 'createdAt' | 'updatedAt' | 'locked' | 'name'> = {
    flags: { ...subscriptionClaim.flags },
    envelopeItemCount: subscriptionClaim.envelopeItemCount,
    teamCount: subscriptionClaim.teamCount,
    memberCount: subscriptionClaim.memberCount,
  };
  return { ...data };
};
```

---

## 三、品牌功能的真实判断逻辑纠正

### 3.1 品牌功能的分层控制

品牌功能由**三个独立flag**分层控制，而非单一的 `allowCustomBranding`：

| Flag | 控制范围 | 检查位置 |
|------|----------|----------|
| `embedSigningWhiteLabel` | 收件人签署页面的自定义CSS/颜色 | `loadRecipientBrandingByTeamId()` |
| `embedAuthoringWhiteLabel` | 嵌入编辑页面的自定义CSS/颜色 | `embed/v2/authoring/_layout.tsx` |
| `hidePoweredBy` | 所有出口（邮件、证书、页面）的"Powered by"标识 | `getEmailContext()`, `generateCertificatePdf()` |

### 3.2 收件人品牌检查 (loadRecipientBrandingByTeamId)

**真实逻辑** (`packages/lib/server-only/branding/load-recipient-branding.ts:22-55`):

```typescript
export const loadRecipientBrandingByTeamId = async ({ teamId }) => {
  const billingEnabled = IS_BILLING_ENABLED();

  const [settings, claim] = await Promise.all([
    getTeamSettings({ teamId }),
    billingEnabled ? getOrganisationClaimByTeamId({ teamId }).catch(() => null) : null,
  ]);

  // ⚠️ 关键纠正：使用 embedSigningWhiteLabel，而非 allowCustomBranding
  const allowCustomBranding = !billingEnabled || claim?.flags?.embedSigningWhiteLabel === true;
  const hidePoweredBy = !billingEnabled || claim?.flags?.hidePoweredBy === true;

  if (!allowCustomBranding) {
    return {
      allowCustomBranding: false,
      hidePoweredBy,
      colors: null,
      css: null,
    };
  }

  return {
    allowCustomBranding: true,
    hidePoweredBy,
    colors: parsedColors?.success ? parsedColors.data : null,
    css: settings.brandingCss && settings.brandingCss.length > 0 ? settings.brandingCss : null,
  };
};
```

### 3.3 嵌入编辑品牌检查 (embed v2 layout)

**真实逻辑** (`apps/remix/app/routes/embed+/v2+/authoring+/_layout.tsx:66-90`):

```typescript
const allowEmbedAuthoringWhiteLabel = organisationClaim.flags.embedAuthoringWhiteLabel ?? false;

useLayoutEffect(() => {
  // ... 解析hash中的配置
  const { css, cssVars, darkModeDisabled, language } = result.data;

  // ⚠️ 关键纠正：只有白标flag为true时才注入自定义CSS
  if (allowEmbedAuthoringWhiteLabel) {
    injectCss({ css, cssVars });
  }
}, []);
```

### 3.4 邮件品牌检查 (getEmailContext)

**真实逻辑** (`packages/lib/server-only/email/get-email-context.ts:158-168`):

```typescript
return {
  // ...
  branding: organisationGlobalSettingsToBranding(
    organisation.organisationGlobalSettings,
    organisation.id,
    claims.flags.hidePoweredBy ?? false,  // 仅使用 hidePoweredBy
  ),
  // ...
};
```

### 3.5 证书品牌检查 (generateCertificatePdf)

**真实逻辑** (`packages/lib/server-only/pdf/generate-certificate-pdf.ts:141`):

```typescript
const payload = {
  // ...
  hidePoweredBy: organisationClaim.flags.hidePoweredBy ?? false,  // 仅使用 hidePoweredBy
  // ...
};
```

### 3.6 allowCustomBranding 的真实作用

`allowCustomBranding` 更多是一个**元标记**，用于：
1. 管理后台显示"Branding"功能标签
2. 数据库迁移脚本中的历史数据标记
3. 前端UI判断是否显示品牌设置入口

**实际执行控制**始终由各个白标flag完成。

---

## 四、embedAuthoring 的真实判断逻辑纠正

### 4.1 两层检查机制

`embedAuthoring` 功能采用**两层检查**：

| 层级 | Flag | 检查点 | 作用 |
|------|------|--------|------|
| 1 | `embedAuthoring` | `create-embedding-presign-token.ts` | 控制是否允许创建嵌入编辑令牌（功能入口） |
| 2 | `embedAuthoringWhiteLabel` | `embed/v2/authoring/_layout.tsx` | 控制是否允许注入自定义CSS（白标能力） |

### 4.2 第一层：功能入口检查

**位置** (`packages/trpc/server/embedding-router/create-embedding-presign-token.ts:43-52`):

```typescript
if (IS_BILLING_ENABLED()) {
  const organisationClaim = await getOrganisationClaimByTeamId({ teamId: token.teamId });

  if (!organisationClaim.flags.embedAuthoring) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, {
      message: 'Embedded Authoring is not included in your current plan. Please contact support.',
    });
  }
}
```

### 4.3 第二层：白标能力检查

**位置** (`apps/remix/app/routes/embed+/v2+/authoring+/_layout.tsx:66-90`):

```typescript
const allowEmbedAuthoringWhiteLabel = organisationClaim.flags.embedAuthoringWhiteLabel ?? false;

useLayoutEffect(() => {
  // ...
  if (allowEmbedAuthoringWhiteLabel) {
    injectCss({ css, cssVars });  // 仅白标用户可自定义样式
  }
}, []);
```

### 4.4 PLATFORM档位的特殊配置

PLATFORM档位的配置体现了分层设计：
```typescript
{
  embedAuthoring: false,           // 禁用嵌入编辑功能入口
  embedAuthoringWhiteLabel: true,  // 但保留白标能力（给特定客户的特殊配置）
  embedSigning: false,
  embedSigningWhiteLabel: true,
}
```

> 这意味着PLATFORM客户默认不能使用嵌入功能，但可以通过管理员单独开启 `embedAuthoring` flag 来启用，同时自动获得白标能力。

---

### 4.5 补充：allowLegacyEnvelopes 检查点

**真实检查点**（前端UI控制）：`apps/remix/app/components/general/folder/folder-grid.tsx:101`

```typescript
{organisation.organisationClaim.flags.allowLegacyEnvelopes && <DocumentUploadButtonLegacy type={type} />}
```

**作用**：控制是否显示"旧版文档上传"按钮，用于向下兼容旧版信封功能。

### 4.6 13个功能开关实际使用总览

| 序号 | Flag名称 | 实际执行检查点 | 控制场景 |
|------|----------|----------------|----------|
| 1 | `allowCustomBranding` | 管理后台元标记 | UI显示标签，不直接控制执行 |
| 2 | `hidePoweredBy` | 邮件模板、PDF生成器、签署页面 | 隐藏"Powered by Documenso"标识 |
| 3 | `unlimitedDocuments` | `getServerLimits()` | 绕过月度文档数量限制 |
| 4 | `emailDomains` | 未找到实际检查点 | 预留企业版功能 |
| 5 | `embedAuthoring` | `create-embedding-presign-token.ts` | 嵌入编辑功能入口控制 |
| 6 | `embedAuthoringWhiteLabel` | `embed/v2/authoring/_layout.tsx` | 嵌入编辑自定义CSS注入 |
| 7 | `embedSigning` | 签署页面路由 | 嵌入签署功能入口控制 |
| 8 | `embedSigningWhiteLabel` | `load-recipient-branding.ts` | 收件人签署页自定义品牌 |
| 9 | `cfr21` | 6个服务端模块 + 8个前端UI | 21 CFR合规认证要求 |
| 10 | `hipaa` | 未找到实际检查点 | 预留合规标记 |
| 11 | `authenticationPortal` | 未找到实际检查点 | 预留企业版功能 |
| 12 | `allowLegacyEnvelopes` | `folder-grid.tsx` | 旧版文档上传按钮显示 |
| 13 | `signingReminders` | 未找到实际检查点 | 预留功能 |

> **备注**：`emailDomains`、`hipaa`、`authenticationPortal`、`signingReminders` 4个flag已在schema中定义，但暂未找到实际执行检查点，属于预留功能。

---

## 五、团队功能协作面：额度检查点跨模块映射

### 5.1 teamCount（团队数量上限）检查点

**唯一检查点**：`packages/lib/server-only/team/create-team.ts:77-90`

```typescript
// Validate they have enough team slots. 0 means they can create unlimited teams.
if (organisation.organisationClaim.teamCount !== 0 && IS_BILLING_ENABLED()) {
  const teamCount = await prisma.team.count({
    where: { organisationId },
  });

  if (teamCount >= organisation.organisationClaim.teamCount) {
    throw new AppError(AppErrorCode.LIMIT_EXCEEDED, {
      message: 'You have reached the maximum number of teams for your plan.',
    });
  }
}
```

**协作逻辑**:
- 读取 `organisationClaim.teamCount` 作为上限
- `0` 表示无限制
- 计费未启用时（自托管）跳过检查
- 统计当前组织下已有团队数量进行对比
- **各档位限制**：FREE/INDIVIDUAL/TEAM/PLATFORM=1，ENTERPRISE/EARLY_ADOPTER=0（无限）

---

### 5.2 memberCount（成员数量上限）检查点

**检查点分布在2个模块、3个位置**：

#### 检查点1：后端服务端断言
**位置**：`packages/ee/server-only/stripe/update-subscription-item-quantity.ts:65-89`

```typescript
export const assertMemberCountWithinCap = async (
  subscription: Subscription,
  organisationClaim: OrganisationClaim,
  quantity: number,  // 拟新增后的总成员数（含待邀请）
) => {
  const maximumMemberCount = organisationClaim.memberCount;

  if (maximumMemberCount === 0) return; // 0 = unlimited

  // Seats-based plans 无硬限制，由Stripe计量
  const isSeatsBased = await isPriceSeatsBased(subscription.priceId);
  if (isSeatsBased) return;

  if (quantity > maximumMemberCount) {
    throw new AppError(AppErrorCode.LIMIT_EXCEEDED, {
      message: 'Maximum member count reached',
    });
  }
};
```

#### 检查点2：Stripe座位同步
**位置**：`packages/ee/server-only/stripe/update-subscription-item-quantity.ts:102-136`

```typescript
export const syncMemberCountWithStripeSeatPlan = async (
  subscription: Subscription,
  organisationClaim: OrganisationClaim,
  quantity: number,
) => {
  if (organisationClaim.memberCount === 0) return; // 无限座位无需同步

  const isSeatsBased = await isPriceSeatsBased(subscription.priceId);
  if (!isSeatsBased) return;

  // 同步到Stripe
  await updateSubscriptionItemQuantity({
    priceId: subscription.priceId,
    subscriptionId: subscription.planId,
    quantity,
  });

  // 同步更新本地避免竞态
  await prisma.organisationClaim.update({
    where: { id: organisationClaim.id },
    data: { memberCount: quantity },
  });
};
```

#### 检查点3：调用入口（成员邀请）
**位置**：`packages/lib/server-only/organisation/create-organisation-member-invites.ts:124-135`

```typescript
const totalMemberCountWithInvites = numberOfCurrentMembers + numberOfCurrentInvites + numberOfNewInvites;

if (subscription) {
  await assertMemberCountWithinCap(subscription, organisationClaim, totalMemberCountWithInvites);
  await syncMemberCountWithStripeSeatPlan(subscription, organisationClaim, totalMemberCountWithInvites);
}
```

#### 检查点4：前端UI预检
**位置**：`apps/remix/app/components/dialogs/organisation-member-invite-dialog.tsx:157-179`

```typescript
const dialogState = useMemo(() => {
  if (!IS_BILLING_ENABLED()) return 'form';
  if (fullOrganisation.organisationClaim.memberCount === 0) return 'form';
  if (fullOrganisation.members.length < fullOrganisation.organisationClaim.memberCount) return 'form';

  // 特殊处理：TEAM档位即使满员也显示表单（可能走seats-based计费）
  if (fullOrganisation.organisationClaim.originalSubscriptionClaimId !== INTERNAL_CLAIM_ID.TEAM) {
    return 'alert';  // 显示升级提示
  }

  return 'form';
}, [fullOrganisation]);
```

**各档位限制**：FREE/INDIVIDUAL=1，TEAM=5，ENTERPRISE/EARLY_ADOPTER/PLATFORM=0（无限）

---

### 5.3 envelopeItemCount（信封文档数量上限）检查点

**检查点分布在2个模块**：

#### 检查点1：新增信封文档
**位置**：`packages/trpc/server/envelope-router/create-envelope-items.ts:74-83`

```typescript
const organisationClaim = envelope.team.organisation.organisationClaim;

const remainingEnvelopeItems = organisationClaim.envelopeItemCount 
  - envelope.envelopeItems.length 
  - files.length;

if (remainingEnvelopeItems < 0) {
  throw new AppError('ENVELOPE_ITEM_LIMIT_EXCEEDED', {
    message: `You cannot upload more than ${organisationClaim.envelopeItemCount} envelope items`,
    statusCode: 400,
  });
}
```

#### 检查点2：嵌入编辑更新信封
**位置**：`packages/trpc/server/embedding-router/update-embedding-envelope.ts:177-186`

```typescript
const organisationClaim = envelope.team.organisation.organisationClaim;
const resultingEnvelopeItemCount =
  envelope.envelopeItems.length 
  - envelopeItemIdsToDelete.length 
  + envelopeItemsToCreate.length;

if (resultingEnvelopeItemCount > organisationClaim.envelopeItemCount) {
  throw new AppError('ENVELOPE_ITEM_LIMIT_EXCEEDED', {
    message: `You cannot upload more than ${organisationClaim.envelopeItemCount} envelope items`,
    statusCode: 400,
  });
}
```

#### 检查点3：额度返回
**位置**：`packages/ee/server-only/limits/server.ts:44`

```typescript
const maximumEnvelopeItemCount = organisation.organisationClaim.envelopeItemCount;

return {
  quota,
  remaining,
  maximumEnvelopeItemCount,  // 返回给前端用于UI控制
};
```

**各档位限制**：FREE/INDIVIDUAL/TEAM/EARLY_ADOPTER=5，PLATFORM/ENTERPRISE=10

---

### 5.4 cfr21（21 CFR合规）检查点

**检查点分布在6个服务端模块 + 多个前端UI控制点**：

#### 服务端检查点（共6处）

| 模块 | 文件位置 | 检查场景 |
|------|----------|----------|
| 1 | `packages/lib/server-only/envelope/create-envelope.ts:231-238` | 创建信封时设置全局action auth |
| 2 | `packages/lib/server-only/envelope/update-envelope.ts:110-115` | 更新信封时修改全局action auth |
| 3 | `packages/lib/server-only/recipient/create-envelope-recipients.ts:76-81` | 创建收件人时设置action auth |
| 4 | `packages/lib/server-only/recipient/set-document-recipients.ts:99-104` | 设置文档收件人时设置action auth |
| 5 | `packages/lib/server-only/recipient/set-template-recipients.ts:56-61` | 设置模板收件人时设置action auth |
| 6 | `packages/lib/server-only/recipient/update-envelope-recipients.ts:80-85` | 更新收件人时设置action auth |

**典型检查逻辑**（以create-envelope为例）：
```typescript
const recipientsHaveActionAuth = data.recipients?.some(
  (recipient) => recipient.actionAuth && recipient.actionAuth.length > 0,
);

if (
  (authOptions.globalActionAuth.length > 0 || recipientsHaveActionAuth) &&
  !team.organisation.organisationClaim.flags.cfr21
) {
  throw new AppError(AppErrorCode.UNAUTHORIZED, {
    message: 'You do not have permission to set the action auth',
  });
}
```

#### 前端UI控制点（共8处）

| 模块 | 文件位置 | 控制内容 |
|------|----------|----------|
| 1 | `ui/primitives/document-flow/add-settings.tsx:301` | 文档设置中显示高级认证选项 |
| 2 | `ui/primitives/document-flow/add-signers.tsx:790,920` | 收件人表单中显示高级认证设置 |
| 3 | `ui/primitives/template-flow/add-template-settings.tsx:427` | 模板设置中显示高级认证选项 |
| 4 | `ui/primitives/template-flow/add-template-placeholder-recipients.tsx:680,801` | 模板收件人中显示高级认证 |
| 5 | `apps/remix/.../envelope-editor-settings-dialog.tsx:851` | 信封编辑器设置对话框 |
| 6 | `apps/remix/.../envelope-editor-recipient-form.tsx:634,637,996` | 信封编辑器收件人表单 |

**典型UI控制逻辑**：
```tsx
{organisation.organisationClaim.flags.cfr21 && (
  <div>
    {/* 高级认证设置UI */}
  </div>
)}
```

**适用档位**：仅ENTERPRISE档位默认开启

---

### 5.5 文档/模板月度额度检查

**核心函数**：`getServerLimits()` (`packages/ee/server-only/limits/server.ts:16-118`)

```typescript
export const getServerLimits = async ({ userId, teamId }) => {
  // 1. 通过teamId查找组织
  const organisation = await prisma.organisation.findFirst({...});

  // 2. 初始化免费版配额
  const quota = structuredClone(FREE_PLAN_LIMITS);
  const remaining = structuredClone(FREE_PLAN_LIMITS);

  // 3. 特殊情况处理（优先级从高到低）
  if (!IS_BILLING_ENABLED()) {
    return { quota: SELFHOSTED_PLAN_LIMITS, ... }; // 自托管无限制
  }

  // ⚠️ 硬编码判断：企业版即使过期也无限制
  if (organisation.organisationClaimId === INTERNAL_CLAIM_ID.ENTERPRISE) {
    return { quota: PAID_PLAN_LIMITS, ... };
  }

  if (subscription?.status === SubscriptionStatus.INACTIVE) {
    return { quota: INACTIVE_PLAN_LIMITS, ... }; // 过期归零
  }

  if (organisation.organisationClaim.flags.unlimitedDocuments) {
    return { quota: PAID_PLAN_LIMITS, ... }; // 有flag则无限制
  }

  // 4. 统计当月使用量
  const [documents, directTemplates] = await Promise.all([
    prisma.envelope.count({ /* 当月文档数，排除模板直链 */ }),
    prisma.envelope.count({ /* 带直链的模板数 */ }),
  ]);

  // 5. 计算剩余量
  remaining.documents = Math.max(remaining.documents - documents, 0);
  remaining.directTemplates = Math.max(remaining.directTemplates - directTemplates, 0);

  return { quota, remaining, maximumEnvelopeItemCount };
};
```

#### 配额常量定义

```typescript
export const FREE_PLAN_LIMITS: TLimitsSchema = {
  documents: 5,         // 每月5个文档
  recipients: 10,       // 10个收件人（实际未使用）
  directTemplates: 3,   // 3个直链模板
};

export const PAID_PLAN_LIMITS: TLimitsSchema = {
  documents: Infinity,
  recipients: Infinity,
  directTemplates: Infinity,
};
```

---

## 六、关键协作机制总结

### 6.1 权限检查的统一模式

所有权限检查都遵循以下模式：

```typescript
// 1. 获取组织声明
const organisationClaim = await getOrganisationClaimByTeamId({ teamId });

// 2. 检查计费是否启用
if (IS_BILLING_ENABLED()) {
  // 3. 检查对应flag或配额
  if (!organisationClaim.flags.[featureFlag]) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, ...);
  }
  // 或检查配额
  if (usage >= organisationClaim.[quotaField]) {
    throw new AppError(AppErrorCode.LIMIT_EXCEEDED, ...);
  }
}
// 自托管环境跳过检查
```

### 6.2 组织-团队权限继承关系

```
Organisation (1)
    ↓ has
OrganisationClaim (1)
    ↓ contains
teamCount, memberCount, envelopeItemCount, flags
    ↓ applies to
Team (N)
    ↓ belongs to
Organisation (1)
```

**关键点**:
- 权限是**组织级**的，不是团队级的
- 一个组织下的所有团队共享同一套 `organisationClaim`
- `teamCount` 限制组织下可创建的团队数量
- `memberCount` 限制组织下的总成员数量（含待邀请）

### 6.3 额度判断的优先级顺序

`getServerLimits()` 中的判断优先级（从高到低）：

1. **自托管模式** → 无限制
2. **ENTERPRISE档位** → 无限制（即使过期）
3. **订阅INACTIVE** → 全部归零
4. **unlimitedDocuments flag** → 无限制
5. **实际用量统计** → 计算剩余额度

> **设计问题**：ENTERPRISE判断采用硬编码 `organisationClaimId === INTERNAL_CLAIM_ID.ENTERPRISE`，而非通过flags组合判断，扩展性较差。

### 6.4 错误码与异常处理

| 错误码 | 场景 |
|--------|------|
| `UNAUTHORIZED` | 功能未授权（flag未开启） |
| `LIMIT_EXCEEDED` | 配额超限（团队数/成员数/文档数） |
| `ENVELOPE_ITEM_LIMIT_EXCEEDED` | 信封文档数量超限 |
| `NOT_FOUND` | 组织/团队不存在 |

---

## 七、代码优化建议

### 7.1 已发现的问题

1. **硬编码档位判断**：`getServerLimits()` 中直接判断 `organisationClaimId === INTERNAL_CLAIM_ID.ENTERPRISE`，新增档位需修改代码
2. **重复查询**：多个检查点独立调用 `getOrganisationClaimByTeamId()`，导致重复数据库查询
3. **额度计算性能**：`getServerLimits()` 每次调用都重新统计当月用量，高频访问下有性能问题
4. **PLATFORM档位矛盾配置**：`embedAuthoring: false` 但 `embedAuthoringWhiteLabel: true`，设计意图不清晰
5. **allowCustomBranding名不副实**：flag名称与实际使用不符，易引起误解
6. **backport job字段不完整**：`backport-subscription-claims` Job schema缺少 `emailDomains`、`authenticationPortal`、`allowLegacyEnvelopes` 3个字段，导致回溯更新时这3个flag无法批量同步
7. **预留flag无检查点**：`emailDomains`、`hipaa`、`authenticationPortal`、`signingReminders` 4个flag已在schema定义但无实际执行检查点

### 7.2 优化建议

**建议1：抽象化档位判断**

```typescript
// 替代硬编码的ENTERPRISE判断
const hasUnlimitedAccess = (claim: OrganisationClaim) => {
  return claim.flags.unlimitedDocuments && 
         claim.teamCount === 0 && 
         claim.memberCount === 0;
};
```

**建议2：引入权限上下文中间件**

```typescript
// 在TRPC上下文或请求中间件中预加载organisationClaim
export const createTrpcContext = async (opts) => {
  const teamId = opts.req.headers.get('team-id');
  const organisationClaim = teamId ? await getOrganisationClaimByTeamId({ teamId }) : null;

  return {
    ...opts,
    organisationClaim, // 后续procedure直接从ctx读取
  };
};
```

**建议3：用量统计缓存**

```typescript
// 使用LRU缓存月度用量，减少DB查询
const monthlyUsageCache = new LRUCache({ max: 1000, ttl: 5 * 60 * 1000 });

const getMonthlyUsage = async (organisationId: string) => {
  const cacheKey = `${organisationId}:${DateTime.utc().startOf('month').toISO()}`;
  const cached = monthlyUsageCache.get(cacheKey);
  if (cached) return cached;

  const usage = await prisma.envelope.count({...});
  monthlyUsageCache.set(cacheKey, usage);
  return usage;
};
```

**建议4：重命名或废弃allowCustomBranding**

将 `allowCustomBranding` 明确为元标记，或统一使用白标flags进行控制，避免混淆。

**建议5：补全backport job schema字段**

在 `packages/lib/jobs/definitions/internal/backport-subscription-claims.ts` 中补充缺失的3个字段：

```typescript
flags: z.object({
  unlimitedDocuments: z.literal(true).optional(),
  allowCustomBranding: z.literal(true).optional(),
  hidePoweredBy: z.literal(true).optional(),
  emailDomains: z.literal(true).optional(),           // 补充
  embedAuthoring: z.literal(true).optional(),
  embedAuthoringWhiteLabel: z.literal(true).optional(),
  embedSigning: z.literal(true).optional(),
  embedSigningWhiteLabel: z.literal(true).optional(),
  cfr21: z.literal(true).optional(),
  hipaa: z.literal(true).optional(),
  authenticationPortal: z.literal(true).optional(),  // 补充
  allowLegacyEnvelopes: z.literal(true).optional(),  // 补充
  signingReminders: z.literal(true).optional(),
}),
```

**建议6：建立flags完整性检查机制**

在CI/CD中添加校验脚本，确保：
1. `ZClaimFlagsSchema` 与 `SUBSCRIPTION_CLAIM_FEATURE_FLAGS` 字段一致
2. `backport-subscription-claims` schema 与 `ZClaimFlagsSchema` 字段一致
3. 每个flag都有对应的实际检查点（或明确标记为预留）

---

## 八、参考文件清单

| 文件路径 | 说明 |
|----------|------|
| `packages/lib/types/subscription.ts` | 档位定义、flags定义、内置档位配置 |
| `packages/prisma/schema.prisma:260-289` | SubscriptionClaim和OrganisationClaim数据模型 |
| `packages/ee/server-only/limits/server.ts` | 服务端额度计算核心逻辑 |
| `packages/ee/server-only/limits/constants.ts` | 各档位配额常量 |
| `packages/ee/server-only/stripe/webhook/on-subscription-created.ts` | 新订阅Webhook处理 |
| `packages/ee/server-only/stripe/webhook/on-subscription-updated.ts` | 订阅更新Webhook处理 |
| `packages/ee/server-only/stripe/webhook/on-subscription-deleted.ts` | 订阅删除Webhook处理 |
| `packages/ee/server-only/stripe/update-subscription-item-quantity.ts` | 成员配额检查与Stripe同步 |
| `packages/lib/server-only/team/create-team.ts` | 团队创建时的teamCount检查 |
| `packages/lib/server-only/organisation/create-organisation-member-invites.ts` | 成员邀请时的memberCount检查 |
| `packages/lib/server-only/organisation/create-organisation.ts` | 组织创建与claim初始化 |
| `packages/lib/server-only/envelope/create-envelope.ts` | 信封创建时的cfr21检查 |
| `packages/lib/server-only/envelope/update-envelope.ts` | 信封更新时的cfr21检查 |
| `packages/lib/server-only/recipient/create-envelope-recipients.ts` | 收件人创建时的cfr21检查 |
| `packages/lib/server-only/recipient/set-document-recipients.ts` | 文档收件人设置时的cfr21检查 |
| `packages/lib/server-only/recipient/set-template-recipients.ts` | 模板收件人设置时的cfr21检查 |
| `packages/lib/server-only/recipient/update-envelope-recipients.ts` | 收件人更新时的cfr21检查 |
| `packages/trpc/server/envelope-router/create-envelope-items.ts` | 新增信封文档时的envelopeItemCount检查 |
| `packages/trpc/server/embedding-router/update-embedding-envelope.ts` | 嵌入更新时的envelopeItemCount检查 |
| `packages/trpc/server/embedding-router/create-embedding-presign-token.ts` | 嵌入编辑的embedAuthoring检查 |
| `packages/trpc/server/admin-router/swap-organisation-subscription.ts` | 组织间订阅转移 |
| `packages/lib/server-only/branding/load-recipient-branding.ts` | 品牌功能检查（embedSigningWhiteLabel） |
| `packages/lib/server-only/email/get-email-context.ts` | 邮件品牌检查（hidePoweredBy） |
| `packages/lib/server-only/pdf/generate-certificate-pdf.ts` | 证书品牌检查（hidePoweredBy） |
| `packages/lib/jobs/definitions/internal/backport-subscription-claims.ts` | 档位回溯更新Job |
| `apps/remix/app/routes/embed+/v2+/authoring+/_layout.tsx` | 嵌入编辑白标检查（embedAuthoringWhiteLabel） |
| `apps/remix/app/components/dialogs/organisation-member-invite-dialog.tsx` | 成员邀请前端预检 |
