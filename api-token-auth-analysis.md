# Documenso API Token 鉴权边界分析报告

## 1. 概述

本报告深入分析 Documenso 系统中 API Token 的鉴权边界，包括 **API V1** 和 **tRPC/API V2** 两条请求拦截路径、令牌创建机制、权限判定逻辑、失败返回对齐以及过期令牌在不同路径中的状态码来源。

**核心文件路径**:
- 令牌创建: `packages/lib/server-only/public-api/create-api-token.ts`
- 令牌验证: `packages/lib/server-only/public-api/get-api-token-by-token.ts`
- API V1 中间件: `packages/api/v1/middleware/authenticated.ts`
- tRPC 中间件: `packages/trpc/server/trpc.ts`
- 错误处理: `packages/lib/errors/app-error.ts`
- tRPC 错误处理器: `packages/trpc/utils/trpc-error-handler.ts`
- OpenAPI 处理器: `packages/trpc/utils/openapi-fetch-handler.ts`

---

## 2. 两条请求拦截路径对比

### 2.1 API V1 路径 (独立中间件)

**路径**: `packages/api/v1/middleware/authenticated.ts`

```typescript
export const authenticatedMiddleware = <T, R>(handler: ...) => {
  return async (args: T, { request }: B) => {
    try {
      // 1. 从 Authorization header 提取 token
      const { authorization } = args.headers;
      const [token] = (authorization || '').split('Bearer ').filter((s) => s.length > 0);

      if (!token) {
        throw new AppError(AppErrorCode.UNAUTHORIZED, {
          message: 'API token was not provided',
        });
      }

      // 2. 验证 token 有效性 (含过期检查)
      const apiToken = await getApiTokenByToken({ token });

      // 3. 检查用户是否被禁用 ✅
      if (apiToken.user.disabled) {
        throw new AppError(AppErrorCode.UNAUTHORIZED, {
          message: 'User is disabled',
        });
      }

      // 4. 注入用户/团队信息到 handler
      return await handler({ ...args, req: request }, apiToken.user, apiToken.team, options);
    } catch (err) {
      // 5. 统一错误返回: 硬编码 401 状态码
      let message = 'Unauthorized';
      if (err instanceof AppError) {
        message = err.message;
      }

      return {
        status: 401,
        body: { message },
      } as const;
    }
  };
};
```

**关键特性**:
- ✅ 独立的 try-catch 错误处理
- ✅ 所有认证失败统一返回 **HTTP 401**（硬编码）
- ✅ 检查 `user.disabled` 状态
- ✅ 不使用 `AppError.toRestAPIError()` 进行错误码映射
- ✅ 日志记录请求元数据和用户信息

---

### 2.2 tRPC/API V2 路径 (tRPC 中间件)

**路径**: `packages/trpc/server/trpc.ts`

```typescript
export const authenticatedMiddleware = t.middleware(async ({ ctx, next, path, meta }) => {
  const authorizationHeader = ctx.req.headers.get('authorization');

  const isApiV2 = Boolean(meta?.openapi?.path);

  // 分支 1: 有 Authorization 头且是 API V2 请求
  if (authorizationHeader && isApiV2) {
    const [token] = (authorizationHeader || '').split('Bearer ').filter((s) => s.length > 0);

    if (!token) {
      // 子场景 B: 有头但 token 为空 → 抛出普通 Error
      throw new Error('Token was not provided for authenticated middleware');
    }

    const apiToken = await getApiTokenByToken({ token });

    // ⚠️ 注意: 这里缺少 user.disabled 检查！
    // ❌ 没有 if (apiToken.user.disabled) 的检查逻辑

    return await next({
      ctx: {
        ...ctx,
        user: apiToken.user,
        teamId: apiToken.teamId,
        session: null,
        metadata: { ...ctx.metadata, auth: 'api' },
      },
    });
  }

  // 分支 2: 无 Authorization 头 或 不是 API V2 请求
  // 子场景 A: 无 Authorization 头 → 进入 Session 检查分支
  if (!ctx.session) {
    throw new TRPCError({
      code: 'UNAUTHORIZED',
      message: 'Invalid session or API token.',
    });
  }

  return await next({ /* session 上下文 */ });
});
```

