# Documenso 口令保护与会话续期机制分析

## 概述

本文档深入分析 Documenso 中"访问口令保护文档下载"机制与"会话续期"之间的关系。通过对照源码，理清访问认证（Access Auth）与操作认证（Action Auth）的严格区分、四种认证模式（ACCOUNT、PASSKEY、PASSWORD、TWO_FACTOR_AUTH）在下载链路中的作用、会话注入和续期触发点、以及各类过期处理的完整流程。

## 一、核心概念澄清

### 1.1 术语对应关系

| 用户术语 | 代码中的实际实现 | 说明 |
|---------|----------------|------|
| 访问口令 | `TWO_FACTOR_AUTH` (Email 2FA) | 文档访问时的邮箱验证码保护 |
| 会话 | `Session` 表 + Cookie | 用户登录后的会话状态 |
| 会话续期 | `validateSessionToken` 中的自动续期逻辑 | 会话到期前自动延长 |

> **重要发现**：代码中没有 "passcode" 术语，用户所说的"访问口令"实际上是文档访问认证中的 `TWO_FACTOR_AUTH`（双因素认证），具体实现为邮箱验证码。

### 1.2 Access Auth vs Action Auth：严格的类型边界

**这是理解整套机制的关键**。Documenso 将文档认证严格分为两类，且**可用的认证类型完全不同**：

| 维度 | Access Auth（访问认证） | Action Auth（操作认证） |
|-----|------------------------|----------------------|
| **用途** | 控制谁能**查看/加载**文档页面 | 控制谁能**签署/操作**字段 |
| **配置字段** | `accessAuth` / `globalAccessAuth` | `actionAuth` / `globalActionAuth` |
| **触发时机** | 页面加载时 (`type: 'ACCESS'`) | 签署字段时 (`type: 'ACTION'`) |
| **完成时额外检查** | `type: 'ACCESS_2FA'`（仅 TWO_FACTOR_AUTH） | 无 |
| **支持的认证类型** | `ACCOUNT`、`TWO_FACTOR_AUTH` | `ACCOUNT`、`PASSKEY`、`PASSWORD`、`TWO_FACTOR_AUTH`、`EXPLICIT_NONE` |

**关键类型定义**（`packages/lib/types/document-auth.ts:53-111`）：

```typescript
// Access Auth 仅支持两种类型
export const ZDocumentAccessAuthTypesSchema = z
  .enum([DocumentAuth.ACCOUNT, DocumentAuth.TWO_FACTOR_AUTH]);

// Action Auth 支持五种类型
export const ZDocumentActionAuthTypesSchema = z
  .enum([DocumentAuth.ACCOUNT, DocumentAuth.PASSKEY,
         DocumentAuth.TWO_FACTOR_AUTH, DocumentAuth.PASSWORD]);

// Recipient Action Auth 额外支持 EXPLICIT_NONE
export const ZRecipientActionAuthTypesSchema = z
  .enum([DocumentAuth.ACCOUNT, DocumentAuth.PASSKEY,
         DocumentAuth.TWO_FACTOR_AUTH, DocumentAuth.PASSWORD,
         DocumentAuth.EXPLICIT_NONE]);
```

> **🔴 核心发现**：`PASSWORD` 和 `PASSKEY` **不能**配置为 Access Auth，只能配置为 Action Auth。它们仅在签署/操作字段时校验，**不影响文档查看和下载**。

### 1.3 认证体系分层

```
┌─────────────────────────────────────────────────────────────────┐
│  用户会话层 (User Session)                                      │
│  - 针对注册用户的登录状态管理                                   │
│  - 基于 Cookie + 数据库 Session 表                              │
│  - 30天有效期，自动续期（剩余<15天时续期30天）                   │
│  - 供 ACCOUNT/PASSKEY/PASSWORD 认证使用                        │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ 完全独立，无直接关联
                         │
┌────────────────────────▼────────────────────────────────────────┐
│  文档访问认证层 (Access Auth)                                   │
│  - 控制谁能加载文档页面                                         │
│  - 仅支持 ACCOUNT 和 TWO_FACTOR_AUTH                            │
│  - ACCOUNT：需登录且邮箱匹配                                     │
│  - TWO_FACTOR_AUTH：页面加载时放行，完成时校验                   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ 同属文档认证但分工不同
                         │
┌────────────────────────▼────────────────────────────────────────┐
│  文档操作认证层 (Action Auth)                                   │
│  - 控制谁能签署/操作字段                                         │
│  - 支持 ACCOUNT/PASSKEY/PASSWORD/TWO_FACTOR_AUTH/EXPLICIT_NONE  │
│  - 仅在操作具体字段时校验                                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、口令校验完整实现流程

### 2.1 认证类型定义

**文件**：`packages/lib/types/document-auth.ts`

```typescript
export const DocumentAuth = {
  ACCOUNT: 'ACCOUNT',
  PASSKEY: 'PASSKEY',
  PASSWORD: 'PASSWORD',
  TWO_FACTOR_AUTH: 'TWO_FACTOR_AUTH',
  EXPLICIT_NONE: 'EXPLICIT_NONE',
} as const;
```

### 2.2 认证方法提取

**文件**：`packages/lib/utils/document-auth.ts:22-45`

`extractDocumentAuthMethods` 函数合并文档级和收件人级的认证配置：

```typescript
const derivedRecipientAccessAuth: TRecipientAccessAuthTypes[] =
  recipientAuthOption.accessAuth.length > 0 
    ? recipientAuthOption.accessAuth 
    : documentAuthOption.globalAccessAuth;
