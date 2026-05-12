# tRPC 认证权限中间件架构分析

## 一、认证入口分类

Documenso 的 tRPC 架构采用四层认证中间件体系，区分公开访问、登录会话、API Key 三类入口：

### 1.1 中间件层级结构

```
packages/trpc/server/trpc.ts
├── procedure (基础，无认证)
│   └── procedureMiddleware - 日志记录，无权限校验
├── authenticatedProcedure (强认证)
│   └── authenticatedMiddleware - 会话认证或 API Key 认证
├── maybeAuthenticatedProcedure (可选认证)
│   └── maybeAuthenticatedMiddleware - 可匿名也可登录
└── adminProcedure (管理员)
    └── adminMiddleware - 仅限管理员角色访问
```

### 1.2 认证方式分支

`authenticatedMiddleware` 内部实现了双路认证逻辑：

**路径 A：API Key 认证（OpenAPI 路由）**
```typescript
// 检测条件：meta.openapi.path 存在且 authorization header 存在
if (authorizationHeader && isApiV2) {
  // 支持两种格式：
  // 1. Authorization: Bearer api_xxx
  // 2. Authorization: api_xxx
  
  const apiToken = await getApiTokenByToken({ token });
  
  // 上下文增强：
  // - user: 关联的用户对象
  // - teamId: 关联的团队 ID
  // - session: null（非会话模式）
  // - metadata.auth = 'api'
}
```

**路径 B：会话认证（Web 应用）**
```typescript
if (!ctx.session) {
  throw new TRPCError({ code: 'UNAUTHORIZED' });
}

// 上下文增强：
// - user: 会话用户
// - teamId: 从 x-team-id header 解析
// - session: 完整会话对象
// - metadata.auth = 'session'
```

### 1.3 可选认证模式

`maybeAuthenticatedMiddleware` 支持：
- 匿名访问：user = null，auth = null
- API Key 认证：同 authenticatedMiddleware
- 会话认证：同 authenticatedMiddleware

主要用于需要区分用户但不强制登录的场景（如公开文档访问）。

---

## 二、团队成员角色校验

### 2.1 角色体系定义

`packages/lib/constants/teams.ts` 定义了三级角色权限模型：

```typescript
// 角色层级：ADMIN > MANAGER > MEMBER
export const TEAM_MEMBER_ROLE_HIERARCHY = {
  [TeamMemberRole.ADMIN]:   [ADMIN, MANAGER, MEMBER],
  [TeamMemberRole.MANAGER]: [MANAGER, MEMBER],
  [TeamMemberRole.MEMBER]:  [MEMBER],
};

// 操作权限映射
export const TEAM_MEMBER_ROLE_PERMISSIONS_MAP = {
  DELETE_TEAM:   [ADMIN],
  MANAGE_TEAM:   [ADMIN, MANAGER],
};
```

### 2.2 权限校验三层防护

**第一层：数据库查询层过滤（buildTeamWhereQuery）**

```typescript
// packages/lib/utils/teams.ts
export const buildTeamWhereQuery = ({ teamId, userId, roles }) => {
  return {
    id: teamId,
    teamGroups: {
      some: {
        organisationGroup: {
          organisationGroupMembers: {
            some: { organisationMember: { userId } }
          }
        },
        teamRole: roles ? { in: roles } : undefined
      }
    }
  };
};

// 使用示例：确保用户有 MANAGE_TEAM 权限才能查询
const team = await prisma.team.findFirst({
  where: buildTeamWhereQuery({
    teamId,
    userId: user.id,
    roles: TEAM_MEMBER_ROLE_PERMISSIONS_MAP['MANAGE_TEAM'],
  })
});
```

**第二层：角色层级校验（getMemberRoles + isTeamRoleWithinUserHierarchy）**

