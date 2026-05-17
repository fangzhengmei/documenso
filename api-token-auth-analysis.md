# Documenso API Token 鉴权边界分析报告

## 1. 概述

本报告深入分析 Documenso 系统中 API Token 的鉴权边界，包括 **API V1**、**tRPC authenticated** 和 **tRPC maybeAuthenticated** 三条请求拦截路径、令牌创建机制、权限判定逻辑、失败返回对齐以及过期令牌在不同路径中的状态码来源。

**核心文件路径**:
- 令牌创建: `packages/lib/server-only/public-api/create-api-token.ts`
- 令牌验证: `packages/lib/server-only/public-api/get-api-token-by-token.ts`
- API V1 中间件: `packages/api/v1/middleware/authenticated.ts`
- tRPC 中间件: `packages/trpc/server/trpc.ts` (authenticated + maybeAuthenticated)
- 错误处理: `packages/lib/errors/app-error.ts`
- tRPC 错误处理器: `packages/trpc/utils/trpc-error-handler.ts`
- OpenAPI 处理器: `packages/trpc/utils/openapi-fetch-handler.ts`

---

## 2. 三条请求拦截路径对比

### 2.1 API V1 路径 (独立中间件)

**路径**: `packages/api/v1/middleware/authenticated.ts`

```typescript
export const authenticatedMiddleware = <T, R>(handler: ...) => {
  return async (args: T, { request }: B) => {
    try {
      const { authorization } = args.headers;
      const [token] = (authorization || '').split('Bearer ').filter((s) => s.length > 0);

      if (!token) {
        throw new AppError(AppErrorCode.UNAUTHORIZED, {
          message: 'API token was not provided',
        });
      }

      const apiToken = await getApiTokenByToken({ token });

      if (apiToken.user.disabled) {
        throw new AppError(AppErrorCode.UNAUTHORIZED, {
          message: 'User is disabled',
        });
      }

      return await handler({ ...args, req: request }, apiToken.user, apiToken.team, options);
    } catch (err) {
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
- ✅ token 缺失时直接拦截，返回 401
- ✅ 不使用 `AppError.toRestAPIError()` 进行错误码映射

---

### 2.2 tRPC authenticated 路径

**路径**: `packages/trpc/server/trpc.ts:72-163`

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

**关键特性**:
- ✅ 通过 `meta.openapi.path` 区分 API V2 和 tRPC 内部调用
- ✅ 共享相同的 `getApiTokenByToken()` 验证逻辑
- ⚠️ 两条分支处理 token 缺失的方式不同
- ❌ **缺少 `user.disabled` 检查！已禁用用户可以正常通过认证**

---

### 2.3 tRPC maybeAuthenticated 路径

**路径**: `packages/trpc/server/trpc.ts:165-250`

```typescript
export const maybeAuthenticatedMiddleware = t.middleware(async ({ ctx, next, path, meta }) => {
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
  // ⚠️ 关键差异: 没有 session 检查！直接继续执行！
  return await next({
    ctx: {
      ...ctx,
      user: ctx.user,        // user 可能为 null/undefined
      session: ctx.session,  // session 可能为 null
      metadata: {
        ...ctx.metadata,
        auth: ctx.session ? 'session' : null,  // auth 为 null
      },
    },
  });
});
```

**关键特性**:
- ✅ 与 authenticated 相同的 Token 提取逻辑
- ✅ 与 authenticated 相同的 token 为空错误处理
- ⚠️ **无 Authorization 头时不拦截！直接继续执行，user 可能为 null**
- ❌ **缺少 `user.disabled` 检查！已禁用用户可以正常通过认证**
- ❌ **没有 session 检查！允许匿名访问**

---

## 3. tRPC 错误格式化器

```typescript
const t = initTRPC.meta<TrpcRouteMeta>().context<TrpcContext>().create({
  transformer: dataTransformer,
  errorFormatter(opts) {
    const { shape, error, ctx } = opts;
    const originalError = error.cause;

    let data: Record<string, unknown> = shape.data;

    if (originalError instanceof AppError) {
      if (originalError.headers && ctx) {
        // 设置响应头
      }

      data = {
        ...data,
        appError: AppError.toJSON(originalError),
        code: originalError.code,
        httpStatus: originalError.statusCode ?? 
                    genericErrorCodeToTrpcErrorCodeMap[originalError.code]?.status ?? 
                    400,
      };
    }

    return { ...shape, data };
  },
});
```

---

## 4. 令牌创建与验证

### 4.1 核心验证函数

**路径**: `packages/lib/server-only/public-api/get-api-token-by-token.ts`

```typescript
export const getApiTokenByToken = async ({ token }: { token: string }) => {
  const hashedToken = hashString(token);

  const apiToken = await prisma.apiToken.findFirst({
    where: { token: hashedToken },
    include: {
      team: { include: { organisation: { include: { owner: true } } } },
      user: { select: { id: true, name: true, email: true, disabled: true } },
    },
  });

  if (!apiToken) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, {
      message: 'Invalid token',
      statusCode: 401,
    });
  }

  if (apiToken.expires && apiToken.expires < new Date()) {
    throw new AppError(AppErrorCode.EXPIRED_CODE, {
      message: 'Expired token',
      statusCode: 401,
    });
  }

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

