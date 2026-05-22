# 嵌入式签署组件通信链路分析

## 一、架构概述

Documenso 嵌入式签署组件采用 **iframe + postMessage** 跨域通信架构，通过三层安全验证机制实现宿主页面与签署组件之间的可信通信。整个系统分为三个核心部分：

1. **宿主页面（Parent）**：集成方业务系统，负责发起签署请求、接收状态回调
2. **预签名授权层（Presign）**：基于 JWT 的短期授权令牌机制
3. **嵌入式签署组件（Iframe）**：Documenso 提供的签署 UI，运行在隔离沙箱中

```
┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐
│   宿主页面       │        │  Documenso API  │        │  嵌入式签署组件   │
│   (Parent)      │────────│   (Presign)     │────────│   (Iframe)      │
└─────────────────┘        └─────────────────┘        └─────────────────┘
          │                        │                         │
          └──────────postMessage─────────────────────────────┘
                     (跨域双向通信)
```

---

## 二、初始化握手流程

### 2.1 握手时序图

```
宿主页面                          Documenso API                          嵌入式组件
    │                                   │                                   │
    │ 1. 请求预签名令牌                  │                                   │
    │ POST /api/v2/embedding/create-presign-token                         │
    │ Header: Authorization: Bearer api_xxx │                                   │
    │──────────────────────────────────>│                                   │
    │                                   │ 2. 验证 API Token                │
    │                                   │ 3. 生成 JWT 预签名令牌            │
    │                                   │    (HS256, 有效期默认1小时)       │
    │ <──────────────────────────────────│                                   │
    │ 4. 返回 { token, expiresAt }      │                                   │
    │                                                                       │
    │ 5. 构建 iframe URL                │                                   │
    │    /embed/v2/authoring/envelope/create?token=<presign>#<hash>         │
    │                                   │                                   │
    │ 6. 创建 iframe 加载签署页面        │                                   │
    │──────────────────────────────────────────────────────────────────────>│
    │                                   │ 7. 服务端 Loader 验证令牌         │
    │                                   │    verifyEmbeddingPresignToken    │
    │                                   │ <──────────────────────────────────│
    │                                   │ 8. 返回用户上下文 + 团队配置       │
    │                                   │──────────────────────────────────>│
    │                                   │                                   │ 9. 解析 URL Hash 配置
    │                                   │                                   │    (Base64(JSON))
    │                                   │                                   │ 10. 发送 document-ready 事件
    │ <──────────────────────────────────────────────────────────────────────│
    │ 11. 握手完成，进入签署流程          │                                   │
```

### 2.2 URL Hash 配置协议

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

### 3.1 双令牌体系

系统采用 **API Token + 预签名 Token** 双令牌机制，实现权限的安全传递和最小化暴露。

| 令牌类型 | 用途 | 有效期 | 存储位置 | 安全性 |
|---------|------|--------|----------|--------|
| API Token | 宿主后端调用 Documenso API | 长期 | 宿主后端数据库 | 高，永不暴露给前端 |
| 预签名 Token | 前端 iframe 认证 | 5分钟~24小时 | URL 查询参数 | 中，短期有效，绑定用户/团队 |

### 3.2 预签名令牌生成

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

### 3.3 预签名令牌验证

**验证流程**（`packages/lib/server-only/embedding-presign/verify-embedding-presign-token.ts:12`）：

```typescript
const verifyEmbeddingPresignToken = async ({ token, scope }) => {
  // 1. 解码 JWT 获取 Claims（不验证签名）
  const decodedToken = decodeJwt<JWTPayload>(token);

  // 2. 验证必要 Claims 存在且格式正确
  if (!decodedToken.sub || !decodedToken.aud) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, 'Invalid presign token format');
  }

  // 3. 通过 sub 查找原始 API Token
  const apiToken = await prisma.apiToken.findFirst({
    where: { id: Number(decodedToken.sub) },
    include: { user: true },
  });

  // 4. 验证 aud 与 API Token 所属团队/用户匹配
  if (audienceId !== apiToken.teamId && audienceId !== apiToken.userId) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, 'Audience mismatch');
  }

  // 5. 验证 scope 匹配（如果提供）
  if (decodedToken.scope && scope && decodedToken.scope !== scope) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, 'Scope not matched');
  }

  // 6. 使用 API Token 作为密钥验证 JWT 签名
  const secret = new TextEncoder().encode(apiToken.token);
  await jwtVerify(token, secret);

  return { ...apiToken, userId, user };
};
```

