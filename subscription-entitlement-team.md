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

### 1.2 内置档位定义

系统在 `packages/lib/types/subscription.ts:126-218` 中预定义了6种内置档位：

```typescript
export enum INTERNAL_CLAIM_ID {
  FREE = 'free',           // 免费版
  INDIVIDUAL = 'individual', // 个人版
  TEAM = 'team',           // 团队版
  PLATFORM = 'platform',   // 平台版
  ENTERPRISE = 'enterprise', // 企业版
  EARLY_ADOPTER = 'earlyAdopter', // 早期用户
}
```

各档位的配额与功能对比：

| 档位 | teamCount | memberCount | envelopeItemCount | 核心flags |
|------|-----------|-------------|-------------------|-----------|
| FREE | 1 | 1 | 5 | 无 |
| INDIVIDUAL | 1 | 1 | 5 | unlimitedDocuments, signingReminders |
| TEAM | 1 | 5 | 5 | unlimitedDocuments, allowCustomBranding, embedSigning, signingReminders |
| PLATFORM | 1 | 0（无限） | 10 | unlimitedDocuments, allowCustomBranding, hidePoweredBy, embedAuthoringWhiteLabel, embedSigningWhiteLabel |
| ENTERPRISE | 0（无限） | 0（无限） | 10 | 全部功能开启 |
| EARLY_ADOPTER | 0（无限） | 0（无限） | 5 | unlimitedDocuments, allowCustomBranding, hidePoweredBy, embedSigning, embedSigningWhiteLabel |

### 1.3 功能开关（Flags）定义

在 `packages/lib/types/subscription.ts:9-38` 中定义了12个功能开关：

```typescript
export const ZClaimFlagsSchema = z.object({
  allowCustomBranding: z.boolean().optional(),      // 允许自定义品牌
  hidePoweredBy: z.boolean().optional(),            // 隐藏"Powered by"标识
  unlimitedDocuments: z.boolean().optional(),       // 无限文档
  emailDomains: z.boolean().optional(),             // 邮箱域名管理（企业版）
  embedAuthoring: z.boolean().optional(),           // 嵌入编辑功能（企业版）
  embedAuthoringWhiteLabel: z.boolean().optional(), // 嵌入编辑白标（企业版）
  embedSigning: z.boolean().optional(),             // 嵌入签署功能
  embedSigningWhiteLabel: z.boolean().optional(),   // 嵌入签署白标
  cfr21: z.boolean().optional(),                    // 21 CFR合规（企业版）
  hipaa: z.boolean().optional(),                    // HIPAA合规（企业版）
  authenticationPortal: z.boolean().optional(),     // 认证门户（企业版）
  allowLegacyEnvelopes: z.boolean().optional(),     // 允许旧版信封
  signingReminders: z.boolean().optional(),         // 签署提醒
});
```

---

## 二、订阅变更驱动权限更新的数据流

### 2.1 完整数据流图

```
Stripe Webhook
    ↓
on-subscription-created.ts / on-subscription-updated.ts
    ↓ 提取price.metadata.claimId
extractStripeClaim() → 从数据库查找SubscriptionClaim
    ↓
createOrganisationClaimUpsertData() → 转换为OrganisationClaim数据
    ↓
prisma.organisationClaim.update() → 更新组织权限
    ↓
后续所有功能检查点读取organisationClaim.flags进行判断
```

### 2.2 Stripe Webhook 处理流程

**新订阅创建** (`packages/ee/server-only/stripe/webhook/on-subscription-created.ts`)

1. 从Stripe Subscription对象提取 `customerId`
2. 从 `subscription.items.data[0].price.metadata.claimId` 提取档位ID
   - 优先读取Price metadata，若不存在则读取Product metadata
3. 根据 `subscription.metadata.organisationCreateData` 判断是新建组织还是更新现有组织
4. 新建组织时调用 `createOrganisation()`，传入对应的claim
5. 更新组织时调用 `handleOrganisationUpdate()` 更新organisationClaim
6. 最后upsert Subscription记录，设置状态（ACTIVE/INACTIVE/PAST_DUE）

**订阅更新** (`packages/ee/server-only/stripe/webhook/on-subscription-updated.ts`)

1. 通过customerId查找对应组织
2. 对比 `previousAttributes` 和当前subscription的price变化
3. 若claimId发生变化（如升级/降级），则更新organisationClaim
4. 更新Subscription状态和periodEnd时间

### 2.3 关键转换函数

`createOrganisationClaimUpsertData()` (`packages/lib/server-only/organisation/create-organisation.ts:187-201`)

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

## 三、团队功能协作面：额度检查点

### 3.1 团队创建额度检查

**位置**: `packages/lib/server-only/team/create-team.ts:77-90`

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

### 3.2 成员邀请额度检查

