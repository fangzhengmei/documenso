# 邮件发送系统架构分析

## 概述

Documenso 的邮件发送系统采用三层架构设计：

1. **邮件 Provider 适配层** - 支持多种邮件发送服务
2. **模板渲染层** - React Email 组件化模板 + 国际化
3. **变量替换层** - 自定义消息模板变量注入

---

## 第一层：邮件 Provider 适配层

### 核心文件

- `packages/email/mailer.ts`
- `packages/email/transports/mailchannels.ts`

### 设计思想

基于 **Nodemailer** 框架，通过工厂模式根据环境变量动态选择邮件传输器（Transport）。

### 支持的 Provider 类型

#### 1. MailChannels
```typescript
if (transport === 'mailchannels') {
  return createTransport(
    MailChannelsTransport.makeTransport({
      apiKey: env('NEXT_PRIVATE_MAILCHANNELS_API_KEY'),
      endpoint: env('NEXT_PRIVATE_MAILCHANNELS_ENDPOINT'),
    }),
  );
}
```
**所需环境变量：**
- `NEXT_PRIVATE_MAILCHANNELS_API_KEY`
- `NEXT_PRIVATE_MAILCHANNELS_ENDPOINT`（可选，默认 `https://api.mailchannels.net/tx/v1/send`）
- `NEXT_PRIVATE_MAILCHANNELS_DKIM_DOMAIN`
- `NEXT_PRIVATE_MAILCHANNELS_DKIM_SELECTOR`
- `NEXT_PRIVATE_MAILCHANNELS_DKIM_PRIVATE_KEY`

#### 2. Resend
```typescript
if (transport === 'resend') {
  return createTransport(
    ResendTransport.makeTransport({
      apiKey: env('NEXT_PRIVATE_RESEND_API_KEY'),
    }),
  );
}
```
**所需环境变量：**
- `NEXT_PRIVATE_RESEND_API_KEY`

#### 3. SMTP API
```typescript
if (transport === 'smtp-api') {
  return createTransport({
    host: env('NEXT_PRIVATE_SMTP_HOST'),
    port: Number(env('NEXT_PRIVATE_SMTP_PORT')) || 587,
    secure: env('NEXT_PRIVATE_SMTP_SECURE') === 'true',
    auth: {
      user: env('NEXT_PRIVATE_SMTP_APIKEY_USER') ?? 'apikey',
      pass: env('NEXT_PRIVATE_SMTP_APIKEY') ?? '',
    },
  });
}
```
**所需环境变量：**
- `NEXT_PRIVATE_SMTP_HOST`
- `NEXT_PRIVATE_SMTP_APIKEY`
- `NEXT_PRIVATE_SMTP_APIKEY_USER`（默认 `apikey`）
- `NEXT_PRIVATE_SMTP_PORT`（默认 587）
- `NEXT_PRIVATE_SMTP_SECURE`

#### 4. SMTP Auth（默认）
```typescript
return createTransport({
  host: env('NEXT_PRIVATE_SMTP_HOST') ?? '127.0.0.1:2500',
  port: Number(env('NEXT_PRIVATE_SMTP_PORT')) || 587,
  secure: env('NEXT_PRIVATE_SMTP_SECURE') === 'true',
  ignoreTLS: env('NEXT_PRIVATE_SMTP_UNSAFE_IGNORE_TLS') === 'true',
  auth: env('NEXT_PRIVATE_SMTP_USERNAME')
    ? {
        user: env('NEXT_PRIVATE_SMTP_USERNAME'),
        pass: env('NEXT_PRIVATE_SMTP_PASSWORD') ?? '',
      }
    : undefined,
  ...(env('NEXT_PRIVATE_SMTP_SERVICE') ? { service: env('NEXT_PRIVATE_SMTP_SERVICE') } : {}),
});
```

### 自定义 Transport 实现示例

以 `MailChannelsTransport` 为例，实现 Nodemailer 的 `Transport` 接口：