### 3.4 服务端路由权限校验

在嵌入式路由的 Loader 中，每次请求都会验证令牌：

**Layout 层验证**（`apps/remix/app/routes/embed+/v2+/authoring+/_layout.tsx:26`）：

```typescript
export const loader = async ({ request }) => {
  const token = url.searchParams.get('token');
  if (!token) throw new Response('Invalid token', { status: 404 });

  // 验证预签名令牌
  const result = await verifyEmbeddingPresignToken({ token }).catch(() => null);
  if (!result) throw new Response('Invalid token', { status: 404 });

  // 获取组织权限声明
  const organisationClaim = await getOrganisationClaimByTeamId({
    teamId: result.teamId,
  });

  return { token, userId: result.userId, teamId: result.teamId, organisationClaim };
};
```

**TRPC API 层验证**（`packages/trpc/server/embedding-router/create-embedding-document.ts:17`）：

```typescript
export const createEmbeddingDocumentRoute = procedure
  .input(ZCreateEmbeddingDocumentRequestSchema)
  .mutation(async ({ input, ctx: { req } }) => {
    const authorizationHeader = req.headers.get('authorization');
    const [presignToken] = (authorizationHeader || '').split('Bearer ').filter(Boolean);

    if (!presignToken) {
      throw new AppError(AppErrorCode.UNAUTHORIZED, 'No presign token provided');
    }

    // 验证预签名令牌，获取用户上下文
    const apiToken = await verifyEmbeddingPresignToken({ token: presignToken });

    // 使用令牌关联的用户ID执行后续操作
    const envelope = await createEnvelope({
      userId: apiToken.userId,
      teamId: apiToken.teamId ?? undefined,
      // ...
    });
  });
```

### 3.5 功能权限粒度控制

通过 `buildEmbeddedEditorOptions` 实现细粒度功能权限控制：

**权限配置合并**（`packages/lib/utils/embed-config.ts:14`）：

```typescript
export const buildEmbeddedEditorOptions = (
  features: DeepPartial<EnvelopeEditorConfig>,
  embedded: EnvelopeEditorConfig['embedded'],
): EnvelopeEditorConfig => {
  return {
    embedded,
    ...buildEmbeddedFeatures(features),
  };
};
```

**功能权限分组**：
- `general`：通用功能（标题配置、步骤控制、侧边栏控制）
- `settings`：设置权限（签名类型、语言、日期格式、时区等）
- `actions`：操作权限（附件、分发、下载、删除等）
- `envelopeItems`：文档项权限（标题、排序、上传、替换等）
- `recipients`：收件人权限（AI 检测、签署顺序、角色配置等）
- `fields`：字段权限（AI 检测等）

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

#### 5.2.3 跨域事件中 Token 的敏感性重新评估

**关键发现**：`document-completed` 和 `document-rejected` 事件中携带的 `token` 字段是 `recipient.token`，这是一个**高度敏感**的认证凭证，而非普通标识符。

**Token 可执行的操作**（`packages/lib/server-only/field/sign-field-with-token.ts:50`）：

```typescript
// 仅需 token 和 fieldId 即可签署字段，无需其他认证
export const signFieldWithToken = async ({
  token,           // recipient.token
  fieldId,
  value,
  isBase64,
  // userId 和 authOptions 是可选的！
}: SignFieldWithTokenOptions) => {
  const recipient = await prisma.recipient.findFirstOrThrow({
    where: { token },  // 仅通过 token 查找收件人
  });

  // ... 执行字段签署，包括签名字段
};
```

**完整权限清单**：

| 操作 | 所需参数 | 代码位置 |
|------|---------|---------|
| 签署任意字段 | `token` + `fieldId` + `value` | `sign-field-with-token.ts:50` |
| 完成文档签署 | `token` + `documentId` | `completeDocumentWithToken` API |
| 查看文档内容 | `token` | `getDocumentAndSenderByToken.ts` |
| 获取字段列表 | `token` | `getFieldsForToken.ts` |
| 标记已查看 | `token` | `viewedDocument.ts` |
| 拒绝文档 | `token` + `reason` | `rejectDocumentWithToken` API |
| 批量签署 | `tokens[]` + `signature` | `apply-multi-sign-signature.ts:15` |