```typescript
// packages/lib/server-only/team/get-member-roles.ts
export const getMemberRoles = async ({ teamId, reference }) => {
  // reference 支持两种类型：
  // - { type: 'User', id: number } - 通过用户 ID 查角色
  // - { type: 'Member', id: string } - 通过成员 ID 查角色
  
  const team = await prisma.team.findUnique({
    where: { id: teamId },
    include: { teamGroups: { ... } }
  });
  
  return {
    teamRole: getHighestTeamRoleInGroup(team.teamGroups)
  };
};

// 层级比较 - 防止低权限用户修改高权限用户
if (!isTeamRoleWithinUserHierarchy(currentUserRole, targetUserRole)) {
  throw new AppError(AppErrorCode.UNAUTHORIZED, {
    message: 'Cannot update a member with a higher role'
  });
}
```

**第三层：操作权限校验（canExecuteTeamAction）**

```typescript
export const canExecuteTeamAction = (action, role) => {
  return TEAM_MEMBER_ROLE_PERMISSIONS_MAP[action].some(r => r === role);
};
```

### 2.3 完整校验流程（以 updateTeamMember 为例）

```typescript
// packages/trpc/server/team-router/update-team-member.ts
export const updateTeamMemberRoute = authenticatedProcedure
  .mutation(async ({ ctx, input }) => {
    const { teamId, memberId, data } = input;
    const userId = ctx.user.id;

    // 1. 数据库层过滤：确保用户是团队成员且有 MANAGE_TEAM 权限
    const team = await prisma.team.findFirst({
      where: {
        AND: [
          buildTeamWhereQuery({
            teamId,
            userId,
            roles: TEAM_MEMBER_ROLE_PERMISSIONS_MAP['MANAGE_TEAM'],
          }),
          { organisation: { members: { some: { id: memberId } } } }
        ]
      }
    });

    // 2. 获取当前操作者角色
    const { teamRole: currentUserTeamRole } = await getMemberRoles({
      teamId,
      reference: { type: 'User', id: userId },
    });

    // 3. 获取目标成员当前角色
    const { teamRole: currentMemberToUpdateTeamRole } = await getMemberRoles({
      teamId,
      reference: { type: 'Member', id: memberId },
    });

    // 4. 层级校验：不能操作比自己权限高的用户
    if (!isTeamRoleWithinUserHierarchy(currentUserTeamRole, currentMemberToUpdateTeamRole)) {
      throw new AppError(AppErrorCode.UNAUTHORIZED, {
        message: 'Cannot update a member with a higher role',
      });
    }

    // 5. 层级校验：不能将用户提升到比自己还高的角色
    if (!isTeamRoleWithinUserHierarchy(currentUserTeamRole, data.role)) {
      throw new AppError(AppErrorCode.UNAUTHORIZED, {
        message: 'Cannot update a member to a role higher than your own',
      });
    }

    // 执行更新...
  });
```

---

## 三、上下文（Context）共享机制

### 3.1 Context 创建流程

`packages/trpc/server/context.ts` 定义了 tRPC 上下文的生命周期：

```typescript
export const createTrpcContext = async ({ c, requestSource }) => {
  // 1. 从 Hono 请求中提取会话（如果有）
  const { session, user } = await getOptionalSession(c);

  // 2. 解析团队 ID（从 x-team-id header）
  const rawTeamId = req.headers.get('x-team-id');
  const teamId = z.coerce.number().optional().catch(undefined).parse(rawTeamId);

  // 3. 初始化元数据
  const metadata: ApiRequestMetadata = {
    requestMetadata,
    source: requestSource, // 'app' | 'apiV1' | 'apiV2'
    auth: null, // 后续中间件会更新此字段
  };

  // 4. 创建 logger 子实例（带请求 ID）
  const trpcLogger = logger.child({
    ipAddress: requestMetadata.ipAddress,
    userAgent: requestMetadata.userAgent,
    requestId: alphaid(),
  });

  return {
    logger: trpcLogger,
    session,
    user,
    teamId,
    req,
    res,
    metadata,
  };
};
```

### 3.2 Context 类型定义

```typescript
export type TrpcContext = (
  | { session: null; user: null }      // 未认证
  | { session: Session; user: SessionUser }  // 已认证
) & {
  teamId: number | undefined;    // 当前操作的团队 ID
  req: Request;                  // 原始请求
  res: Response;                 // 响应对象
  metadata: ApiRequestMetadata;  // 审计元数据
  logger: Logger;                // 带上下文的日志器
};
```