**tRPC 错误格式化器**:
```typescript
const t = initTRPC.meta<TrpcRouteMeta>().context<TrpcContext>().create({
  transformer: dataTransformer,
  errorFormatter(opts) {
    const { shape, error, ctx } = opts;
    const originalError = error.cause;

    let data: Record<string, unknown> = shape.data;

    // 如果原始错误是 AppError
    if (originalError instanceof AppError) {
      if (originalError.headers && ctx) {
        // 设置响应头
      }

      data = {
        ...data,
        appError: AppError.toJSON(originalError),
        code: originalError.code,
        // 优先级: 自定义 statusCode > 映射表 > 默认 400
        httpStatus: originalError.statusCode ?? 
                    genericErrorCodeToTrpcErrorCodeMap[originalError.code]?.status ?? 
                    400,
      };
    }

    return { ...shape, data };
  },
});
```

**关键特性**:
- ✅ 通过 `meta.openapi.path` 区分 API V2 和 tRPC 内部调用
- ✅ 共享相同的 `getApiTokenByToken()` 验证逻辑
- ✅ 使用 tRPC 错误格式化器进行状态码映射
- ✅ 支持 `AppError` 中的自定义 `statusCode` 覆盖映射表
- ⚠️ 两条分支处理 token 缺失的方式不同
- ❌ **缺少 `user.disabled` 检查！已禁用用户可以正常通过认证**

---

## 3. 令牌创建机制

**路径**: `packages/lib/server-only/public-api/create-api-token.ts`

```typescript
export const createApiToken = async ({ userId, teamId, tokenName, expiresIn }) => {
  // 1. 生成原始 token: api_ 前缀 + 16位随机字符
  const apiToken = `api_${alphaid(16)}`;
  
  // 2. SHA512 哈希存储 (数据库不存明文)
  const hashedToken = hashString(apiToken);

  // 3. 权限检查: 验证用户是否有 MANAGE_TEAM 权限
  const team = await prisma.team.findFirst({
    where: buildTeamWhereQuery({
      teamId,
      userId,
      roles: TEAM_MEMBER_ROLE_PERMISSIONS_MAP['MANAGE_TEAM'],
    }),
  });

  if (!team) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, {
      message: 'You do not have permission to create a token for this team',
    });
  }

  // 4. 存储到数据库
  const storedToken = await prisma.apiToken.create({
    data: {
      name: tokenName,
      token: hashedToken,
      expires: expiresIn ? DateTime.now().plus(timeConstants[expiresIn]).toJSDate() : null,
      userId,
      teamId,
    },
  });

  // 仅创建时返回原始 token 明文
  return { id: storedToken.id, token: apiToken };
};
```

---

## 4. 令牌验证与过期处理

### 4.1 核心验证函数

**路径**: `packages/lib/server-only/public-api/get-api-token-by-token.ts`

```typescript
export const getApiTokenByToken = async ({ token }: { token: string }) => {
  // 1. 哈希输入 token 进行查询
  const hashedToken = hashString(token);

  // 2. 数据库查询
  const apiToken = await prisma.apiToken.findFirst({
    where: { token: hashedToken },
    include: {
      team: { include: { organisation: { include: { owner: true } } } },
      user: { select: { id: true, name: true, email: true, disabled: true } },
    },
  });

  // 3. Token 不存在: 抛出 UNAUTHORIZED, 显式设置 statusCode: 401
  if (!apiToken) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, {
      message: 'Invalid token',
      statusCode: 401,  // ✅ 显式设置
    });
  }

  // 4. Token 过期: 抛出 EXPIRED_CODE, 显式设置 statusCode: 401
  if (apiToken.expires && apiToken.expires < new Date()) {
    throw new AppError(AppErrorCode.EXPIRED_CODE, {
      message: 'Expired token',
      statusCode: 401,  // ✅ 显式设置
    });
  }

  // 5. 兼容旧数据: 团队 token 没有 user 时用组织 owner
  if (apiToken.team && !apiToken.user) {
    apiToken.user = apiToken.team.organisation.owner;
  }

  const { user } = apiToken;

  if (!user) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, {
      message: 'Invalid token',
      statusCode: 401,
    });
  }

  return { ...apiToken, user };
};
```

### 4.2 过期令牌状态码来源分析

| 位置 | EXPIRED_CODE 映射 | 优先级 | 说明 |
|------|-------------------|--------|------|
| `getApiTokenByToken.ts:48-53` | **401** (显式设置) | 最高 | `throw new AppError(EXPIRED_CODE, { statusCode: 401 })` |
| `app-error.ts:35` | **400** (映射表) | 中等 | `genericErrorCodeToTrpcErrorCodeMap[EXPIRED_CODE] = { status: 400 }` |
| `app-error.ts:228-256` | **500** (otherwise) | 最低 | `toRestAPIError()` 未处理 EXPIRED_CODE |

**状态码解析优先级 (tRPC 路径)**:
```
AppError.statusCode (401) 
  > genericErrorCodeToTrpcErrorCodeMap[code].status (400) 
    > 默认值 400
```