```

**优先级**：收件人级配置 > 文档级全局配置

### 2.3 口令校验核心逻辑

**文件**：`packages/lib/server-only/document/is-recipient-authorized.ts`

`isRecipientAuthorized` 函数支持三种校验类型：

| 类型 | 触发时机 | 适用认证 | 说明 |
|-----|---------|---------|------|
| `ACCESS` | 页面加载时 | ACCOUNT, TWO_FACTOR_AUTH | TWO_FACTOR_AUTH 直接放行（延迟到完成时） |
| `ACCESS_2FA` | 点击"完成"时 | TWO_FACTOR_AUTH | 对 TWO_FACTOR_AUTH 执行真正校验 |
| `ACTION` | 签署/操作字段时 | ACCOUNT, PASSKEY, PASSWORD, TWO_FACTOR_AUTH, EXPLICIT_NONE | 操作认证 |

**关键代码**（第76-79行）：
```typescript
// Early true return for ACCESS auth if all methods are 2FA 
// since validation happens in ACCESS_2FA.
if (type === 'ACCESS' && authMethods.every((method) => method === DocumentAuth.TWO_FACTOR_AUTH)) {
  return true;
}
```

### 2.4 四种认证模式的校验实现

#### 2.4.1 ACCOUNT 认证

```typescript
// is-recipient-authorized.ts:94-105
.with({ type: DocumentAuth.ACCOUNT }, async () => {
  if (!userId) {
    return false;
  }

  const recipientUser = await getUserByEmail(recipient.email);

  if (!recipientUser) {
    return false;
  }

  return recipientUser.id === userId;
})
```

**流程**：
1. 检查是否有登录用户（`userId`）
2. 根据收件人邮箱查找对应用户
3. 比较用户 ID 是否匹配

**作用域**：可用于 Access Auth 和 Action Auth

#### 2.4.2 PASSKEY 认证（仅 Action Auth）

**文件**：`packages/lib/server-only/document/is-recipient-authorized.ts:107-117, 172-273`

```typescript
.with({ type: DocumentAuth.PASSKEY }, async ({ authenticationResponse, tokenReference }) => {
  if (!userId) {
    return false;
  }

  return await isPasskeyAuthValid({
    userId,
    authenticationResponse,
    tokenReference,
  });
})
```

**PASSKEY 校验详细流程**（`isPasskeyAuthValid` 内部）：
1. 调用 `buildPasskeyVerificationToken(userId)` 生成并存储验证令牌
2. 前端通过 WebAuthn API 获取 `authenticationResponse`
3. 后端根据 `credentialId` 和 `userId` 查找 Passkey 记录
4. 删除并验证 `verificationToken`（一次性令牌）
5. 检查令牌是否过期
6. 使用 `verifyAuthenticationResponse` 验证 WebAuthn 响应
7. 更新 Passkey 的 `lastUsedAt` 和 `counter`

**关键过期点**：
```typescript
if (verificationToken.expires < new Date()) {
  throw new AppError(AppErrorCode.EXPIRED_CODE, {
    message: 'Token expired',
  });
}
```

**作用域**：仅用于 Action Auth（签署/操作字段时）

#### 2.4.3 PASSWORD 认证（仅 Action Auth）

**文件**：`packages/lib/server-only/2fa/verify-password.ts`

```typescript
export const verifyPassword = async ({ userId, password }: VerifyPasswordOptions) => {
  const user = await prisma.user.findUnique({
    where: { id: userId },
  });

  if (!user || !user.password) {
    return false;
  }

  return await compare(password, user.password);
};
```

**流程**：
1. 根据 `userId` 查找用户
2. 检查用户是否存在且设置了密码
3. 使用 bcrypt 比较密码哈希

**作用域**：仅用于 Action Auth（签署/操作字段时）

#### 2.4.4 TWO_FACTOR_AUTH (Email 2FA)

**文件**：`packages/lib/server-only/2fa/email/validate-2fa-token-from-email.ts`

```typescript
for (let i = 0; i < window; i++) {
  const counter = Math.floor(now / period); // period = 30秒
  const hotp = await generateHOTP(secret, counter);
  if (code === hotp) return true;
  now -= period;
}
```

**特点**：
- 基于 TOTP（基于时间的一次性密码）
- 密钥与 `email` 和 `envelopeId` 绑定
- 校验窗口：10 个窗口 = 5 分钟（10 × 30秒）
- 不需要存储验证码，使用相同参数即可重新生成校验

**作用域**：可用于 Access Auth 和 Action Auth

### 2.5 2FA Email 验证码生成

**文件**：`packages/lib/server-only/2fa/email/generate-2fa-credentials-from-email.ts`

```typescript
const identity = `email-2fa|v1|email:${email}|id:${envelopeId}`;
const secret = hmac(sha256, DOCUMENSO_ENCRYPTION_KEY, identity);
const uri = createTOTPKeyURI(ISSUER, email, secret);
```

---

## 三、下载链路中的鉴权入口

### 3.1 下载路径总览

Documenso 提供多种下载路径，每条路径有不同的鉴权机制：

```
┌────────────────────────────────────────────────────────────────────┐
│                        下载路径分类                                │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. 收件人下载（Token-based）                                      │
│     ├─ GET /api/files/token/:token/envelopeItem/:id/download      │
│     └─ GET /api/files/token/:token/envelope/:eid/...              │
│     鉴权：仅匹配 recipient token 或 qrToken                      │
│     ❌ 不检查 Access Auth                                          │
│                                                                    │
│  2. 注册用户下载（Session-based）                                  │
│     ├─ GET /api/files/envelope/:eid/envelopeItem/:id/download     │
│     └─ GET /api/files/envelope/:eid/envelopeItem/:id/dataId/...   │
│     鉴权：Session + Team 成员关系                                  │
│     ❌ 不检查 Access Auth                                          │
│                                                                    │
│  3. API V2 下载（API Key-based）                                   │
│     ├─ GET /api/v2/download/envelope/item/:id/download            │
│     └─ GET /api/v2/download/document/:id/download                 │
│     鉴权：API Token + Team 成员关系                                 │
│     ❌ 不检查 Access Auth                                          │
│                                                                    │
│  4. tRPC 下载（Session-based）                                     │
│     └─ authenticatedProcedure 查询                                 │
│     鉴权：Session + Team 成员关系                                  │
│     ❌ 不检查 Access Auth                                          │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 3.2 收件人下载（Token-based）

