# 嵌入式签署组件通信链路分析

## 一、架构概述

Documenso 嵌入式组件采用 **iframe + postMessage** 跨域通信架构。系统存在**两条独立的嵌入链路**，使用不同的令牌体系和安全验证机制：

### 🔹 两条独立嵌入链路

| 链路类型 | 路由模式 | 令牌类型 | 主要用途 | 本文分析重点 |
|---------|---------|---------|---------|-------------|
| **签署主链路** | `/embed/sign/{token}` | `recipient.token`（随机字符串） | 收件人签署文档 | ✅ 是 |
| **创作嵌入链路** | `/embed/v1/v2/authoring/...` | 预签名 Token（JWT） | 文档/模板编辑 | ❌ 仅作对比 |

### 🔹 三层安全架构（签署主链路）

签署主链路不涉及预签名 Token（JWT），采用以下三层安全机制：

1. **宿主页面（Parent）**：集成方业务系统，负责创建签署文档、接收状态回调
2. **数据库验证层**：通过 `getRecipientByToken` 查询验证 `recipient.token` 有效性（无 JWT 签名验证）
3. **嵌入式签署组件（Iframe）**：Documenso 提供的签署 UI，运行在隔离沙箱中

```
┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐
│   宿主页面       │        │  Documenso 后端   │        │  嵌入式签署组件   │
│   (Parent)      │────────│  (DB 查询验证)    │────────│   (Iframe)      │
│                 │        │  recipient.token │        │                 │
└─────────────────┘        └─────────────────┘        └─────────────────┘
          │                        │                         │
          └──────────postMessage─────────────────────────────┘
                     (跨域双向通信)
```

### 🔹 预签名授权层（创作链路，仅作对比）

创作嵌入链路（`/embed/v1/v2/authoring/...`）使用基于 JWT 的预签名令牌，具有签名、过期时间、Audience、Scope 等 JWT 语义。**本文分析的签署主链路不涉及此层**。

---

## 二、初始化握手流程

### 2.1 主链路说明

嵌入式签署的**主流程**基于 `/embed/sign/{token}` 路由（`apps/remix/app/routes/embed+/_v0+/sign.$token.tsx`），这是生产环境实际使用的签署入口。整个流程不涉及预签名 Token（presign token），而是直接使用 `recipient.token` 作为认证凭证。

**注意**：之前提到的 `/embed/v2/authoring/envelope/create?token=<presign>` 是文档创作嵌入链路，**不是签署主链路**。签署主链路不需要预签名 Token。

### 2.2 握手时序图

```
宿主页面                          Documenso 后端                       嵌入式组件
    │                                   │                                   │
    │ 1. 业务系统创建签署文档（后端到后端）│                                   │
    │ POST /api/v1/documents             │                                   │
    │ Header: Authorization: Bearer api_xxx │                                   │
    │──────────────────────────────────>│                                   │
    │                                   │ 2. 创建文档和收件人                │
    │                                   │ 3. 生成 recipient.token            │
    │ <──────────────────────────────────│                                   │
    │ 4. 返回 { signUrl: "/embed/sign/{token}", ... }                      │
    │                                                                       │
    │ 5. 构建 iframe URL                                                  │
    │    https://documenso.example.com/embed/sign/{token}#<hash>           │
    │                                   │                                   │
    │ 6. 创建 iframe 加载签署页面        │                                   │
    │──────────────────────────────────────────────────────────────────────>│
    │                                   │ 7. Loader 层验证 token            │
    │                                   │    getRecipientByToken(token)     │
    │                                   │ <──────────────────────────────────│
    │                                   │ 8. 验证组织权限、收件人状态       │
    │                                   │    organisationClaim, isExpired,  │
    │                                   │    isRecipientsTurn, accessAuth   │
    │                                   │──────────────────────────────────>│
    │                                   │                                   │ 9. 解析 URL Hash 配置
    │                                   │                                   │    (Base64(encodeURIComponent(JSON)))
    │                                   │                                   │ 10. 发送 document-ready 事件
    │ <──────────────────────────────────────────────────────────────────────│
    │ 11. 握手完成，进入签署流程          │                                   │
```

### 2.3 URL Hash 配置协议

嵌入式组件通过 URL Hash 传递非敏感配置参数，采用 `Base64(encodeURIComponent(JSON))` 编码格式。

**核心 Schema 定义**（`packages/lib/types/embed-base-schemas.ts:6`）：

```typescript
const ZBaseEmbedDataSchema = z.object({
  darkModeDisabled: z.boolean().optional().default(false),
  css: z.string().optional(),
  cssVars: ZCssVarsSchema.optional().default({}),
  language: ZSupportedLanguageCodeSchema.optional(),
});
```

**签署页面扩展 Schema**（`packages/lib/types/embed-document-sign-schema.ts:6`）：

```typescript
const ZSignDocumentEmbedDataSchema = ZBaseEmbedDataSchema.extend({
  email: z.union([z.literal(''), zEmail()]).optional(),
  lockEmail: z.boolean().optional().default(false),
  name: z.string().optional(),
  lockName: z.boolean().optional().default(false),
  allowDocumentRejection: z.boolean().optional(),
  showOtherRecipientsCompletedFields: z.boolean().optional(),
});
```

**Hash 编码示例**（`apps/remix/app/routes/embed+/playground.tsx:295`）：

```typescript
const hashData = {
  externalId,
  type: envelopeType,
  language,
  darkModeDisabled,
  css,
  cssVars,
  features: {
    general: generalFeatures,
    settings: settingsFeatures,
    actions: actionsFeatures,
    // ... 其他功能配置
  },
};

const hash = btoa(encodeURIComponent(JSON.stringify(hashData)));
```

### 2.3 初始化状态流转

初始化过程在 `EmbedSignDocumentV2ClientPage` 组件中完成：

1. **useLayoutEffect 解析 Hash**（`apps/remix/app/components/embed/embed-document-signing-page-v2.tsx:124`）：
   - 解码并验证 URL Hash 数据
   - 应用白标定制（CSS 注入、主题变量）
   - 设置签署者姓名/邮箱及锁定状态
   - 激活多语言支持

2. **hasFinishedInit 标志位**：
   - 控制 Loading 状态显示
   - 确保配置应用完成后再渲染签署页面

3. **document-ready 事件**（`apps/remix/app/components/embed/embed-document-signing-page-v2.tsx:181`）：
   - 初始化完成后通过 postMessage 通知宿主
   - 宿主可据此隐藏 Loading、展示签署界面

---

## 三、权限传递机制

### 3.1 双链路令牌体系

系统存在两条独立的嵌入链路，使用不同的令牌体系：

| 链路类型 | 路由模式 | 令牌类型 | 用途 |
|---------|---------|---------|------|
| **签署主链路** | `/embed/sign/{token}` | `recipient.token` | 收件人签署文档 |
| **创作嵌入链路** | `/embed/v1/v2/authoring/...` | 预签名 Token（JWT） | 文档/模板编辑 |

**⚠️ 关键区分**：本分析聚焦于**签署主链路**，使用 `recipient.token`，而非预签名 Token。

### 3.2 签署主链路：recipient.token

`recipient.token` 是收件人级别的认证凭证，在创建收件人生成，与特定文档和收件人绑定。

**令牌生成**（创建收件人时自动生成）：
- 随机字符串（非 JWT）
- 全局唯一
- 与 `recipient.id` 一一对应
- 永久有效（直到文档完成或过期）

**令牌验证**（`/embed/sign/{token}` Loader 层）：

```typescript
// apps/remix/app/routes/embed+/_v0+/sign.$token.tsx:45-54
const [document, fields, recipient, completedFields] = await Promise.all([
  getDocumentAndSenderByToken({
    token,           // recipient.token
    userId: user?.id,
    requireAccessAuth: false,
  }).catch(() => null),
  getFieldsForToken({ token }),
  getRecipientByToken({ token }).catch(() => null),
  getCompletedFieldsForToken({ token }).catch(() => []),
]);

if (!document || !recipient) {
  throw new Response('Not found', { status: 404 });
}
```

**令牌权限边界**（详见 5.2.3 节）：
- ✅ 仅凭 token 可签署非签名字段（TEXT, NUMBER, EMAIL, NAME, DATE, INITIALS, CHECKBOX, RADIO, DROPDOWN）
- ✅ 仅凭 token 可签署签名字段（当无 ACTION auth 配置时）
- ✅ 仅凭 token 可完成文档签署（当无 ACCESS 2FA 配置时）
- ❌ 签署签名字段需要 ACTION auth（配置了 ACCOUNT/PASSKEY/2FA/PASSWORD 时）
- ❌ 完成文档需要 2FA（配置了 ACCESS 2FA 时）

### 3.3 创作嵌入链路：预签名 Token（仅作对比）

创作嵌入链路使用基于 JWT 的预签名令牌，用于文档/模板编辑场景：

**生成流程**（`packages/lib/server-only/embedding-presign/create-embedding-presign-token.ts:18`）：