✅ **实际结果**: 过期令牌在两条路径中均返回 **HTTP 401**

---

## 5. 失败返回对齐对比表（逐条代码验证）

### 5.1 tRPC/API V2 Token 缺失双子场景深度分析

#### 子场景 A: 无 Authorization 头

**触发条件**:
- `authorizationHeader = null` (请求没有 Authorization 头)
- `isApiV2 = true`

**执行路径**:
```
trpc.ts:86
  if (authorizationHeader && isApiV2)
  → null && true → false
  → 跳过 API Token 认证分支
  ↓
trpc.ts:127
  if (!ctx.session)
  → !null → true
  ↓
trpc.ts:128-131
  throw new TRPCError({
    code: 'UNAUTHORIZED',
    message: 'Invalid session or API token.',
  });
```

**最终结果**:
- 抛出 `TRPCError`，code = `UNAUTHORIZED`
- tRPC 框架映射为 **HTTP 401 Unauthorized**

---

#### 子场景 B: 有 Authorization 头但 token 为空

**触发条件**:
- `authorizationHeader = "Bearer "` 或 `""` 或其他空值
- `isApiV2 = true`

**执行路径**:
```
trpc.ts:86
  if (authorizationHeader && isApiV2)
  → "Bearer " && true → true
  → 进入 API Token 认证分支
  ↓
trpc.ts:88
  const [token] = (authorizationHeader || '').split('Bearer ').filter((s) => s.length > 0);
  例: "Bearer " → ["", ""] → filter → [] → token = undefined
  ↓
trpc.ts:90-91
  if (!token) → true
  throw new Error('Token was not provided for authenticated middleware');
  ↓
  抛出普通 Error，不是 AppError 或 TRPCError
  ↓
tRPC errorFormatter:
  originalError instanceof AppError? → false
  → httpStatus 未被 AppError 逻辑设置
  → TRPCError code 默认为 INTERNAL_SERVER_ERROR
```

**最终结果**:
- 抛出普通 `Error`
- errorFormatter 不识别
- 映射为 **HTTP 500 Internal Server Error**

---

### 5.2 逐条代码核对结果（含子场景拆分）

| 错误场景 | API V1 路径 (代码位置) | tRPC/API V2 路径 (代码位置) | 实际返回 | 一致性 |
|---------|----------------------|---------------------------|---------|--------|
| **Token 缺失 (子场景 A: 无 Authorization 头)** | `authenticated.ts:57-60` 抛出 `AppError(UNAUTHORIZED)`，catch 块硬编码返回 401 | `trpc.ts:127-131` 进入 Session 分支，抛出 `TRPCError(UNAUTHORIZED)` | 401 | ✅ 一致 |
| **Token 缺失 (子场景 B: 有头但 token 为空)** | `authenticated.ts:57-60` 抛出 `AppError(UNAUTHORIZED)`，catch 块硬编码返回 401 | `trpc.ts:90-91` 抛出 `Error("Token was not provided...")`，Error 不是 AppError，进入 default 分支 | V1: 401<br>tRPC: 500 | ❌ 不一致 |
| **Token 无效** | `getApiTokenByToken.ts:41-46` 抛出 `AppError(UNAUTHORIZED, {statusCode:401})`，V1 catch 块返回 401 | 相同的 `getApiTokenByToken` 抛出，tRPC errorFormatter 识别 `statusCode:401` | 401 | ✅ 一致 |
| **Token 过期** | `getApiTokenByToken.ts:48-53` 抛出 `AppError(EXPIRED_CODE, {statusCode:401})`，V1 catch 块返回 401 | 相同的 `getApiTokenByToken` 抛出，tRPC errorFormatter 识别 `statusCode:401` | 401 | ✅ 一致 |
| **用户被禁用** | `authenticated.ts:65-69` 抛出 `AppError(UNAUTHORIZED)`，catch 块返回 401 | `trpc.ts:94-124` **没有检查** `apiToken.user.disabled`，直接进入 next | V1: 401<br>tRPC: 200 ✅ 通过 | ❌ **严重不一致（安全漏洞）** |
| **无资源权限** | 业务层返回 404 | 业务层返回 404 | 404 | ✅ 一致 |
| **业务逻辑错误** | 400/404/500 | 400/404/500 | 相同 | ✅ 一致 |

### 5.3 关键不一致点深度分析

**问题 1: 子场景 B - token 为空时抛出 Error 而非 TRPCError/AppError**