```typescript
export class MailChannelsTransport implements Transport<SentMessageInfo> {
  public name = 'CloudflareMailTransport';
  public version = VERSION;

  public static makeTransport(options: Partial<MailChannelsTransportOptions>) {
    return new MailChannelsTransport(options);
  }

  public send(mail: MailMessage, callback: (_err: Error | null, _info: SentMessageInfo) => void) {
    // 1. 地址格式转换
    const mailTo = this.toMailChannelsAddresses(mail.data.to);
    
    // 2. 构建 API 请求
    fetch(this._options.endpoint, {
      method: 'POST',
      headers: requestHeaders,
      body: JSON.stringify({
        from: from,
        subject: mail.data.subject,
        content: [
          { type: 'text/plain', value: mail.data.text?.toString('utf-8') ?? '' },
          { type: 'text/html', value: mail.data.html?.toString('utf-8') ?? '' },
        ],
      }),
    })
    .then((res) => {
      // 3. 回调处理
      callback(null, {
        messageId: '',
        envelope: { from: mail.data.from, to: mail.data.to },
        accepted: mail.data.to,
        rejected: [],
        pending: [],
      });
    });
  }

  private toMailChannelsAddresses(address: NodeMailerAddress): Array<MailChannelsAddress> {
    // 地址格式适配
  }
}
```

**关键设计点：**
- 实现 `Transport` 接口的 `send` 方法
- 将 Nodemailer 的标准邮件格式转换为目标 API 的格式
- 使用回调方式通知发送结果

---

## 第二层：模板渲染层

### 核心文件

- `packages/email/render.tsx`
- `packages/lib/utils/render-email-with-i18n.tsx`
- `packages/email/templates/*.tsx`
- `packages/email/template-components/*.tsx`
- `packages/email/providers/branding.tsx`

### 技术栈

- **@react-email/render** - 核心渲染引擎
- **@lingui/react** - 国际化（i18n）
- **Tailwind CSS** - 样式系统
- **React Context** - 品牌信息注入

### 渲染流程

#### 1. 模板组件结构

**模板文件**（`templates/document-invite.tsx`）：
```typescript
export const DocumentInviteEmailTemplate = ({
  inviterName = 'Lucas Smith',
  documentName = 'Open Source Pledge.pdf',
  signDocumentLink = 'https://documenso.com',
  customBody,
  role,
  selfSigner = false,
  organisationType,
  teamName,
  includeSenderDetails,
}: DocumentInviteEmailTemplateProps) => {
  const { _ } = useLingui();
  const branding = useBranding();

  const action = _(RECIPIENT_ROLES_DESCRIPTION[role].actionVerb).toLowerCase();

  return (
    <Html>
      <Head />
      <Preview>{_(previewText)}</Preview>
      <Body>
        {branding.brandingEnabled && branding.brandingLogo ? (
          <Img src={branding.brandingLogo} alt="Branding Logo" />
        ) : (
          <Img src={getAssetUrl('/static/logo.png')} alt="Documenso Logo" />
        )}
        <TemplateDocumentInvite
          inviterName={inviterName}
          documentName={documentName}
          signDocumentLink={signDocumentLink}
          role={role}
          // ...
        />
      </Body>
    </Html>
  );
};
```

**模板组件**（`template-components/template-document-invite.tsx`）：
```typescript
export const TemplateDocumentInvite = ({
  inviterName,
  documentName,
  signDocumentLink,
  role,
  selfSigner,
  // ...
}: TemplateDocumentInviteProps) => {
  const { _ } = useLingui();
  const { actionVerb } = RECIPIENT_ROLES_DESCRIPTION[role];

  return (
    <>
      <Text className="mx-auto mb-0 max-w-[80%] text-center font-semibold text-lg text-primary">
        {match({ selfSigner, organisationType, includeSenderDetails, teamName })
          .with({ selfSigner: true }, () => (
            <Trans>
              Please {_(actionVerb).toLowerCase()} your document
              <br />"{documentName}"
            </Trans>
          ))
          .otherwise(() => (
            <Trans>
              {inviterName} has invited you to {_(actionVerb).toLowerCase()}
              <br />"{documentName}"
            </Trans>
          ))}
      </Text>
      <Button href={signDocumentLink}>
        {match(role)
          .with(RecipientRole.SIGNER, () => <Trans>View Document to sign</Trans>)
          .with(RecipientRole.VIEWER, () => <Trans>View Document</Trans>)
          .exhaustive()}
      </Button>
    </>
  );
};
```

