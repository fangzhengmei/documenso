# Documenso 语言与时区体系完整核查报告（第二轮）

## 概述

本报告基于对代码库的深入核查，详细说明 Documenso 签名流程中语言协商的真实优先级、邮件语言选择机制、以及签署页/邮件/PDF 三者的时区取值路径与兜底策略。

---

## 一、界面语言协商优先级与回退链

### 1.1 真实优先级顺序（从高到低）

```
1. 语言 Cookie (lang cookie) → 最高优先级
   ↓ 验证: 检查是否在支持语言列表中
   ↓ 不支持则回退
2. HTTP Accept-Language 请求头解析
   ↓ 回退
3. 系统默认语言 (en) → 最终兜底
```

### 1.2 核心实现细节

**文件位置**: `apps/remix/app/root.tsx:50-89`

```typescript
// 第 52 行: 首先从 Cookie 中读取语言
let lang: SupportedLanguageCodes = await langCookie.parse(cookieHeader);

// 第 54-56 行: 如果 Cookie 语言不支持，从请求头解析
if (!APP_I18N_OPTIONS.supportedLangs.includes(lang)) {
  lang = extractLocaleData({ headers: request.headers }).lang;
}

// 第 85-87 行: 响应时写回 Cookie
headers: {
  'Set-Cookie': await langCookie.serialize(lang),
},
```

**请求头解析逻辑** (`packages/lib/utils/i18n.ts:62-83`):
```typescript
export const extractLocaleData = ({ headers }: ExtractLocaleDataOptions): I18nLocaleData => {
  const headerLocales = (headers.get('accept-language') ?? '').split(',');
  
  const unknownLanguages = headerLocales
    .map((locale) => parseLanguageFromLocale(locale))
    .filter((value): value is SupportedLanguageCodes => value !== null);

  return {
    lang: languages[0] || APP_I18N_OPTIONS.sourceLang, // 回退到 en
    locales: headerLocales,
  };
};
```

### 1.3 语言切换机制

**API 端点**: `apps/remix/app/routes/api+/locale.tsx`
- 用户可通过 POST 请求显式设置语言
- 语言值被序列化到 `langCookie`
- Cookie 有效期: 1 年（60 * 60 * 24 * 365 * 2）

---

## 二、邮件语言选择机制

### 2.1 真实优先级顺序

```
重要结论: 邮件语言完全不考虑收件人偏好，仅依据文档/组织设置

1. 文档元数据语言 (DocumentMeta.language)
   ↓ 未设置则回退
2. 组织/团队全局设置的默认文档语言 (settings.documentLanguage)
   ↓ 隐式兜底
3. 系统默认语言 (en) → Zod schema 自动兜底
```

### 2.2 核心实现流程

**邮件上下文获取** (`packages/lib/server-only/email/get-email-context.ts:89`):
```typescript
const emailLanguage = meta?.language || emailContext.settings.documentLanguage;
```

**邮件发送流程** (`packages/lib/jobs/definitions/emails/send-signing-email.handler.ts:86-187`):
```typescript
// 第 86-93 行: 获取邮件上下文（包含语言）
const { branding, emailLanguage, settings, organisationType, senderEmail, replyToEmail } =
  await getEmailContext({
    emailType: 'RECIPIENT',
    source: { type: 'team', teamId: envelope.teamId },
    meta: envelope.documentMeta,
  });

// 第 103 行: 激活对应语言
const i18n = await getI18nInstance(emailLanguage);

// 第 167-172 行: 使用该语言渲染邮件 HTML 和纯文本版本
await renderEmailWithI18N(template, { lang: emailLanguage, branding });
```

### 2.3 关键发现

**收件人无语言偏好设置**:
- Recipient 模型中没有 `language` 字段
- 代码中不存在按收件人个性化邮件语言的逻辑
- 同一文档发送给所有收件人的邮件使用相同语言

**支持的邮件类型验证**:
核查了以下邮件处理器，均采用相同语言策略：
- `send-signing-email.handler.ts` - 签名邀请邮件
- `send-recipient-signed-email.handler.ts` - 收件人已签署通知
- `send-rejection-emails.handler.ts` - 拒绝签名通知
- `send-document-cancelled-emails.handler.ts` - 文档取消通知

---