```typescript
// API V1 路径: authenticated.ts:57-60
throw new AppError(AppErrorCode.UNAUTHORIZED, { message: 'API token was not provided' });
// 被 catch 块捕获 → 返回 401

// tRPC 路径: trpc.ts:90-91
throw new Error('Token was not provided for authenticated middleware');
// error.cause 是 Error 而非 AppError/TRPCError → errorFormatter 不识别
// → 返回 INTERNAL_SERVER_ERROR (500)
```

**问题 2: tRPC 路径缺少 user.disabled 检查 - 安全漏洞！**

```typescript
// API V1 路径: authenticated.ts:65-69
if (apiToken.user.disabled) {
  throw new AppError(AppErrorCode.UNAUTHORIZED, { message: 'User is disabled' });
}
// → 禁用用户被拦截，返回 401

// tRPC 路径: trpc.ts:94-124
const apiToken = await getApiTokenByToken({ token });
// ❌ 这里缺少 user.disabled 检查！
return await next({
  ctx: {
    ...ctx,
    user: apiToken.user,  // 可能是 disabled = true 的用户！
    teamId: apiToken.teamId,
    ...
  },
});
// → 禁用用户可以正常调用 API！
```

**影响**: 已被管理员禁用的用户仍然可以通过 API V2 路径使用旧的 API Token 访问系统。

---

## 6. 错误处理流程详解

### 6.1 API V1 错误处理流程

```
请求到达
  ↓
authenticatedMiddleware 拦截
  ↓
┌─────────────────────────────────────────┐
│  try 块                                  │
│  ├─ 提取 token                          │
│  ├─ 验证 token (getApiTokenByToken)    │
│  └─ 检查 user.disabled                  │
└─────────────────────────────────────────┘
  ↓ 成功?
  ├─ 是 → 执行业务 handler
  └─ 否 → catch 块
          ↓
          硬编码返回 { status: 401, body: { message } }
          (忽略 AppError 中的 statusCode)
```

⚠️ **注意**: API V1 中间件 **忽略** `AppError` 中的自定义 `statusCode`，所有认证错误统一返回 401。

### 6.2 tRPC/API V2 错误处理流程

```
请求到达
  ↓
createOpenApiFetchHandler 处理
  ↓
authenticatedMiddleware 拦截
  ↓
┌───────────────────────────────────────────────────────┐
│  有 Authorization 头?                                  │
│  ├─ 是 → 子场景 B: 提取 token → 验证                  │
│  │     ├─ token 空 → 抛出 Error → 500                 │
│  │     ├─ token 无效/过期 → 抛出 AppError → 401       │
│  │     └─ ❌ 缺少 user.disabled 检查                   │
│  └─ 否 → 子场景 A: 进入 Session 分支                  │
│        └─ session 空 → 抛出 TRPCError(UNAUTHORIZED)   │
│           → 401                                       │
└───────────────────────────────────────────────────────┘
  ↓
tRPC errorFormatter 处理错误
  ↓
┌─────────────────────────────────────────┐
│  解析 originalError = error.cause       │
│  └─ 是 AppError?                        │
│     ├─ 是 → 使用 appError.statusCode ?? │
│     │        mappedStatus ?? 400        │
│     └─ 否 → TRPCError 或普通 Error      │
│          → TRPCError code 映射 HTTP 状态码
│          → UNAUTHORIZED → 401
│          → 其他 → 默认可能 500          │
└─────────────────────────────────────────┘
  ↓
返回最终 HTTP 响应
```

✅ **注意**: tRPC 路径 **尊重** `AppError` 中的自定义 `statusCode`，优先级高于映射表。

---

## 7. 权限判定机制

### 7.1 两层认证模型

| 层级 | API V1 | tRPC/API V2 |
|-----|--------|-------------|
| **认证层** | `authenticatedMiddleware` | `authenticatedMiddleware` |
| **Token 验证** | ✅ `getApiTokenByToken()` | ✅ `getApiTokenByToken()` |
| **用户禁用检查** | ✅ 有 | ❌ **无** |
| **Session 验证** | ❌ | ✅ 有 |
| **注入** | `user`, `team` 对象 | `user`, `teamId`, `session`, `metadata` |

### 7.2 业务层权限检查

**文档访问示例**:
```typescript
// 两条路径都通过 userId/teamId 过滤查询
const { envelopeWhereInput } = await getEnvelopeWhereInput({
  id: { type: 'documentId', id: Number(documentId) },
  type: EnvelopeType.DOCUMENT,
  userId: user.id,      // 认证层注入的用户 ID
  teamId: team.id,      // 认证层注入的团队 ID
});
```

**安全策略**: 无权限时返回 **404 Not Found** 而非 **403 Forbidden**，避免泄露资源存在性。

