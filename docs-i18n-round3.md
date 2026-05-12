# Documenso 语言与时区体系核查报告（第三轮 - 可核验版）

## 概述

本报告基于代码级精确核查，修正前序报告中的不精确表述，逐项核验关键逻辑，并明确区分已确认事实与暂不确定点。

---

## 一、界面语言 Cookie 有效期精确核验

### 1.1 代码精确核对

**文件位置**: `apps/remix/app/storage/lang-cookie.server.ts:4-9`

```typescript
export const langCookie = createCookie('lang', {
  path: '/',
  maxAge: 60 * 60 * 24 * 365 * 2,  // 第 6 行
  httpOnly: true,
  secure: env('NODE_ENV') === 'production',
});
```

### 1.2 精确计算

| 表达式 | 计算结果 | 含义 |
|-------|---------|-----|
| 60 * 60 | 3,600 | 1 小时（秒） |
| 3,600 * 24 | 86,400 | 1 天（秒） |
| 86,400 * 365 | 31,536,000 | 1 年（秒） |
| 31,536,000 * 2 | **63,072,000 秒** | **2 年（秒）** |

### 1.3 修正后结论

✅ **Cookie 有效期为 2 年（730 天）**，前序报告中"1 年"表述错误，特此修正。

---

## 二、PDF 证书时间展示链路精确拆解

### 2.1 核心代码核查

**文件位置**: `packages/lib/server-only/pdf/render-certificate.ts:420-461`

```typescript
// 第 423-431 行: Sent 时间显示
value: recipient.logs.emailed
  ? DateTime.fromJSDate(recipient.logs.emailed.createdAt)
      .setLocale(APP_I18N_OPTIONS.defaultLocale)  // 仅设置 locale
      .toFormat('yyyy-MM-dd hh:mm:ss a (ZZZZ)')   // 格式化，但无 setZone
  : recipient.logs.sent
    ? DateTime.fromJSDate(recipient.logs.sent.createdAt)
        .setLocale(APP_I18N_OPTIONS.defaultLocale)
        .toFormat('yyyy-MM-dd hh:mm:ss a (ZZZZ)')
    : i18n._(msg`Unknown`),

// 第 435-439 行: Viewed 时间显示
value: recipient.logs.opened
  ? DateTime.fromJSDate(recipient.logs.opened.createdAt)
      .setLocale(APP_I18N_OPTIONS.defaultLocale)
      .toFormat('yyyy-MM-dd hh:mm:ss a (ZZZZ)')
  : i18n._(msg`Unknown`),

// 第 445-459 行: Rejected/Signed 时间显示
value: DateTime.fromJSDate(recipient.logs.rejected.createdAt)
  .setLocale(APP_I18N_OPTIONS.defaultLocale)
  .toFormat('yyyy-MM-dd hh:mm:ss a (ZZZZ)')
```

### 2.2 链路精确拆解

```
数据库存储 (UTC 时间戳)
    ↓
DateTime.fromJSDate() → Luxon DateTime 对象（默认系统时区）
    ↓
.setLocale(APP_I18N_OPTIONS.defaultLocale) → 仅设置语言（如 'en-US'），不改变时区
    ↓
.toFormat('yyyy-MM-dd hh:mm:ss a (ZZZZ)') → 格式化输出
    ↓
最终显示: 使用服务器运行环境的默认时区，格式中 (ZZZZ) 显示其时区名称
```

### 2.3 关键发现

❌ **PDF 证书中的时间戳不使用文档时区**

对比签署/邮件环节的典型用法：
```typescript
// 签署时写入日期字段（使用文档时区）
DateTime.now()
  .setZone(documentMeta?.timezone ?? DEFAULT_DOCUMENT_TIME_ZONE)  // ✅ 有 setZone
  .toFormat(documentMeta?.dateFormat ?? DEFAULT_DOCUMENT_DATE_FORMAT)

// PDF 证书显示（不使用文档时区）
DateTime.fromJSDate(createdAt)
  .setLocale(APP_I18N_OPTIONS.defaultLocale)                    // ❌ 无 setZone
  .toFormat('yyyy-MM-dd hh:mm:ss a (ZZZZ)')
```

---

## 三、PDF 审计日志时间展示链路精确拆解

### 3.1 核心代码核查

**文件位置**: `packages/lib/server-only/pdf/render-audit-logs.ts:239-250`