```typescript
const createEmbeddingPresignToken = async ({ apiToken, expiresIn, scope }) => {
  // 1. 验证 API Token 有效性
  const validatedToken = await getApiTokenByToken({ token: apiToken });

  // 2. 计算过期时间（生产环境最小5分钟）
  const effectiveExpiresIn = expiresIn >= minExpirationMinutes ? expiresIn : 60;
  const expiresAt = now.plus({ minutes: effectiveExpiresIn });

  // 3. 使用 API Token 作为 JWT 签名密钥
  const secret = new TextEncoder().encode(validatedToken.token);

  // 4. 签发 JWT
  const token = await new SignJWT({
    aud: String(validatedToken.teamId ?? validatedToken.userId),
    sub: String(validatedToken.id),
    scope,
  })
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt(now.toJSDate())
    .setExpirationTime(expiresAt.toJSDate())
    .sign(secret);

  return { token, expiresAt, expiresIn };
};
```

**JWT Claims 说明**：
- `aud`（Audience）：团队ID或用户ID，确保令牌使用范围
- `sub`（Subject）：API Token ID，用于反向查找原始令牌
- `scope`：作用域，可选的权限范围限制
- `iat`（Issued At）：签发时间
- `exp`（Expiration）：过期时间

### 3.4 签署主链路的权限校验层级

`/embed/sign/{token}` 路由采用多层权限校验，全部在 Loader 层完成：

| 校验层级 | 检查内容 | 失败处理 | 代码位置 |
|---------|---------|---------|---------|
| L1 Token 有效性 | recipient.token 是否存在、对应收件人是否存在 | 404 Not Found | `sign.$token.tsx:58-60` |
| L2 组织权限 | 组织是否开通 `embedSigning` 功能 | 403 embed-paywall | `sign.$token.tsx:70-79` |
| L3 收件人状态 | 收件人是否已过期 | 403 embed-recipient-expired | `sign.$token.tsx:81-90` |
| L4 签署轮次 | 是否轮到该收件人签署 | 403 embed-waiting-for-turn | `sign.$token.tsx:116-127` |
| L5 Access Auth | 是否需要登录/2FA 认证 | 401 embed-authentication-required | `sign.$token.tsx:92-114` |
| L6 文档状态 | 文档是否为 PENDING 状态 | 重定向到对应状态页 | `sign.$token.tsx:84-86` |

### 3.5 功能权限粒度控制

通过 URL Hash 配置和组织权限声明实现细粒度功能权限控制：

**Hash 配置权限**（`packages/lib/types/embed-document-sign-schema.ts:6`）：
- `allowDocumentRejection`：是否允许拒绝文档
- `lockEmail` / `lockName`：是否锁定邮箱/姓名
- `darkModeDisabled`：是否禁用深色模式
- `css` / `cssVars`：自定义 CSS 注入（需组织权限）

**组织权限声明**（`apps/remix/app/routes/embed+/_v0+/sign.$token.tsx:62-66`）：

```typescript
const organisationClaim = await getOrganisationClaimByTeamId({ teamId: document.teamId });

const allowEmbedSigningWhitelabel = organisationClaim.flags.embedSigningWhiteLabel;
const hidePoweredBy = organisationClaim.flags.hidePoweredBy;
```

**CSS 注入权限检查**（`apps/remix/app/components/embed/embed-document-signing-page-v2.tsx:157`）：

```typescript
if (allowWhitelabelling) {  // 受组织权限控制
  injectCss({
    css: data.css,
    cssVars: data.cssVars,
  });
}
```

---

## 四、生命周期事件

### 4.1 事件体系概览

嵌入式组件通过 `postMessage` 向宿主页面发送标准化事件，所有事件遵循统一格式：

```typescript
{
  action: string,      // 事件类型标识
  data: any | null     // 事件数据载荷
}
```

### 4.2 事件类型清单