---

## 8. 审计与日志

| 功能 | API V1 | tRPC |
|-----|--------|------|
| 请求 ID | ✅ nanoid 生成 | ✅ alphaid 生成 |
| IP 地址 | ✅ 记录 | ✅ 记录 |
| User Agent | ✅ 记录 | ✅ 记录 |
| 用户 ID | ✅ 记录 | ✅ 记录 |
| Token ID | ✅ 记录 | ✅ 记录 |
| 错误日志 | ❌ 简单 console.log | ✅ `handleTrpcRouterError` 统一处理 |

---

## 9. 问题与修复建议

### 9.1 已发现问题列表

| 问题 | 影响 | 严重程度 | 代码位置 |
|-----|------|---------|---------|
| tRPC 路径缺少 `user.disabled` 检查 | 已禁用用户仍可通过 API Token 访问 | **高** | `packages/trpc/server/trpc.ts:94-124` |
| tRPC 路径 token 为空时抛出 `Error` 而非 `TRPCError`/`AppError` | 返回 500 而非 401 | 中 | `packages/trpc/server/trpc.ts:90-91` |
| API V1 中间件忽略 `AppError.statusCode`，统一返回 401 | 过期 token 无法通过状态码区分 | 低 | `packages/api/v1/middleware/authenticated.ts:108-113` |
| `toRestAPIError()` 未处理 `EXPIRED_CODE`，返回 500 | 业务层直接调用时可能返回错误状态码 | 中 | `packages/lib/errors/app-error.ts:228-256` |

### 9.2 代码修复建议

**建议 1: 在 tRPC 中间件添加 user.disabled 检查** ⚠️ 高优先级
```typescript
// 位置: packages/trpc/server/trpc.ts (第94行之后)
const apiToken = await getApiTokenByToken({ token });

// 新增: 检查用户是否被禁用
if (apiToken.user.disabled) {
  throw new AppError(AppErrorCode.UNAUTHORIZED, {
    message: 'User is disabled',
    statusCode: 401,
  });
}

ctx.logger.info({ ... });
```

**建议 2: 统一 tRPC 中间件的错误抛出类型 - 使用 TRPCError**
```typescript
// 修改前 (第90-91行)
throw new Error('Token was not provided for authenticated middleware');

// 修改后 (与 Session 分支保持一致)
throw new TRPCError({
  code: 'UNAUTHORIZED',
  message: 'API token was not provided.',
});
```

**建议 3: 在 `toRestAPIError()` 中添加 `EXPIRED_CODE` 处理**
```typescript
const status = match(error.code)
  .with(AppErrorCode.INVALID_BODY, ..., () => 400)
  .with(AppErrorCode.UNAUTHORIZED, AppErrorCode.EXPIRED_CODE, () => 401)  // 添加 EXPIRED_CODE
  .with(AppErrorCode.FORBIDDEN, () => 403)
  .with(AppErrorCode.NOT_FOUND, () => 404)
  .with(AppErrorCode.NOT_IMPLEMENTED, () => 501)
  .otherwise(() => 500);
```

---

## 10. 总结

| 对比项 | API V1 路径 | tRPC/API V2 路径 |
|-------|-------------|-----------------|
| **认证入口** | 独立中间件 | tRPC middleware + openapi meta |
| **Token 验证** | `getApiTokenByToken()` | `getApiTokenByToken()` |
| **user.disabled 检查** | ✅ 有 | ❌ **无** |
| **过期令牌状态码** | 401 (硬编码) | 401 (显式设置 statusCode) |
| **错误处理** | try-catch 硬编码 | tRPC errorFormatter 映射 |
| **statusCode 优先级** | 忽略，强制 401 | 自定义 > 映射表 > 默认 |
| **Token 缺失 (无 Authorization 头)** | 401 UNAUTHORIZED | 401 UNAUTHORIZED |
| **Token 缺失 (有头但 token 为空)** | 401 UNAUTHORIZED | ❌ 500 Internal Server Error |
| **安全策略** | 无权限返回 404 | 无权限返回 404 |
| **代码复用** | 独立实现 | 部分复用（注释说明取自 V1） |

**总体一致性**: ⚠️ **存在关键不一致** - 核心验证逻辑共享，但缺少 `user.disabled` 检查可能导致**禁用用户绕过认证**，且 token 为空的子场景返回 500 而非 401。建议优先修复这两个问题。

**最终核对结论**: tRPC/API V2 路径 **没有** `user.disabled` 拦截逻辑，且 token 缺失时有两条不同的执行路径，这是两条路径最大的差异，也是安全隐患。