```typescript
// 第 239-244 行: Created At 显示
text: DateTime.fromJSDate(envelope.createdAt)
  .setLocale(APP_I18N_OPTIONS.defaultLocale)
  .toFormat('yyyy-MM-dd hh:mm:ss a (ZZZZ)'),

// 第 246-250 行: Last Updated 显示
text: DateTime.fromJSDate(envelope.updatedAt)
  .setLocale(APP_I18N_OPTIONS.defaultLocale)
  .toFormat('yyyy-MM-dd hh:mm:ss a (ZZZZ)'),
```

**文件位置**: `packages/lib/server-only/pdf/render-audit-logs.ts:223-228`

```typescript
// 第 223-228 行: 时区信息仅作静态文本显示
const timeZoneLabel = renderOverviewCardLabels({
  label: i18n._(msg`Time Zone`),
  text: envelope.documentMeta?.timezone || 'N/A',  // 仅显示，不影响时间格式化
  width: columnWidth,
  groupX: columnWidth + columnSpacing,
});
```

### 3.2 链路精确拆解

```
数据库存储 (UTC 时间戳)
    ↓
DateTime.fromJSDate() → Luxon DateTime 对象（默认系统时区）
    ↓
.setLocale(APP_I18N_OPTIONS.defaultLocale) → 仅设置语言
    ↓
.toFormat('yyyy-MM-dd hh:mm:ss a (ZZZZ)') → 格式化输出
    ↓
最终显示: 使用服务器运行环境的默认时区

同时:
  独立展示文档时区字段 → envelope.documentMeta?.timezone || 'N/A'
  （仅作信息展示，不影响任何时间戳的格式化）
```

### 3.3 关键发现

❌ **PDF 审计日志中的时间戳同样不使用文档时区**

✅ 文档时区字段仅作静态信息展示，与时间格式化逻辑完全分离

---

## 四、全场景时区链路对比表

| 场景 | 是否调用 .setZone() | 时区来源 | 格式化格式 | 精确性 |
|-----|---------------------|---------|-----------|-------|
| **签署页 - 日期字段显示** | ✅ 是 | documentMeta.timezone ?? UTC | 用户自定义格式 | 精确 |
| **签署页 - 签署时写入** | ✅ 是 | documentMeta.timezone ?? UTC | 用户自定义格式 | 精确 |
| **邮件 - 签署时日期字段** | ✅ 是 | documentMeta.timezone ?? UTC | 用户自定义格式 | 精确 |
| **邮件 - 完成文档时间** | ✅ 是 | documentMeta.timezone ?? UTC | 用户自定义格式 | 精确 |
| **PDF 证书 - 所有时间戳** | ❌ 否 | 服务器运行环境默认时区 | 硬编码固定格式 | 不使用文档时区 |
| **PDF 审计日志 - 所有时间戳** | ❌ 否 | 服务器运行环境默认时区 | 硬编码固定格式 | 不使用文档时区 |
| **PDF 审计日志 - 时区字段显示** | - | documentMeta.timezone || 'N/A' | 纯文本 | 仅展示，不影响格式化 |

### 4.1 一致性结论修正

**前序报告结论修正**:
- ❌ 错误: "三者时区来源完全统一"
- ✅ 正确: **签署页与邮件使用文档时区；PDF 时间戳使用服务器默认时区，仅独立显示文档时区作为文本**

---

## 五、语言体系关键点复核

### 5.1 界面语言协商（已确认）

**优先级顺序已确认正确**:
```
1. Cookie 语言 → 验证通过则直接使用
   ↓ 不支持则
2. Accept-Language 请求头解析
   ↓ 回退
3. en（默认）
```

**关键代码行**: `apps/remix/app/root.tsx:52-56`
```typescript
let lang: SupportedLanguageCodes = await langCookie.parse(cookieHeader);
if (!APP_I18N_OPTIONS.supportedLangs.includes(lang)) {
  lang = extractLocaleData({ headers: request.headers }).lang;
}
```

### 5.2 邮件语言选择（已确认）

**无收件人偏好机制已确认正确**:
```
1. DocumentMeta.language
   ↓
2. settings.documentLanguage
   ↓
3. en（Zod schema 兜底）
```

**关键代码行**: `packages/lib/server-only/email/get-email-context.ts:89`
```typescript
const emailLanguage = meta?.language || emailContext.settings.documentLanguage;
```

**Recipient 模型核查**: 无 `language` 字段，代码中无按收件人差异化邮件语言的逻辑。

---

## 六、已确认事实清单

以下结论均有明确代码支撑，可通过指定文件行号核验：

