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

### 支持的 Provider 类型及配置说明

#### 1. MailChannels

**代码实现**：
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

**DKIM 参数实际使用**（`transports/mailchannels.ts:81-83`）：
```typescript
dkim_domain: env('NEXT_PRIVATE_MAILCHANNELS_DKIM_DOMAIN') || undefined,
dkim_selector: env('NEXT_PRIVATE_MAILCHANNELS_DKIM_SELECTOR') || undefined,
dkim_private_key: env('NEXT_PRIVATE_MAILCHANNELS_DKIM_PRIVATE_KEY') || undefined,
```

**配置说明（基于代码实现）**：

| 环境变量 | 必填/可选 | 依据 | 默认值 |
|---------|----------|------|--------|
| `NEXT_PRIVATE_MAILCHANNELS_API_KEY` | 可选 | 仅在请求头中设置，无为空检查 | 无 |
| `NEXT_PRIVATE_MAILCHANNELS_ENDPOINT` | 可选 | 构造函数中有默认值 | `https://api.mailchannels.net/tx/v1/send` |
| `NEXT_PRIVATE_MAILCHANNELS_DKIM_DOMAIN` | 可选 | 使用 `|| undefined`，空值时不传 | 无 |
| `NEXT_PRIVATE_MAILCHANNELS_DKIM_SELECTOR` | 可选 | 使用 `|| undefined`，空值时不传 | 无 |
| `NEXT_PRIVATE_MAILCHANNELS_DKIM_PRIVATE_KEY` | 可选 | 使用 `|| undefined`，空值时不传 | 无 |

**语义解释**：
- `apiKey`：用于自定义 Worker 代理的认证，不是 MailChannels API 本身强制要求
- `endpoint`：支持通过 Cloudflare Worker 代理转发 MailChannels 请求
- DKIM 参数：**全部为可选**，使用 `|| undefined` 模式，意味着环境变量为空或未设置时，JSON 中不包含这些字段
- MailChannels API 允许不携带 DKIM 配置，此时由 MailChannels 自行处理邮件签名

---

#### 2. Resend

**代码实现**：
```typescript
if (transport === 'resend') {
  if (!env('NEXT_PRIVATE_RESEND_API_KEY')) {
    throw new Error('Resend transport requires NEXT_PRIVATE_RESEND_API_KEY');
  }

  return createTransport(
    ResendTransport.makeTransport({
      apiKey: env('NEXT_PRIVATE_RESEND_API_KEY'),
    }),
  );
}
```

**配置说明（基于代码实现）**：

| 环境变量 | 必填/可选 | 依据 |
|---------|----------|------|
| `NEXT_PRIVATE_RESEND_API_KEY` | **必填** | 有显式 `if (!env(...)) throw new Error(...)` 检查 |

**语义解释**：
- Resend Provider 强制要求 API Key，缺失时在初始化阶段直接抛出异常
- 属于"启动时校验"模式，避免运行时才发现配置缺失

---

#### 3. SMTP API

