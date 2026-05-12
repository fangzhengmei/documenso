# Documenso 语言与时区体系完整报告

## 概述

本报告详细说明了 Documenso 签名流程中的语言处理机制和时区处理方案，包括界面语言协商、邮件语言渲染、文档语言字段、时区显示一致性等核心功能的优先级与回退路径。

---

## 一、语言处理体系

### 1.1 支持的语言列表

系统支持以下语言（定义于 `packages/lib/constants/locales.ts`）：

| 语言代码 | 语言名称 |
|---------|---------|
| de | 德语 |
| en | 英语（默认） |
| fr | 法语 |
| es | 西班牙语 |
| it | 意大利语 |
| nl | 荷兰语 |
| pl | 波兰语 |
| pt-BR | 葡萄牙语（巴西） |
| ja | 日语 |
| ko | 韩语 |
| zh | 中文 |

### 1.2 界面语言协商机制

#### 优先级顺序

```
1. 用户语言 Cookie 存储 (lang cookie)
   ↓
2. HTTP Accept-Language 请求头解析
   ↓
3. 系统默认语言 (en)
```

#### 实现细节

**语言解析流程** (`packages/lib/utils/i18n.ts`):

1. **从请求头解析**:
   - 从 `Accept-Language` 头中提取语言列表
   - 解析语言代码（如 `zh-CN` → `zh`）
   - 验证语言是否在支持列表中
   - 验证格式有效性（使用 `Intl.Locale`）

2. **回退机制**:
   - 若解析的语言不支持 → 使用默认语言 `en`
   - 若请求头为空 → 使用默认语言 `en`

**关键代码**:
```typescript
// 解析语言代码
const parseLanguageFromLocale = (locale: string): SupportedLanguageCodes | null => {
  const [language, _country] = locale.split('-');
  // 验证是否在支持列表中
};

// 提取语言数据
export const extractLocaleData = ({ headers }: ExtractLocaleDataOptions): I18nLocaleData => {
  const headerLocales = (headers.get('accept-language') ?? '').split(',');
  // 解析并验证语言
  return {
    lang: languages[0] || APP_I18N_OPTIONS.sourceLang, // 回退到 en
    locales: headerLocales,
  };
};
```

### 1.3 文档语言字段处理

#### 优先级顺序

```
1. 用户显式设置的文档语言
   ↓
2. 模板预设语言
   ↓
3. 组织/团队设置的默认语言
   ↓
4. 系统默认语言 (en)
```

#### 处理流程

**文档元数据语言字段**:
- 存储位置: `DocumentMeta.language` 字段
- 数据类型: `SupportedLanguageCodes`
- Schema 验证: `ZDocumentMetaLanguageSchema`

**创建流程** (`packages/lib/utils/document.ts:28-64`):

```typescript
export const extractDerivedDocumentMeta = (
  settings: Omit<OrganisationGlobalSettings, 'id'>,
  overrideMeta: Partial<DocumentMeta> | undefined | null,
) => {
  return {
    // 优先级: 覆盖值 > 组织设置
    language: meta.language || settings.documentLanguage,
    // ... 其他字段
  };
};
```

**从模板创建时的特殊处理** (`packages/lib/server-only/template/create-document-from-template.ts:518`):

```typescript
language: 
  override?.language ||          // 1. 用户覆盖
  template.documentMeta?.language || // 2. 模板预设
  settings.documentLanguage,    // 3. 团队/组织设置
  // (隐式回退到 en)
```

### 1.4 收件人邮件语言渲染

#### 优先级顺序

```
1. 文档元数据中设置的语言 (DocumentMeta.language)
   ↓
2. 组织/团队设置的默认文档语言
   ↓
3. 系统默认语言 (en)
```

#### 实现流程

**邮件上下文获取** (`packages/lib/server-only/email/get-email-context.ts:89`):

```typescript
const emailLanguage = meta?.language || settings.documentLanguage;
```

**邮件渲染** (`packages/lib/utils/render-email-with-i18n.tsx`):

1. 接收语言参数
2. 验证语言代码有效性
3. 获取对应语言的 i18n 实例
4. 激活语言环境
5. 使用 Lingui 渲染邮件组件

```typescript
export const renderEmailWithI18N = async (component, options) => {
  const lang = isValidLanguageCode(providedLang) 
    ? providedLang 
    : APP_I18N_OPTIONS.sourceLang;

  const i18n = await getI18nInstance(lang);
  i18n.activate(lang);

  return renderWithI18N(component, { i18n, ...otherOptions });
};
```

---

## 二、时区处理体系

### 2.1 支持的时区

系统使用 `@vvo/tzdb` 时区数据库，支持:
- 所有标准时区（IANA 时区数据库）
- 默认时区: `Etc/UTC`（协调世界时）
- 完整时区列表: `TIME_ZONES` 数组

关键常量 (`packages/lib/constants/time-zones.ts`):
```typescript
export const DEFAULT_DOCUMENT_TIME_ZONE = 'Etc/UTC';
export const TIME_ZONES = ['Etc/UTC', ...timeZonesNames];
```

### 2.2 文档时区字段处理

#### 优先级顺序

```
1. 用户显式设置的文档时区
   ↓
2. 模板预设时区
   ↓
3. 组织/团队设置的默认时区
   ↓
4. 系统默认时区 (Etc/UTC)
```

#### 实现流程

**文档元数据时区字段**:
- 存储位置: `DocumentMeta.timezone` 字段
- Schema 验证: `ZDocumentMetaTimezoneSchema`
- 用于: 日期字段显示、签名时间戳格式化

