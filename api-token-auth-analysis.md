# Documenso API Token 鉴权边界分析报告

## 1. 概述

本报告分析 Documenso 系统中 API Token 的鉴权边界，包括令牌创建、权限判定、请求拦截和失败返回四个核心环节的实现机制与关系。

**核心文件路径：**
- 令牌创建：`packages/lib/server-only/public-api/create-api-token.ts`
- 令牌验证：`packages/lib/server-only/public-api/get-api-token-by-token.ts`
- 认证中间件：`packages/api/v1/middleware/authenticated.ts`
- API 实现：`packages/api/v1/implementation.ts`
- 错误处理：`packages/lib/errors/app-error.ts`
- 哈希工具：`packages/lib/server-only/auth/hash.ts`

---

## 2. 令牌创建机制

### 2.1 Token 生成流程

```typescript
// create-api-token.ts
export const createApiToken = async ({ userId, teamId, tokenName, expiresIn }: CreateApiTokenInput) => {
  // 1. 生成原始 token：api_ 前缀 + 16位随机字符
  const apiToken = `api_${alphaid(16)}`;
  
  // 2. SHA512 哈希存储
  const hashedToken = hashString(apiToken);
  
  // 3. 权限检查：验证用户是否有 MANAGE_TEAM 权限
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

  return {
    id: storedToken.id,
    token: apiToken,  // 仅创建时返回原始 token
  };
};
```

### 2.2 安全特性

| 特性 | 实现方式 | 说明 |
|------|---------|------|
| **Token 格式** | `api_` + 16位随机字符 | 便于识别和区分其他类型 token |
| **存储方式** | SHA512 哈希 | 数据库不存储明文，防止泄露 |
| **权限控制** | MANAGE_TEAM 角色检查 | 只有团队管理员可以创建 API Token |
| **过期支持** | 可选过期时间 | 支持设置 token 有效期 |

---

## 3. 请求拦截机制

### 3.1 认证中间件流程

```typescript
// authenticated.ts
export const authenticatedMiddleware = <T, R>(handler: ...) => {
  return async (args: T, { request }: B) => {
    // 1. 提取请求元数据用于日志
    const requestMetadata = extractRequestMetadata(request);

    try {
      // 2. 从 Authorization header 提取 token
      // 支持两种格式：
      // - Authorization: Bearer api_xxx
      // - Authorization: api_xxx
      const { authorization } = args.headers;
      const [token] = (authorization || '').split('Bearer ').filter((s) => s.length > 0);

      if (!token) {
        throw new AppError(AppErrorCode.UNAUTHORIZED, {
          message: 'API token was not provided',
        });
      }

      // 3. 验证 token 有效性（含过期检查）
      const apiToken = await getApiTokenByToken({ token });

      // 4. 检查用户是否被禁用
      if (apiToken.user.disabled) {
        throw new AppError(AppErrorCode.UNAUTHORIZED, {
          message: 'User is disabled',
        });
      }

      // 5. 注入用户和团队信息到 handler
      return await handler(
        { ...args, req: request },
        apiToken.user,
        apiToken.team,
        { metadata, logger: apiLogger },
      );
    } catch (err) {
      // 6. 统一错误返回
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

### 3.2 Token 验证函数

```typescript
// get-api-token-by-token.ts
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

  // 3. 不存在则抛出 401
  if (!apiToken) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, {
      message: 'Invalid token',
      statusCode: 401,
    });
  }

  // 4. 检查是否过期
  if (apiToken.expires && apiToken.expires < new Date()) {
    throw new AppError(AppErrorCode.EXPIRED_CODE, {
      message: 'Expired token',
      statusCode: 401,
    });
  }

  return apiToken;
};
```

---

## 4. 权限判定机制

### 4.1 两层权限模型

**第一层：认证中间件（全局）**
- 验证 Token 有效性
- 验证用户状态（未禁用）
- 注入 `user` 和 `team` 对象

**第二层：业务逻辑权限（端点级）**
- 每个 API 端点根据 `userId` 和 `teamId` 进行资源访问控制
- 使用 `buildTeamWhereQuery` 构建团队权限查询
- 验证用户对具体资源的所有权

### 4.2 示例：文档访问控制

```typescript
// implementation.ts - getDocument 端点
getDocument: authenticatedMiddleware(async (args, user, team) => {
  const { id: documentId } = args.params;

  try {
    // 使用 getEnvelopeWhereInput 构建权限查询
    const { envelopeWhereInput } = await getEnvelopeWhereInput({
      id: { type: 'documentId', id: Number(documentId) },
      type: EnvelopeType.DOCUMENT,
      userId: user.id,      // 注入的用户ID
      teamId: team.id,      // 注入的团队ID
    });

    const envelope = await prisma.envelope.findFirstOrThrow({
      where: envelopeWhereInput,  // 权限条件已包含在查询中
      // ...
    });

    return { status: 200, body: { ... } };
  } catch (err) {
    // 查询失败返回 404（隐藏资源存在性）
    return { status: 404, body: { message: 'Document not found' } };
  }
});
```

### 4.3 团队权限映射

```typescript
// 权限检查使用 TEAM_MEMBER_ROLE_PERMISSIONS_MAP
// 例如：创建 API Token 需要 MANAGE_TEAM 权限
const team = await prisma.team.findFirst({
  where: buildTeamWhereQuery({
    teamId,
    userId,
    roles: TEAM_MEMBER_ROLE_PERMISSIONS_MAP['MANAGE_TEAM'],
  }),
});
```

---

## 5. 失败返回机制

### 5.1 错误码映射

| AppErrorCode | HTTP 状态码 | 场景 |
|-------------|------------|------|
| `UNAUTHORIZED` | 401 | Token 缺失、无效、用户禁用 |
| `EXPIRED_CODE` | 401 | Token 已过期 |
| `FORBIDDEN` | 403 | 权限不足 |
| `NOT_FOUND` | 404 | 资源不存在或无权限访问 |

### 5.2 统一错误处理

```typescript
// app-error.ts
export class AppError extends Error {
  code: string;
  statusCode?: number;