**代码实现**：
```typescript
if (transport === 'smtp-api') {
  if (!env('NEXT_PRIVATE_SMTP_HOST') || !env('NEXT_PRIVATE_SMTP_APIKEY')) {
    throw new Error('SMTP API transport requires NEXT_PRIVATE_SMTP_HOST and NEXT_PRIVATE_SMTP_APIKEY');
  }

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

**配置说明（基于代码实现）**：

| 环境变量 | 必填/可选 | 依据 | 默认值 |
|---------|----------|------|--------|
| `NEXT_PRIVATE_SMTP_HOST` | **必填** | 有显式 `if (!env(...)) throw new Error(...)` 检查 | 无 |
| `NEXT_PRIVATE_SMTP_APIKEY` | **必填** | 同上 | 无 |
| `NEXT_PRIVATE_SMTP_APIKEY_USER` | 可选 | 使用 `??` 提供默认值 | `apikey` |
| `NEXT_PRIVATE_SMTP_PORT` | 可选 | 使用 `\|\|` 提供默认值 | 587 |
| `NEXT_PRIVATE_SMTP_SECURE` | 可选 | 直接比较字符串，默认 false | false |

**语义解释**：
- 核心配置（Host + API Key）强制校验
- 认证用户名通常为 `apikey`（如 SendGrid 等服务）
- `secure: true` 表示使用 SSL/TLS 加密连接

---

#### 4. SMTP Auth（默认）

**代码实现**：
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

**配置说明（基于代码实现）**：

| 环境变量 | 必填/可选 | 依据 | 默认值 |
|---------|----------|------|--------|
| `NEXT_PRIVATE_SMTP_HOST` | 可选 | 使用 `??` 提供默认值 | `127.0.0.1:2500` |
| `NEXT_PRIVATE_SMTP_PORT` | 可选 | 使用 `\|\|` 提供默认值 | 587 |
| `NEXT_PRIVATE_SMTP_SECURE` | 可选 | 直接比较字符串 | false |
| `NEXT_PRIVATE_SMTP_UNSAFE_IGNORE_TLS` | 可选 | 直接比较字符串 | false |
| `NEXT_PRIVATE_SMTP_USERNAME` | 可选 | 存在时才创建 `auth` 对象 | 无 |
| `NEXT_PRIVATE_SMTP_PASSWORD` | 可选（依赖） | 仅当 `USERNAME` 存在时使用 `?? ''` | 空字符串 |
| `NEXT_PRIVATE_SMTP_SERVICE` | 可选 | 使用条件展开 `...(foo ? {foo} : {})` | 无 |

**语义解释**：
- 默认适配本地开发环境（`127.0.0.1:2500`），通常配合 MailHog 等工具
- **认证是可选的**：不设置 `USERNAME` 时，`auth` 为 `undefined`，允许连接不需要认证的 SMTP 服务器
- `ignoreTLS`："不安全"选项，忽略 TLS 证书校验（仅用于特殊网络环境）
- `service`：Nodemailer 内置服务快捷配置（如 "gmail"、"outlook" 等）

---

### Provider 必填校验策略总结

| Provider | 校验时机 | 校验方式 | 失败处理 |
|---------|---------|---------|---------|
| MailChannels | 无强制校验 | 运行时按需传入 | 依赖服务端返回错误 |
| Resend | 初始化时 | 显式 `if (!env) throw` | 立即抛出异常 |
| SMTP API | 初始化时 | 显式 `if (!env) throw` | 立即抛出异常 |
| SMTP Auth（默认） | 无强制校验 | 提供默认值 / 可选 | 静默回退到默认配置 |

---

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

**核心函数**（`render-custom-email-template.ts`）：
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

**正则解析**：`/\{(\S+)\}/g`
- 匹配字面量 `{`
- 捕获组：一个或多个非空白字符（`\S+`）
- 匹配字面量 `}`
- 全局匹配（`g`）

**变量查找逻辑**：
```typescript
if (key in variables) {
  return variables[key];    // 命中：返回变量值
}
return key;                  // 未命中：返回原始 key 字符串（不含花括号）
```

---

**未提供变量时的回退行为（实际输出语义）**：

| 场景 | 模板字符串 | 变量对象 | 实际输出 | 解释 |
|------|-----------|---------|---------|------|
| 变量存在 | `"Hello {name}"` | `{name: "Alice"}` | `"Hello Alice"` | 正常替换 |
| 变量不存在 | `"Hello {name}"` | `{other: "value"}` | `"Hello name"` | **保留 key 名称，去掉花括号** |
| 变量值为空字符串 | `"Hello {name}"` | `{name: ""}` | `"Hello "` | 替换为空字符串 |
| 多个变量混合 | `"{a} and {b}"` | `{a: "X"}` | `"X and b"` | `{a}` 替换，`{b}` 回退为 `b` |
| 无变量占位符 | `"Hello World"` | `{name: "Alice"}` | `"Hello World"` | 原样输出 |
| 双花括号 | `"Hello {{name}}"` | `{name: "Alice"}` | `"Hello {name}"` | 贪婪匹配导致 `{{name}}` 整体被匹配，捕获组为 `{name}` |
| 含空白字符 | `"Hello {name age}"` | `{name: "Alice"}` | `"Hello {name age}"` | `\S+` 不匹配空白，整体不被识别为占位符 |
| 未定义变量 key | `"Hello {undefined}"` | `{name: "Alice"}` | `"Hello undefined"` | 匹配到 `{undefined}`，但 key 不存在，回退为 `undefined` |

---

### 占位符语法边界说明

当前正则：`/\{(\S+)\}/g`

**语法拆解**：
| 符号 | 含义 |
|------|------|
| `\{` | 匹配字面量左花括号 |
| `(\S+)` | 捕获组：一个或多个**非空白字符** |
| `\}` | 匹配字面量右花括号 |
| `g` | 全局匹配 |

---

#### 边界场景 1：双花括号 `{{name}}`

**匹配过程**：
```
输入: Hello {{name}}
       ↑ ↑      ↑ ↑
       │ │      │ │
       │ └──────┘ │
       │    │     │
       │  \S+    │
       │         │
       \{       \}

