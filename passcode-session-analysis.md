# Documenso 口令保护与会话续期机制分析

## 概述

本文档深入分析 Documenso 中"访问口令保护文档下载"机制与"会话续期"之间的关系。通过对照源码，理清口令校验、会话注入和过期处理的完整流程。

## 一、核心概念澄清

在深入分析之前，需要澄清几个关键概念：

### 1.1 术语对应关系

| 用户术语 | 代码中的实际实现 | 说明 |
|---------|----------------|------|
| 访问口令 | `TWO_FACTOR_AUTH` (Email 2FA) | 文档访问时的邮箱验证码保护 |
| 会话 | `Session` 表 + Cookie | 用户登录后的会话状态 |
| 会话续期 | `validateSessionToken` 中的自动续期逻辑 | 会话到期前自动延长 |

> **重要发现**：代码中没有 "passcode" 术语，用户所说的"访问口令"实际上是文档访问认证中的 `TWO_FACTOR_AUTH`（双因素认证），具体实现为邮箱验证码。

### 1.2 认证体系分层

Documenso 的认证体系分为两层，**完全独立运作**：

```
┌─────────────────────────────────────────────────────────┐
│  用户会话层 (User Session)                             │
│  - 针对注册用户的登录状态管理                          │
│  - 基于 Cookie + 数据库 Session 表                     │
│  - 30天有效期，自动续期                                │
└───────────────────┬─────────────────────────────────────┘
                    │
                    │ 完全独立，无直接关联
                    │
┌───────────────────▼─────────────────────────────────────┐
│  文档访问认证层 (Document Access Auth)                 │
│  - 针对特定文档的访问权限控制                          │
│  - 支持 ACCOUNT (账户) 和 TWO_FACTOR_AUTH (邮箱验证码) │
│  - 每次访问/签署时校验                                 │
└─────────────────────────────────────────────────────────┘
```

---

## 二、口令校验（2FA Email）实现流程

### 2.1 认证类型定义

**文件**：`packages/lib/types/document-auth.ts`

```typescript
// 访问认证支持的类型
export const ZDocumentAccessAuthTypesSchema = z.enum([
  DocumentAuth.ACCOUNT,        // 需要登录匹配邮箱的账户
  DocumentAuth.TWO_FACTOR_AUTH // 需要输入邮箱验证码
]);
```

认证可以配置在两个层级：
- **全局文档级**：`globalAccessAuth` - 对所有收件人生效
- **收件人级**：`accessAuth` - 仅对特定收件人生效

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

| 类型 | 触发时机 | 2FA 处理方式 |
|-----|---------|-------------|
| `ACCESS` | 页面加载时 | **直接返回 true**（真正校验延迟到完成时） |
| `ACCESS_2FA` | 点击"完成"时 | 真正执行 2FA 校验 |
| `ACTION` | 签署操作时 | 执行操作认证校验 |

**关键代码**（第76-79行）：
```typescript
// Early true return for ACCESS auth if all methods are 2FA 
// since validation happens in ACCESS_2FA.
if (type === 'ACCESS' && authMethods.every((method) => method === DocumentAuth.TWO_FACTOR_AUTH)) {
  return true;
}
```

> 💡 **不直观的设计**：页面加载时的 `ACCESS` 校验对 2FA 直接放行，用户看到文档内容但实际认证在点击"完成"时才执行。

### 2.4 2FA Email 验证码生成与校验

#### 验证码生成
**文件**：`packages/lib/server-only/2fa/email/generate-2fa-credentials-from-email.ts`

使用基于时间的一次性密码（TOTP）算法：

```typescript
const identity = `email-2fa|v1|email:${email}|id:${envelopeId}`;
const secret = hmac(sha256, DOCUMENSO_ENCRYPTION_KEY, identity);
const uri = createTOTPKeyURI(ISSUER, email, secret);
```

**特点**：
- 密钥与 `email` 和 `envelopeId` 绑定
- 不需要存储验证码，使用相同参数即可重新生成校验

#### 验证码校验
**文件**：`packages/lib/server-only/2fa/email/validate-2fa-token-from-email.ts`

```typescript
for (let i = 0; i < window; i++) {
  const counter = Math.floor(now / period); // period = 30秒
  const hotp = await generateHOTP(secret, counter);
  if (code === hotp) return true;
  now -= period;
}
```

**校验窗口**：默认 10 个窗口 = 5 分钟（10 × 30秒）

---

## 三、会话注入机制