---

## 5. 失败返回对齐对比表（三条路径完整对比）

### 5.1 子场景 A: 无 Authorization 头

**触发条件**:
- `authorizationHeader = null` (请求没有 Authorization 头)
- `isApiV2 = true` (带 openapi meta)

| 路径 | 执行分支 | 实际返回 | 说明 |
|-----|---------|---------|------|
| **API V1** | Token 分支 → token 为空 → 抛出 `AppError(UNAUTHORIZED)` → catch 返回 401 | **401** | 硬编码返回，强制认证 |
| **tRPC authenticated** | Token 分支条件不满足 → 进入 Session 分支 → 抛出 `TRPCError(UNAUTHORIZED)` | **401** | Session 缺失拦截 |
| **tRPC maybeAuthenticated** | Token 分支条件不满足 → 无 Session 检查 → 直接 next | **200** ✅ 通过 | **不拦截！user 为 null，允许匿名访问** |

---

### 5.2 子场景 B: 有 Authorization 头但 token 为空

**触发条件**:
- `authorizationHeader = "Bearer "` 或 `""` 或其他空值
- `isApiV2 = true` (带 openapi meta)

| 路径 | 执行分支 | 实际返回 | 说明 |
|-----|---------|---------|------|
| **API V1** | Token 分支 → token 为空 → 抛出 `AppError(UNAUTHORIZED)` → catch 返回 401 | **401** | 硬编码返回 |
| **tRPC authenticated** | Token 分支 → token 为空 → 抛出普通 `Error` → errorFormatter 不识别 | **500** | 抛出 Error 而非 TRPCError/AppError |
| **tRPC maybeAuthenticated** | Token 分支 → token 为空 → 抛出普通 `Error` → errorFormatter 不识别 | **500** | 与 authenticated 相同的 bug |

---

### 5.3 完整失败返回矩阵

| 错误场景 | API V1 | tRPC authenticated | tRPC maybeAuthenticated | 一致性 |
|---------|--------|-------------------|------------------------|--------|
| **Token 缺失 (无 Authorization 头)** | 401 UNAUTHORIZED | 401 UNAUTHORIZED | **200 通过** | ❌ 不一致 |
| **Token 缺失 (有头但 token 为空)** | 401 UNAUTHORIZED | 500 Internal Error | 500 Internal Error | ❌ V1 vs tRPC |
| **Token 无效** | 401 UNAUTHORIZED | 401 UNAUTHORIZED | 401 UNAUTHORIZED | ✅ 一致 |
| **Token 过期** | 401 UNAUTHORIZED | 401 UNAUTHORIZED | 401 UNAUTHORIZED | ✅ 一致 |
| **用户被禁用** | 401 拦截 | **200 通过** | **200 通过** | ❌ **安全漏洞** |
| **无资源权限** | 404 Not Found | 404 Not Found | 404 Not Found | ✅ 一致 |

---

## 6. 三条路径执行流程图