#### 2. 渲染函数

**基础渲染**（`render.tsx`）：
```typescript
export const render = async (element: React.ReactNode, options?: RenderOptions) => {
  const { branding, ...otherOptions } = options ?? {};

  return ReactEmail.render(
    <BrandingProvider branding={branding}>
      <Tailwind
        config={{
          theme: {
            extend: {
              colors,
            },
          },
        }}
      >
        {element}
      </Tailwind>
    </BrandingProvider>,
    otherOptions,
  );
};

export const renderWithI18N = async (element: React.ReactNode, options?: RenderOptions) => {
  const { branding, i18n, ...otherOptions } = options ?? {};

  return ReactEmail.render(
    <I18nProvider i18n={i18n}>
      <BrandingProvider branding={branding}>
        <Tailwind config={{...}}>
          {element}
        </Tailwind>
      </BrandingProvider>
    </I18nProvider>,
    otherOptions,
  );
};
```

**带语言加载的渲染**（`render-email-with-i18n.tsx`）：
```typescript
export const renderEmailWithI18N = async (
  component: React.ReactNode,
  options?: RenderOptions & { lang?: SupportedLanguageCodes },
) => {
  const { lang: providedLang, ...otherOptions } = options ?? {};

  const lang = isValidLanguageCode(providedLang) ? providedLang : APP_I18N_OPTIONS.sourceLang;

  const i18n = await getI18nInstance(lang);
  i18n.activate(lang);

  return renderWithI18N(component, { i18n, ...otherOptions });
};
```

#### 3. Context Provider 注入

**品牌信息 Provider**（`providers/branding.tsx`）：
```typescript
type BrandingContextValue = {
  brandingEnabled: boolean;
  brandingUrl: string;
  brandingLogo: string;
  brandingCompanyDetails: string;
  brandingHidePoweredBy: boolean;
};

export const BrandingProvider = (props: { branding?: BrandingContextValue; children: React.ReactNode }) => {
  return (
    <BrandingContext.Provider value={props.branding ?? defaultBrandingContextValue}>
      {props.children}
    </BrandingContext.Provider>
  );
};

export const useBranding = () => {
  const ctx = useContext(BrandingContext);
  if (!ctx) throw new Error('Branding context not found');
  return ctx;
};
```

### 国际化机制

使用 `@lingui/react` 实现多语言支持：

1. **模板定义**：
   ```typescript
   import { msg } from '@lingui/core/macro';
   import { Trans } from '@lingui/react/macro';
   
   const previewText = msg`Please sign your document`;
   
   <Trans>
     {inviterName} has invited you to sign
     <br />"{documentName}"
   </Trans>
   ```

2. **渲染时激活**：
   ```typescript
   const i18n = await getI18nInstance(lang);
   i18n.activate(lang);
   ```

3. **组件内使用**：
   ```typescript
   const { _ } = useLingui();
   const action = _(RECIPIENT_ROLES_DESCRIPTION[role].actionVerb).toLowerCase();
   ```

---

## 第三层：变量替换层

### 核心文件

- `packages/lib/utils/render-custom-email-template.ts`
- `packages/lib/server-only/email/get-email-context.ts`

### 两类变量替换

#### 1. 自定义邮件内容变量替换