**提取逻辑** (`packages/lib/utils/document.ts:38`):

```typescript
timezone: 
  meta.timezone ||                // 1. 用户覆盖
  settings.documentTimezone ||    // 2. 组织设置
  DEFAULT_DOCUMENT_TIME_ZONE,     // 3. 默认 UTC
```

### 2.3 日期字段的时区显示

#### 处理流程

**日期字段组件** (`packages/apps/remix/app/components/general/document-signing/document-signing-date-field.tsx`):

```typescript
// 1. 获取时区参数（从文档元数据传入）
timezone = DEFAULT_DOCUMENT_TIME_ZONE

// 2. 转换为本地系统格式显示
const localDateString = convertToLocalSystemFormat(
  field.customText, 
  dateFormat, 
  timezone
);

// 3. 显示时区差异提示
const isDifferentTime = field.inserted && localDateString !== field.customText;
const tooltipText = `"${field.customText}" will appear on the document as it has a timezone of "${timezone || ''}".`;
```

#### 核心原则

1. **存储一致性**: 所有日期时间统一使用 UTC 存储
2. **显示本地化**: 前端根据文档时区转换为本地时间显示
3. **提示明确**: 当显示时间与存储时间因时区不同而有差异时，显示提示信息
4. **格式统一**: 日期格式与时区配合，确保跨时区用户看到一致的日期表示

---

## 三、语言与时区优先级总览

### 3.1 完整优先级矩阵

| 层级 | 语言字段 | 时区字段 | 来源位置 |
|-----|---------|---------|---------|
| 1（最高） | 用户界面设置 | 用户界面设置 | AddSettingsForm, DocumentPreferencesForm |
| 2 | API/覆盖参数 | API/覆盖参数 | create-document-from-template.ts override |
| 3 | 模板 DocumentMeta | 模板 DocumentMeta | Template.documentMeta |
| 4 | 组织/团队设置 | 组织/团队设置 | OrganisationGlobalSettings, TeamSettings |
| 5（最低） | en (默认) | Etc/UTC (默认) | APP_I18N_OPTIONS, DEFAULT_DOCUMENT_TIME_ZONE |

### 3.2 各场景下的语言选择

#### 场景 1: 创建新文档（空白文档）

```
用户在设置页面选择语言 → 存储到 DocumentMeta.language
  ↓
发送邮件时使用该语言渲染
  ↓
收件人看到对应语言的邮件通知
```

#### 场景 2: 从模板创建文档

```
检查是否有覆盖参数 (override.language) → 是则使用
  ↓ 否则
使用模板预设的语言 (template.documentMeta.language)
  ↓ 否则
使用团队/组织的默认文档语言 (settings.documentLanguage)
  ↓ 否则
回退到英语 (en)
```

#### 场景 3: 收件人访问签名页面

```
浏览器 Accept-Language 头 → 解析支持的语言
  ↓ 否则
使用语言 Cookie 中的值
  ↓ 否则
回退到英语 (en)
```

#### 场景 4: 邮件发送给收件人

```
使用 DocumentMeta.language（发件人设的文档语言）
  ↓ 否则
使用组织/团队设置的默认语言
  ↓ 否则
回退到英语 (en)
```

---

## 四、关键实现文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `packages/lib/constants/locales.ts` | 支持的语言代码列表和默认值 |
| `packages/lib/constants/i18n.ts` | 语言常量和验证函数 |
| `packages/lib/utils/i18n.ts` | 语言解析、i18n 激活等工具函数 |
| `packages/lib/utils/render-email-with-i18n.tsx` | 带 i18n 的邮件渲染 |
| `packages/lib/utils/document.ts` | 文档元数据提取（含语言和时区） |
| `packages/lib/server-only/email/get-email-context.ts` | 邮件上下文（含语言） |
| `packages/lib/server-only/template/create-document-from-template.ts` | 从模板创建文档时的语言处理 |
| `packages/lib/constants/time-zones.ts` | 时区常量和时区列表 |
| `packages/apps/remix/app/storage/lang-cookie.server.ts` | 语言 Cookie 处理 |
| `packages/lib/client-only/providers/i18n.client.tsx` | 客户端 i18n Provider |
| `packages/lib/client-only/providers/i18n-server.tsx` | 服务端 i18n Provider |
| `packages/apps/remix/app/components/general/document-signing/document-signing-date-field.tsx` | 日期字段时区显示 |

---

## 五、设计原则总结

### 5.1 语言处理原则

1. **三层回退**: 用户设置 → 组织设置 → 系统默认
2. **邮件与界面分离**: 邮件使用文档语言，界面使用浏览器/用户设置语言
3. **模板继承**: 从模板创建文档时继承模板语言配置
4. **统一验证**: 所有语言代码统一通过 `isValidLanguageCode` 验证

### 5.2 时区处理原则

1. **UTC 存储**: 所有时间数据统一使用 UTC 存储
2. **文档级时区**: 每个文档有独立的时区设置
3. **本地显示**: 前端根据文档时区转换为本地时间显示
4. **差异提示**: 时区差异时提供明确的用户提示

### 5.3 一致性保障

- 语言和时区字段在整个文档生命周期保持一致
- 邮件通知、PDF 证书、审计日志使用相同的语言与时区设置
- 从模板创建文档时完整继承语言与时区配置
- 所有时间格式化操作统一使用文档时区参数

---

*报告生成时间: 2026-01-11*
*基于 Documenso 代码库分析*