**风险等级评估**：
- **敏感性**：🔴 极高 - 窃取后可伪造完整签署流程
- **可滥用性**：🔴 极高 - 仅需 token 即可完成所有操作，无二次认证
- **影响范围**：🟠 中 - 仅影响对应收件人的特定文档
- **时效性**：🟡 中 - 文档签署完成或过期后失效

**事件暴露情况对比**：

| 事件类型 | 是否暴露 token | 敏感性 |
|---------|---------------|--------|
| `document-ready` | ❌ 否 | 低 |
| `field-signed` | ❌ 否 | 低 |
| `field-unsigned` | ❌ 否 | 低 |
| `document-completed` | ✅ 是 | 极高 |
| `document-rejected` | ✅ 是 | 极高 |
| `document-error` | ❌ 否 | 低 |
| `document-waiting-for-turn` | ❌ 否 | 低 |
| `recipient-expired` | ❌ 否 | 低 |
| `all-documents-completed` | ✅ 是（每个文档） | 极高 |

**对宿主端回调处理策略的影响**：

1. **Origin 验证从「建议」升级为「必须」**：
   - 原结论：`event.origin` 验证是建议性的安全加固
   - 新结论：**必须**验证，否则恶意父窗口可窃取 `recipient.token` 并伪造签署

2. **Token 处理策略**：
   - 禁止在前端日志中输出完整 token
   - 禁止将 token 存储在 localStorage/sessionStorage
   - 建议：宿主端收到 token 后立即通过后端 API 验证并作废（如适用）
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

### 5.5 Token 安全边界

#### 5.5.1 Token 暴露风险

| 风险点 | 说明 | 缓解措施 |
|-------|------|---------|
| URL 查询参数 | 预签名 Token 出现在 URL 中，可能被历史记录、日志捕获 | 1. 短期有效期<br>2. 仅可单次使用场景<br>3. HTTPS 传输 |
| Referer 泄露 | iframe 请求可能通过 Referer 头泄露 Token | 1. 设置 Referrer-Policy<br>2. 服务端验证 Token 有效期 |
| XSS 窃取 | 宿主页面 XSS 可能窃取 iframe Token | 1. Token 绑定用户/团队<br>2. 短期有效期<br>3. 独立子域名部署 |

#### 5.5.2 令牌过期策略

```typescript
// 生产环境：最小5分钟，默认60分钟
const isDevelopment = env('NODE_ENV') !== 'production';
const minExpirationMinutes = isDevelopment ? 0 : 5;
const effectiveExpiresIn = expiresIn >= minExpirationMinutes ? expiresIn : 60;
```

### 5.6 路由访问控制

嵌入式路由通过多层防护确保安全：

1. **Loader 层 Token 验证**：每次页面加载验证预签名 Token
2. **Layout 层权限声明**：获取组织功能权限声明
3. **TRPC 中间件验证**：每个 API 调用再次验证 Token
4. **Context 层环境标记**：通过 React Context 标记嵌入环境

---

## 六、握手失败场景下的权限收敛处理

### 6.1 握手失败场景分类与处理机制

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
- ❌ 不发送 `document-ready` 事件（若解析失败发生在初始化早期）
- ✅ 仍会渲染签署页面，但使用默认配置
- ⚠️ 风险：用户可能在未授权配置下完成签署，但使用最保守权限

#### 场景 2：预签名 Token 验证失败

**触发条件**：
- Token 格式错误
- Token 已过期
- Token 签名无效
- Token 所属 API Token 已被吊销
- Audience（团队/用户ID）不匹配

**代码位置**：
- Layout 层：`apps/remix/app/routes/embed+/v2+/authoring+/_layout.tsx:35-39`
- API 层：`packages/lib/server-only/embedding-presign/verify-embedding-presign-token.ts:12`

```typescript
// Layout 层验证
export const loader = async ({ request }) => {
  const token = url.searchParams.get('token');
  if (!token) throw new Response('Invalid token', { status: 404 });

  const result = await verifyEmbeddingPresignToken({ token }).catch(() => null);
  if (!result) throw new Response('Invalid token', { status: 404 });
  // ...
};
```