**文件**：`apps/remix/server/api/files/files.ts:298-348`

```typescript
// 基于收件人 Token 的 PDF 下载
.get('/token/:token/envelopeItem/:envelopeItemId/download/:version?', async (c) => {
  const { token, envelopeItemId, version } = c.req.valid('param');

  // 通过 recipient token 或 qrToken 查询
  let envelopeWhereQuery = {
    id: envelopeItemId,
    envelope: {
      recipients: { some: { token } },
    },
  };

  if (token.startsWith('qr_')) {
    envelopeWhereQuery = {
      id: envelopeItemId,
      envelope: { qrToken: token },
    };
  }

  // ⚠️ 关键：不做任何 Access Auth 检查
  const envelopeItem = await prisma.envelopeItem.findUnique({
    where: envelopeWhereQuery,
    include: { envelope: true, documentData: true },
  });

  if (!envelopeItem) {
    return c.json({ error: 'Envelope item not found' }, 404);
  }
  // ...直接返回文件
});
```

**安全影响**：
- 如果配置了 `ACCOUNT` Access Auth：收件人必须登录才能**查看页面**，但如果拿到了 token 仍可直接**下载**
- 如果配置了 `TWO_FACTOR_AUTH` Access Auth：收件人无需任何认证即可**下载**，但**完成签署**时需要验证码
- `PASSKEY`/`PASSWORD` 作为 Action Auth：不影响下载，只影响签署操作

### 3.3 注册用户下载（Session-based）

**文件**：`apps/remix/server/api/files/files.ts:145-244`

```typescript
.get('/envelope/:envelopeId/envelopeItem/:envelopeItemId/download/:version?', async (c) => {
  // 1. 读取会话
  const session = await getOptionalSession(c);

  if (!session.user) {
    return c.json({ error: 'Unauthorized' }, 401);
  }

  // 2. 检查团队访问权限
  const hasDownloadAccess = await checkEnvelopeFileAccess({
    userId: session.user.id,
    teamId: envelope.teamId,
    envelopeType: envelope.type,
    templateType: envelope.templateType,
  });

  if (!hasDownloadAccess) {
    return c.json({ error: 'User does not have access...' }, 403);
  }
  // ...
});
```

**鉴权流程**：
1. 读取用户会话（`getOptionalSession`）→ 触发 `validateSessionToken` → 可能触发自动续期
2. 验证会话中是否有用户
3. 检查用户是否属于文档所属的团队

### 3.4 API V2 下载（API Key-based）

**文件**：`apps/remix/server/api/download/download.ts:18-212`