### 语言相关
1. ✅ Cookie 有效期为 2 年（63,072,000 秒）
   - 核验位置: `apps/remix/app/storage/lang-cookie.server.ts:6`
2. ✅ 界面语言 Cookie 优先于请求头，且会验证支持性
   - 核验位置: `apps/remix/app/root.tsx:52-56`
3. ✅ 邮件语言仅基于文档语言或组织设置，无收件人语言偏好
   - 核验位置: `packages/lib/server-only/email/get-email-context.ts:89`
4. ✅ Recipient 模型无 language 字段
   - 核验位置: Prisma schema 及相关查询代码

### 时区相关
5. ✅ 签署时写入日期字段使用文档时区
   - 核验位置: `packages/lib/server-only/field/sign-field-with-token.ts:197-201`
6. ✅ PDF 证书时间戳不调用 .setZone()，使用服务器默认时区
   - 核验位置: `packages/lib/server-only/pdf/render-certificate.ts:420-461`
7. ✅ PDF 审计日志时间戳不调用 .setZone()，使用服务器默认时区
   - 核验位置: `packages/lib/server-only/pdf/render-audit-logs.ts:239-250`
8. ✅ PDF 审计日志中的时区字段仅作静态文本展示，不影响格式化
   - 核验位置: `packages/lib/server-only/pdf/render-audit-logs.ts:223-228`
9. ✅ 默认文档时区常量为 Etc/UTC
   - 核验位置: `packages/lib/constants/time-zones.ts:5`

---

## 七、暂不确定点清单（待进一步核查）

以下内容现有代码无法完全确认，需额外信息或进一步核查：

### 环境相关
1. ❓ 生产环境服务器的默认时区是什么？
   - 影响 PDF 时间戳的实际显示值
   - 代码中无显式设置，依赖部署环境配置

### PDF 渲染相关
2. ❓ APP_I18N_OPTIONS.defaultLocale 的具体值是什么？
   - 影响日期格式的本地化（如月份名称语言）
   - 不影响时区计算

3. ❓ PDF 中日期字段（已签署的）如何展示？
   - 签署时已按时区格式化为字符串存储
   - PDF 渲染时是直接显示字符串还是重新格式化？
   - 需核查 PDF 字段渲染逻辑

### 多语言完整性
4. ❓ 所有邮件模板是否都正确支持 i18n？
   - 已核查 signing-email，其他邮件类型待逐一确认

### 边缘场景
5. ❓ 文档创建后修改时区，历史签署时间如何展示？
   - 日期字段已存储为格式化字符串
   - 审计日志使用服务器时区
   - 时区字段显示最新值而非历史值

---

## 八、核心问题汇总

### 已发现的不一致点

1. **时区不一致**: PDF 时间戳不使用文档时区，签署页与邮件使用文档时区
2. **格式不一致**: PDF 使用硬编码格式（`yyyy-MM-dd hh:mm:ss a (ZZZZ)`），签署页/邮件使用用户自定义格式
3. **语言不一致**: PDF locale 硬编码为 APP_I18N_OPTIONS.defaultLocale，不随文档语言变化

### 潜在影响

- 用户在文档设置中选择了特定时区，但 PDF 证书时间显示可能与此不符
- 跨时区协作时，审计日志时间可能产生混淆
- 时区字段显示的值与实际时间戳格式化使用的时区可能不一致

---

## 九、核心文件核验索引

| 文件路径 | 关键行 | 核验点 | 状态 |
|---------|-------|--------|-----|
| `apps/remix/app/storage/lang-cookie.server.ts` | 6 | Cookie 有效期 | ✅ 已核验 |
| `apps/remix/app/root.tsx` | 52-56 | 语言协商优先级 | ✅ 已核验 |
| `packages/lib/server-only/email/get-email-context.ts` | 89 | 邮件语言来源 | ✅ 已核验 |
| `packages/lib/server-only/field/sign-field-with-token.ts` | 197-201 | 签署时区处理 | ✅ 已核验 |
| `packages/lib/server-only/pdf/render-certificate.ts` | 420-461 | 证书时间格式化 | ✅ 已核验 |
| `packages/lib/server-only/pdf/render-audit-logs.ts` | 223-228, 239-250 | 审计日志时间格式化 | ✅ 已核验 |
| `packages/lib/constants/time-zones.ts` | 5 | 默认时区常量 | ✅ 已核验 |

---

*报告生成时间: 2026-05-12*
*核验方法: 精确代码行级追踪与对比*
*所有结论均可通过提供的文件路径与行号独立复核验证*