**权限收敛策略**：

| 验证阶段 | 失败处理 | 权限收敛结果 |
|---------|---------|-------------|
| Token 格式检查 | 抛出 404 响应 | ❌ 完全拒绝访问 |
| API Token 存在性检查 | 抛出 401 错误 | ❌ 完全拒绝访问 |
| Audience 匹配检查 | 抛出 401 错误 | ❌ 完全拒绝访问 |
| Scope 匹配检查 | 抛出 401 错误 | ❌ 完全拒绝访问 |
| JWT 签名验证 | 抛出 401 错误 | ❌ 完全拒绝访问 |
| Token 过期检查 | 抛出 401 错误 | ❌ 完全拒绝访问 |

**生命周期事件收敛**：
- ❌ 不发送任何生命周期事件
- ❌ 不加载任何签署组件
- ✅ 通过 ErrorBoundary 渲染通用错误页面

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
    // ... 其他错误类型
  }

  return <div><Trans>Not Found</Trans></div>;
}
```

#### 场景 3：组织功能权限不足

**触发条件**：
- 组织未开通嵌入式签署功能
- 组织未开通白标定制功能
- 账单状态异常

**代码位置**：`packages/trpc/server/embedding-router/create-embedding-presign-token.ts:34-52`

```typescript
if (IS_BILLING_ENABLED()) {
  const organisationClaim = await getOrganisationClaimByTeamId({
    teamId: token.teamId,
  });

  if (!organisationClaim.flags.embedAuthoring) {
    throw new AppError(AppErrorCode.UNAUTHORIZED, {
      message: 'Embedded Authoring is not included in your current plan.',
    });
  }
}
```

**权限收敛策略**：
- ❌ 不生成预签名 Token
- ❌ API 调用直接失败，返回 401 错误
- ✅ 宿主端应终止嵌入流程，展示升级提示

**生命周期事件收敛**：
- ❌ 不创建 iframe
- ❌ 不发送任何事件
- ✅ 宿主端负责处理错误展示

#### 场景 4：收件人状态异常

**触发条件**：
- 收件人已完成签署
- 收件人已拒绝签署
- 签署链接已过期
- 尚未轮到该收件人签署

**代码位置**：`apps/remix/app/routes/_recipient+/sign.$token+/_index.tsx:137-147`

```typescript
// Loader 层状态检查
if (recipient.signingStatus === SigningStatus.REJECTED) {
  throw redirect(`/sign/${token}/rejected`);
}

if (isRecipientExpired(recipient)) {
  throw redirect(`/sign/${token}/expired`);
}

if (document.status === DocumentStatus.COMPLETED || 
    recipient.signingStatus === SigningStatus.SIGNED) {
  throw redirect(documentMeta?.redirectUrl || `/sign/${token}/complete`);
}
```

**权限收敛策略**：

| 收件人状态 | 处理方式 | 权限收敛结果 |
|-----------|---------|-------------|
| 已签署 | 重定向到 `/complete` | ✅ 渲染完成页面，不允许重复签署 |
| 已拒绝 | 重定向到 `/rejected` | ✅ 渲染拒绝页面，不允许更改 |
| 已过期 | 重定向到 `/expired` | ✅ 渲染过期页面，不允许签署 |
| 等待轮次 | 重定向到 `/waiting` | ✅ 渲染等待页面，不允许签署 |

**生命周期事件收敛**（状态页面）：

```typescript
// 等待轮次页面 - apps/remix/app/components/embed/embed-document-waiting-for-turn.tsx:7
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

### 6.2 握手失败的状态流转图

```
握手开始
    │
    ├─► Token 验证失败 ──► 404 ErrorBoundary ──► 终止（无事件）
    │
    ├─► 权限不足 ───────► 401 API Error ───────► 终止（宿主处理）
    │
    ├─► 收件人状态异常 ──► 状态页面 ───────────► 发送状态事件（无签署权限）
    │
    ├─► Hash 解析失败 ───► 默认配置 ───────────► document-ready（保守权限）
    │
    └─► 全部成功 ───────► 完整配置 ───────────► document-ready（完整权限）
```

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