```typescript
.get('/envelope/item/:envelopeItemId/download', async (c) => {
  // 1. 提取 API Token
  const authorizationHeader = c.req.header('authorization');
  const [token] = (authorizationHeader || '').split('Bearer ').filter((s) => s.length > 0);

  // 2. 验证 API Token
  const apiToken = await getApiTokenByToken({ token });

  if (apiToken.user.disabled) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, {
      message: 'User is disabled',
    });
  }

  // 3. 检查团队访问权限
  const envelopeItem = await prisma.envelopeItem.findFirst({
    where: {
      id: envelopeItemId,
      envelope: {
        team: buildTeamWhereQuery({ teamId: apiToken.teamId, userId: apiToken.user.id }),
      },
    },
  });
});
```

### 3.5 tRPC 下载（Session-based）

**文件**：`packages/trpc/server/document-router/download-document-beta.ts`

```typescript
export const downloadDocumentBetaRoute = authenticatedProcedure
  .input(ZDownloadDocumentRequestSchema)
  .query(async ({ input, ctx }) => {
    const { teamId, user } = ctx;
    // authenticatedProcedure 已经验证了用户会话
    const envelope = await getEnvelopeById({
      id: { type: 'documentId', id: documentId },
      type: EnvelopeType.DOCUMENT,
      userId: user.id,
      teamId,
    });
    // 返回 S3 预签名 URL
  });
```

### 3.6 下载鉴权入口总结

| 下载路径 | 鉴权方式 | 会话读取 | Access Auth 检查 | Action Auth 检查 |
|---------|---------|---------|-----------------|-----------------|
| 收件人 Token | Token 匹配 | ❌ 否 | ❌ 否 | ❌ 否 |
| 注册用户 | Session + Team | ✅ `getOptionalSession` | ❌ 否 | ❌ 否 |
| API V2 | API Key | ❌ 否 | ❌ 否 | ❌ 否 |
| tRPC | Session | ✅ `authenticatedProcedure` | ❌ 否 | ❌ 否 |

> **🔴 核心发现**：**所有下载路径都不检查文档的 Access Auth 或 Action Auth**。这意味着：
> - Access Auth（ACCOUNT/TWO_FACTOR_AUTH）仅保护**页面加载**，不保护**文件下载**
> - Action Auth（PASSKEY/PASSWORD）仅保护**签署操作**，不保护**文件下载**
> - 拥有收件人 token 的任何人都可以直接下载文档

---

## 四、会话注入机制

### 4.1 会话创建流程

**文件**：`packages/auth/server/lib/utils/authorizer.ts`

```typescript
export const onAuthorize = async (user: AuthorizeUser, c: Context<HonoAuthContext>) => {
  const metadata = c.get('requestMetadata');
  const sessionToken = generateSessionToken();
  await createSession(sessionToken, user.userId, metadata);
  await setSessionCookie(c, sessionToken);
};
```

**会话创建仅在以下场景触发**：
- 用户邮箱密码登录（`email-password.ts`）
- Passkey 登录（`passkey.ts`）
- OAuth 回调（`handle-oauth-callback-url.ts`）

### 4.2 关键发现：文档认证 ≠ 会话注入

**核心结论**：**文档认证（包括 ACCOUNT、PASSKEY、PASSWORD、TWO_FACTOR_AUTH）验证成功后，不会创建或注入用户会话！**

证据：
1. `onAuthorize` 仅在用户主动登录时调用
2. `completeDocumentWithToken` 中进行 TWO_FACTOR_AUTH 校验后，没有调用任何会话创建逻辑
3. `signFieldWithToken` 中进行 PASSKEY/PASSWORD 校验后，没有调用任何会话创建逻辑
4. TWO_FACTOR_AUTH 认证甚至不要求用户拥有 Documenso 账户

### 4.3 PASSWORD/PASSKEY 在签署流程中的完整调用链

```
用户在签署页面点击签名字段
    │
    ▼
  DocumentSigningAuthProvider.executeActionAuthProcedure()
    │
    ├─ 显示认证对话框（PASSKEY/PASSWORD/2FA 选项）
    │
    ▼
  用户选择认证方式并提交
    │
    ├─ PASSKEY → navigator.credentials.get() → WebAuthn 响应
    ├─ PASSWORD → 输入密码
    └─ 2FA → 输入邮箱验证码
    │
    ▼
  signFieldWithToken({ authOptions: { ... } })
    │
    └─► validateFieldAuth()
         └─► isRecipientAuthorized({ type: 'ACTION' })
              │
              ├─ PASSKEY → isPasskeyAuthValid()
              │   ├─ 查找 Passkey 记录
              │   ├─ 验证 verificationToken（一次性）
              │   ├─ 检查过期
              │   └─ verifyAuthenticationResponse()
              │
              ├─ PASSWORD → verifyPassword()
              │   └─ bcrypt.compare()
              │
              └─ TWO_FACTOR_AUTH → validate2FAFromEmail()
                  └─ TOTP 校验
    │
    └─► ⚠️  不会创建或续期用户会话！
```