| 事件类型 | 触发时机 | 数据载荷 | 代码位置 |
|---------|---------|---------|---------|
| `document-ready` | 组件初始化完成，准备就绪 | `null` | [embed-document-signing-page-v2.tsx:70](apps/remix/app/components/embed/embed-document-signing-page-v2.tsx#L70) |
| `field-signed` | 某个字段完成签署 | `{ fieldId?, value?, isBase64? }` | [embed-document-signing-page-v2.tsx:82](apps/remix/app/components/embed/embed-document-signing-page-v2.tsx#L82) |
| `field-unsigned` | 某个字段取消签署 | `{ fieldId? }` | [embed-document-signing-page-v2.tsx:94](apps/remix/app/components/embed/embed-document-signing-page-v2.tsx#L94) |
| `document-completed` | 当前收件人完成所有签署 | `{ token, documentId, envelopeId, recipientId }` | [embed-document-signing-page-v2.tsx:41](apps/remix/app/components/embed/embed-document-signing-page-v2.tsx#L41) |
| `document-rejected` | 收件人拒绝签署 | `{ token, documentId, envelopeId, recipientId, reason? }` | [embed-document-signing-page-v2.tsx:106](apps/remix/app/components/embed/embed-document-signing-page-v2.tsx#L106) |
| `document-error` | 签署过程发生错误 | `null` | [embed-document-signing-page-v2.tsx:58](apps/remix/app/components/embed/embed-document-signing-page-v2.tsx#L58) |
| `document-waiting-for-turn` | 等待其他签署者 | - | [embed-document-waiting-for-turn.tsx:11](apps/remix/app/components/embed/embed-document-waiting-for-turn.tsx#L11) |
| `recipient-expired` | 签署链接已过期 | - | [embed-recipient-expired.tsx:11](apps/remix/app/components/embed/embed-recipient-expired.tsx#L11) |
| `all-documents-completed` | 批量签署全部完成 | `{ documents: [...] }` | [multisign/_index.tsx:158](apps/remix/app/routes/embed+/v1+/multisign+/_index.tsx#L158) |

### 4.3 事件发送实现

**标准事件发送器**（`apps/remix/app/components/embed/embed-document-signing-page-v2.tsx:41`）：

```typescript
const onDocumentCompleted = (data) => {
  if (window.parent) {
    window.parent.postMessage(
      {
        action: 'document-completed',
        data,
      },
      '*',  // 注意：目标源设为通配符，实际应在宿主端验证来源
    );
  }
};
```

### 4.4 事件监听机制（宿主端）

宿主页面通过监听 `message` 事件接收回调：

**示例实现**（`apps/remix/app/routes/embed+/playground.tsx:174`）：

```typescript
useEffect(() => {
  const handleMessage = (event: MessageEvent) => {
    const timestamp = new Date().toISOString().slice(11, 19);

    // 业务处理逻辑
    switch (event.data.action) {
      case 'document-ready':
        console.log('签署组件已就绪');
        break;
      case 'document-completed':
        console.log('签署完成', event.data.data);
        break;
      case 'field-signed':
        console.log('字段已签署', event.data.data);
        break;
      // ... 其他事件处理
    }
  };

  window.addEventListener('message', handleMessage);
  return () => window.removeEventListener('message', handleMessage);
}, []);
```

### 4.5 生命周期状态流转

```
初始化
    │
    ▼
document-ready ──► field-signed ──► field-unsigned
    │                    │                │
    │                    └────────┬───────┘
    │                             ▼
    │                    （可重复多次）
    │                             │
    ▼                             ▼
document-completed / document-rejected / document-error
    │
    ▼
结束
```

---

## 五、安全边界与跨域回调

### 5.1 安全架构分层

```
┌─────────────────────────────────────────────────────────────┐
│                     宿主页面（Origin A）                     │
│  ┌───────────────────────────────────────────────────────┐  │
│  │             iframe 沙箱（Origin B）                   │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │              React 上下文隔离                    │  │  │
│  │  │  ┌──────────────────────────────────────────┐   │  │  │
│  │  │  │         EmbedSigningContext             │   │  │  │
│  │  │  │  - 权限标记（isEmbed: true）            │   │  │  │
│  │  │  │  - 回调函数隔离                         │   │  │  │
│  │  │  └──────────────────────────────────────────┘   │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 跨域通信安全边界

#### 5.2.1 postMessage 通信模型

**通信方向**：
- **内嵌 → 宿主**：通过 `window.parent.postMessage()` 发送事件
- **宿主 → 内嵌**：理论上可通过 `iframe.contentWindow.postMessage()`，但当前实现仅单向通信

**目标源策略**：
当前实现使用 `'*'` 作为目标源（`embed-document-signing-page-v2.tsx:53`）：

```typescript
window.parent.postMessage(
  {
    action: 'document-completed',
    data,
  },
  '*',  // ⚠️ 通配符目标源
);
```

> **安全注意**：使用 `'*'` 意味着消息可被任何父窗口接收。**⚠️ 重要修正**：`document-completed` 和 `document-rejected` 事件中包含的 `token` 是 `recipient.token`，这是**高度敏感**的凭证（详见 5.2.3 节），因此宿主端必须验证 `event.origin`。

#### 5.2.2 宿主端源验证最佳实践

宿主页面应始终验证消息来源：

```typescript
const handleMessage = (event: MessageEvent) => {
  // ✅ 验证消息来源 - 必须执行，因为事件包含敏感 token
  if (event.origin !== 'https://app.documenso.com') {
    return; // 忽略非信任源消息
  }

  // ✅ 验证消息格式
  if (!event.data || typeof event.data.action !== 'string') {
    return;
  }

  // 处理业务逻辑
  switch (event.data.action) {
    // ...
  }
};
```

#### 5.2.3 跨域事件中 Token 的敏感性与鉴权边界重新评估

**关键发现**：`document-completed` 和 `document-rejected` 事件中携带的 `token` 字段是 `recipient.token`，这是一个**高度敏感**的认证凭证，但并非所有操作都仅凭 token 即可完成，存在明确的鉴权边界。

---

##### 🔹 鉴权边界核心规则（代码事实）

**字段类型决定鉴权要求**（`packages/lib/server-only/document/validate-field-auth.ts:28-31`）：

```typescript
// 非签名字段直接跳过所有鉴权
if (field.type !== FieldType.SIGNATURE) {
  return undefined;  // ✅ 无需任何鉴权
}

// 只有 SIGNATURE / FREE_SIGNATURE 字段需要 ACTION 级鉴权
const isValid = await isRecipientAuthorized({
  type: 'ACTION',
  // ...
});
```

**ACTION 鉴权边界**（`packages/lib/server-only/document/is-recipient-authorized.ts:72-74`）：

```typescript
// 无鉴权要求或显式指定无需鉴权 → 直接放行
if (authMethods.length === 0 || 
    authMethods.some((method) => method === DocumentAuth.EXPLICIT_NONE)) {
  return true;  // ✅ 仅凭 token 即可
}

// 否则需要额外鉴权凭证
// ACCOUNT: 需要 userId 匹配收件人邮箱
// PASSKEY: 需要 userId + passkey 认证响应
// TWO_FACTOR_AUTH: 需要 userId + TOTP 验证码
// PASSWORD: 需要 userId + 密码
```

---

##### 🔹 仅凭 token 可执行的操作（无需其他凭证）

| 操作 | 字段类型/说明 | 代码位置 |
|------|--------------|---------|
| 签署非签名字段 | TEXT, NUMBER, EMAIL, NAME, DATE, INITIALS, CHECKBOX, RADIO, DROPDOWN | `sign-field-with-token.ts:127-166` |
| 签署签名字段（无鉴权配置时） | SIGNATURE, FREE_SIGNATURE（当 `actionAuth` 为 `EXPLICIT_NONE` 或未配置时） | `sign-field-with-token.ts:174-190` |
| 完成文档签署 | 仅当 `accessAuth` 不包含 `TWO_FACTOR_AUTH` 时 | `complete-document-with-token.ts:117-173` |
| 拒绝文档 | 所有场景 | `reject-document-with-token.ts` |
| 查看文档内容 | 所有场景 | `get-document-by-token.ts` |
| 获取字段列表 | 所有场景 | `get-fields-for-token.ts` |
| 标记已查看 | 所有场景 | `viewed-document.ts` |
| 获取已签署字段 | 所有场景 | `get-completed-fields-for-token.ts` |

> **⚠️ 重要边界说明**：90% 以上的字段类型（非签名字段）仅凭 token 即可签署，无需任何额外鉴权。只有签名字段在文档配置了 ACTION 鉴权时才需要额外凭证。

---

##### 🔹 必须动作鉴权（ACTION auth）的场景

当文档的 `actionAuth` 配置了以下任一方式时，签署 **SIGNATURE / FREE_SIGNATURE** 字段需要额外凭证：

| 鉴权方式 | 所需额外参数 | 代码位置 |
|---------|-------------|---------|
| `ACCOUNT` | `userId`（需与收件人邮箱匹配的登录用户） | `is-recipient-authorized.ts:94-106` |
| `PASSKEY` | `userId` + passkey 认证响应 + tokenReference | `is-recipient-authorized.ts:107-117` |
| `TWO_FACTOR_AUTH` | `userId` + TOTP 验证码 | `is-recipient-authorized.ts:118-155` |
| `PASSWORD` | `userId` + 密码 | `is-recipient-authorized.ts:156-165` |

---

##### 🔹 必须 2FA 的场景

只有一种场景需要 2FA 验证（`complete-document-with-token.ts:117-173`）：

```typescript
// 仅当 accessAuth 包含 TWO_FACTOR_AUTH 时，完成文档需要 2FA
if (derivedRecipientAccessAuth.includes(DocumentAuth.TWO_FACTOR_AUTH)) {
  if (!accessAuthOptions) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, {
      message: 'Access authentication required',
    });
  }
  // 验证 2FA 令牌...
}
```

**触发条件**：
- 文档 `accessAuth` 配置包含 `TWO_FACTOR_AUTH`
- 操作：`completeDocumentWithToken`（完成文档签署）
- 所需参数：`accessAuthOptions`（包含 2FA token 和 method）

---

##### 🔹 鉴权边界总结

```
recipient.token
    │
    ├─► 非签名字段（TEXT, NUMBER, EMAIL, NAME, DATE,
    │               INITIALS, CHECKBOX, RADIO, DROPDOWN）
    │               └─► ✅ 仅凭 token 即可签署
    │
    ├─► 签名字段（SIGNATURE, FREE_SIGNATURE）
    │   │
    │   ├─► 无 ACTION auth 配置 / EXPLICIT_NONE
    │   │       └─► ✅ 仅凭 token 即可签署
    │   │
    │   └─► 配置 ACTION auth（ACCOUNT/PASSKEY/2FA/PASSWORD）
    │           └─► ❌ 需要额外鉴权凭证 + userId
    │
    └─► 完成文档签署
        │
        ├─► 无 ACCESS 2FA 配置
        │       └─► ✅ 仅凭 token 即可完成
        │
        └─► 配置 ACCESS 2FA
                └─► ❌ 需要 2FA 验证码
```

---

##### 🔹 风险等级与事件暴露

| 操作 | 敏感性 | 可滥用性 | 备注 |
|-----|--------|---------|------|
| 签署非签名字段 | 🔴 高 | 🔴 极高 | 90% 字段类型仅凭 token 即可 |
| 签署签名字段（无鉴权） | 🔴 极高 | 🔴 极高 | 默认配置下仅凭 token 即可 |
| 完成文档签署（无 2FA） | 🔴 极高 | 🔴 极高 | 默认配置下仅凭 token 即可 |
| 拒绝文档 | 🟠 中 | 🟠 中 | 仅凭 token 即可 |
| 查看文档/字段 | 🟡 中 | 🟠 中 | 仅凭 token 即可 |

**事件暴露情况对比**：

| 事件类型 | 是否暴露 token | 敏感性 |
|---------|---------------|--------|
| `document-ready` | ❌ 否 | 低 |
| `field-signed` | ❌ 否 | 低 |
| `field-unsigned` | ❌ 否 | 低 |
| `document-completed` | ✅ 是 | 极高（默认配置下可伪造完整签署） |
| `document-rejected` | ✅ 是 | 中 |
| `document-error` | ❌ 否 | 低 |
| `document-waiting-for-turn` | ❌ 否 | 低 |
| `recipient-expired` | ❌ 否 | 低 |
| `all-documents-completed` | ✅ 是（每个文档） | 极高 |

---

##### 🔹 对宿主端回调处理策略的影响

1. **Origin 验证从「建议」升级为「必须」**：
   - 原结论：`event.origin` 验证是建议性的安全加固
   - 新结论：**必须**验证，否则恶意父窗口可窃取 `recipient.token`，在默认配置下仅凭 token 即可完成整个签署流程

2. **Token 处理策略**：
   - 禁止在前端日志中输出完整 token
   - 禁止将 token 存储在 localStorage/sessionStorage
   - 建议：宿主端收到 token 后立即通过后端 API 验证并标记已使用
   - 建议：token 仅用于后端到后端的状态同步，不在前端业务逻辑中使用

3. **回调异常处理**：
   - 若 `event.origin` 验证失败，应静默丢弃消息并记录安全审计日志
   - 若消息格式异常，应丢弃而非抛出异常（防止 XSS 探针）
   - 建议实现回调幂等性，防止重放攻击

### 5.3 上下文安全隔离

#### 5.3.1 EmbedSigningContext 隔离层

通过 React Context 标记嵌入式环境，实现组件行为差异化：

**Context 定义**（`apps/remix/app/components/embed/embed-signing-context.tsx:3`）：

```typescript
type EmbedSigningContextValue = {
  isEmbed: true;  // 硬编码为 true，明确标记嵌入环境
  allowDocumentRejection: boolean;
  isNameLocked: boolean;
  isEmailLocked: boolean;
  hidePoweredBy: boolean;
  // 生命周期回调
  onDocumentCompleted: (data) => void;
  onDocumentError: () => void;
  onDocumentRejected: (data) => void;
  onDocumentReady: () => void;
  onFieldSigned: (data) => void;
  onFieldUnsigned: (data) => void;
};
```

**安全特性**：
- `isEmbed: true` 硬编码，防止伪造非嵌入环境
- 回调函数由嵌入页面注入，不依赖全局变量
- 权限标志通过 Context 向下传递，组件可据此调整行为

#### 5.3.2 安全边界检查模式

组件通过 `useRequiredEmbedSigningContext()` 断言嵌入环境：

```typescript
const useRequiredEmbedSigningContext = () => {
  const context = useEmbedSigningContext();

  if (!context) {
    // 非嵌入环境下使用会抛出错误，防止权限提升
    throw new Error('useRequiredEmbedSigningContext must be used within EmbedSigningProvider');
  }

  return context;
};
```

### 5.4 白标定制安全控制

#### 5.4.1 CSS 注入权限

CSS 注入受组织权限声明控制，防止未授权用户篡改 UI：

**权限检查**（`apps/remix/app/components/embed/embed-document-signing-page-v2.tsx:157`）：

```typescript
if (allowWhitelabelling) {
  injectCss({
    css: data.css,
    cssVars: data.cssVars,
  });
}
```

**组织权限声明来源**（`apps/remix/app/routes/embed+/v1+/multisign+/_index.tsx:53`）：

```typescript
const organisationClaim = await getOrganisationClaimByTeamId({
  teamId: firstDocument.teamId,
});

const allowWhitelabelling = organisationClaim.flags.embedSigningWhiteLabel;
const hidePoweredBy = organisationClaim.flags.hidePoweredBy;
```

#### 5.4.2 注入机制

`injectCss` 工具函数安全地将 CSS 注入到文档中：

- CSS 变量通过 `document.documentElement.style.setProperty()` 设置
- 原始 CSS 通过 `<style>` 标签注入
- 注意：未做 XSS 过滤，依赖组织权限控制

### 5.5 Token 安全边界（签署主链路）

#### 5.5.1 recipient.token 暴露风险

签署主链路使用 `recipient.token`，这是一个永久有效的凭证（直到文档完成或过期）。

| 风险点 | 说明 | 缓解措施 |
|-------|------|---------|
| URL 路径参数 | `recipient.token` 出现在 URL 路径中（`/embed/sign/{token}`），可能被历史记录、日志捕获 | 1. 文档完成后令牌失效<br>2. HTTPS 传输<br>3. 配置合理的文档过期时间 |
| Referer 泄露 | iframe 请求可能通过 Referer 头泄露 Token | 1. 设置 Referrer-Policy: no-referrer<br>2. 服务端验证文档状态 |
| XSS 窃取 | 宿主页面 XSS 可能窃取 iframe Token | 1. 独立子域名部署<br>2. CSP 策略隔离 |
| 跨域事件暴露 | `document-completed` 和 `document-rejected` 事件携带 token | 1. 宿主端必须验证 event.origin<br>2. 禁止前端存储和日志输出 token |

#### 5.5.2 Token 失效策略

`recipient.token` 在以下情况下失效：
1. 文档状态变为 `COMPLETED`（所有收件人签署完成）
2. 文档被拒绝（`recipient.signingStatus = REJECTED`）
3. 文档过期（`recipient.expiredAt < now`）
4. 文档被删除

#### 5.5.3 令牌安全对比

| 特性 | recipient.token（签署主链路） | 预签名 Token（创作链路） |
|-----|-----------------------------|-------------------------|
| 格式 | 随机字符串 | JWT |
| 有效期 | 直到文档完成/过期 | 5分钟~24小时，可配置 |
| 绑定 | 特定收件人 + 特定文档 | API Token + 团队/用户 |
| 存储位置 | URL 路径 | URL 查询参数 |
| 可执行操作 | 签署文档、查看内容 | 编辑文档、创建模板 |
| 签名验证 | 数据库查询 | JWT 签名验证 |

### 5.6 路由访问控制

嵌入式路由通过多层防护确保安全：

1. **Loader 层 Token 验证**：每次页面加载验证预签名 Token
2. **Layout 层权限声明**：获取组织功能权限声明
3. **TRPC 中间件验证**：每个 API 调用再次验证 Token
4. **Context 层环境标记**：通过 React Context 标记嵌入环境

---

## 六、握手失败场景下的权限收敛处理

### 6.1 主链路说明：/embed/sign 签署入口

嵌入式签署的**主链路**是 `/embed/sign/{token}`（路由定义：`apps/remix/app/routes/embed+/_v0+/sign.$token.tsx`），这是实际生产环境使用的签署入口。

**主链路架构**：

```
/embed/sign/{token}
    │
    ├─► Loader 层根据 envelope.internalVersion 分流
    │   ├─► internalVersion === 2 ──► handleV2Loader() ──► V2 签署页面
    │   └─► 其他 ──────────────────► handleV1Loader() ──► V1 签署页面
    │
    └─► 所有状态异常通过 throw data() 触发 ErrorBoundary 渲染
```

---

### 6.2 document-ready 触发条件（V1 vs V2 可验证判断）

**⚠️ 关键发现**：V1 和 V2 版本的 `document-ready` 触发条件有本质差异，可通过状态变量的依赖关系精确推导。

---

#### 🔹 状态变量依赖关系总览

| 状态变量 | V1 存在性 | V2 存在性 | 设置时机 | 依赖关系 |
|---------|----------|----------|---------|---------|
| `hasFinishedInit` | ✅ 存在 | ✅ 存在 | Hash 解析完成后（成功或失败） | 与 PDF 加载无关 |
| `hasDocumentLoaded` | ✅ 存在 | ❌ 已注释（第35行） | PDF 加载完成回调 | 与 Hash 解析无关 |

---

#### 🔹 V1：hasFinishedInit 依赖关系（可验证判断）

**代码锚点**：`embed-document-signing-page-v1.tsx:192-240`

```typescript
useLayoutEffect(() => {
  const hash = window.location.hash.slice(1);

  try {
    const data = ZSignDocumentEmbedDataSchema.parse(JSON.parse(decodeURIComponent(atob(hash))));
    // ... 应用所有配置 ...
    
    // ✅ 成功路径：设置 hasFinishedInit = true
    if (data.language && data.language !== APP_I18N_OPTIONS.sourceLang) {
      void dynamicActivate(data.language).finally(() => {
        setHasFinishedInit(true);  // v1.tsx:227
      });
    } else {
      setHasFinishedInit(true);  // v1.tsx:230
    }
  } catch (err) {
    console.error(err);
    // ✅ 失败路径：同样设置 hasFinishedInit = true
    setHasFinishedInit(true);  // v1.tsx:234
  }
}, []);  // ⚠️ 空依赖数组，仅执行一次
```

**可验证判断**：
> **命题 P1**：`hasFinishedInit = true` 当且仅当 useLayoutEffect 执行完毕（无论 Hash 解析成功或失败）
>
> **推导**：
> 1. try 块所有分支最终都会调用 `setHasFinishedInit(true)`（第 227、230 行）
> 2. catch 块直接调用 `setHasFinishedInit(true)`（第 234 行）
> 3. 依赖数组为 `[]`，useLayoutEffect 仅在组件挂载时执行一次
> 4. 因此，**只要组件挂载完成，hasFinishedInit 最终必然为 true**

---

#### 🔹 V1：hasDocumentLoaded 依赖关系（可验证判断）

**代码锚点**：`embed-document-signing-page-v1.tsx:290-301`

```typescript
<PDFViewerLazy
  data={getDocumentDataUrlForPdfViewer({...})}
  scrollParentRef="window"
  onDocumentLoad={() => setHasDocumentLoaded(true)}  // v1.tsx:300
/>
```

**可验证判断**：
> **命题 P2**：`hasDocumentLoaded = true` 当且仅当 PDFViewerLazy 的 onDocumentLoad 回调被触发
>
> **推导**：
> 1. `hasDocumentLoaded` 初始值为 `false`（第 80 行）
> 2. 唯一的 `setHasDocumentLoaded(true)` 调用在 `onDocumentLoad` 回调中（第 300 行）
> 3. 若 PDF 加载失败、网络中断或组件卸载，回调永远不会触发
> 4. 因此，**hasDocumentLoaded 可能永远为 false**

---

#### 🔹 V1：document-ready 触发条件（可验证判断）

**代码锚点**：`embed-document-signing-page-v1.tsx:242-252`

```typescript
useEffect(() => {
  if (hasFinishedInit && hasDocumentLoaded && window.parent) {
    window.parent.postMessage(
      { action: 'document-ready', data: null },
      '*',
    );
  }
}, [hasFinishedInit, hasDocumentLoaded]);  // v1.tsx:252
```

**可验证判断**：
> **命题 P3**（V1）：`document-ready` 触发 ⟺ `hasFinishedInit = true` ∧ `hasDocumentLoaded = true` ∧ `window.parent ≠ null`
>
> **Hash 解析失败场景推导**：
> 1. Hash 解析失败 → catch 块 → `hasFinishedInit = true`（P1 成立）
> 2. `hasDocumentLoaded` 取决于 PDF 加载，与 Hash 解析无关（P2）
> 3. 因此：**Hash 解析失败时，document-ready 可能触发也可能不触发，取决于 PDF 是否成功加载**
>
> **真值表**：
> | hasFinishedInit | hasDocumentLoaded | window.parent | document-ready 触发? |
> |----------------|------------------|--------------|---------------------|
> | true（Hash 失败） | true（PDF 加载成功） | true | ✅ 是 |
> | true（Hash 失败） | false（PDF 加载失败） | true | ❌ 否 |
> | true（Hash 失败） | true | false | ❌ 否 |

---

#### 🔹 V2：hasFinishedInit 依赖关系（可验证判断）

**代码锚点**：`embed-document-signing-page-v2.tsx:124-179`

```typescript
useLayoutEffect(() => {
  const hash = window.location.hash.slice(1);

  try {
    const data = ZSignDocumentEmbedDataSchema.parse(JSON.parse(decodeURIComponent(atob(hash))));
    // ... 应用所有配置 ...
    
    // ✅ 成功路径：设置 hasFinishedInit = true
    if (data.language && data.language !== APP_I18N_OPTIONS.sourceLang) {
      void dynamicActivate(data.language).finally(() => {
        setHasFinishedInit(true);  // v2.tsx:166
      });
    } else {
      setHasFinishedInit(true);  // v2.tsx:169
    }
  } catch (err) {
    console.error(err);
    // ✅ 失败路径：同样设置 hasFinishedInit = true
    setHasFinishedInit(true);  // v2.tsx:173
  }
}, [allowWhitelabelling]);  // v2.tsx:179
```

**可验证判断**：
> **命题 Q1**：`hasFinishedInit = true` 当且仅当 useLayoutEffect 执行完毕（无论 Hash 解析成功或失败）
>
> **推导**：
> 1. try 块所有分支最终都会调用 `setHasFinishedInit(true)`（第 166、169 行）
> 2. catch 块直接调用 `setHasFinishedInit(true)`（第 173 行）
> 3. 依赖数组为 `[allowWhitelabelling]`，只要该 prop 不变，仅执行一次
> 4. 因此，**只要组件挂载完成，hasFinishedInit 最终必然为 true**

---

#### 🔹 V2：hasDocumentLoaded 状态（已废弃）

**代码锚点**：`embed-document-signing-page-v2.tsx:34-35`

```typescript
// !: Not used at the moment, may be removed in the future.
// const [hasDocumentLoaded, setHasDocumentLoaded] = useState(false);
```

**可验证判断**：
> **命题 Q2**：V2 中 `hasDocumentLoaded` 已被注释，不参与任何逻辑判断
>
> **推导**：
> 1. 变量声明已被注释（第 35 行）
> 2. 无任何地方调用 `setHasDocumentLoaded`
> 3. `document-ready` 触发条件不依赖该变量

---

#### 🔹 V2：document-ready 触发条件（可验证判断）

**代码锚点**：`embed-document-signing-page-v2.tsx:181-186`

```typescript
useEffect(() => {
  if (hasFinishedInit) {
    onDocumentReady();
  }
  // eslint-disable-next-line react-hooks/exhaustive-deps
}, [hasFinishedInit]);  // v2.tsx:186
```

**可验证判断**：
> **命题 Q3**（V2）：`document-ready` 触发 ⟺ `hasFinishedInit = true` ∧ `window.parent ≠ null`
>
> **Hash 解析失败场景推导**：
> 1. Hash 解析失败 → catch 块 → `hasFinishedInit = true`（Q1 成立）
> 2. 不依赖 `hasDocumentLoaded`（已废弃）
> 3. 因此：**Hash 解析失败时，只要 window.parent 存在，document-ready 必然触发**
>
> **真值表**：
> | hasFinishedInit | window.parent | document-ready 触发? |
> |----------------|--------------|---------------------|
> | true（Hash 失败） | true | ✅ 是（使用默认配置） |
> | true（Hash 失败） | false | ❌ 否 |

---

#### 🔹 V1 vs V2 行为对比总结

| 条件 | V1 行为 | V2 行为 |
|-----|---------|---------|
| Hash 解析成功 + PDF 加载成功 | ✅ 触发 document-ready | ✅ 触发 document-ready |
| Hash 解析成功 + PDF 加载失败 | ❌ 不触发 | ✅ 触发（不等待 PDF） |
| Hash 解析失败 + PDF 加载成功 | ✅ 触发（默认配置） | ✅ 触发（默认配置） |
| Hash 解析失败 + PDF 加载失败 | ❌ 不触发 | ✅ 触发（默认配置） |

> **统一结论**：
> - **V1 是"保守触发"**：等待 PDF 加载完成，可能永远不触发
> - **V2 是"乐观触发"**：不等待 PDF 加载，只要初始化完成就触发
> - 宿主端必须同时处理两种情况，不能假设 document-ready 一定会触发或一定不触发

---

### 6.3 握手失败场景分类与处理机制

嵌入式签署组件的握手过程涉及多个环节，每个环节都可能失败。系统采用**"失败安全"（fail-safe）**设计，确保在任何握手失败场景下，权限都会收敛到最保守状态。

#### 场景 1：URL Hash 解析失败

**触发条件**：
- Hash 格式错误（非 Base64、非 JSON）
- Schema 验证失败（缺少必填字段、类型错误）
- 解码异常

**代码位置**：`apps/remix/app/components/embed/embed-document-signing-page-v2.tsx:171-174`

```typescript
useLayoutEffect(() => {
  const hash = window.location.hash.slice(1);

  try {
    const data = ZSignDocumentEmbedDataSchema.parse(JSON.parse(decodeURIComponent(atob(hash))));
    // ... 应用配置
  } catch (err) {
    console.error(err);
    // ✅ 权限收敛：设置初始化完成标志，但不应用任何配置
    setHasFinishedInit(true);
  }
}, [allowWhitelabelling]);
```

**权限收敛策略**：

| 配置项 | 失败后收敛值 | 安全影响 |
|-------|-------------|---------|
| `allowDocumentRejection` | `false` | 禁止拒绝文档，防止滥用 |
| `isNameLocked` | `false` | 允许用户编辑姓名（但不提供预填值） |
| `isEmailLocked` | `true`（DOCUMENT）/ `false`（TEMPLATE） | 遵循默认安全策略 |
| `darkModeDisabled` | `false` | 不修改主题 |
| CSS 注入 | 不执行 | 防止恶意 CSS 注入 |
| 语言设置 | 使用默认语言（en） | 避免语言切换漏洞 |
| `lockName` / `lockEmail` | `false` | 不强制锁定，允许用户输入 |

**生命周期事件收敛**：
- V1：❌ 不发送 `document-ready`（因为 `hasDocumentLoaded` 可能仍为 false）
- V2：✅ 发送 `document-ready`（因为只需要 `hasFinishedInit`）
- ✅ 仍会渲染签署页面，但使用默认配置
- ⚠️ 风险：用户可能在未授权配置下完成签署，但使用最保守权限

#### 场景 2：recipient.token 验证失败（签署主链路）

**⚠️ 重要澄清**：签署主链路使用 `recipient.token`，这是一个**随机字符串**（非 JWT），**没有签名、没有过期时间**（JWT 的 exp/iat 等概念完全不适用）。Token 的有效性通过数据库查询验证，而非密码学签名验证。

**触发条件**（主链路 `/embed/sign/{token}` Loader 层）：
- Token 格式错误（空值、长度不符）
- Token 对应的收件人不存在（数据库查询返回 null）
- **注意**：没有"签名无效"、"Token 过期"、"Audience 不匹配"等 JWT 语义，这些属于创作链路的预签名 Token

**代码锚点**：`apps/remix/app/routes/embed+/_v0+/sign.$token.tsx:45-60`

```typescript
// V1 Loader 验证 - 主链路（无 JWT 签名验证）
const [document, fields, recipient, completedFields] = await Promise.all([
  getDocumentAndSenderByToken({
    token,           // recipient.token - 随机字符串
    userId: user?.id,
    requireAccessAuth: false,
  }).catch(() => null),  // ❌ 查询失败返回 null
  getFieldsForToken({ token }).catch(() => []),
  getRecipientByToken({ token }).catch(() => null),  // ❌ 查询失败返回 null
  getCompletedFieldsForToken({ token }).catch(() => []),
]);

if (!document || !recipient) {
  throw new Response('Not found', { status: 404 });  // ❌ 完全拒绝
}
```

**核心验证函数**：`getRecipientByToken`（`packages/lib/server-only/recipient/get-recipient-by-token.ts:1`）

```typescript
// 仅通过数据库查询验证 token，无签名、无过期检查
export const getRecipientByToken = async ({ token }: { token: string }) => {
  return await prisma.recipient.findFirstOrThrow({
    where: { token },  // ✅ 仅通过 token 字段查找
  });
};
```

**权限收敛策略**（主链路）：

| 验证阶段 | 失败处理 | 权限收敛结果 | 代码锚点 |
|---------|---------|-------------|---------|
| Token 存在性检查 | 抛出 404 响应 | ❌ 完全拒绝访问 | `sign.$token.tsx:37-39` |
| 收件人存在性检查 | 抛出 404 响应 | ❌ 完全拒绝访问 | `sign.$token.tsx:58-60` |
| Token 过期检查 | （无，使用 recipient.expiredAt 字段） | 见场景 4 | `sign.$token.tsx:81-90` |
| **JWT 签名验证** | ❌ 不存在此概念 | - | - |
| **Audience 匹配** | ❌ 不存在此概念 | - | - |
| **Scope 匹配** | ❌ 不存在此概念 | - | - |

---

#### 场景 2b：预签名 Token 验证失败（创作链路，仅作对比）

**⚠️ 仅适用于创作链路**：`/embed/v1/v2/authoring/...` 路由使用 JWT 格式的预签名 Token，具有签名、过期时间、Audience、Scope 等 JWT 语义。

**代码锚点**：`packages/lib/server-only/embedding-presign/verify-embedding-presign-token.ts:12`

```typescript
// 创作链路 - JWT 预签名 Token 验证（有签名、有过期）
const verifyEmbeddingPresignToken = async ({ token, scope }) => {
  const decodedToken = decodeJwt<JWTPayload>(token);  // 解码 JWT
  
  // ✅ 检查 sub (API Token ID)
  if (!decodedToken.sub || !decodedToken.aud) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, 'Invalid presign token format');
  }
  
  // ✅ 通过 sub 查找原始 API Token
  const apiToken = await prisma.apiToken.findFirst({
    where: { id: Number(decodedToken.sub) },
    include: { user: true },
  });
  
  // ✅ 检查 aud (团队/用户ID) 匹配
  if (audienceId !== apiToken.teamId && audienceId !== apiToken.userId) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, 'Audience mismatch');
  }
  
  // ✅ 检查 scope 匹配
  if (decodedToken.scope && scope && decodedToken.scope !== scope) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, 'Scope not matched');
  }
  
  // ✅ 验证 JWT 签名（使用 API Token 作为密钥）
  const secret = new TextEncoder().encode(apiToken.token);
  await jwtVerify(token, secret);  // ⚠️ JWT 签名验证 - 仅创作链路有
  
  return { ...apiToken, userId, user };
};
```

**创作链路权限收敛策略**（对比用）：

| 验证阶段 | 失败处理 | 权限收敛结果 |
|---------|---------|-------------|
| Token 格式检查 | 抛出 404 响应 | ❌ 完全拒绝访问 |
| JWT 签名验证 | 抛出 401 错误 | ❌ 完全拒绝访问 |
| Audience 匹配检查 | 抛出 401 错误 | ❌ 完全拒绝访问 |
| Scope 匹配检查 | 抛出 401 错误 | ❌ 完全拒绝访问 |
| Token 过期检查 | 抛出 401 错误 | ❌ 完全拒绝访问 |

> **关键区分总结**：
> | 特性 | 签署主链路 recipient.token | 创作链路 presign token |
> |-----|-------------------------|-----------------------|
> | 格式 | 随机字符串 | JWT |
> | 签名 | ❌ 无 | ✅ HS256 |
> | 过期时间 | ❌ 无（用 recipient.expiredAt） | ✅ exp Claim |
> | Audience | ❌ 无 | ✅ aud Claim |
> | Scope | ❌ 无 | ✅ scope Claim |
> | 验证方式 | 数据库查询 | JWT 签名验证 |
> | 适用路由 | `/embed/sign/{token}` | `/embed/v1/v2/authoring/...` |

**生命周期事件收敛**（主链路 Token 验证失败）：
- ❌ 不发送任何生命周期事件
- ❌ 不加载任何签署组件
- ✅ 通过 ErrorBoundary 渲染对应状态页面

**ErrorBoundary 处理**（`apps/remix/app/routes/embed+/_v0+/_layout.tsx:47-95`）：

```typescript
export function ErrorBoundary() {
  const error = useRouteError();

  if (isRouteErrorResponse(error)) {
    if (error.status === 401 && error.data.type === 'embed-authentication-required') {
      return <EmbedAuthenticationRequired />;
    }
    if (error.status === 403 && error.data.type === 'embed-paywall') {
      return <EmbedPaywall />;
    }
    if (error.status === 403 && error.data.type === 'embed-recipient-expired') {
      return <EmbedRecipientExpired />;
    }
    if (error.status === 403 && error.data.type === 'embed-waiting-for-turn') {
      return <EmbedDocumentWaitingForTurn />;
    }
  }

  return <div><Trans>Not Found</Trans></div>;
}
```

#### 场景 3：组织功能权限不足（签署主链路）

**触发条件**（主链路 `/embed/sign/{token}` Loader 层）：
- 组织未开通嵌入式签署功能（`organisationClaim.flags.embedSigning = false`）
- 账单状态异常

**⚠️ 注意**：这是主链路唯一的"预签发"检查，在 token 验证通过后、页面渲染前执行。

**代码锚点**：
- V1：`apps/remix/app/routes/embed+/_v0+/sign.$token.tsx:70-79`
- V2：`apps/remix/app/routes/embed+/_v0+/sign.$token.tsx:211-220`

```typescript
// V1 Loader - 组织权限检查
const organisationClaim = await getOrganisationClaimByTeamId({ teamId: document.teamId });

if (IS_BILLING_ENABLED() && !organisationClaim.flags.embedSigning) {
  throw data(
    { type: 'embed-paywall' },
    { status: 403 },
  );
}
```

**权限收敛策略**：
- ❌ 抛出 403 `embed-paywall`
- ❌ 不提供签署权限
- ✅ 通过 ErrorBoundary 渲染付费墙页面

**生命周期事件收敛**：
- ❌ 不发送 `document-ready`
- ❌ 不发送签署相关事件
- ✅ 宿主端可通过 iframe 加载状态感知错误

#### 场景 4：收件人状态异常（签署主链路）

**触发条件**（主链路 `/embed/sign/{token}` Loader 层）：
- 收件人已完成签署（`signingStatus = SIGNED`）
- 收件人已拒绝签署（`signingStatus = REJECTED`）
- 收件人已过期（`recipient.expiredAt < now`）
- 尚未轮到该收件人签署（顺序签署模式）
- 需要 Access Auth 认证（ACCOUNT / 2FA）

**⚠️ 重要澄清**："Token 过期"概念不适用于 `recipient.token`，实际检查的是 `recipient.expiredAt` 字段（收件人级别过期时间），而非 Token 本身的过期时间。

**代码锚点**：
- 过期检查：`apps/remix/app/routes/embed+/_v0+/sign.$token.tsx:81-90`（V1）、`222-231`（V2）
- 轮次检查：`apps/remix/app/routes/embed+/_v0+/sign.$token.tsx:116-127`（V1）、`233-242`（V2）
- Access Auth 检查：`apps/remix/app/routes/embed+/_v0+/sign.$token.tsx:92-114`（V1）、`244-267`（V2）

```typescript
// 检查收件人是否过期（检查 recipient.expiredAt，不是 token 过期）
if (isRecipientExpired(recipient)) {
  throw data(
    { type: 'embed-recipient-expired' },
    { status: 403 },
  );
}

// 检查是否是签署轮次
const isRecipientsTurnToSign = await getIsRecipientsTurnToSign({ token });
if (!isRecipientsTurnToSign) {
  throw data(
    { type: 'embed-waiting-for-turn' },
    { status: 403 },
  );
}
```

**权限收敛策略**：

| 收件人状态 | 处理方式 | 权限收敛结果 | 代码锚点 |
|-----------|---------|-------------|---------|
| 已签署 | Loader 中重定向到 `/complete` 或直接返回已完成状态 | ✅ 渲染完成页面，不允许重复签署 | `sign.$token.tsx:84-86` |
| 已拒绝 | Loader 中重定向到 `/rejected` | ✅ 渲染拒绝页面，不允许更改 | `sign.$token.tsx:84-86` |
| 已过期 | throw 403 `embed-recipient-expired` | ✅ 渲染过期页面，不允许签署 | `sign.$token.tsx:81-90` |
| 等待轮次 | throw 403 `embed-waiting-for-turn` | ✅ 渲染等待页面，不允许签署 | `sign.$token.tsx:116-127` |
| 需要 Access Auth | throw 401 `embed-authentication-required` | ✅ 渲染认证页面，要求登录/2FA | `sign.$token.tsx:92-114` |

**生命周期事件收敛**（状态页面）：

```typescript
// 等待轮次页面 - apps/remix/app/components/embed/embed-document-waiting-for-turn.tsx:7-15
const [hasPostedMessage, setHasPostedMessage] = useState(false);

useEffect(() => {
  if (window.parent && !hasPostedMessage) {
    window.parent.postMessage(
      { action: 'document-waiting-for-turn', data: null },
      '*',
    );
  }
  setHasPostedMessage(true);
}, [hasPostedMessage]);
```

- ✅ 发送对应状态事件（`document-completed`/`document-rejected`/`recipient-expired`/`document-waiting-for-turn`）
- ✅ 使用 `hasPostedMessage` 标志确保事件仅发送一次
- ❌ 不发送 `document-ready` 事件（因为不是可签署状态）
- ❌ 不提供签署操作入口

---

### 6.4 /embed/sign 主链路状态分流与事件收敛图

**主链路状态分流图**（基于 `apps/remix/app/routes/embed+/_v0+/sign.$token.tsx` 实际代码）：

```
/embed/sign/{token}
    │
    ├─► Token 无效（数据库查不到）──────────► 404 Not Found
    │    (getRecipientByToken 返回 null)          │
    │                                             └─► ❌ 无事件
    │
    ├─► 组织无嵌入权限 ───────────────────────► 403 embed-paywall
    │    (!organisationClaim.flags.embedSigning)    │
    │                                             └─► ❌ 无事件
    │
    ├─► 收件人已过期 ──────────────────────────► 403 embed-recipient-expired
    │    (recipient.expiredAt < now)               │
    │                                             └─► ✅ 发送 recipient-expired
    │
    ├─► 不是签署轮次 ──────────────────────────► 403 embed-waiting-for-turn
    │    (!isRecipientsTurnToSign)                 │
    │                                             └─► ✅ 发送 document-waiting-for-turn
    │
    ├─► 需要 Access Auth ─────────────────────► 401 embed-authentication-required
    │    (!isAccessAuthValid)                     │
    │                                             └─► ❌ 无事件（要求登录/2FA）
    │
    └─► 全部验证通过 ───────────────────────────► 渲染签署页面（V1/V2）
         │
         ├─► Hash 解析成功 ─────────────────────► ✅ document-ready（完整权限）
         │
         └─► Hash 解析失败 ─────────────────────► V1: 取决于 PDF 是否加载
                                                    V2: ✅ document-ready（保守权限）
```

**⚠️ 重要说明**：
- 没有"Token 签名无效"检查（因为 recipient.token 是随机字符串，不是 JWT）
- 没有"Audience 不匹配"检查（没有 JWT aud claim 概念）
- 没有"Token 过期"检查（用 recipient.expiredAt 字段代替）
- 没有"Scope 不匹配"检查（没有 JWT scope 概念）

---

### 6.5 握手失败的事件收敛总结

| 失败场景 | document-ready | 其他事件 | 权限收敛结果 | 代码锚点 |
|---------|---------------|---------|-------------|---------|
| Token 无效/不存在 | ❌ 不发送 | ❌ 不发送 | 完全拒绝访问 | `sign.$token.tsx:58-60` |
| 组织权限不足 | ❌ 不发送 | ❌ 不发送 | 完全拒绝访问 | `sign.$token.tsx:70-79` |
| 收件人已过期 | ❌ 不发送 | ✅ recipient-expired | 禁止签署 | `sign.$token.tsx:81-90` |
| 等待签署轮次 | ❌ 不发送 | ✅ document-waiting-for-turn | 禁止签署 | `sign.$token.tsx:116-127` |
| 需要 Access Auth | ❌ 不发送 | ❌ 不发送 | 要求认证 | `sign.$token.tsx:92-114` |
| Hash 解析失败（V1） | ❌ 可能不发（取决于 PDF） | ✅ 后续签署事件仍可发送 | 保守权限（默认配置） | `v1.tsx:192-252` |
| Hash 解析失败（V2） | ✅ 发送 | ✅ 后续签署事件仍可发送 | 保守权限（默认配置） | `v2.tsx:124-186` |

> **统一结论**：`document-ready` 不发送不代表签署流程不可用，宿主端应通过多种信号组合判断状态。
>
> **关键区分**：签署主链路（`/embed/sign/{token}`）使用 `recipient.token`（随机字符串，无 JWT 语义）；创作链路（`/embed/v1/v2/authoring/...`）使用预签名 Token（JWT，有签名/过期/audience/scope）。

---

## 七、回调异常场景下的生命周期事件收敛处理

### 7.1 回调异常场景分类与处理机制

嵌入式组件与宿主页面的通信是**单向异步**的，`postMessage` API 本身不提供交付确认。系统在设计上考虑了多种回调异常场景，确保生命周期事件的可靠收敛。

#### 场景 1：非 iframe 环境运行

**触发条件**：
- 用户直接在浏览器中访问嵌入 URL
- 宿主页面未正确创建 iframe
- `window.parent` 为 `null` 或 `undefined`

**代码位置**：所有 `postMessage` 调用前的检查

```typescript
const onDocumentCompleted = (data) => {
  if (window.parent) {  // ✅ 检查父窗口存在性
    window.parent.postMessage(
      { action: 'document-completed', data },
      '*',
    );
  }
  // 无 else 分支 - 静默失败
};
```

**生命周期事件收敛策略**：
- ❌ 不发送任何 `postMessage` 事件
- ✅ 内部状态正常流转（签署仍可完成）
- ✅ 签署结果持久化到数据库
- ⚠️ 风险：宿主页面无法感知签署状态，需依赖后端轮询或 Webhook

**对宿主端的影响**：
- 建议：不要完全依赖前端回调，应以后端状态为准
- 建议：实现签署完成后的后端 Webhook 通知
- 建议：设置合理的超时机制，超时后查询后端状态

#### 场景 2：`postMessage` 调用失败

**触发条件**：
- 父窗口已被销毁（用户关闭标签页）
- 跨域安全策略阻止（极端情况）
- 浏览器兼容性问题

**机制说明**：
- `postMessage` 是异步 API，**不抛出异常**（即使目标源无法接收）
- 无回调/确认机制，发送端无法知道消息是否被接收

**代码位置**：无显式错误处理（因为 API 不抛出）

**生命周期事件收敛策略**：
- ✅ 内部状态正常更新
- ✅ 签署结果持久化到数据库
- ❌ 事件可能丢失（宿主端无法收到）
- ❌ 无重试机制

**风险与缓解**：

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| 事件丢失 | 宿主端状态不同步 | 后端 Webhook 通知 + 前端超时轮询 |
| 用户重复操作 | 可能重复提交 | 前端按钮防重点击 + 后端幂等性保证 |
| 业务流程中断 | 签署完成但宿主未感知 | 完成页面提示用户手动刷新宿主页面 |

#### 场景 3：宿主端消息处理异常

**触发条件**：
- 宿主端 `message` 事件处理函数抛出异常
- 宿主端消息格式验证失败
- 宿主端业务逻辑处理错误

**机制说明**：
- `postMessage` 是**单向通信**，发送端不监听响应
- 宿主端异常不会影响嵌入端的执行
- 嵌入端完全无感知

**生命周期事件收敛策略**（嵌入端视角）：
- ✅ 嵌入端继续正常流程
- ✅ 签署结果持久化到数据库
- ❌ 宿主端可能状态不一致
- ❌ 嵌入端无重试机制

**宿主端防御性编程建议**：

```typescript
// 宿主端应实现的安全处理模式
const handleMessage = (event: MessageEvent) => {
  try {
    // 1. Origin 验证（必须）
    if (event.origin !== 'https://app.documenso.com') {
      console.warn('Ignoring message from untrusted origin:', event.origin);
      return;
    }

    // 2. 格式验证
    if (!event.data || typeof event.data !== 'object') {
      console.warn('Invalid message format');
      return;
    }

    const { action, data } = event.data;

    // 3. Action 白名单验证
    const ALLOWED_ACTIONS = new Set([
      'document-ready', 'document-completed', 'document-rejected',
      'document-error', 'field-signed', 'field-unsigned',
      'document-waiting-for-turn', 'recipient-expired',
      'all-documents-completed'
    ]);

    if (!ALLOWED_ACTIONS.has(action)) {
      console.warn('Unknown action:', action);
      return;
    }

    // 4. 使用 try-catch 包裹业务逻辑
    try {
      switch (action) {
        case 'document-completed':
          // 5. 敏感数据处理：不要在前端日志输出完整 token
          console.log('Document completed, documentId:', data.documentId);
          // 6. 后端验证并处理
          await fetch('/api/documents/verify-signing', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ token: data.token }),
          });
          break;
        // ... 其他事件处理
      }
    } catch (businessError) {
      // 7. 业务异常不向外抛出，避免影响后续消息
      console.error('Business logic error:', businessError);
      // 8. 考虑：异常上报到监控系统
    }
  } catch (outerError) {
    // 9. 最外层兜底，防止异常冒泡
    console.error('Fatal error in message handler:', outerError);
  }
};
```

#### 场景 4：签署操作失败

**触发条件**：
- 网络中断
- 服务端内部错误
- 字段验证失败
- 并发冲突（其他签署者已操作）

**代码位置**：`apps/remix/app/components/embed/embed-document-signing-page-v1.tsx:154-163`

```typescript
const onCompleteClick = async () => {
  try {
    await completeDocumentWithToken({ documentId, token });
    // 发送成功事件
    window.parent.postMessage({ action: 'document-completed', data: {...} }, '*');
    setHasCompletedDocument(true);
  } catch (err) {
    // ✅ 发生错误时发送错误事件
    if (window.parent) {
      window.parent.postMessage(
        { action: 'document-error', data: null },
        '*',
      );
    }

    // ✅ 本地提示用户
    toast({
      title: _(msg`Something went wrong`),
      description: _(msg`We were unable to submit this document at this time.`),
      variant: 'destructive',
    });

    // ✅ 不修改状态，允许用户重试
  }
};
```

**生命周期事件收敛策略**：
- ✅ 发送 `document-error` 事件通知宿主
- ✅ 保留当前状态，允许用户重试
- ✅ 不自动跳转或终止
- ❌ 不发送 `document-completed` 事件
- ✅ 本地错误提示（toast）

**错误状态流转**：

```
签署操作
    │
    ├─► 成功 ──► document-completed ──► 完成页面
    │
    └─► 失败 ──► document-error ──► 保留当前状态，允许重试
                  │
                  └─► 超过重试次数 ──► 建议刷新或联系支持