| 版本 | 路由模式 | 用途 |
|-----|---------|------|
| V0 | `/embed/v0/sign/:token` | 早期签署嵌入 |
| V0 | `/embed/v0/direct/:token` | 直接模板嵌入 |
| V1 | `/embed/v1/authoring/...` | 文档/模板编辑嵌入 |
| V1 | `/embed/v1/multisign/` | 批量签署嵌入 |
| V2 | `/embed/v2/authoring/envelope/...` | 信封编辑嵌入 |

---

## 九、关键代码索引

| 功能模块 | 文件路径 | 关键行 |
|---------|---------|--------|
| 预签名令牌生成 | `packages/lib/server-only/embedding-presign/create-embedding-presign-token.ts` | 18 |
| 预签名令牌验证 | `packages/lib/server-only/embedding-presign/verify-embedding-presign-token.ts` | 12 |
| 签署页面 V2 | `apps/remix/app/components/embed/embed-document-signing-page-v2.tsx` | 1 |
| 嵌入上下文 | `apps/remix/app/components/embed/embed-signing-context.tsx` | 3 |
| 配置合并 | `packages/lib/utils/embed-config.ts` | 14 |
| 签署数据 Schema | `packages/lib/types/embed-document-sign-schema.ts` | 6 |
| V2 创作布局 | `apps/remix/app/routes/embed+/v2+/authoring+/_layout.tsx` | 26 |
| 批量签署 | `apps/remix/app/routes/embed+/v1+/multisign+/_index.tsx` | 1 |
| 测试 playground | `apps/remix/app/routes/embed+/playground.tsx` | 26 |
| 创建嵌入式文档 | `packages/trpc/server/embedding-router/create-embedding-document.ts` | 14 |
| 创建预签名令牌 API | `packages/trpc/server/embedding-router/create-embedding-presign-token.ts` | 17 |

---

## 十、安全建议（补充完善）

### 10.1 核心安全措施

1. **🔴 宿主端 Origin 验证（必须执行）**：
   - 始终验证 `event.origin`，避免接受伪造消息
   - 原因：`document-completed` 和 `document-rejected` 事件包含高度敏感的 `recipient.token`

2. **Token 有效期控制**：
   - 根据业务场景设置最短必要的有效期
   - 生产环境最小 5 分钟，默认 60 分钟
   - 建议：根据签署流程预估时间，设置 1.5 倍余量

3. **HTTPS 强制**：
   - 生产环境必须使用 HTTPS，防止传输窃听
   - Token 出现在 URL 查询参数中，HTTP 环境下完全暴露

4. **Referrer Policy**：
   - 设置 `Referrer-Policy: no-referrer` 避免 Token 通过 Referer 头泄露
   - 系统已默认设置 `strict-origin-when-cross-origin`

5. **独立域名部署**：
   - 使用独立子域名部署 Documenso，实现跨源隔离
   - 防止宿主页面 XSS 漏洞直接影响嵌入组件

### 10.2 Token 安全处理

6. **禁止前端存储 Token**：
   - 禁止在 `localStorage`/`sessionStorage` 中存储 `recipient.token`
   - 禁止在前端日志中输出完整 Token
   - 建议：Token 仅用于后端到后端的状态同步

7. **Token 验证机制**：
   - 宿主端收到 `document-completed` 事件后，应立即通过后端 API 验证 Token 有效性
   - 不要直接信任前端回调数据，以防伪造

8. **定期轮换**：
   - API Token 定期轮换，降低泄露影响
   - 预签名 Token 本身为短期有效，无需额外轮换

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

### 10.5 集成检查清单

✅ **集成前检查**：
- [ ] 已启用 HTTPS
- [ ] 已配置正确的 CSP 策略
- [ ] 已实现 `event.origin` 验证
- [ ] 已规划后端 Webhook 接收签署通知
- [ ] 已设计超时和重试机制
- [ ] 已确认 Token 不会被记录到应用日志

✅ **上线前检查**：
- [ ] 渗透测试确认无 XSS 漏洞
- [ ] 安全审计确认 Token 处理符合规范
- [ ] 异常场景测试通过（网络中断、超时、重复提交）
- [ ] 监控告警已配置（签署失败率、异常错误率）