**位置**: `packages/lib/server-only/organisation/create-organisation-member-invites.ts:124-135`

```typescript
const totalMemberCountWithInvites = numberOfCurrentMembers + numberOfCurrentInvites + numberOfNewInvites;

// Enforce the seat cap and sync billing for seat based plans.
if (subscription) {
  await assertMemberCountWithinCap(subscription, organisationClaim, totalMemberCountWithInvites);
  await syncMemberCountWithStripeSeatPlan(subscription, organisationClaim, totalMemberCountWithInvites);
}
```

#### 额度断言逻辑

`assertMemberCountWithinCap()` (`packages/ee/server-only/stripe/update-subscription-item-quantity.ts:65-89`)

```typescript
export const assertMemberCountWithinCap = async (
  subscription: Subscription,
  organisationClaim: OrganisationClaim,
  quantity: number,
) => {
  const maximumMemberCount = organisationClaim.memberCount;
  
  if (maximumMemberCount === 0) return; // 0 = unlimited
  
  // Seats-based plans don't have a hard cap; Stripe meters the usage.
  const isSeatsBased = await isPriceSeatsBased(subscription.priceId);
  if (isSeatsBased) return;
  
  if (quantity > maximumMemberCount) {
    throw new AppError(AppErrorCode.LIMIT_EXCEEDED, {
      message: 'Maximum member count reached',
    });
  }
};
```

#### Stripe座位同步逻辑

`syncMemberCountWithStripeSeatPlan()` (`packages/ee/server-only/stripe/update-subscription-item-quantity.ts:102-136`)

- 仅对seats-based的订阅进行同步
- 调用Stripe API更新subscription item的quantity
- 同时更新本地organisationClaim.memberCount以避免竞态条件

### 3.3 文档/模板月度额度检查

**位置**: `packages/ee/server-only/limits/server.ts:16-118`

`getServerLimits()` 是核心的额度计算函数，返回当前配额和剩余量：

```typescript
export const getServerLimits = async ({ userId, teamId }: GetServerLimitsOptions) => {
  // 1. 通过teamId查找组织及其subscription和organisationClaim
  const organisation = await prisma.organisation.findFirst({...});
  
  // 2. 初始化免费版配额
  const quota = structuredClone(FREE_PLAN_LIMITS);
  const remaining = structuredClone(FREE_PLAN_LIMITS);
  
  // 3. 特殊情况处理（优先级从高到低）
  if (!IS_BILLING_ENABLED()) {
    return { quota: SELFHOSTED_PLAN_LIMITS, ... }; // 自托管无限制
  }
  
  if (organisation.organisationClaimId === INTERNAL_CLAIM_ID.ENTERPRISE) {
    return { quota: PAID_PLAN_LIMITS, ... }; // 企业版无限制，即使过期
  }
  
  if (subscription?.status === SubscriptionStatus.INACTIVE) {
    return { quota: INACTIVE_PLAN_LIMITS, ... }; // 过期订阅归零
  }
  
  if (organisation.organisationClaim.flags.unlimitedDocuments) {
    return { quota: PAID_PLAN_LIMITS, ... }; // 有无限文档flag则无限制
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

`packages/ee/server-only/limits/constants.ts`

```typescript
export const FREE_PLAN_LIMITS: TLimitsSchema = {
  documents: 5,         // 每月5个文档
  recipients: 10,       // 10个收件人
  directTemplates: 3,   // 3个直链模板
};

export const INACTIVE_PLAN_LIMITS: TLimitsSchema = {
  documents: 0,
  recipients: 0,
  directTemplates: 0,
};

