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

> **安全注意**：使用 `'*'` 意味着消息可被任何父窗口接收。虽然事件数据不包含敏感信息（仅包含 token、ID 等标识符），但建议宿主端验证 `event.origin`。

#### 5.2.2 宿主端源验证最佳实践

宿主页面应始终验证消息来源：

```typescript
const handleMessage = (event: MessageEvent) => {
  // ✅ 验证消息来源
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

## 六、版本差异

### 6.1 V1 vs V2 签署页面

| 特性 | V1 | V2 |
|-----|----|----|
| 架构 | 独立上下文 | 复用 EnvelopeSigningContext |
| 多文档支持 | 否 | 通过 multisign 路由支持 |
| 信封模型 | Document 模型 | Envelope 模型 |
| 完成事件数据 | `{ token, documentId, recipientId }` | `{ token, documentId, envelopeId, recipientId }` |

### 6.2 嵌入路由版本

| 版本 | 路由模式 | 用途 |
|-----|---------|------|
| V0 | `/embed/v0/sign/:token` | 早期签署嵌入 |
| V0 | `/embed/v0/direct/:token` | 直接模板嵌入 |
| V1 | `/embed/v1/authoring/...` | 文档/模板编辑嵌入 |
| V1 | `/embed/v1/multisign/` | 批量签署嵌入 |
| V2 | `/embed/v2/authoring/envelope/...` | 信封编辑嵌入 |

---

## 七、关键代码索引

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

## 八、安全建议

1. **宿主端 Origin 验证**：始终验证 `event.origin`，避免接受伪造消息
2. **Token 有效期控制**：根据业务场景设置最短必要的有效期
3. **HTTPS 强制**：生产环境必须使用 HTTPS，防止传输窃听
4. **Referrer Policy**：设置 `Referrer-Policy: no-referrer` 避免 Token 泄露
5. **独立域名部署**：使用独立子域名部署 Documenso，实现跨源隔离
6. **审计日志**：记录所有嵌入操作，用于安全审计
7. **定期轮换**：API Token 定期轮换，降低泄露影响