### 3.3 元数据（metadata）字段详解

```typescript
type ApiRequestMetadata = {
  requestMetadata: {
    ipAddress: string;      // 客户端 IP
    userAgent: string;      // User-Agent
  };
  source: 'app' | 'apiV1' | 'apiV2';  // 请求来源
  auth: 'session' | 'api' | null;     // 认证方式
  
  // 审计用户信息（用于日志和审计追踪）
  auditUser?: {
    id: string | null;      // 用户 ID（API 模式下为 null）
    email: string | null;   // 邮箱（API 模式下为 null）
    name: string;           // 用户名或团队名
  };
};
```

### 3.4 中间件对 Context 的增强

```
┌─────────────────────────────────────────────────────────────┐
│ 初始 Context (createTrpcContext)                            │
│  ├── logger: 基础日志器                                      │
│  ├── session: 可能为 null                                    │
│  ├── user: 可能为 null                                      │
│  ├── teamId: 从 header 解析                                 │
│  └── metadata.auth: null                                    │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ authenticatedMiddleware 增强                                │
│  ├── 路径 A (API Key):                                     │
│  │   ├── user = apiToken.user                              │
│  │   ├── teamId = apiToken.teamId                          │
│  │   ├── session = null                                    │
│  │   └── metadata.auth = 'api'                             │
│  └── 路径 B (Session):                                     │
│      ├── user = session 用户                                │
│      ├── teamId = ctx.teamId || -1                         │
│      ├── session = 完整会话对象                             │
│      └── metadata.auth = 'session'                         │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 业务路由 Handler 可访问                                      │
│  ├── ctx.user.id - 操作者 ID                                │
│  ├── ctx.teamId - 操作目标团队                              │
│  ├── ctx.logger.info(...) - 带上下文日志                    │
│  └── ctx.metadata.auditUser - 审计追踪                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 四、请求执行链路时序分析

### 4.1 分支判定矩阵

根据 `packages/trpc/server/trpc.ts` 第 81-125 行的真实实现，认证分支的判定逻辑如下：

| authorization | meta.openapi.path | session 存在 | 执行分支 | 最终 metadata.auth | 最终 teamId 来源 |
|--------------|-------------------|-------------|----------|-------------------|------------------|
| ✓ 存在 | ✓ 存在 | 任意 | **API Key 认证分支** | 'api' | apiToken.teamId (覆盖 header) |
| ✗ 不存在 | ✗ 不存在 | ✓ 存在 | **会话认证分支** | 'session' | x-team-id header (或 -1) |
| ✗ 不存在 | ✗ 不存在 | ✗ 不存在 | authenticated → **抛出 UNAUTHORIZED** <br> maybeAuth → **继续匿名** | null | undefined |
| ✓ 存在 | ✗ 不存在 | ✓ 存在 | **回落到会话认证** | 'session' | x-team-id header (或 -1) |
| ✓ 存在 | ✗ 不存在 | ✗ 不存在 | authenticated → **抛出 UNAUTHORIZED** <br> maybeAuth → **继续匿名** | null | undefined |
| ✗ 不存在 | ✓ 存在 | ✓ 存在 | **会话认证分支** (OpenAPI 路由但无 API Key) | 'session' | x-team-id header (或 -1) |
| ✗ 不存在 | ✓ 存在 | ✗ 不存在 | **抛出 UNAUTHORIZED** | - | - |

**核心规则**（代码第 86 行）：`if (authorizationHeader && isApiV2)` → 只有两个条件**同时满足**才走 API Key 分支，否则回落到会话校验（authenticated）或匿名（maybeAuth）。

---

### 4.2 完整请求执行时序（会话认证模式）

以下是一次真实 Web 应用请求的完整执行链路（以 `updateTeamMember` 路由为例，使用 `authenticatedProcedure`）：

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: HTTP 请求入口 (Hono Server)                                 │
├─────────────────────────────────────────────────────────────────────┤
│ 路径: POST /api/trpc/team.updateTeamMember                         │
│ Headers:                                                            │
│   - Cookie: documenso.session=xxx (会话凭证)                        │
│   - x-team-id: 12345                                                │
│   - Content-Type: application/json                                  │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Context 初始化 (createTrpcContext)                         │
├─────────────────────────────────────────────────────────────────────┤
│ 调用: getOptionalSession(c) 从 Cookie 解析会话                      │
│                                                                     │
│ 初始 Context 状态:                                                  │
│   ├── logger: { ipAddress, userAgent, requestId: 'abc123' }        │
│   ├── session: Session { id: 'sess_xxx', userId: 1001 }            │
│   ├── user: { id: 1001, name: 'Alice', email: 'alice@documenso.com' }│
│   ├── teamId: 12345 (从 x-team-id header 解析)                     │
│   ├── req: Request 对象                                            │
│   ├── res: Response 对象                                           │
│   └── metadata: { source: 'app', auth: null }                      │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: 路由匹配与 Procedure 分派                                  │
├─────────────────────────────────────────────────────────────────────┤
│ 路由定义 (packages/trpc/server/team-router/updateTeamMember.ts):   │
│   export const updateTeamMemberRoute = authenticatedProcedure       │
│     .input(ZUpdateTeamMemberRequestSchema)                          │
│     .output(ZUpdateTeamMemberResponseSchema)                        │
│     .mutation(handler)                                              │
│                                                                     │
│ 关键: 此路由无 .meta({ openapi: ... }) → meta.openapi.path = undefined│
│                                                                     │
│ 分派结果: 触发 authenticatedMiddleware                              │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: authenticatedMiddleware 执行分支判断                        │
├─────────────────────────────────────────────────────────────────────┤
│ 代码第 81-86 行判定:                                                │
│   const authorizationHeader = ctx.req.headers.get('authorization')  │
│   const isApiV2 = Boolean(meta?.openapi?.path)                      │
│                                                                     │
│ 实际值:                                                             │
│   - authorizationHeader? → undefined (无 API Key)                   │
│   - isApiV2 → Boolean(undefined) → false                            │
│   → 条件 `authorizationHeader && isApiV2` = false ❌                │
│   → 跳过 API Key 分支，进入【会话认证校验】                         │
│                                                                     │
│ 代码第 127-132 行校验:                                              │
│   if (!ctx.session) → 会话存在，校验通过                            │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 5: Context 增强处理（会话模式）                               │
├─────────────────────────────────────────────────────────────────────┤
│ 代码第 134-162 行执行:                                              │
│                                                                     │
│ 1. Logger 子实例创建:                                               │
│    ctx.logger.child({ nonBatchedRequestId: alphaid() })             │
│                                                                     │
│ 2. Metadata 更新 (第 152-160 行):                                  │
│    ctx.metadata.auth = 'session'                                    │
│    ctx.metadata.auditUser = { id: 1001, name: 'Alice', email: '...' }│
│                                                                     │
│ 3. Team ID 规范化 (第 148 行):                                      │
│    ctx.teamId = ctx.teamId || -1  // 12345 (来自 header)            │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 6: 业务 Handler 执行                                          │
├─────────────────────────────────────────────────────────────────────┤
│ Handler 接收的最终 Context:                                         │
│   ctx.user → { id: 1001, name: 'Alice', email: 'alice@documenso.com' }│
│   ctx.teamId → 12345 (来自 x-team-id header)                        │
│   ctx.session → Session 对象                                        │
│   ctx.metadata.auth → 'session'                                     │
│                                                                     │
│ 业务逻辑: 调用 buildTeamWhereQuery + getMemberRoles 校验权限...    │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 4.3 对照时序 A：匿名 token 访问（真实 maybeAuthenticatedProcedure）

以仓库真实的 `getEnvelopeItemsByTokenRoute` 路由为例（packages/trpc/server/envelope-router/get-envelope-items-by-token.ts）：

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: HTTP 请求入口                                               │
├─────────────────────────────────────────────────────────────────────┤
│ 路径: POST /api/trpc/envelope.getEnvelopeItemsByToken              │
│ Headers: 无 Cookie，无 Authorization                                 │
│          x-team-id: 无                                              │
│ Body: { envelopeId: 'env_xxx', access: { type: 'recipient', token: 'tok_xxx' } }│
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Context 初始化                                              │
├─────────────────────────────────────────────────────────────────────┤
│ getOptionalSession(c) → { session: null, user: null }              │
│                                                                     │
│ 初始 Context:                                                       │
│   session: null                                                     │
│   user: null                                                        │
│   teamId: undefined                                                 │
│   metadata: { source: 'app', auth: null }                           │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: Procedure 分派 → maybeAuthenticatedProcedure              │
├─────────────────────────────────────────────────────────────────────┤
│ 路由定义 (第 15 行):                                                │
│   export const getEnvelopeItemsByTokenRoute = maybeAuthenticatedProcedure│
│                                                                     │
│ 关键: 无 .meta({ openapi: ... }) → isApiV2 = false                 │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: maybeAuthenticatedMiddleware 执行                          │
├─────────────────────────────────────────────────────────────────────┤
│ 代码第 179-223 行判定:                                              │
│   const authorizationHeader = ctx.req.headers.get('authorization')  │
│   const isApiV2 = Boolean(meta?.openapi?.path)                      │
│                                                                     │
│ 实际值:                                                             │
│   - authorizationHeader? → undefined                                │
│   - isApiV2 → false                                                 │
│   → 条件 `authorizationHeader && isApiV2` = false ❌                │
│   → 跳过 API Key 分支                                               │
│                                                                     │
│ 代码第 225-249 行: 不校验 session! 直接继续执行                    │
│   ❗ 与 authenticatedMiddleware 核心区别: 无 if (!ctx.session) throw│
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 5: Context 增强（匿名模式）                                   │
├─────────────────────────────────────────────────────────────────────┤
│ 代码第 231-248 行执行:                                              │
│   ctx.metadata.auth = ctx.session ? 'session' : null  → null        │
│   ctx.metadata.auditUser = ctx.user ? { ... } : undefined → undefined│
│   ctx.teamId = undefined (保持原值)                                 │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 6: Handler 执行（匿名 token 访问）                            │
├─────────────────────────────────────────────────────────────────────┤
│ Handler 接收 Context (第 19 行):                                    │
│   const { teamId, user } = ctx;  // teamId = undefined, user = null│
│                                                                     │
│ 业务逻辑 (第 30-56 行):                                             │
│   if (access.type === 'user') {                                     │
│     // 需要认证: 手动校验 user/teamId                               │
│     if (!user || !teamId) throw new AppError(UNAUTHORIZED);         │
│   } else {                                                          │
│     // 匿名访问: 通过 recipient token 查询，不校验用户              │
│     return handleGetEnvelopeItemsByToken(envelopeId, access.token); │
│   }                                                                 │
└─────────────────────────────────────────────────────────────────────┘
```

