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
      // 支持两种格式:
      // - Authorization: Bearer api_xxx
      // - Authorization: api_xxx
      const { authorization } = args.headers;
      const [token] = (authorization || '').split('Bearer ').filter((s) => s.length > 0);

      if (!token) {
        throw new AppError(AppErrorCode.UNAUTHORIZED, {
          message: 'API token was not provided',
        });
      }

      // 2. 验证 token 有效性 (含过期检查)
      const apiToken = await getApiTokenByToken({ token });

      // 3. 检查用户是否被禁用
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
- ✅ 不使用 `AppError.toRestAPIError()` 进行错误码映射
- ✅ 日志记录请求元数据和用户信息

---

### 2.2 tRPC/API V2 路径 (tRPC 中间件)

**路径**: `packages/trpc/server/trpc.ts`

```typescript
export const authenticatedMiddleware = t.middleware(async ({ ctx, next, path, meta }) => {
  const authorizationHeader = ctx.req.headers.get('authorization');

  // 通过 meta.openapi.path 判断是否为 API V2 请求
  const isApiV2 = Boolean(meta?.openapi?.path);

  if (authorizationHeader && isApiV2) {
    // 1. 提取 token (与 V1 相同逻辑)
    const [token] = (authorizationHeader || '').split('Bearer ').filter((s) => s.length > 0);

    if (!token) {
      // ⚠️ 注意: 这里抛出的是通用 Error 而非 AppError
      throw new Error('Token was not provided for authenticated middleware');
    }

    // 2. 验证 token (调用相同的 getApiTokenByToken)
    const apiToken = await getApiTokenByToken({ token });

    // 3. 注入用户/团队信息
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

  // 非 API V2 请求: 走 Session 认证路径
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
- ⚠️ token 缺失时抛出 `Error` 而非 `AppError`（可能导致 500）

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

  return apiToken;
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

✅ **实际结果**: 过期令牌在 tRPC 路径中返回 **HTTP 401**（因为 `getApiTokenByToken` 显式设置了 `statusCode: 401`）

---

## 5. 失败返回对齐对比表

| 错误场景 | API V1 路径 | tRPC/API V2 路径 | 一致性 |
|---------|-------------|-----------------|--------|
| **Token 缺失** | 401 `UNAUTHORIZED` | ❓ 抛出 `Error` (可能 500) | ❌ 不一致 |
| **Token 无效** | 401 `UNAUTHORIZED` | 401 `UNAUTHORIZED` | ✅ 一致 |
| **Token 过期** | 401 `EXPIRED_CODE` | 401 `EXPIRED_CODE` | ✅ 一致 |
| **用户被禁用** | 401 `UNAUTHORIZED` | 401 `UNAUTHORIZED` | ✅ 一致 |
| **无资源权限** | 404 `NOT_FOUND` | 404 `NOT_FOUND` | ✅ 一致 |
| **业务逻辑错误** | 400/404/500 | 400/404/500 | ✅ 一致 |

### 5.1 关键不一致点分析

**问题 1: Token 缺失时的错误类型不一致**

```typescript
// API V1 路径: 抛出 AppError
throw new AppError(AppErrorCode.UNAUTHORIZED, { message: 'API token was not provided' });

// tRPC 路径: 抛出通用 Error
throw new Error('Token was not provided for authenticated middleware');
```

**影响**: tRPC 路径中 token 缺失时可能返回 **HTTP 500 INTERNAL_SERVER_ERROR** 而非 401。

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
│  └─ 检查用户状态                        │
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
┌─────────────────────────────────────────┐
│  token 验证 (getApiTokenByToken)        │
│  └─ 失败 → 抛出 AppError                │
└─────────────────────────────────────────┘
  ↓
tRPC errorFormatter 处理错误
  ↓
┌─────────────────────────────────────────┐
│  解析 originalError = error.cause       │
│  └─ 是 AppError?                        │
│     ├─ 是 → 使用 appError.statusCode ?? │
│     │        mappedStatus ?? 400        │
│     └─ 否 → 使用 TRPCError.code 映射    │
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
| **职责** | Token 验证、用户状态检查 | Token 验证、Session 验证、用户状态检查 |
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

## 9. 问题与建议

### 9.1 已发现问题

| 问题 | 影响 | 严重程度 |
|-----|------|---------|
| tRPC 路径中 token 缺失时抛出 `Error` 而非 `AppError` | 可能返回 500 而非 401 | 中 |
| API V1 中间件忽略 `AppError.statusCode`，统一返回 401 | 过期 token 无法通过状态码区分 | 低 |
| `toRestAPIError()` 未处理 `EXPIRED_CODE`，返回 500 | 业务层直接调用时可能返回错误状态码 | 中 |

### 9.2 修复建议

**建议 1: 统一 tRPC 中间件的错误抛出类型**
```typescript
// 修改前
throw new Error('Token was not provided for authenticated middleware');

// 修改后
throw new AppError(AppErrorCode.UNAUTHORIZED, {
  message: 'API token was not provided',
  statusCode: 401,
});
```

**建议 2: 在 `toRestAPIError()` 中添加 `EXPIRED_CODE` 处理**
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
| **过期令牌状态码** | 401 (硬编码) | 401 (显式设置 statusCode) |
| **错误处理** | try-catch 硬编码 | tRPC errorFormatter 映射 |
| **statusCode 优先级** | 忽略，强制 401 | 自定义 > 映射表 > 默认 |
| **安全策略** | 无权限返回 404 | 无权限返回 404 |
| **代码复用** | 独立实现 | 部分复用（注释说明取自 V1） |

**总体一致性**: ✅ **基本一致**，核心验证逻辑共享，过期令牌最终都返回 401。仅在边缘场景（如 token 缺失）存在不一致风险。