### 3.1 会话创建流程

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

### 3.2 关键发现：口令验证 ≠ 会话注入

**核心结论**：**访问口令（2FA）验证成功后，不会创建或注入用户会话！**

证据：
1. `onAuthorize` 仅在用户主动登录时调用
2. `completeDocumentWithToken` 中进行 2FA 校验后，没有调用任何会话创建逻辑
3. 2FA 认证甚至不要求用户拥有 Documenso 账户

### 3.3 2FA 访问认证的完整流程

```
用户访问 /sign/{token}
    │
    ▼
  Loader 执行
    │
    ├─► getOptionalSession(request) → 检查是否已有登录会话
    │
    ├─► getEnvelopeForRecipientSigning()
    │    └─► isRecipientAuthorized({ type: 'ACCESS' })
    │         └─► 对 TWO_FACTOR_AUTH 直接返回 true ※
    │
    └─► 返回文档数据给前端（用户此时已能看到文档）
    │
    ▼
  用户填写字段后点击"完成"
    │
    ▼
  检测到需要 2FA → 显示验证码输入框
    │
    ▼
  用户输入验证码 → onTwoFactorFormSubmit()
    │
    ▼
  调用 completeDocumentWithToken()
    │
    └─► isRecipientAuthorized({ type: 'ACCESS_2FA' })
         └─► 真正执行 2FA 校验
         └─► 校验通过 → 完成文档签署
         └─► ⚠️  不会创建用户会话！
```

> 💡 **不直观的设计**：用户在输入"访问口令"并验证成功后，系统不会记住这个认证状态。如果刷新页面或重新访问，需要再次输入验证码。

---

## 四、会话续期机制

### 4.1 会话生命周期配置

**文件**：`packages/auth/server/config.ts`

```typescript
export const AUTH_SESSION_LIFETIME = 1000 * 60 * 60 * 24 * 30; // 30 天
```

### 4.2 自动续期逻辑

**文件**：`packages/auth/server/lib/session/session.ts:105-116`

```typescript
// 如果会话剩余时间少于 15 天，自动续期 30 天
if (Date.now() >= session.expiresAt.getTime() - 1000 * 60 * 60 * 24 * 15) {
  session.expiresAt = new Date(Date.now() + 1000 * 60 * 60 * 24 * 30);
  
  await prisma.session.update({
    where: { id: session.id },
    data: { expiresAt: session.expiresAt },
  });
}
```

**续期触发时机**：每次调用 `validateSessionToken` 时检查

### 4.3 客户端会话刷新

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

---

## 五、过期处理逻辑

### 5.1 多层次过期机制

系统中有四种独立的过期时间：

| 过期类型 | 有效期 | 处理方式 |
|---------|-------|---------|
| 2FA Email 验证码 | 5 分钟 | 校验失败，提示重新发送 |
| 收件人签署窗口 | 可配置（`recipient.expiresAt`） | 重定向到过期页面 |
| 用户会话 | 30 天（自动续期） | 删除会话，需要重新登录 |
| 文档状态 | 无固定期限 | 完成/拒绝后无法访问 |

### 5.2 2FA 验证码过期

**文件**：`packages/lib/server-only/2fa/email/constants.ts`
```typescript
export const TWO_FACTOR_EMAIL_EXPIRATION_MINUTES = 5;
```

虽然前端显示倒计时，但实际校验是通过 TOTP 时间窗口实现的，验证码在发送后约 5 分钟内有效。

### 5.3 收件人签署过期

**文件**：`packages/lib/utils/recipients.ts:118-120`

```typescript
export const isRecipientExpired = (recipient: { expiresAt: Date | null }) => {
  return Boolean(recipient.expiresAt && new Date(recipient.expiresAt) <= new Date());
};
```

在文档加载时检查，过期则重定向到 `/sign/{token}/expired`。

### 5.4 会话过期

**文件**：`packages/auth/server/lib/session/session.ts:100-103`

```typescript
if (Date.now() >= session.expiresAt.getTime()) {
  await prisma.session.delete({ where: { id: sessionId } });
  return { session: null, user: null, isAuthenticated: false };
}
```

---

## 六、口令保护与会话续期的关系

### 6.1 核心结论：两者完全独立，无直接关联