**核心函数**：
```typescript
export const renderCustomEmailTemplate = <T extends Record<string, string>>(
  template: string,
  variables: T
): string => {
  return template.replace(/\{(\S+)\}/g, (_, key) => {
    if (key in variables) {
      return variables[key];
    }
    return key;
  });
};
```

**语法**：`{variable_name}`

**支持的变量**：
```typescript
const customEmailTemplate = {
  'signer.name': name,        // 收件人姓名
  'signer.email': email,      // 收件人邮箱
  'document.name': envelope.title,  // 文档名称
};
```

**使用示例**（`send-signing-email.handler.ts`）：
```typescript
const customEmailTemplate = {
  'signer.name': name,
  'signer.email': email,
  'document.name': envelope.title,
};

// 替换邮件主题
subject: renderCustomEmailTemplate(
  documentMeta?.subject || emailSubject,
  customEmailTemplate
),

// 替换邮件正文
customBody: renderCustomEmailTemplate(emailMessage, customEmailTemplate),
```

**用户输入示例**：
- 用户在界面输入主题：`请 {signer.name} 签署 {document.name}`
- 实际发送时替换为：`请 张三 签署 合作协议.pdf`

#### 2. 邮件上下文注入

**邮件上下文获取**（`get-email-context.ts`）：
```typescript
type EmailContextResponse = {
  allowedEmails: OrganisationEmail[];        // 允许使用的发件邮箱
  branding: BrandingSettings;                // 品牌设置
  settings: Omit<OrganisationGlobalSettings, 'id'>;  // 组织全局设置
  claims: OrganisationClaim;                 // 组织权限声明
  organisationType: OrganisationType;        // 组织类型
  senderEmail: { name: string; address: string };  // 发件人邮箱
  replyToEmail: string | undefined;          // 回复邮箱
  emailLanguage: string;                     // 邮件语言
};
```

**上下文获取逻辑**：
```typescript
export const getEmailContext = async (options: GetEmailContextOptions) => {
  const { source, meta } = options;

  let emailContext;
  if (source.type === 'organisation') {
    emailContext = await handleOrganisationEmailContext(source.organisationId);
  } else {
    emailContext = await handleTeamEmailContext(source.teamId);
  }

  const emailLanguage = meta?.language || emailContext.settings.documentLanguage;

  // 内部邮件使用 Documenso 官方邮箱
  if (options.emailType === 'INTERNAL') {
    return {
      ...emailContext,
      senderEmail: DOCUMENSO_INTERNAL_EMAIL,
      replyToEmail: undefined,
      emailLanguage,
    };
  }

  // 收件人邮件：根据组织设置选择自定义发件邮箱
  const replyToEmail = meta?.emailReplyTo || emailContext.settings.emailReplyTo || undefined;
  
  const senderEmailId = match(meta?.emailId)
    .with(P.string, (emailId) => emailId)           // 显式指定邮箱ID
    .with(undefined, () => emailContext.settings.emailId)  // 使用继承的设置
    .with(null, () => null)                        // 显式使用 Documenso 邮箱
    .exhaustive();

  const foundSenderEmail = emailContext.allowedEmails.find((email) => email.id === senderEmailId);

  const senderEmail = foundSenderEmail
    ? { name: foundSenderEmail.emailName, address: foundSenderEmail.email }
    : DOCUMENSO_INTERNAL_EMAIL;

  return { ...emailContext, senderEmail, replyToEmail, emailLanguage };
};
```

---

## 完整调用链示例

以 `send-signing-email.handler.ts` 为例，展示从数据准备到发送的完整流程：

### 步骤 1：获取邮件上下文
```typescript
const { branding, emailLanguage, settings, organisationType, senderEmail, replyToEmail } = 
  await getEmailContext({
    emailType: 'RECIPIENT',
    source: { type: 'team', teamId: envelope.teamId },
    meta: envelope.documentMeta,
  });
```

### 步骤 2：准备自定义变量
```typescript
const customEmailTemplate = {
  'signer.name': name,
  'signer.email': email,
  'document.name': envelope.title,
};
```

