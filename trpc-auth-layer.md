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

### 4.1 完整请求执行时序（会话认证模式）

以下是一次真实 Web 应用请求的完整执行链路（以 `findTeamMembers` 路由为例）：

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: HTTP 请求入口 (Hono Server)                                 │
├─────────────────────────────────────────────────────────────────────┤
│ 路径: POST /api/trpc/team.findTeamMembers                          │
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
│ 路由定义:                                                           │
│   export const findTeamMembersRoute = authenticatedProcedure       │
│     .input(ZFindTeamMembersRequestSchema)                           │
│     .output(ZFindTeamMembersResponseSchema)                         │
│     .query(handler)                                                 │
│                                                                     │
│ 分派结果: 使用 authenticatedProcedure → 触发 authenticatedMiddleware│
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: authenticatedMiddleware 执行分支判断                        │
├─────────────────────────────────────────────────────────────────────┤
│ 判断条件:                                                           │
│   - authorizationHeader? → undefined (无 API Key)                   │
│   - isApiV2 = Boolean(meta.openapi.path) → false (非 OpenAPI 路由) │
│   → 进入【会话认证分支】                                             │
│                                                                     │
│ 校验: if (!ctx.session) → 通过，会话存在                            │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 5: Context 增强处理                                           │
├─────────────────────────────────────────────────────────────────────┤
│ 1. Logger 子实例创建 (带 nonBatchedRequestId)                      │
│    const trpcSessionLogger = ctx.logger.child({                    │
│      nonBatchedRequestId: alphaid()  // e.g. 'def456'              │
│    })                                                               │
│                                                                     │
│ 2. Metadata 更新:                                                   │
│    ctx.metadata.auth = 'session'                                    │
│    ctx.metadata.auditUser = {                                       │
│      id: 1001,                                                      │
│      name: 'Alice',                                                 │
│      email: 'alice@documenso.com'                                   │
│    }                                                                │
│                                                                     │
│ 3. Team ID 规范化:                                                  │
│    ctx.teamId = ctx.teamId || -1  // 12345                         │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 6: 业务 Handler 执行 (findTeamMembers)                        │
├─────────────────────────────────────────────────────────────────────┤
│ Handler 可访问的最终 Context 状态:                                  │
│                                                                     │
│  ctx = {                                                            │
│    logger: LoggerChild({                                            │
│      ipAddress: '192.168.1.1',                                      │
│      userAgent: 'Chrome/120.0',                                     │
│      requestId: 'abc123',        // Hono 层请求 ID                  │
│      nonBatchedRequestId: 'def456'  // tRPC 子请求 ID               │
│    })                                                               │
│    session: Session { id: 'sess_xxx', userId: 1001, ... }          │
│    user: { id: 1001, name: 'Alice', email: 'alice@documenso.com' } │
│    teamId: 12345                                                    │
│    req: Request                                                     │
│    res: Response                                                    │
│    metadata: {                                                      │
│      source: 'app',                                                 │
│      auth: 'session',                                               │
│      auditUser: { id: 1001, name: 'Alice', email: '...' }          │
│    }                                                                │
│  }                                                                  │
│                                                                     │
│ 业务逻辑: 使用 ctx.user.id 和 ctx.teamId 进行数据库查询...         │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 4.2 对照时序 A：匿名访问（maybeAuthenticatedProcedure）