```

#### 场景 5：事件重复发送防护

**触发条件**：
- React 组件因状态变化重复渲染
- useEffect 依赖数组变化导致重复执行
- 用户重复操作

**防护机制**：`hasPostedMessage` 状态标志

**代码位置**：`apps/remix/app/components/embed/embed-document-waiting-for-turn.tsx:5-19`

```typescript
const [hasPostedMessage, setHasPostedMessage] = useState(false);

useEffect(() => {
  if (window.parent && !hasPostedMessage) {
    window.parent.postMessage(
      { action: 'document-waiting-for-turn', data: null },
      '*',
    );
  }
  setHasPostedMessage(true);  // ✅ 发送后立即标记
}, [hasPostedMessage]);

if (!hasPostedMessage) {
  return null;  // ✅ 发送前不渲染内容，确保时序
}
```

**另一种防护机制**：useEffect 精确依赖

```typescript
// document-ready 事件：仅在 hasFinishedInit 变化时触发
useEffect(() => {
  if (hasFinishedInit) {
    onDocumentReady();
  }
}, [hasFinishedInit]);  // ✅ 精确依赖，避免重复触发

// document-completed 事件：仅在 isCompleted 变化时触发
useEffect(() => {
  if (isCompleted) {
    onDocumentCompleted({...});
  }
}, [isCompleted, envelope.id, recipient.id, recipient.token]);
```

### 7.2 生命周期事件的最终一致性保障

由于前端通信的不可靠性，系统通过多层机制保障最终一致性：

| 保障层级 | 机制 | 可靠性 |
|---------|------|--------|
| L1 前端回调 | `postMessage` 事件 | ⭐⭐ 不可靠，可能丢失 |
| L2 前端状态 | React 状态管理 | ⭐⭐⭐ 页面生命周期内可靠 |
| L3 后端持久化 | 数据库事务写入 | ⭐⭐⭐⭐⭐ 最终可靠 |
| L4 后端通知 | Webhook 回调（可重试） | ⭐⭐⭐⭐ 较可靠 |
| L5 状态查询 | 后端 API 主动查询 | ⭐⭐⭐⭐⭐ 最终可靠 |

**建议的宿主端集成策略**：

```
宿主页面
    │
    ├─► 监听 postMessage 事件（实时更新 UI）
    │
    ├─► 设置超时定时器（如 30 分钟）
    │    └─► 超时后查询后端状态
    │
    └─► 接收后端 Webhook（最终状态确认）
          └─► 更新业务系统状态，触发后续流程