> **🔴 PASSWORD 和 PASSKEY 的特殊性**：它们需要用户**先登录**（获取 userId），但认证成功后**不续期会话**。如果用户长时间停留在签署页面，会话可能过期，导致后续操作需要重新登录。

### 4.4 ACCOUNT Access Auth 的页面加载流程

```
用户访问 /sign/{token}（配置了 ACCOUNT Access Auth）
    │
    ▼
  V1 Loader (handleV1Loader):
    │
    ├─ getOptionalSession(request) → user
    │
    ├─ extractDocumentAuthMethods() → derivedRecipientAccessAuth
    │
    ├─ isAccessAuthValid = derivedRecipientAccessAuth.every((auth) =>
    │    match(auth)
    │      .with(DocumentAccessAuth.ACCOUNT, () => user && user.email === recipient.email)
    │      .with(DocumentAccessAuth.TWO_FACTOR_AUTH, () => true)
    │      .exhaustive()
    │  )
    │
    └─ 如果 isAccessAuthValid 为 false：
         ├─ 查询 recipientHasAccount
         └─ 返回 isDocumentAccessValid: false → 显示认证页面
    │
    ▼
  V2 Loader (handleV2Loader):
    │
    ├─ getEnvelopeForRecipientSigning({ token, userId })
    │   └─ isRecipientAuthorized({ type: 'ACCESS' })
    │       └─ ACCOUNT → 检查 userId 匹配 recipient.email 对应用户
    │
    ├─ 如果 UNAUTHORIZED → getEnvelopeRequiredAccessData()
    │   └─ 返回 recipientEmail, recipientHasAccount
    │
    └─ 同上 isAccessAuthValid 检查
```

**关键代码**（`apps/remix/app/routes/_recipient+/sign.$token+/_index.tsx:103-113, 217-222`）：
```typescript
// V1 和 V2 Loader 中完全相同的逻辑
const isAccessAuthValid = derivedRecipientAccessAuth.every((accesssAuth) =>
  match(accesssAuth)
    .with(DocumentAccessAuth.ACCOUNT, () => user && user.email === recipient.email)
    .with(DocumentAccessAuth.TWO_FACTOR_AUTH, () => true) // 直接放行
    .exhaustive(),
);
```

> **注意**：此 match 语句仅覆盖 `ACCOUNT` 和 `TWO_FACTOR_AUTH`，因为 Access Auth 的类型定义中只有这两种。PASSWORD 和 PASSKEY 不在 Access Auth 范围内，所以此处不会出现。

---

## 五、会话读取与续期触发点

### 5.1 会话读取入口

会话在以下位置被读取，每次读取都会触发 `validateSessionToken`：

#### 5.1.1 Remix Loader / Action

**文件**：`apps/remix/app/routes/_recipient+/sign.$token+/_index.tsx:48, 175`

```typescript
const { user } = await getOptionalSession(request);
```

#### 5.1.2 API 路由

**文件**：`apps/remix/server/api/files/files.ts:78, 154`

```typescript
// 注册用户下载路由
const session = await getOptionalSession(c);
```

#### 5.1.3 tRPC 上下文

**文件**：`packages/trpc/server/trpc.ts`

```typescript
export const authenticatedProcedure = t.procedure.use(async ({ ctx, next }) => {
  const session = ctx.session;
  if (!session?.user) {
    throw new TRPCError({ code: 'UNAUTHORIZED' });
  }
  return next();
});
```

### 5.2 会话续期触发点

**文件**：`packages/auth/server/lib/session/session.ts:68-119`

```typescript
export const validateSessionToken = async (token: string): Promise<SessionValidationResult> => {
  const sessionId = encodeHexLowerCase(sha256(new TextEncoder().encode(token)));

  const result = await prisma.session.findUnique({
    where: { id: sessionId },
    include: { user: true },
  });

  if (!result) {
    return { session: null, user: null, isAuthenticated: false };
  }

  const { session, user } = result;

  // 检查会话是否过期
  if (Date.now() >= session.expiresAt.getTime()) {
    await prisma.session.delete({ where: { id: sessionId } });
    return { session: null, user: null, isAuthenticated: false };
  }

  // 自动续期：如果剩余时间少于 15 天，续期 30 天
  if (Date.now() >= session.expiresAt.getTime() - 1000 * 60 * 60 * 24 * 15) {
    session.expiresAt = new Date(Date.now() + 1000 * 60 * 60 * 24 * 30);
    
    await prisma.session.update({
      where: { id: session.id },
      data: { expiresAt: session.expiresAt },
    });
  }

  return { session, user, isAuthenticated: true };
};
```

**续期条件**：
- 每次调用 `validateSessionToken` 时检查
- 仅当剩余时间 < 15 天时才续期
- 续期后延长 30 天

### 5.3 文档签署流程中的会话续期时机