```
                    请求到达 (带 openapi meta)
                        ↓
              ┌───────────────────────────┐
              │  有 Authorization 头?      │
              └──────────────┬────────────┘
                    ┌───────┴───────────────┐
                    │                        │
                    ▼                        ▼
            ┌───────────────┐        ┌─────────────────┐
            │  有 Header     │        │  无 Header       │
            └───────┬───────┘        └──────────┬──────┘
                    │                               │
                    ▼                               ▼
        ┌───────────────────────────┐    ┌─────────────────────────┐
        │ 提取 token 并检查非空      │    │  API V1: 抛出 AppError   │
        │  token 空 → 抛出 Error     │    │           ↓             │
        │           ↓                │    │  catch → 硬编码返回 401 │
        │  token 非空 → 验证 token   │    └─────────────────────────┘
        │           ↓                │    ┌─────────────────────────┐
        │  token 有效/过期 → 401     │    │ authenticated: 检查 session │
        │           ↓                │    │            ↓             │
        │  token 有效 → 注入 user    │    │ session 空 → TRPCError 401 │
        └───────────────────────────┘    └─────────────────────────┘
                    │                          ┌─────────────────────────┐
                    │                          │ maybeAuthenticated: 无检查│
                    │                          │            ↓             │
                    │                          │ 直接 next，user 为 null  │
                    │                          └─────────────────────────┘
                    ▼
        ┌───────────────────────────────────┐
        │  ⚠️  三条路径都缺少 user.disabled 检查 │
        └───────────────────────────────────┘
```

---

## 7. 问题与修复建议

### 7.1 已发现问题列表

| 问题 | 影响 | 严重程度 | 代码位置 |
|-----|------|---------|---------|
| tRPC 两条路径缺少 `user.disabled` 检查 | 已禁用用户仍可通过 API Token 访问 | **高** | `trpc.ts:94-124` & `192-223` |
| tRPC 两条路径 token 为空时抛出 `Error` 而非 `TRPCError` | 返回 500 而非 401 | 中 | `trpc.ts:90-91` & `188-189` |
| maybeAuthenticated 无 Authorization 头时不拦截 | 匿名用户可以访问需要认证的 API | **高** | `trpc.ts:231-249` |
| API V1 中间件忽略 `AppError.statusCode` | 过期 token 无法通过状态码区分 | 低 | `authenticated.ts:108-113` |

### 7.2 代码修复建议

**建议 1: 在 tRPC 两条中间件添加 user.disabled 检查** ⚠️ 高优先级
```typescript
// 位置: packages/trpc/server/trpc.ts (第94行和第192行之后)
const apiToken = await getApiTokenByToken({ token });

// 新增: 检查用户是否被禁用
if (apiToken.user.disabled) {
  throw new AppError(AppErrorCode.UNAUTHORIZED, {
    message: 'User is disabled',
    statusCode: 401,
  });
}
```

**建议 2: 统一 tRPC 中间件的错误抛出类型 - 使用 TRPCError**
```typescript
// 修改前 (第90-91行和第188-189行)
throw new Error('Token was not provided for authenticated middleware');

// 修改后
throw new TRPCError({
  code: 'UNAUTHORIZED',
  message: 'API token was not provided.',
});
```

**建议 3: maybeAuthenticated 路径行为澄清**
- 如预期为可选认证：文档需明确说明哪些 API 允许匿名访问
- 如预期为强制认证：需在 maybeAuthenticated 中添加 session 检查或统一逻辑

---

## 8. 总结

| 对比项 | API V1 | tRPC authenticated | tRPC maybeAuthenticated |
|-------|--------|-------------------|------------------------|
| **Token 验证** | `getApiTokenByToken()` | `getApiTokenByToken()` | `getApiTokenByToken()` |
| **user.disabled 检查** | ✅ 有 | ❌ 无 | ❌ 无 |
| **无 Header 时行为** | 401 拦截 | 401 Session 拦截 | **200 通过** |
| **有 Header token 空时** | 401 AppError | 500 Error | 500 Error |
| **过期令牌状态码** | 401 (硬编码) | 401 (显式设置) | 401 (显式设置) |
| **Session 支持** | ❌ 无 | ✅ 有 | ✅ 有但不强制 |
| **匿名访问** | ❌ 不允许 | ❌ 不允许 | ✅ 允许 |

**最终核对结论**:
1. `maybeAuthenticated` 在无 Authorization 头时 **不拦截**，直接允许匿名访问（user = null）
2. tRPC 两条路径在 token 为空时都返回 500（bug）
3. 两条 tRPC 路径都缺少 `user.disabled` 检查（安全漏洞）
4. API V1 行为最严格，全部场景都返回 401