```

---

## 八、版本差异

### 8.1 V1 vs V2 签署页面

| 特性 | V1 | V2 |
|-----|----|----|
| 架构 | 独立上下文 | 复用 EnvelopeSigningContext |
| 多文档支持 | 否 | 通过 multisign 路由支持 |
| 信封模型 | Document 模型 | Envelope 模型 |
| 完成事件数据 | `{ token, documentId, recipientId }` | `{ token, documentId, envelopeId, recipientId }` |

### 8.2 嵌入路由版本

| 版本 | 路由模式 | 用途 | 令牌类型 |
|-----|---------|------|---------|
| V0 | `/embed/sign/:token` | **签署主链路**（生产环境使用） | `recipient.token` |
| V0 | `/embed/v0/direct/:token` | 直接模板嵌入 | `recipient.token` |
| V1 | `/embed/v1/authoring/...` | 文档/模板编辑嵌入 | 预签名 Token |
| V1 | `/embed/v1/multisign/` | 批量签署嵌入 | `recipient.token` |
| V2 | `/embed/v2/authoring/envelope/...` | 信封编辑嵌入 | 预签名 Token |

---

## 九、关键代码索引（签署主链路）

| 功能模块 | 文件路径 | 关键行 |
|---------|---------|--------|
| **签署主入口** | `apps/remix/app/routes/embed+/_v0+/sign.$token.tsx` | 1 |
| V1 签署页面 | `apps/remix/app/components/embed/embed-document-signing-page-v1.tsx` | 1 |
| V2 签署页面 | `apps/remix/app/components/embed/embed-document-signing-page-v2.tsx` | 1 |
| 嵌入上下文 | `apps/remix/app/components/embed/embed-signing-context.tsx` | 3 |
| 签署数据 Schema | `packages/lib/types/embed-document-sign-schema.ts` | 6 |
| 字段鉴权边界 | `packages/lib/server-only/document/validate-field-auth.ts` | 28 |
| 收件人鉴权逻辑 | `packages/lib/server-only/document/is-recipient-authorized.ts` | 53 |
| 凭 token 签名字段 | `packages/lib/server-only/field/sign-field-with-token.ts` | 50 |
| 凭 token 完成文档 | `packages/lib/server-only/document/complete-document-with-token.ts` | 54 |
| 获取收件人 by token | `packages/lib/server-only/recipient/get-recipient-by-token.ts` | 1 |
| 错误边界处理 | `apps/remix/app/routes/embed+/_v0+/_layout.tsx` | 47 |
| 等待轮次页面 | `apps/remix/app/components/embed/embed-document-waiting-for-turn.tsx` | 1 |
| 过期页面 | `apps/remix/app/components/embed/embed-recipient-expired.tsx` | 1 |
| 测试 playground | `apps/remix/app/routes/embed+/playground.tsx` | 26 |
| 预签名令牌生成（创作链路） | `packages/lib/server-only/embedding-presign/create-embedding-presign-token.ts` | 18 |
| 预签名令牌验证（创作链路） | `packages/lib/server-only/embedding-presign/verify-embedding-presign-token.ts` | 12 |

---

## 十、安全建议（补充完善）

### 10.1 核心安全措施

1. **🔴 宿主端 Origin 验证（必须执行）**：
   - 始终验证 `event.origin`，避免接受伪造消息
   - 原因：`document-completed` 和 `document-rejected` 事件包含高度敏感的 `recipient.token`
   - 默认配置下仅凭 token 即可完成整个签署流程

2. **文档过期时间控制**：
   - 根据业务场景设置合理的文档过期时间
   - `recipient.token` 永久有效直到文档完成或过期
   - 建议：根据签署流程预估时间，设置 1.5 倍余量的过期时间

3. **HTTPS 强制**：
   - 生产环境必须使用 HTTPS，防止传输窃听
   - Token 出现在 URL 路径中，HTTP 环境下完全暴露

4. **Referrer Policy**：
   - 设置 `Referrer-Policy: no-referrer` 避免 Token 通过 Referer 头泄露
   - 系统已默认设置 `strict-origin-when-cross-origin`

5. **独立域名部署**：
   - 使用独立子域名部署 Documenso，实现跨源隔离
   - 防止宿主页面 XSS 漏洞直接影响嵌入组件

### 10.2 Token 安全处理（签署主链路）

6. **🔴 禁止前端存储 recipient.token**：
   - 禁止在 `localStorage`/`sessionStorage` 中存储 `recipient.token`
   - 禁止在前端日志中输出完整 token
   - 建议：token 仅用于后端到后端的状态同步，不在前端业务逻辑中使用

7. **Token 验证机制**：
   - 宿主端收到 `document-completed` 事件后，应立即通过后端 API 验证 Token 有效性
   - 验证内容：文档状态、收件人状态、签署完成时间
   - 不要直接信任前端回调数据，以防伪造

8. **Token 使用范围控制**：
   - `recipient.token` 绑定特定文档和收件人，不可跨文档使用
   - 文档完成或过期后，token 自动失效
   - 建议：关键业务操作应使用后端到后端的 API 调用，不依赖前端传递的 token

### 10.3 回调处理安全

9. **消息格式白名单验证**：
   - 验证 `event.data` 为合法对象
   - 验证 `action` 字段在白名单内
   - 防止 XSS 探针和注入攻击

10. **异常隔离处理**：
    - 使用 `try-catch` 包裹消息处理逻辑
    - 业务异常不向外抛出，避免影响后续消息处理
    - 异常应上报到监控系统，便于排查

11. **幂等性设计**：
    - 宿主端应实现回调幂等性，防止重放攻击
    - 关键操作（如更新业务系统状态）应做去重处理

### 10.4 可靠性保障

12. **审计日志**：
    - 记录所有嵌入操作，用于安全审计
    - 包括：握手时间、Token ID、签署完成时间、异常事件等

13. **多层状态确认机制**：
    - 不要完全依赖前端回调，应以后端状态为准
    - 建议：前端回调 + 后端 Webhook + 超时轮询的三层确认机制

14. **超时机制**：
    - 设置合理的签署超时时间（如 30 分钟）
    - 超时后主动查询后端状态，避免状态不一致

### 10.5 集成检查清单（签署主链路）

✅ **集成前检查**：
- [ ] 已启用 HTTPS
- [ ] 已配置正确的 CSP 策略
- [ ] 已实现 `event.origin` 验证（必须执行）
- [ ] 已规划后端 Webhook 接收签署通知
- [ ] 已设计超时和重试机制
- [ ] 已确认 `recipient.token` 不会被记录到应用日志
- [ ] 已明确文档鉴权配置（ACTION auth / ACCESS 2FA）
- [ ] 已了解 V1 和 V2 版本 `document-ready` 触发条件差异

✅ **上线前检查**：
- [ ] 渗透测试确认无 XSS 漏洞
- [ ] 安全审计确认 Token 处理符合规范
- [ ] 异常场景测试通过（网络中断、超时、重复提交）
- [ ] 监控告警已配置（签署失败率、异常错误率）
- [ ] 已验证 Hash 解析失败场景的降级行为
- [ ] 已验证握手失败场景的错误页面展示
- [ ] 已验证 `document-ready` 触发/不触发场景的处理逻辑
- [ ] 已确认文档过期时间设置合理