```
用户访问 /sign/{token}
    │
    ├─ Loader → getOptionalSession() → validateSessionToken() ✅ 可能续期
    │
    ▼
  页面加载完成
    │
    ├─ 客户端 focus 事件 → refreshSession() → validateSessionToken() ✅ 可能续期
    ├─ 客户端路由变化 → refreshSession() → validateSessionToken() ✅ 可能续期
    │
    ▼
  用户点击"完成"
    │
    ├─ completeDocumentWithToken()
    │   ├─ isRecipientAuthorized({ type: 'ACCESS_2FA' })
    │   │   └─ TWO_FACTOR_AUTH 校验
    │   └─ ❌ 不调用 validateSessionToken → ❌ 不会续期
    │
    ▼
  用户操作签名字段
    │
    ├─ signFieldWithToken()
    │   ├─ isRecipientAuthorized({ type: 'ACTION' })
    │   │   ├─ PASSKEY/PASSWORD 校验（需 userId）
    │   │   └─ ❌ 不调用 validateSessionToken → ❌ 不会续期
    │   └─ ...
    │
    ▼
  用户下载文档（收件人路径）
    │
    └─ GET /api/files/token/:token/... → ❌ 不读会话 → ❌ 不会续期
```

### 5.4 客户端会话刷新

**文件**：`packages/lib/client-only/providers/session.tsx`

前端会在以下时机主动刷新会话：
1. 窗口获得焦点时（`focus` 事件）
2. 路由导航时（`location.pathname` 变化）

```typescript
useEffect(() => {
  const onFocus = () => { void refreshSession(); };
  window.addEventListener('focus', onFocus);
  return () => window.removeEventListener('focus', onFocus);
}, [refreshSession]);

useEffect(() => {
  void refreshSession();
}, [location.pathname]);
```

### 5.5 会话生命周期配置

**文件**：`packages/auth/server/config.ts`

```typescript
export const AUTH_SESSION_LIFETIME = 1000 * 60 * 60 * 24 * 30; // 30 天
```

---

## 六、过期处理逻辑

### 6.1 多层次过期机制

系统中有六种独立的过期时间：

| 过期类型 | 有效期 | 处理方式 | 触发检查点 |
|---------|-------|---------|-----------|
| 2FA Email 验证码 | 5 分钟 | 校验失败，提示重新发送 | 签署/完成时 |
| PASSKEY 验证令牌 | 短暂（一次性） | 校验失败，提示重试 | 操作字段时 |
| 收件人签署窗口 | 可配置（`recipient.expiresAt`） | 重定向到过期页面 | 页面加载时 |
| 用户会话 | 30 天（剩余<15天时续期30天） | 删除会话，需要重新登录 | 每次请求 |
| API Token | 可配置 | 返回 401 | 每次 API 请求 |
| 文档状态 | 无固定期限 | 完成/拒绝后无法访问 | 操作文档时 |

### 6.2 2FA 验证码过期

**文件**：`packages/lib/server-only/2fa/email/constants.ts`
```typescript
export const TWO_FACTOR_EMAIL_EXPIRATION_MINUTES = 5;
```

虽然前端显示倒计时，但实际校验是通过 TOTP 时间窗口实现的，验证码在发送后约 5 分钟内有效。

### 6.3 PASSKEY 验证令牌过期

**文件**：`packages/lib/server-only/document/is-recipient-authorized.ts:238-241`

```typescript
if (verificationToken.expires < new Date()) {
  throw new AppError(AppErrorCode.EXPIRED_CODE, {
    message: 'Token expired',
  });
}
```

PASSKEY 认证使用一次性验证令牌，过期后需要重新生成。令牌在 `buildPasskeyVerificationToken` 中创建并存储。

### 6.4 收件人签署过期

**文件**：`packages/lib/utils/recipients.ts:118-133`

```typescript
export const isRecipientExpired = (recipient: { expiresAt: Date | null }) => {
  return Boolean(recipient.expiresAt && new Date(recipient.expiresAt) <= new Date());
};

export const assertRecipientNotExpired = (recipient: { expiresAt: Date | null }) => {
  if (isRecipientExpired(recipient)) {
    throw new AppError(AppErrorCode.RECIPIENT_EXPIRED, {
      message: 'Recipient signing window has expired',
    });
  }
};
```

在以下位置检查：
- 文档加载时（`getEnvelopeForRecipientSigning`）
- 签署字段时（`signFieldWithToken`）
- 完成文档时（`completeDocumentWithToken`）

### 6.5 会话过期

**文件**：`packages/auth/server/lib/session/session.ts:100-103`

```typescript
if (Date.now() >= session.expiresAt.getTime()) {
  await prisma.session.delete({ where: { id: sessionId } });
  return { session: null, user: null, isAuthenticated: false };
}
```

### 6.6 文档状态过期

**文件**：`packages/lib/server-only/field/sign-field-with-token.ts:104-109`

```typescript
if (envelope.deletedAt) {
  throw new Error(`Document ${envelope.id} has been deleted`);
}

if (envelope.status !== DocumentStatus.PENDING) {
  throw new Error(`Document ${envelope.id} must be pending for signing`);
}
```