实际匹配: {{name}}      ← 贪婪匹配从第一个 { 到最后一个 }
捕获组 1: {name}        ← 注意：key 变成了 "{name}"
```

**变量查找**：
```typescript
// 捕获组 key = "{name}"
if ('{name}' in {name: 'Alice'})  // false！因为 key 是 '{name}' 不是 'name'
  return variables['{name}'];
else
  return '{name}';                 // 输出: {name}
```

**结果**：
- 输入：`Hello {{name}}`
- 变量：`{name: 'Alice'}`
- 输出：`Hello {name}`

**注意**：如果变量对象恰好有一个 key 叫 `'{name}'`（含花括号），则会被替换：
- 变量：`{'{name}': 'Bob'}`
- 输出：`Hello Bob`

---

#### 边界场景 2：包含空白字符的占位符 `{name age}`

**匹配过程**：
```
输入: Hello {name age}
           ↑  ↑
           │  │
           │  空格（\s）不被 \S+ 匹配
           │
           \{ 匹配，但 \S+ 到空格停止，后面没有 \} 闭合

实际匹配: 无匹配（因为 {name 后面是空格，找不到 }）
```

**`\S` 定义**：匹配任何**非空白字符**，等价于 `[^\r\n\t\f\v ]`

**不被识别为占位符的情况**：
| 输入 | 原因 | 是否匹配 |
|------|------|---------|
| `{name age}` | 中间有空格 | ❌ 不匹配 |
| `{name\nage}` | 中间有换行 | ❌ 不匹配 |
| `{name\tag}` | 中间有制表符 | ❌ 不匹配 |
| `{ name}` | 左花括号后有空格 | ❌ 不匹配 |
| `{name }` | 右花括号前有空格 | ❌ 不匹配 |
| `{}` | 空的花括号（`\S+` 要求至少一个字符） | ❌ 不匹配 |
| `{signer.name}` | 点号属于非空白字符 | ✅ 匹配（key = `signer.name`） |
| `{document_name}` | 下划线属于非空白字符 | ✅ 匹配（key = `document_name`） |

**不匹配的后果**：字符串原样输出，不进入替换逻辑。

---

#### 边界场景 3：未定义变量 key `{undefined}`

**匹配过程**：
```
输入: Hello {undefined}
       ↑     ↑        ↑
       │     │        │
       │  捕获组    │
       │            │
       \{          \}

实际匹配: {undefined}
捕获组 1: undefined
```

**变量查找与回退**：
```typescript
// 变量对象: {name: 'Alice'}
// key = 'undefined'

if ('undefined' in {name: 'Alice'})  // false
  return variables['undefined'];
else
  return 'undefined';               // 注意：不带花括号