| 维度 | 口令保护（2FA） | 会话续期 |
|-----|----------------|---------|
| **目的** | 保护特定文档的访问 | 保持用户登录状态 |
| **适用对象** | 文档收件人（可以是未注册用户） | 已注册并登录的用户 |
| **生命周期** | 单次操作有效（点击"完成"时校验） | 30天，自动续期 |
| **存储方式** | 无状态（TOTP 基于时间计算） | Cookie + 数据库 Session 表 |
| **触发时机** | 文档访问/签署时 | 每次请求验证会话时 |
| **关联点** | ❌ 无直接关联 | ❌ 无直接关联 |

### 6.2 为什么关系"不直观"

造成用户困惑的设计决策：

1. **延迟校验**：页面加载时不校验 2FA，用户能看到文档内容，但点击"完成"时才校验
2. **无状态认证**：2FA 校验通过后不会创建会话，刷新后需要重新验证
3. **命名混淆**：用户说的"访问口令"在代码中是 `TWO_FACTOR_AUTH`，与"会话"是完全不同的概念
4. **两种认证模式混合**：
   - `ACCOUNT` 类型：依赖用户会话，需要登录匹配邮箱
   - `TWO_FACTOR_AUTH` 类型：不依赖会话，仅需邮箱验证码

### 6.3 ACCOUNT vs TWO_FACTOR_AUTH 对比

| 特性 | ACCOUNT 认证 | TWO_FACTOR_AUTH 认证 |
|-----|-------------|---------------------|
| 需要注册账户 | ✅ 是 | ❌ 否 |
| 依赖会话 | ✅ 是 | ❌ 否 |
| 校验时机 | 页面加载时 | 点击"完成"时 |
| 验证后状态 | 会话保持登录 | 无状态，需重新验证 |
| 自动续期 | ✅ 会话自动续期 | ❌ 无 |

### 6.4 潜在问题与改进建议

**当前设计的问题**：

1. **用户体验不一致**：使用 ACCOUNT 认证的用户登录后可持续访问，使用 2FA 认证的用户每次访问都需要输入验证码
2. **校验时机迷惑**：用户能看到文档内容，但签署时才被要求输入验证码
3. **术语不统一**：UI 中的"访问口令/访问码"与代码中的 `TWO_FACTOR_AUTH` 不一致

**可能的改进方向**：

1. 2FA 验证通过后，可以考虑创建一个短期的"文档访问会话"（与用户登录会话分离）
2. 在页面加载时就进行 2FA 校验，而不是延迟到签署时
3. 统一术语，避免"口令"、"访问码"、"验证码"等多种说法

---

## 七、关键代码位置索引

| 功能模块 | 文件路径 | 关键行 |
|---------|---------|-------|
| 认证类型定义 | `packages/lib/types/document-auth.ts` | 8, 53-59 |
| 认证方法提取 | `packages/lib/utils/document-auth.ts` | 22-45 |
| 授权校验核心 | `packages/lib/server-only/document/is-recipient-authorized.ts` | 53-170 |
| 2FA 密钥生成 | `packages/lib/server-only/2fa/email/generate-2fa-credentials-from-email.ts` | 20-37 |
| 2FA 校验 | `packages/lib/server-only/2fa/email/validate-2fa-token-from-email.ts` | 13-37 |
| 文档完成时校验 | `packages/lib/server-only/document/complete-document-with-token.ts` | 117-173 |
| 会话创建 | `packages/auth/server/lib/utils/authorizer.ts` | 14-21 |
| 会话验证与续期 | `packages/auth/server/lib/session/session.ts` | 68-119 |
| 会话 Cookie 管理 | `packages/auth/server/lib/session/session-cookies.ts` | 46-74 |
| 客户端会话管理 | `packages/lib/client-only/providers/session.tsx` | 57-129 |
| 2FA 表单组件 | `apps/remix/app/components/general/document-signing/access-auth-2fa-form.tsx` | 36-300 |
| 签署完成对话框 | `apps/remix/app/components/general/document-signing/document-signing-complete-dialog.tsx` | 180-188 |
| 签署页面 Loader | `apps/remix/app/routes/_recipient+/sign.$token+/_index.tsx` | 45-309 |

---

## 八、总结

"访问口令保护文档下载"与"会话续期"是 Documenso 中两个**完全独立**的安全机制：

1. **访问口令（2FA）** 是针对特定文档的一次性访问控制，不依赖用户账户和会话
2. **会话续期** 是针对注册用户的登录状态管理，与文档访问无关
3. 两者之间唯一的间接联系是当使用 `ACCOUNT` 类型的访问认证时，需要依赖用户会话来验证身份

理解这种分离设计对于排查问题和改进用户体验至关重要。