export const PAID_PLAN_LIMITS: TLimitsSchema = {
  documents: Infinity,
  recipients: Infinity,
  directTemplates: Infinity,
};
```

---

## 四、团队功能协作面：功能开关检查点

### 4.1 CFR21 合规功能检查

**位置**: `packages/lib/server-only/envelope/create-envelope.ts:231-238`

```typescript
// Check if user has permission to set the global action auth.
if (
  (authOptions.globalActionAuth.length > 0 || recipientsHaveActionAuth) &&
  !team.organisation.organisationClaim.flags.cfr21
) {
  throw new AppError(AppErrorCode.UNAUTHORIZED, {
    message: 'You do not have permission to set the action auth',
  });
}
```

**协作逻辑**:
- 当用户设置操作认证（如访问密码、短信验证等）时，需要检查 `cfr21` flag
- 仅企业版档位默认开启此功能

### 4.2 自定义品牌功能检查

**位置**: `packages/lib/server-only/branding/load-recipient-branding.ts:22-55`

```typescript
export const loadRecipientBrandingByTeamId = async ({ teamId }) => {
  const billingEnabled = IS_BILLING_ENABLED();
  
  const [settings, claim] = await Promise.all([
    getTeamSettings({ teamId }),
    billingEnabled ? getOrganisationClaimByTeamId({ teamId }).catch(() => null) : null,
  ]);
  
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
  
  // 返回自定义品牌配置
  return { allowCustomBranding: true, hidePoweredBy, colors, css };
};
```

**协作逻辑**:
- `embedSigningWhiteLabel` flag 控制是否允许自定义品牌颜色和CSS
- `hidePoweredBy` flag 控制是否隐藏"Powered by Documenso"标识
- 自托管环境下默认全部允许

### 4.3 嵌入编辑功能检查

**位置**: `packages/trpc/server/embedding-router/create-embedding-presign-token.ts:34-52`

```typescript
if (IS_BILLING_ENABLED()) {
  const token = await getApiTokenByToken({ token: apiToken });
  
  const organisationClaim = await getOrganisationClaimByTeamId({
    teamId: token.teamId,
  });
  
  if (!organisationClaim.flags.embedAuthoring) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, {
      message: 'Embedded Authoring is not included in your current plan. Please contact support.',
    });
  }
}
```

**协作逻辑**:
- 创建嵌入编辑预签名令牌时检查 `embedAuthoring` flag
- 仅企业版和平台版默认开启

### 4.4 其他功能检查点

| 功能 | 检查位置 | Flag | 适用档位 |
|------|----------|------|----------|
| 认证门户 | `enterprise-router/get-organisation-authentication-portal.ts` | `authenticationPortal` | Enterprise |
| 邮箱域名 | `enterprise-router/create-organisation-email-domain.ts` | `emailDomains` | Enterprise |
| 签署提醒 | 前端UI控制 | `signingReminders` | Individual/Team/Platform/Enterprise |
| 证书品牌 | `pdf/generate-certificate-pdf.ts` | `allowCustomBranding` | Team/Platform/Enterprise |

---

## 五、关键协作机制总结

### 5.1 权限检查的统一模式

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

### 5.2 组织-团队权限继承关系

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

### 5.3 错误码与异常处理

| 错误码 | 场景 |
|--------|------|
| `UNAUTHORIZED` | 功能未授权（flag未开启） |
| `LIMIT_EXCEEDED` | 配额超限（团队数/成员数/文档数） |
| `NOT_FOUND` | 组织/团队不存在 |

### 5.4 计费开关的影响

`IS_BILLING_ENABLED()` 是全局开关，决定了是否执行所有订阅相关检查：

- **开启**（SaaS模式）：严格执行所有订阅档位检查
- **关闭**（自托管模式）：所有配额设为Infinity，所有功能flag视为开启

---

## 六、代码优化建议

### 6.1 潜在问题

1. **重复查询**：多个检查点独立调用 `getOrganisationClaimByTeamId()`，导致重复数据库查询
2. **硬编码档位判断**：`getServerLimits()` 中直接判断 `organisationClaimId === INTERNAL_CLAIM_ID.ENTERPRISE`，扩展性差
3. **额度计算时机**：`getServerLimits()` 每次调用都重新统计当月用量，在高频访问场景下可能有性能问题

### 6.2 优化建议

**建议1：引入权限上下文中间件**

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

**建议2：档位判断抽象化**

```typescript
// 替代硬编码的ENTERPRISE判断
const hasUnlimitedAccess = (claim: OrganisationClaim) => {
  return claim.flags.unlimitedDocuments && 
         claim.teamCount === 0 && 
         claim.memberCount === 0;
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

---

## 七、参考文件清单

| 文件路径 | 说明 |
|----------|------|
| `packages/lib/types/subscription.ts` | 档位定义、flags定义、内置档位配置 |
| `packages/prisma/schema.prisma:260-289` | SubscriptionClaim和OrganisationClaim数据模型 |
| `packages/ee/server-only/limits/server.ts` | 服务端额度计算核心逻辑 |
| `packages/ee/server-only/limits/constants.ts` | 各档位配额常量 |
| `packages/ee/server-only/stripe/webhook/on-subscription-created.ts` | 新订阅Webhook处理 |
| `packages/ee/server-only/stripe/webhook/on-subscription-updated.ts` | 订阅更新Webhook处理 |
| `packages/ee/server-only/stripe/update-subscription-item-quantity.ts` | 成员配额检查与Stripe同步 |
| `packages/lib/server-only/team/create-team.ts` | 团队创建时的配额检查 |
| `packages/lib/server-only/organisation/create-organisation-member-invites.ts` | 成员邀请时的配额检查 |
| `packages/lib/server-only/envelope/create-envelope.ts` | 信封创建时的CFR21检查 |
| `packages/lib/server-only/branding/load-recipient-branding.ts` | 品牌功能检查 |
| `packages/trpc/server/embedding-router/create-embedding-presign-token.ts` | 嵌入功能检查 |