```

**结果**：
- 输入：`Hello {undefined}`
- 变量：`{name: 'Alice'}`
- 输出：`Hello undefined`（花括号被去除）

**区分两种 "未定义"**：
| 场景 | 输入 | 变量 | 输出 | 类型 |
|------|------|------|------|------|
| 占位符不匹配 | `{name age}` | 任意 | `{name age}` | 语法层面不识别 |
| key 不存在 | `{undefined}` | `{name: 'Alice'}` | `undefined` | 语义层面回退 |

---

#### 边界场景总结表

| 输入 | 变量对象 | 输出 | 原因 |
|------|---------|------|------|
| `{name}` | `{name: 'Alice'}` | `Alice` | 正常替换 |
| `{name}` | `{other: 'X'}` | `name` | key 不存在，回退为 key |
| `{{name}}` | `{name: 'Alice'}` | `{name}` | 贪婪匹配，key 变成 `{name}` |
| `{{name}}` | `{'{name}': 'Bob'}` | `Bob` | 恰好有 key 叫 `{name}` |
| `{name age}` | `{name: 'Alice'}` | `{name age}` | 含空格，正则不匹配 |
| `{name}` | `{name: ''}` | ` `（空字符串） | 替换为空 |
| `{undefined}` | `{name: 'Alice'}` | `undefined` | key 不存在，回退为 key（去花括号） |
| `{}` | `{name: 'Alice'}` | `{}` | `\S+` 要求至少一个字符，不匹配 |
| `{a}{b}` | `{a: 'X'}` | `Xb` | 两个独立匹配，`{a}` 替换，`{b}` 回退 |

---

**设计决策解读**：
- 采用"**静默回退**"策略，不抛异常、不报错
- 未命中时返回 `key`（变量名）而非原始 `{key}`，便于识别哪些变量未被替换
- `\S+` 的选择意味着**不支持**带空格的变量名，这是有意的约束（变量名通常不包含空格）
- 双花括号 `{{name}}` 的行为是**贪婪匹配**的副作用，非刻意设计
- 属于"尽力而为"（best-effort）的模板渲染模式

---

**支持的变量定义**（以 `send-signing-email.handler.ts:139-143` 为例）：
```typescript
const customEmailTemplate = {
  'signer.name': name,        // 收件人姓名
  'signer.email': email,      // 收件人邮箱
  'document.name': envelope.title,  // 文档名称
};
```

**使用示例**：
```typescript
// 替换邮件主题
subject: renderCustomEmailTemplate(
  documentMeta?.subject || emailSubject,
  customEmailTemplate
),

// 替换邮件正文
customBody: renderCustomEmailTemplate(emailMessage, customEmailTemplate),
```

**用户输入示例**：
- 用户输入主题：`请 {signer.name} 签署 {document.name}`
- 变量对象：`{'signer.name': '张三', 'document.name': '合作协议.pdf'}`
- 实际发送：`请 张三 签署 合作协议.pdf`

**未定义变量示例**：
- 用户输入主题：`请 {signer.name} 查看 {invalid.var}`
- 变量对象：`{'signer.name': '张三'}`
- 实际发送：`请 张三 查看 invalid.var`（注意：`{invalid.var}` 变成了 `invalid.var`，花括号被去除）

---

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

## 关键逻辑落点调用链

### 调用链 1：从发送入口到 MailChannels DKIM 参数处理

```
send-signing-email.handler.ts:175-185
  │
  └─► mailer.sendMail({ from, to, subject, html, text, ... })
        │
        └─► mailer.ts:108
              │
              └─► const mailer = getTransport()
                    │
                    └─► mailer.ts:53: getTransport()
                          │
                          ├─► 判断 transport === 'mailchannels'
                          │     └─► MailChannelsTransport.makeTransport({ apiKey, endpoint })
                          │           │
                          │           └─► transports/mailchannels.ts:34-36
                          │                 │
                          │                 └─► 构造函数接收 Partial 配置，endpoint 有默认值
                          │
                          └─► 实际发送时：transports/mailchannels.ts:47
                                │
                                └─► send(mail, callback)
                                      │
                                      └─► transports/mailchannels.ts:73-96
                                            │
                                            ├─► body 构建
                                            │     │
                                            │     └─► personalizations[0]:
                                            │           │
                                            │           ├─► dkim_domain: env(...) || undefined
                                            │           ├─► dkim_selector: env(...) || undefined
                                            │           └─► dkim_private_key: env(...) || undefined
                                            │
                                            └─► fetch(endpoint, { method: 'POST', body, headers })