---

## 七、口令保护与会话续期的关系

### 7.1 核心结论：两者完全独立，无直接关联

| 维度 | 口令保护（文档认证） | 会话续期 |
|-----|-------------------|---------|
| **目的** | 保护特定文档的访问/操作 | 保持用户登录状态 |
| **适用对象** | 文档收件人 | 已注册并登录的用户 |
| **生命周期** | 单次操作有效（2FA 5分钟，PASSKEY 一次性） | 30天，自动续期 |
| **存储方式** | 无状态（TOTP）或数据库（Passkey/Password） | Cookie + 数据库 Session 表 |
| **触发时机** | 文档访问/签署/操作时 | 每次请求验证会话时 |
| **关联点** | ❌ 无直接关联 | ❌ 无直接关联 |

### 7.2 各认证模式与会话的关系

| 认证模式 | 认证类型 | 需要会话 | 认证后续期会话 | 影响下载 |
|---------|---------|---------|--------------|---------|
| `ACCOUNT` | Access + Action | ✅ 是 | ❌ 否 | ❌ 不影响（下载不检查 Access Auth） |
| `PASSKEY` | Action Only | ✅ 是 | ❌ 否 | ❌ 不影响（仅影响签署） |
| `PASSWORD` | Action Only | ✅ 是 | ❌ 否 | ❌ 不影响（仅影响签署） |
| `TWO_FACTOR_AUTH` | Access + Action | ❌ 否 | N/A | ❌ 不影响（仅页面加载放行） |
| `EXPLICIT_NONE` | Action Only | ❌ 否 | N/A | ❌ 不影响 |

> **🔴 关键发现**：
> 1. **PASSWORD 和 PASSKEY 仅为 Action Auth**：不影响文档加载和下载，仅在签署/操作字段时校验
> 2. **ACCOUNT Access Auth 不保护下载**：页面加载时需要登录，但拥有 token 仍可绕过下载
> 3. **所有认证成功后都不续期会话**：即使 PASSKEY/PASSWORD 需要用户登录，认证成功后也不会触发 `validateSessionToken`
> 4. **TWO_FACTOR_AUTH 不创建会话**：验证通过后无状态，刷新页面需重新验证

### 7.3 为什么关系"不直观"

造成用户困惑的设计决策：

1. **下载路径完全不鉴权**：所有下载路径都不检查文档认证，仅通过 token 或 session 验证身份
2. **Access Auth 与 Action Auth 分离**：用户可能配置了 PASSWORD/PASSKEY 以为能保护下载，但它们仅保护签署操作
3. **延迟校验**：TWO_FACTOR_AUTH 在页面加载时直接放行，用户能看到文档内容，但签署时才校验
4. **无状态认证**：TWO_FACTOR_AUTH 校验通过后不会创建任何形式的会话，刷新后需要重新验证
5. **认证不续期会话**：PASSKEY/PASSWORD 需要用户登录，但认证成功后不续期会话，长时间停留可能导致会话过期

### 7.4 潜在问题与改进建议

**当前设计的问题**：

1. **下载路径不检查 Access Auth**：收件人获得 token 后可直接下载文档，即使配置了 ACCOUNT Access Auth 也能绕过
2. **Action Auth 不保护下载**：PASSWORD/PASSKEY 仅保护签署，不能保护文档内容
3. **认证后不续期会话**：PASSKEY/PASSWORD 认证成功后，会话可能过期导致需要重新登录
4. **术语不统一**：UI 中的"访问口令"、"访问码"、"验证码"与代码中的 `TWO_FACTOR_AUTH` 对应关系不明确

**可能的改进方向**：

1. 在收件人 token 下载路径中添加 Access Auth 检查
2. PASSKEY/PASSWORD 认证成功后显式调用 `validateSessionToken` 触发续期
3. TWO_FACTOR_AUTH 验证通过后创建短期的"文档访问会话"（与用户会话分离）
4. 统一术语，在 UI 中明确区分"访问认证"和"操作认证"

---

## 八、关键代码位置索引