### 步骤 3：创建 React 元素并注入 props
```typescript
const template = createElement(DocumentInviteEmailTemplate, {
  documentName: envelope.title,
  inviterName: user.name || undefined,
  signDocumentLink: `${NEXT_PUBLIC_WEBAPP_URL()}/sign/${recipient.token}`,
  customBody: renderCustomEmailTemplate(emailMessage, customEmailTemplate),
  role: recipient.role,
  selfSigner,
  organisationType,
  teamName: team?.name,
  includeSenderDetails: settings.includeSenderDetails,
});
```

### 步骤 4：渲染 HTML 和纯文本版本
```typescript
const [html, text] = await Promise.all([
  renderEmailWithI18N(template, { lang: emailLanguage, branding }),
  renderEmailWithI18N(template, {
    lang: emailLanguage,
    branding,
    plainText: true,
  }),
]);
```

### 步骤 5：通过 mailer 发送
```typescript
await mailer.sendMail({
  to: { name: recipient.name, address: recipient.email },
  from: senderEmail,
  replyTo: replyToEmail,
  subject: renderCustomEmailTemplate(
    documentMeta?.subject || emailSubject,
    customEmailTemplate
  ),
  html,
  text,
});
```

---

## 架构总结

```
┌─────────────────────────────────────────────────────────────┐
│                    业务调用层 (Handler)                      │
│  - send-signing-email.handler.ts                            │
│  - send-completed-email.ts                                  │
│  - 准备数据、构建模板 props、调用发送                         │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                第三层：变量替换层                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  renderCustomEmailTemplate()                        │   │
│  │  - 语法: {variable_name}                            │   │
│  │  - 变量: signer.name, signer.email, document.name   │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  getEmailContext()                                  │   │
│  │  - 发件人邮箱 (senderEmail)                         │   │
│  │  - 品牌设置 (branding)                              │   │
│  │  - 语言设置 (emailLanguage)                         │   │
│  └─────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                第二层：模板渲染层                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  React Email 模板系统                               │   │
│  │  - templates/document-invite.tsx (容器)            │   │
│  │  - template-components/template-document-invite.tsx│   │
│  │  - 组件化设计、可复用                               │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  国际化 (i18n)                                      │   │
│  │  - @lingui/react + Trans/msg macro                 │   │
│  │  - getI18nInstance(lang) + i18n.activate(lang)     │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  渲染函数                                           │   │
│  │  - renderWithI18N()                                │   │
│  │  - BrandingProvider + I18nProvider + Tailwind      │   │
│  └─────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│              第一层：邮件 Provider 适配层                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  mailer.ts                                          │   │
│  │  - getTransport() 工厂函数                          │   │
│  │  - 根据 NEXT_PRIVATE_SMTP_TRANSPORT 选择            │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  传输器 (Transports)                                │   │
│  │  - MailChannelsTransport (自定义)                   │   │
│  │  - ResendTransport (npm 包)                         │   │
│  │  - SMTP (Nodemailer 内置)                          │   │
│  │  - 实现 Transport 接口，统一 API                    │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 关键设计模式

| 层级 | 设计模式 | 实现位置 |
|------|---------|---------|
| Provider 适配层 | 工厂模式 + 策略模式 | `mailer.ts` + `transports/` |
| 模板渲染层 | 组件化 + Provider 注入 | `templates/` + `render.tsx` |
| 变量替换层 | 模板方法 + 字符串替换 | `render-custom-email-template.ts` |

### 扩展性设计

1. **新增 Provider**：实现 Nodemailer `Transport` 接口，在 `mailer.ts` 的 `getTransport()` 中添加分支
2. **新增模板**：在 `templates/` 中创建 React 组件，在 `template-components/` 中抽离可复用部分
3. **新增变量**：在 `renderCustomEmailTemplate()` 的调用处扩展 `customEmailTemplate` 对象