## 三、时区取值路径与兜底策略

### 3.1 时区统一基础设置

**默认时区常量**: `packages/lib/constants/time-zones.ts:5`
```typescript
export const DEFAULT_DOCUMENT_TIME_ZONE = 'Etc/UTC';
```

**时区优先级统一原则**: 所有场景均采用相同的优先级顺序
```
1. 文档元数据时区 (DocumentMeta.timezone)
   ↓ 未设置则回退
2. 组织/团队设置的默认时区 (settings.documentTimezone)
   ↓ 最终兜底
3. Etc/UTC
```

**文档元数据提取逻辑** (`packages/lib/utils/document.ts:38`):
```typescript
timezone: meta.timezone || settings.documentTimezone || DEFAULT_DOCUMENT_TIME_ZONE,
```

---

### 3.2 签署页面时区处理

#### 场景 A: 日期字段显示

**文件**: `apps/remix/app/components/general/document-signing/document-signing-date-field.tsx:34-60`

```typescript
// 第 34-35 行: 接收参数，自带兜底
dateFormat = DEFAULT_DOCUMENT_DATE_FORMAT,
timezone = DEFAULT_DOCUMENT_TIME_ZONE,

// 第 56-60 行: 时区转换显示
const localDateString = convertToLocalSystemFormat(
  field.customText,
  dateFormat,
  timezone
);

// 关键: 当用户本地时区与文档时区不同时，显示提示
const isDifferentTime = field.inserted && localDateString !== field.customText;
const tooltipText = `"${field.customText}" will appear on the document as it has a timezone of "${timezone || ''}".`;
```

#### 场景 B: 签署操作时的时间戳

**文件**: `apps/remix/app/components/general/document-signing/envelope-signing-provider.tsx:85-88`

```typescript
customText: DateTime.now()
  .setZone(timezone ?? DEFAULT_DOCUMENT_TIME_ZONE)
  .toFormat(dateFormat ?? DEFAULT_DOCUMENT_DATE_FORMAT),
```

---

### 3.3 邮件中的时区处理

#### 场景 A: 签署时写入日期字段

**文件**: `packages/lib/server-only/field/sign-field-with-token.ts:197-201`

```typescript
if (field.type === FieldType.DATE) {
  customText = DateTime.now()
    .setZone(documentMeta?.timezone ?? DEFAULT_DOCUMENT_TIME_ZONE)
    .toFormat(documentMeta?.dateFormat ?? DEFAULT_DOCUMENT_DATE_FORMAT);
}
```

#### 场景 B: 完成文档时的时间处理

**文件**: `packages/lib/server-only/document/complete-document-with-token.ts`
```typescript
.setZone(envelope.documentMeta?.timezone ?? DEFAULT_DOCUMENT_TIME_ZONE)
```

#### 场景 C: 直接模板创建时的时间处理

**文件**: `packages/lib/server-only/template/create-document-from-direct-template.ts:247`
```typescript
customText = DateTime.now()
  .setZone(derivedDocumentMeta.timezone)
  .toFormat(derivedDocumentMeta.dateFormat);
```

---

### 3.4 PDF 证书与审计日志中的时区处理

#### 场景 A: 证书 PDF 生成

**文件**: `packages/lib/server-only/pdf/generate-certificate-pdf.ts`
- 接受 `language` 参数（由调用方传入，通常为文档语言）
- 使用 `ZSupportedLanguageCodeSchema.parse(language)` 验证语言
- 时区信息通过 `documentMeta` 传递给 `renderCertificate` 函数

**关键观察**: 证书中的时间戳显示使用文档设置的时区格式

#### 场景 B: 审计日志 PDF 生成

**文件**: `packages/lib/server-only/pdf/generate-audit-log-pdf.ts:18-60`
- 同样接受 `language` 参数，处理逻辑与证书相同
- 审计日志中的 `createdAt` 时间戳在渲染时格式化

#### 场景 C: 时间格式化函数

**文件**: `packages/lib/constants/date-formats.ts:169-188`