  static toRestAPIError(err: unknown): {
    status: 400 | 401 | 403 | 404 | 500 | 501;
    body: { message: string };
  } {
    const error = AppError.parseError(err);

    const status = match(error.code)
      .with(AppErrorCode.INVALID_BODY, ..., () => 400)
      .with(AppErrorCode.UNAUTHORIZED, () => 401)
      .with(AppErrorCode.FORBIDDEN, () => 403)
      .with(AppErrorCode.NOT_FOUND, () => 404)
      .with(AppErrorCode.NOT_IMPLEMENTED, () => 501)
      .otherwise(() => 500);

    return {
      status,
      body: {
        message: status !== 500 ? error.message : 'Something went wrong',
      },
    };
  }
}
```

### 5.3 "模糊化"安全策略

对于无权限访问的资源，系统返回 **404 Not Found** 而非 **403 Forbidden**，避免泄露资源存在性信息。

```typescript
// 示例：用户B尝试访问用户A的文档
// 返回 404，而非 403
expect(resB.status()).toBe(404);
```

---

## 6. 四者关系图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        API Token 鉴权流程                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────┐    │
│  │  令牌创建    │────▶│  请求拦截    │────▶│   权限判定       │    │
│  │  (Create)    │     │  (Middleware)│     │  (Authorization) │    │
│  └──────────────┘     └──────────────┘     └──────────────────┘    │
│         │                      │                      │             │
│         ▼                      ▼                      ▼             │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────┐    │
│  │  SHA512 哈希 │     │ Token 提取   │     │  资源所有权检查  │    │
│  │  MANAGE_TEAM │     │ 过期检查     │     │  团队角色验证    │    │
│  │  权限检查    │     │ 用户状态检查  │     │  业务逻辑权限    │    │
│  └──────────────┘     └──────────────┘     └──────────────────┘    │
│                              │                      │               │
│                              ▼                      ▼               │
│                        ┌──────────────────────────────────┐         │
│                        │         失败返回机制             │         │
│                        │  (AppError.toRestAPIError)      │         │
│                        │  - 401: 认证失败                │         │
│                        │  - 404: 资源不存在/无权限        │         │
│                        │  - 500: 服务器错误               │         │
│                        └──────────────────────────────────┘         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. 关键安全边界总结

### 7.1 认证边界（Authentication）

| 检查点 | 位置 | 说明 |
|-------|------|------|
| Token 存在性 | `authenticated.ts:55-61` | Authorization header 必须包含有效 token |
| Token 有效性 | `get-api-token-by-token.ts:9-46` | SHA512 哈希匹配数据库记录 |
| Token 过期检查 | `get-api-token-by-token.ts:48-53` | `expires` 字段与当前时间比较 |
| 用户禁用状态 | `authenticated.ts:65-69` | `user.disabled` 必须为 false |

### 7.2 授权边界（Authorization）

| 检查点 | 位置 | 说明 |
|-------|------|------|
| 团队成员角色 | `create-api-token.ts:31-43` | 创建 token 需要 MANAGE_TEAM 权限 |
| 资源所有权 | 各 API 端点 | 通过 `userId`/`teamId` 过滤查询结果 |
| 操作权限 | 业务逻辑层 | 文档发送、删除等操作的权限检查 |

### 7.3 审计与日志边界

```typescript
// authenticated.ts - 请求日志
const apiLogger = logger.child({
  ipAddress: requestMetadata.ipAddress,
  userAgent: requestMetadata.userAgent,
  requestId: nanoid(),
});

apiLogger.info({
  auth: 'api',
  source: 'apiV1',
  userId: apiToken.user.id,
  apiTokenId: apiToken.id,
  path: request.url,
});
```

---

## 8. E2E 测试验证要点

从 `test-unauthorized-api-access.spec.ts` 中验证的安全场景：

| 场景 | 预期结果 |
|------|---------|
| 用户B访问用户A的文档 | 404 Not Found |
| 用户B下载用户A的文档 | 404 Not Found |
| 用户B删除用户A的文档 | 404 Not Found |
| 用户B操作用户A的收件人 | 401 Unauthorized |
| 用户B操作用户A的模板 | 404 Not Found |
| 用户列表只能看到自己的文档 | 200 OK，文档列表过滤正确 |

---

## 9. 结论

Documenso 的 API Token 鉴权体系设计遵循以下安全原则：

1. **最小权限原则**：Token 权限与创建者用户/团队绑定
2. **防御性编程**：无权限返回 404 而非 403，避免信息泄露
3. **分层验证**：中间件层认证 + 业务层授权，双重保障
4. **安全存储**：SHA512 哈希存储，明文仅在创建时返回一次
5. **可观测性**：完整的请求日志和审计追踪

该设计在保障 API 安全的同时，保持了良好的可维护性和扩展性。