以公开文档查看路由为例，使用 `maybeAuthenticatedProcedure`：

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: HTTP 请求入口                                               │
├─────────────────────────────────────────────────────────────────────┤
│ 路径: POST /api/trpc/document.getPublicDocument                     │
│ Headers: 无 Cookie，无 Authorization                                 │
│          x-team-id: 无                                              │
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
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: maybeAuthenticatedMiddleware 执行                          │
├─────────────────────────────────────────────────────────────────────┤
│ 判断条件:                                                           │
│   - authorizationHeader? → undefined                                │
│   - isApiV2? → false                                                │
│   → 无认证信息，保留匿名状态                                        │
│                                                                     │
│ 不抛出异常！这是与 authenticatedMiddleware 的核心区别               │
│                                                                     │
│ Context 状态保持:                                                   │
│   session: null                                                     │
│   user: null                                                        │
│   metadata.auth: null                                               │
│   metadata.auditUser: undefined                                     │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 5: Handler 执行（匿名模式）                                    │
├─────────────────────────────────────────────────────────────────────┤
│ Handler 接收 Context:                                               │
│   ctx.user → null  ❗ 业务代码必须处理 user 为 null 的情况          │
│   ctx.teamId → undefined                                            │
│   ctx.metadata.auth → null                                          │
│                                                                     │
│ 业务逻辑: 仅返回公开文档信息，不返回任何用户私有数据                 │
└─────────────────────────────────────────────────────────────────────┘
```

**关键点**：`maybeAuthenticatedProcedure` 允许 `ctx.user` 为 `null`，业务 Handler 必须自行处理匿名访问逻辑。

---

### 4.3 对照时序 B：API Key 认证（OpenAPI 路由）

以 API V2 文档列表查询为例：

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: HTTP 请求入口                                               │
├─────────────────────────────────────────────────────────────────────┤
│ 路径: GET /api/v2/documents                                         │
│ Headers:                                                            │
│   - Authorization: Bearer api_abc123def456                         │
│   - x-team-id: 67890                                                │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Context 初始化                                              │
├─────────────────────────────────────────────────────────────────────┤
│ getOptionalSession(c) → { session: null, user: null }              │
│                                                                     │
│ 初始 Context:                                                       │
│   session: null                   ❗ 会话为空                       │
│   user: null                      ❗ 用户为空                       │
│   teamId: 67890 (从 header 解析，后续会被 API Token 覆盖)           │
│   metadata: { source: 'apiV2', auth: null }                        │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: Procedure 分派 → authenticatedProcedure                   │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: authenticatedMiddleware 执行 → API Key 分支               │
├─────────────────────────────────────────────────────────────────────┤
│ 判断条件:                                                           │
│   - authorizationHeader? → 'Bearer api_abc123def456' ✓             │
│   - isApiV2 = Boolean(meta.openapi.path) → true ✓                  │
│   → 进入【API Key 认证分支】                                         │
│                                                                     │
│ Token 解析:                                                         │
│   const [token] = authorization.split('Bearer ').filter(Boolean)   │
│   // token = 'api_abc123def456'                                    │
│                                                                     │
│ 数据库查询: getApiTokenByToken({ token })                           │
│   → 查找有效的 API Token 记录                                       │
│   → 关联 user 和 team                                              │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 5: Context 增强（API Key 模式）                               │
├─────────────────────────────────────────────────────────────────────┤
│  ctx = {                                                            │
│    ...                                                              │
│    user: apiToken.user          ❗ 从 API Token 关联的用户          │
│    teamId: apiToken.teamId      ❗ 覆盖 header 中的 teamId         │
│    session: null                 ❗ 始终为 null                     │
│    metadata: {                                                      │
│      source: 'apiV2',                                               │
│      auth: 'api',               ✅ 标记为 API 认证                  │
│      auditUser: {                                                   │
│        id: apiToken.team ? null : apiToken.user.id,                 │
│        email: apiToken.team ? null : apiToken.user.email,           │
│        name: apiToken.team?.name ?? apiToken.user.name              │
│      }                                                              │
│    }                                                                │
│  }                                                                  │
│                                                                     │
│ 注意: 如果是团队级 API Token，auditUser.id/email 为 null            │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 6: Handler 执行（API 模式）                                   │
├─────────────────────────────────────────────────────────────────────┤
│ Handler 接收 Context:                                               │
│   ctx.user → { id: 2002, name: 'API Bot', email: 'api@documenso.com' }│
│   ctx.teamId → 67890 (来自 API Token)                              │
│   ctx.session → null                                               │
│   ctx.metadata.auth → 'api'                                        │
│                                                                     │
│ 业务逻辑: 与会话认证一致，但通过 ctx.metadata.auth 区分审计来源    │
└─────────────────────────────────────────────────────────────────────┘
```

**关键点**：
1. 即使有会话 Cookie，只要提供 Authorization header + OpenAPI meta，优先走 API Key 认证
2. API Token 中的 teamId 会覆盖 x-team-id header，防止越权访问
3. 团队级 API Token 的 auditUser.id/email 为 null，仅保留团队名用于审计

---

### 4.4 三种入口最终状态对比表

| 字段 | 公开访问 (procedure) | 匿名访问 (maybeAuth) | 会话认证 (authed) | API Key 认证 (authed) |
|------|---------------------|---------------------|-------------------|----------------------|
| ctx.session | null / Session | null | Session | null |
| ctx.user | null / User | null | User | User |
| ctx.teamId | undefined | undefined | number (from header) | number (from token) |
| ctx.metadata.source | 'app' / 'apiV2' | 'app' | 'app' | 'apiV2' |
| ctx.metadata.auth | null | null | 'session' | 'api' |
| ctx.metadata.auditUser | undefined | undefined | { id, name, email } | { id?, name, email? } |
| 认证失败行为 | - | 继续执行 | 抛出 UNAUTHORIZED | 抛出 UNAUTHORIZED |

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

## 五、关键设计模式总结

1. **数据库层权限过滤优先**：通过 `buildTeamWhereQuery` 在查询层面就过滤掉无权限数据，避免后续校验遗漏

2. **中间件职责单一**：每层中间件只做一件事，组合使用实现复杂权限控制

3. **Context 渐进式增强**：从基础 Context 开始，每层中间件逐步添加上下文信息

4. **角色层级双向校验**：既校验操作者权限，也校验目标对象权限，防止权限提升攻击

5. **审计信息贯穿全链路**：metadata 携带完整审计信息，所有操作都可溯源