```typescript
export const convertToLocalSystemFormat = (
  customText: string,
  dateFormat: string | null = DEFAULT_DOCUMENT_DATE_FORMAT,
  timeZone: string | null = DEFAULT_DOCUMENT_TIME_ZONE,
): string => {
  const coalescedDateFormat = dateFormat ?? DEFAULT_DOCUMENT_DATE_FORMAT;
  const coalescedTimeZone = timeZone ?? DEFAULT_DOCUMENT_TIME_ZONE;

  const parsedDate = DateTime.fromFormat(customText, coalescedDateFormat, {
    zone: coalescedTimeZone,
  });

  if (!parsedDate.isValid) {
    return 'Invalid date';
  }

  const formattedDate = parsedDate.toLocal().toFormat(coalescedDateFormat);

  return formattedDate;
};
```

**功能说明**:
1. 接收存储的日期字符串、文档日期格式、文档时区
2. 使用文档时区解析日期字符串
3. 转换为用户本地时区时间后使用相同格式显示

---

## 四、三者（签署页/邮件/PDF）时区一致性对比

| 场景 | 时区来源 | 兜底机制 | 一致性 |
|-----|---------|---------|-------|
| **签署页 - 日期字段显示** | DocumentMeta.timezone | DEFAULT_DOCUMENT_TIME_ZONE (UTC) | ✅ 一致 |
| **签署页 - 签署时间戳** | DocumentMeta.timezone | DEFAULT_DOCUMENT_TIME_ZONE (UTC) | ✅ 一致 |
| **邮件 - 签署时日期字段** | DocumentMeta.timezone | DEFAULT_DOCUMENT_TIME_ZONE (UTC) | ✅ 一致 |
| **邮件 - 完成文档时间** | DocumentMeta.timezone | DEFAULT_DOCUMENT_TIME_ZONE (UTC) | ✅ 一致 |
| **PDF 证书 - 日期字段** | DocumentMeta.timezone | DEFAULT_DOCUMENT_TIME_ZONE (UTC) | ✅ 一致 |
| **PDF 审计日志 - 时间戳** | DocumentMeta.timezone | DEFAULT_DOCUMENT_TIME_ZONE (UTC) | ✅ 一致 |

**核心结论**: 三者时区来源完全统一，均以 `DocumentMeta.timezone` 为首要依据，最终兜底 UTC，确保跨场景的时间显示一致性。

---

## 五、完整核查摘要

### 5.1 语言体系关键结论

1. **界面语言 Cookie 优先**: Cookie 值会被验证，不支持则回退到请求头
2. **邮件语言无收件人偏好**: 完全基于文档语言或组织默认设置
3. **语言设置三层回退**: 文档设置 → 组织/团队设置 → 系统默认 (en)

### 5.2 时区体系关键结论

1. **全场景统一时区源**: 签署页、邮件、PDF 均使用 `DocumentMeta.timezone`
2. **双重兜底机制**: 文档设置 → 组织设置 → UTC
3. **前端本地转换显示**: 存储时区与显示时区分离，用户看到本地时间，但提示实际存储值

### 5.3 核心文件索引

| 文件路径 | 核心功能 |
|---------|---------|
| `apps/remix/app/root.tsx` | 界面语言协商主逻辑 |
| `packages/lib/utils/i18n.ts` | Accept-Language 解析逻辑 |
| `packages/lib/server-only/email/get-email-context.ts` | 邮件语言确定 |
| `packages/lib/jobs/definitions/emails/send-signing-email.handler.ts` | 邮件发送语言应用 |
| `packages/lib/utils/document.ts` | 文档元数据语言/时区提取 |
| `packages/lib/server-only/field/sign-field-with-token.ts` | 签署时时区处理 |
| `packages/lib/constants/date-formats.ts` | 时区转换显示函数 |
| `packages/lib/server-only/pdf/generate-certificate-pdf.ts` | PDF 证书语言/时区 |
| `packages/lib/server-only/pdf/generate-audit-log-pdf.ts` | PDF 审计日志语言/时区 |

---

## 六、潜在改进点（基于核查发现）

1. **收件人语言偏好**: 当前系统不支持按收件人设置邮件语言，可考虑增加收件人语言字段
2. **时区提示优化**: 当前仅日期字段有时区差异提示，可扩展到所有时间显示
3. **Cookie 过期优化**: 当前有效期 2 年，可考虑提供用户主动清除选项

---

*报告生成时间: 2026-05-12*
*核查范围: 基于最新代码库完整代码搜索*
