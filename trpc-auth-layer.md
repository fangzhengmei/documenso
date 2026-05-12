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

## 四、API V1 vs tRPC 认证对比

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