```

**落点说明**：
- **DKIM 参数最终处理位置**：`transports/mailchannels.ts:81-83`
- **判断依据**：使用 `|| undefined`，空值 / falsy 值时为 `undefined`，JSON 序列化时自动忽略该字段
- **无前置校验**：不在 `mailer.ts` 初始化阶段校验 DKIM 配置

---

### 调用链 2：从发送入口到自定义变量替换回退逻辑

```
send-signing-email.handler.ts:139-156
  │
  ├─► 准备变量：
  │     const customEmailTemplate = {
  │       'signer.name': name,
  │       'signer.email': email,
  │       'document.name': envelope.title,
  │     };
  │
  ├─► 替换邮件主题（第 182 行）：
  │     subject: renderCustomEmailTemplate(
  │       documentMeta?.subject || emailSubject,
  │       customEmailTemplate
  │     ),
  │     │
  │     └─► render-custom-email-template.ts:1-9
  │           │
  │           ├─► regex: /\{(\S+)\}/g
  │           │
  │           └─► replace callback:
  │                 if (key in variables) → return variables[key]
  │                 else → return key   ←── 回退逻辑落点
  │
  └─► 替换邮件正文（第 155 行）：
        customBody: renderCustomEmailTemplate(emailMessage, customEmailTemplate),
        │
        └─► 同上：render-custom-email-template.ts:1-9
              │
              └─► 同样的回退逻辑：未命中变量时返回 key 名称（无花括号）
```

**落点说明**：
- **变量替换回退逻辑位置**：`render-custom-email-template.ts:7` → `return key`
- **行为特征**：
  - 输入：`{undefined_var}`
  - 正则捕获组：`undefined_var`
  - `key in variables` → false
  - 返回：`undefined_var`（**去掉了花括号**）
- **设计意图**：便于在输出中识别"哪些变量未被替换"，同时避免原始花括号在邮件正文中显得突兀

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
│  │  - 命中: 返回变量值                                  │   │
│  │  - 未命中: 返回 key（去掉花括号）                     │   │
│  │  - 落点: render-custom-email-template.ts:7          │   │
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
│  │  - Resend / SMTP API: 启动时强制校验                │   │
│  │  - MailChannels / SMTP Auth: 无强制校验 + 默认值    │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  MailChannelsTransport 落点                         │   │
│  │  - DKIM 参数: transports/mailchannels.ts:81-83     │   │
│  │  - 模式: env(...) || undefined → 可选               │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  传输器 (Transports)                                │   │
│  │  - MailChannelsTransport (自定义)                   │   │
│  │  - ResendTransport (npm 包)                         │   │
│  │  - SMTP (Nodemailer 内置)                          │   │
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

---

## 修订记录

### 本次修订要点（2026-05-11，第 2 次修订）

5. **双花括号示例修正**：
   - 原表述：`Hello {{name}}` → `Hello {Alice}`
   - 修正后：`Hello {{name}}` → `Hello {name}`
   - 原因：贪婪匹配 `{{name}}` 整体，捕获组为 `{name}`，与 `{name: 'Alice'}` 不匹配，回退为 `{name}`

6. **占位符语法边界说明**：
   - 新增章节：占位符语法边界说明
   - 涵盖三类边界：双花括号、含空白字符、未定义变量 key
   - 详细解释正则 `/\{(\S+)\}/g` 的实际匹配行为

---

### 历史修订要点（第 1 次修订）

1. **MailChannels DKIM 参数修正**：
   - 原表述：错误标记为必填
   - 修正后：明确标注为可选，依据 `|| undefined` 模式
   - 落点：`transports/mailchannels.ts:81-83`

2. **自定义变量回退行为明确**：
   - 原表述：未说明未命中时的行为
   - 修正后：详细说明 `{undefined_var}` → `undefined_var`（去掉花括号）
   - 落点：`render-custom-email-template.ts:7`

3. **Provider 必填/可选依据表格**：
   - 新增 4 个 Provider 的完整配置表格
   - 标注"依据"列：显式 throw / `??` 默认值 / `\|\| undefined` 等

4. **关键逻辑落点调用链**：
   - 新增从 `send-signing-email.handler.ts` 到两处核心逻辑的精简调用链
   - 标注关键代码行号