**关键点**：`maybeAuthenticatedProcedure` 不强制认证，但业务 Handler 可根据访问类型**自行决定**是否需要认证。

---

### 4.4 对照时序 B：Authorization 存在但非 OpenAPI 路由（回落场景）

这是容易忽略的边界情况：请求带 Authorization header，但路由无 openapi meta：

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: HTTP 请求入口                                               │
├─────────────────────────────────────────────────────────────────────┤
│ 路径: POST /api/trpc/team.updateTeamMember                         │
│ Headers:                                                            │
│   - Authorization: Bearer api_abc123def456 (误传)                  │
│   - Cookie: documenso.session=xxx (同时存在会话)                    │
│   - x-team-id: 12345                                                │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Context 初始化                                              │
├─────────────────────────────────────────────────────────────────────┤
│ getOptionalSession(c) → 从 Cookie 解析会话成功                       │
│   session: Session { id: 'sess_xxx', userId: 1001 }                │
│   user: { id: 1001, name: 'Alice', ... }                            │
│   teamId: 12345 (从 header 解析)                                    │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: 路由匹配 → authenticatedProcedure                         │
├─────────────────────────────────────────────────────────────────────┤
│ 关键: updateTeamMember 路由无 .meta({ openapi: ... })              │
│       → meta.openapi.path = undefined                               │
│       → isApiV2 = Boolean(undefined) = false                        │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: authenticatedMiddleware 分支判断 ❗ 重要                   │
├─────────────────────────────────────────────────────────────────────┤
│ 代码第 86 行条件: `if (authorizationHeader && isApiV2)`             │
│                                                                     │
│ 实际值:                                                             │
│   - authorizationHeader → 'Bearer api_abc123def456' ✓ (存在)       │
│   - isApiV2 → false ✗ (不存在)                                     │
│   → 逻辑与结果: true && false = false                               │
│                                                                     │
│ ❗ 结果: 跳过 API Key 分支，回落到会话认证分支!                     │
│        Authorization header 被完全忽略!                            │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 5: 会话认证分支执行                                            │
├─────────────────────────────────────────────────────────────────────┤
│ 代码第 127 行校验: if (!ctx.session) → 通过，会话存在               │
│                                                                     │
│ 最终 Context 状态:                                                  │
│   ctx.user → { id: 1001, ... } (来自 Cookie 会话，非 API Token)    │
│   ctx.teamId → 12345 (来自 x-team-id header，非 API Token)          │
│   ctx.metadata.auth → 'session' (不是 'api'!)                       │
│                                                                     │
│ ❗ 注意: API Token 未被查询、未被验证、完全被丢弃!                  │
└─────────────────────────────────────────────────────────────────────┘
```

**关键结论**：只有**同时满足** `Authorization header 存在` + `路由定义了 meta.openapi.path`，才会走 API Key 认证。缺少任一条件，Authorization header 会被静默忽略。

---

### 4.5 四种场景最终状态对照表

| 场景 | ctx.user 来源 | ctx.teamId 来源 | metadata.auth | session 存在 |
|------|--------------|----------------|---------------|-------------|
| 会话认证（标准） | Cookie 会话 | x-team-id header | 'session' | ✓ |
| 匿名访问（maybeAuth + token） | null | undefined | null | ✗ |
| API Key 认证（OpenAPI 路由） | API Token 关联用户 | API Token.teamId | 'api' | ✗ |
| 回落场景（有 Authorization 但非 OpenAPI） | Cookie 会话 | x-team-id header | 'session' | ✓ |

---

## 五、API V1 vs tRPC 认证对比

### 4.1 API V1 独立中间件

`packages/api/v1/middleware/authenticated.ts` 采用独立实现：

```typescript
// API V1 中间件直接包装路由 handler
export const authenticatedMiddleware = (handler) => {
  return async (args, { request }) => {
    const { authorization } = args.headers;
    const [token] = authorization.split('Bearer ').filter(Boolean);
    
    const apiToken = await getApiTokenByToken({ token });
    
    // 直接传递给 handler
    return handler(args, apiToken.user, apiToken.team, { 
      metadata, 
      logger 
    });
  };
};
```

### 4.2 异同点总结

| 特性 | tRPC authenticatedMiddleware | API V1 authenticatedMiddleware |
|------|-----------------------------|-------------------------------|
| 认证方式 | Session + API Key | 仅 API Key |
| 上下文注入 | tRPC Context 增强 | Handler 参数直接传递 |
| 团队 ID | x-team-id header | 路径参数 /teams/:teamId |
| 日志集成 | Logger child 模式 | Logger child 模式 |
| 审计元数据 | metadata.auditUser | metadata.auditUser |

---

## 六、关键设计模式总结

1. **数据库层权限过滤优先**：通过 `buildTeamWhereQuery` 在查询层面就过滤掉无权限数据，避免后续校验遗漏

2. **中间件职责单一**：每层中间件只做一件事，组合使用实现复杂权限控制

3. **Context 渐进式增强**：从基础 Context 开始，每层中间件逐步添加上下文信息

4. **角色层级双向校验**：既校验操作者权限，也校验目标对象权限，防止权限提升攻击

5. **审计信息贯穿全链路**：metadata 携带完整审计信息，所有操作都可溯源