| 功能模块 | 文件路径 | 关键行 |
|---------|---------|-------|
| **类型定义** | | |
| Access Auth 类型 | `packages/lib/types/document-auth.ts` | 53-59 |
| Action Auth 类型 | `packages/lib/types/document-auth.ts` | 66-76, 96-111 |
| Access/Action 区分 | `packages/lib/types/document-auth.ts` | 48-116 |
| **认证校验** | | |
| 授权校验核心 | `packages/lib/server-only/document/is-recipient-authorized.ts` | 53-273 |
| ACCOUNT 认证 | `packages/lib/server-only/document/is-recipient-authorized.ts` | 94-105 |
| PASSKEY 认证 | `packages/lib/server-only/document/is-recipient-authorized.ts` | 107-117, 172-273 |
| PASSWORD 认证 | `packages/lib/server-only/2fa/verify-password.ts` | 1-19 |
| 2FA 密钥生成 | `packages/lib/server-only/2fa/email/generate-2fa-credentials-from-email.ts` | 20-37 |
| 2FA 校验 | `packages/lib/server-only/2fa/email/validate-2fa-token-from-email.ts` | 13-37 |
| **签署流程** | | |
| 字段签署认证 | `packages/lib/server-only/field/sign-field-with-token.ts` | 174-180 |
| 文档完成时 2FA 校验 | `packages/lib/server-only/document/complete-document-with-token.ts` | 117-173 |
| 字段认证验证 | `packages/lib/server-only/document/validate-field-auth.ts` | 1-50 |
| **页面加载** | | |
| V1 签名页 Loader | `apps/remix/app/routes/_recipient+/sign.$token+/_index.tsx` | 45-168 |
| V2 签名页 Loader | `apps/remix/app/routes/_recipient+/sign.$token+/_index.tsx` | 170-260 |
| Access Auth 有效性检查 | `apps/remix/app/routes/_recipient+/sign.$token+/_index.tsx` | 103-113, 217-222 |
| V1/V2 分发 | `apps/remix/app/routes/_recipient+/sign.$token+/_index.tsx` | 262-309 |
| **下载路径** | | |
| 收件人下载路由 | `apps/remix/server/api/files/files.ts` | 298-348 |
| 用户下载路由 | `apps/remix/server/api/files/files.ts` | 145-244 |
| 下载 URL 构造 | `packages/lib/utils/envelope-download.ts` | 25-41 |
| API V2 下载路由 | `apps/remix/server/api/download/download.ts` | 18-212 |
| tRPC 下载路由 | `packages/trpc/server/document-router/download-document-beta.ts` | 1-93 |
| 文件下载处理 | `apps/remix/server/api/files/files.helpers.ts` | 69-235 |
| **会话管理** | | |
| 会话创建 | `packages/auth/server/lib/utils/authorizer.ts` | 14-21 |
| 会话验证与续期 | `packages/auth/server/lib/session/session.ts` | 68-119 |
| 会话 Cookie 管理 | `packages/auth/server/lib/session/session-cookies.ts` | 46-74 |
| 客户端会话管理 | `packages/lib/client-only/providers/session.tsx` | 57-129 |
| **前端组件** | | |
| 认证 Provider | `apps/remix/app/components/general/document-signing/document-signing-auth-provider.tsx` | 1-100 |
| 认证对话框 | `apps/remix/app/components/general/document-signing/document-signing-auth-dialog.tsx` | 1-177 |
| PASSKEY 认证组件 | `apps/remix/app/components/general/document-signing/document-signing-auth-passkey.tsx` | 1-309 |
| PASSWORD 认证组件 | `apps/remix/app/components/general/document-signing/document-signing-auth-password.tsx` | 1-135 |
| 2FA 表单组件 | `apps/remix/app/components/general/document-signing/access-auth-2fa-form.tsx` | 36-300 |
| 认证页面 | `apps/remix/app/components/general/document-signing/document-signing-auth-page.tsx` | 1-100 |
| 签署完成对话框 | `apps/remix/app/components/general/document-signing/document-signing-complete-dialog.tsx` | 180-188 |

---

## 九、总结

"访问口令保护文档下载"与"会话续期"是 Documenso 中两个**完全独立**的安全机制：

### 9.1 Access Auth vs Action Auth 的关键区分

| | Access Auth | Action Auth |
|--|------------|------------|
| **控制什么** | 文档页面加载 | 字段签署/操作 |
| **可用类型** | `ACCOUNT`, `TWO_FACTOR_AUTH` | `ACCOUNT`, `PASSKEY`, `PASSWORD`, `TWO_FACTOR_AUTH`, `EXPLICIT_NONE` |
| **影响下载** | ❌ 否（下载路径不检查） | ❌ 否（下载路径不检查） |

### 9.2 下载路径完全不受文档认证保护

所有四条下载路径（收件人 Token、注册用户 Session、API V2、tRPC）**均不检查** Access Auth 或 Action Auth。这意味着拥有收件人 token 的任何人都可以直接下载文档。

### 9.3 PASSWORD 和 PASSKEY 仅保护签署操作

- PASSWORD 和 PASSKEY 在类型定义中**仅**为 Action Auth 有效
- 它们仅在签署/操作字段时校验
- 不影响文档加载和下载
- 需要用户先登录（依赖会话），但认证成功后不续期会话

### 9.4 会话续期与文档认证无直接关联

- 会话续期仅在 `validateSessionToken` 被调用时自动触发
- 文档认证（包括 ACCOUNT、PASSKEY、PASSWORD 校验）不调用 `validateSessionToken`
- 客户端通过 `focus` 事件和路由变化主动刷新会话，这是签署页面中唯一的续期触发点

理解 Access Auth / Action Auth 的严格类型边界、下载路径的鉴权缺口、以及会话续期的独立机制，对于排查问题和改进用户体验至关重要。
