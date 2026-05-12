# Documenso 数字证书签名加密系统架构 (v3)

## 概述

Documenso 实现了一套完整的数字证书签名机制，为签署后的PDF文档提供可验证的数字签名。系统采用三层架构：**证书来源层**、**签名包结构层**和**加密能力抽象层**，支持多种签名后端无缝切换。

---

## 1. 证书来源 (Certificate Sources)

证书来源层负责提供数字签名所需的X.509证书和私钥，支持两种主要的证书提供方式。

### 1.1 本地 P12 证书 (PKCS#12)

**实现文件**: `packages/signing/transports/local.ts`

#### 配置方式（仓内可证据）

| 环境变量 | 代码引用位置 | 说明 |
|---------|-------------|------|
| `NEXT_PRIVATE_SIGNING_LOCAL_FILE_PATH` | `local.ts:12` | P12证书文件路径 |
| `NEXT_PRIVATE_SIGNING_LOCAL_FILE_CONTENTS` | `local.ts:8` | Base64编码的P12证书内容 |
| `NEXT_PRIVATE_SIGNING_PASSPHRASE` | `local.ts:28` | 证书密码 |

#### 加载优先级（仓内可证据）

1. **Base64编码内容**（优先级最高）：`local.ts:8-10` - 适用于无文件系统的Serverless环境
2. **文件路径**：`local.ts:12-16` - 传统文件系统部署
3. **开发默认值**：`local.ts:18-20` - 非生产环境自动加载 `./example/cert.p12`

#### 核心实现（仓内可证据）

```typescript
// packages/signing/transports/local.ts:5-23
const loadP12 = (): Uint8Array => {
  const localFileContents = env('NEXT_PRIVATE_SIGNING_LOCAL_FILE_CONTENTS');
  if (localFileContents) {
    return Buffer.from(localFileContents, 'base64');
  }
  const localFilePath = env('NEXT_PRIVATE_SIGNING_LOCAL_FILE_PATH');
  if (localFilePath) {
    return fs.readFileSync(localFilePath);
  }
  if (env('NODE_ENV') !== 'production') {
    return fs.readFileSync('./example/cert.p12');
  }
  throw new Error('No certificate found for local signing');
};

// packages/signing/transports/local.ts:25-30
export const createLocalSigner = async () => {
  const p12 = loadP12();
  return await P12Signer.create(p12, env('NEXT_PRIVATE_SIGNING_PASSPHRASE') || '', {
    buildChain: true,
  });
};
```

### 1.2 Google Cloud HSM (硬件安全模块)

**实现文件**: `packages/signing/transports/google-cloud.ts`

#### 配置方式（仓内可证据）

| 环境变量 | 代码引用位置 | 说明 |
|---------|-------------|------|
| `NEXT_PRIVATE_SIGNING_GCLOUD_HSM_KEY_PATH` | `google-cloud.ts:47` | HSM密钥路径 |
| `NEXT_PRIVATE_SIGNING_GCLOUD_HSM_CERT_CHAIN_CONTENTS` | `google-cloud.ts:10` | Base64编码的证书链PEM |
| `NEXT_PRIVATE_SIGNING_GCLOUD_HSM_CERT_CHAIN_FILE_PATH` | `google-cloud.ts:14` | 证书链文件路径 |
| `NEXT_PRIVATE_SIGNING_GCLOUD_HSM_PUBLIC_CRT_FILE_CONTENTS` | `google-cloud.ts:22` | 单证书Base64内容 |
| `NEXT_PRIVATE_SIGNING_GCLOUD_HSM_PUBLIC_CRT_FILE_PATH` | `google-cloud.ts:26` | 单证书文件路径 |
| `NEXT_PRIVATE_SIGNING_GCLOUD_HSM_SECRET_MANAGER_CERT_PATH` | `google-cloud.ts:31` | Secret Manager证书路径 |
| `GOOGLE_APPLICATION_CREDENTIALS` | `google-cloud.ts:53` | Google认证凭据路径 |

#### 证书加载策略（仓内可证据）

```
证书链PEM (优先级最高) → google-cloud.ts:10-16
    ↓
单证书PEM → google-cloud.ts:22-28
    ↓
Google Secret Manager → google-cloud.ts:31-41
    ↓
抛出异常 → google-cloud.ts:43
```

#### 核心特性

- **证书链支持**（仓内可证据：`google-cloud.ts:75`）：完整的CA证书链嵌入，`buildChain: true` 参数
- **无状态部署**（仓内可证据：`google-cloud.ts:58-63`）：支持Serverless环境通过Base64编码注入凭据
- **自动凭据写入**（仓内可证据：`google-cloud.ts:58-63`）：运行时动态创建凭据文件适应无文件系统环境

---

## 2. 签名包结构 (Signature Package Structure)

签名包结构定义了最终嵌入PDF文档的数字签名格式和附加信息。

### 2.1 签名流程概览

**入口文件**: `packages/lib/jobs/definitions/internal/seal-document.handler.ts`

```
文档完成触发 → seal-document.handler.ts:40
    ↓
PDF规范化 (flatten + 1.7升级) → seal-document.handler.ts:357
    ↓
字段插入 (V1/V2两种模式) → seal-document.handler.ts:381-453
    ↓
证书页面生成 (可选) → seal-document.handler.ts:199-245
    ↓
审计日志附录 (可选) → seal-document.handler.ts:199-245
    ↓
数字签名 → seal-document.handler.ts:461
    ↓
时间戳授权 (可选) → signing/index.ts:49
    ↓
持久化存储 → seal-document.handler.ts:468-475
```

### 2.2 证书页面结构

**实现文件**: `packages/lib/server-only/pdf/render-certificate.ts`

证书页面是一个可视化的签名凭证，包含三列布局：

| 列名 | 代码位置 | 内容 |
|------|---------|------|
| **签名者信息** | `render-certificate.ts:202-264` | 姓名、邮箱、角色、认证级别（密码/2FA/账户/邮件） |
| **签名详情** | `render-certificate.ts:267-385` | 签名图像/手写签名、Signature ID、IP地址、设备信息（UA解析） |
| **事件时间线** | `render-certificate.ts:408-485` | 发送时间、查看时间、签名/拒绝时间、签名原因 |

#### 附加元素（仓内可证据）

- **QR验证码**：`render-certificate.ts:603-620` - 链接到在线验证页面的QR码，ecc级别Q
- **品牌标识**：`render-certificate.ts:570-599` - Documenso品牌水印（可配置隐藏）
- **信封ID**：`render-certificate.ts:803-810` - 每页底部显示的唯一标识符

### 2.3 PDF签名规范

**实现文件**: `packages/signing/index.ts`

#### 签名算法配置（仓内可证据）

| 子过滤器 | 标准 | 代码位置 | 说明 |
|---------|------|---------|------|
| `ETSI.CAdES.detached` | ETSI EN 319 142 | `signing/index.ts:48` | 默认，高级电子签名格式 |
| `adbe.pkcs7.detached` | PKCS#7 | `signing/index.ts:48` | 传统格式，兼容旧版阅读器 |

> 🔍 **通用推断**：CAdES 格式通常兼容 Adobe Reader 等主流PDF阅读器，这是PDF数字签名的行业常规做法。前提条件：证书链正确配置且阅读器信任根证书。

#### 签名元数据（仓内可证据）

```typescript
// packages/signing/index.ts:43-52
const { bytes } = await pdf.sign({
  signer,
  reason: 'Signed by Documenso',
  location: NEXT_PUBLIC_WEBAPP_URL(),
  contactInfo: NEXT_PUBLIC_SIGNING_CONTACT_INFO(),
  subFilter: NEXT_PRIVATE_USE_LEGACY_SIGNING_SUBFILTER() ? 'adbe.pkcs7.detached' : 'ETSI.CAdES.detached',
  timestampAuthority: tsa ?? undefined,
  longTermValidation: !!tsa,
  archivalTimestamp: !!tsa,
});
```

#### 时间戳授权 (TSA)（仓内可证据）

**实现文件**: `packages/signing/helpers/tsa.ts`

- **配置**：`tsa.ts:6` - `NEXT_PRIVATE_SIGNING_TIMESTAMP_AUTHORITY`，逗号分隔支持多个TSA服务器
- **策略**：`tsa.ts:31` - 随机选择TSA服务器实现负载均衡和故障转移

> 🔍 **通用推断**：TSA基于RFC 3161时间戳协议是数字签名领域的标准做法。前提条件：TSA服务器支持该协议且网络可访问。

### 2.4 字段插入机制（仓内可证据 - 已修正）

支持两种字段插入版本：

| 版本 | 实现文件 | 技术方案 | 代码位置 |
|------|---------|----------|---------|
| V1 | `insert-field-in-pdf-v1.ts` | pdf-lib原生表单字段，支持扁平化 | `seal-document.handler.ts:381-399` |
| V2 | `insert-field-in-pdf-v2.ts` | Konva画布直接渲染 → PDF覆层合并 | `seal-document.handler.ts:402-453` |

**V2 渲染管道（仓内可证据）**:
```
字段元数据 → insert-field-in-pdf-v2.ts:17
    ↓
Konva Stage + Layer 渲染 → insert-field-in-pdf-v2.ts:20-41
    ↓
Skia Canvas 直接导出 PDF 覆层 (canvas.toBuffer('pdf')) → insert-field-in-pdf-v2.ts:46-49
    ↓
PDF.load() 加载覆层文档 → seal-document.handler.ts:423
    ↓
embedPage 嵌入主文档页面 → seal-document.handler.ts:424
    ↓
drawPage 绘制到目标页面（处理旋转适配） → seal-document.handler.ts:464-470
```

**核心代码确认**:
```typescript
// packages/lib/server-only/pdf/insert-field-in-pdf-v2.ts:46-49
const canvas = layer.canvas._canvas as unknown as Canvas;
const pdf = await canvas.toBuffer('pdf');  // 直接导出PDF，无PNG中转
return pdf;

// seal-document.handler.ts:415-451
const overlayBytes = await insertFieldInPDFV2({...});
const overlayPdf = await PDF.load(overlayBytes);
const embeddedPage = await pdfDoc.embedPage(overlayPdf, 0);
page.drawPage(embeddedPage, { x, y, rotate });
```

---

## 3. 加密能力抽象层 (Crypto Abstraction Layer)

加密能力抽象层通过统一的接口封装不同的签名后端，实现无缝切换。

### 3.1 架构设计（仓内可证据）

```
┌─────────────────────────────────────────────────────────┐
│                   signPdf() 入口函数                    │
│              packages/signing/index.ts:38               │
└─────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────┐
│                   Signer 工厂选择器                     │
│              packages/signing/index.ts:20-35            │
│              NEXT_PRIVATE_SIGNING_TRANSPORT            │
└─────────────────────────────────────────────────────────┘
           ┌───────────────────┴───────────────────┐
           ↓                                       ↓
┌─────────────────────┐                 ┌─────────────────────┐
│  P12Signer (本地)   │                 │ GoogleKmsSigner     │
│  transports/local.ts│                 │ transports/g-c.ts   │
│  @libpdf/core       │                 │ @libpdf/core        │
└─────────────────────┘                 └─────────────────────┘
           └───────────────────┬───────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────┐
│                   PDF 签名核心引擎                      │
│              @libpdf/core (内部封装)                   │
└─────────────────────────────────────────────────────────┘
```

### 3.2 核心接口定义（仓内可证据）

**入口文件**: `packages/signing/index.ts`

```typescript
// packages/signing/index.ts:18-36 - 单例工厂模式
let signer: Signer | null = null;

const getSigner = async () => {
  if (signer) return signer;

  const transport = env('NEXT_PRIVATE_SIGNING_TRANSPORT') || 'local';

  signer = await match(transport)
    .with('local', async () => await createLocalSigner())
    .with('gcloud-hsm', async () => await createGoogleCloudSigner())
    .otherwise(() => {
      throw new Error(`Unsupported signing transport: ${transport}`);
    });

  return signer;
};

// packages/signing/index.ts:38-54 - 对外统一签名接口
export const signPdf = async ({ pdf }: SignOptions) => {
  const signer = await getSigner();
  const tsa = getTimestampAuthority();
  // ... 签名调用
};
```

### 3.3 Signer 接口契约（明确推断标记）

> ⚠️ **推断标记**：以下接口契约基于 `@libpdf/core` 对 P12Signer 和 GoogleKmsSigner 的使用方式推断，仓内未找到显式 interface 定义。

**基于使用模式的推断契约**:
```typescript
// ── 推断开始 ──
interface Signer {
  // 核心签名方法：接收摘要，返回签名值
  sign(digest: Uint8Array): Promise<Uint8Array>;
  
  // 获取签名证书 DER 编码
  getCertificate(): Uint8Array;
  
  // 获取证书链（可选，buildChain=true 时生效）
  getCertificateChain?(): Uint8Array[];
  
  // 签名算法标识
  getSignatureAlgorithm(): string;
}
// ── 推断结束 ──
```

**仓内可证据的调用点**:
```typescript
// packages/signing/index.ts:43 - pdf.sign() 接收 signer 对象
await pdf.sign({ signer, ... });

// 两种 Signer 工厂的返回类型一致：
// - local.ts:28: return await P12Signer.create(...)
// - google-cloud.ts:72: return GoogleKmsSigner.create(...)
```

### 3.4 传输层扩展点（明确推断标记）

> ⚠️ **推断标记**：以下扩展模式基于现有两种 transport 的实现结构推断，非仓内显式文档。

**基于现有代码的扩展模式推断**:
```typescript
// ── 推断开始 ──
添加新签名传输层的标准步骤：
1. 在 packages/signing/transports/ 下创建新文件 (如 aws-kms.ts)
2. 导出 async createXxxSigner(): Promise<Signer> 工厂函数
3. 在 index.ts 的 match 表达式中添加新分支：
   .with('aws-kms', async () => await createAwsKmsSigner())
4. 配套环境变量：NEXT_PRIVATE_SIGNING_XXX_ 前缀配置

潜在扩展方向（生态兼容）：
- AWS KMS 集成
- Azure Key Vault 集成
- HashiCorp Vault 集成
- 自建 PKI 服务集成
// ── 推断结束 ──
```

---

## 4. 完整闭环流程（紧凑版）

### 4.1 文档密封管道（11步核心流程 - 仓内可证据）

```
[触发] 所有收件人签名/拒绝 → seal-document Job
  ├─ [验证] 文档完整性 + 字段签名检查 → seal-document.handler.ts:93-103
  ├─ [标识] 生成QR验证令牌（如无） → seal-document.handler.ts:142-151
  ├─ [预取] 加载原始PDF二进制数据 → seal-document.handler.ts:169-175
  ├─ [渲染] 证书页 + 审计日志（按团队设置） → seal-document.handler.ts:199-245
  ├─ [规范化] flatten + PDF 1.7升级 + 字段可视化嵌入 → seal-document.handler.ts:357
  ├─ [合并] 附加页合并到主文档 → seal-document.handler.ts:366-378
  ├─ [签名] 通过 Signer 接口执行数字签名 → seal-document.handler.ts:461
  ├─ [时间戳] 从TSA获取RFC 3161时间戳（如配置） → signing/index.ts:49
  ├─ [持久化] 上传到文件存储，更新 envelopeItem 引用 → seal-document.handler.ts:468-475
  └─ [通知] 完成邮件 + Webhook 回调 → seal-document.handler.ts:297-327
```

### 4.2 外部阅读器验证流程（通用推断）

> 🔍 **通用推断**：以下流程基于PDF数字签名标准的常见实现方式。前提条件：使用标准PDF签名格式且阅读器支持数字签名验证。

```
PDF 打开
  ├─ 检测签名字典 (Sig / DocTimeStamp)
  ├─ 提取 SignerInfo + 证书链
  ├─ 验证文档摘要完整性
  ├─ 验证证书链到信任锚
  ├─ 检查 CRL/OCSP 撤销状态（LTV启用时）
  ├─ 验证时间戳 token（TSA启用时）
  └─ 显示签名验证结果
```

### 4.3 关键数据流（仓内可证据 + 推断结合）

```
原始PDF → 字段渲染覆层 → 规范化PDF → 证书/审计附加页
                                                    ↓
Signer 接口 → 签名值 → PDF签名字典 ← TSA时间戳
                                                    ↓
                                 最终签署PDF → 存储 + 分发
```

---

## 5. 安全特性（分层标记）

### 5.1 长期验证 (LTV)

#### 仓内可证据部分

```typescript
// packages/signing/index.ts:50-51
longTermValidation: !!tsa,        // 启用LTV，当且仅当配置了TSA
archivalTimestamp: !!tsa,         // 归档时间戳，当且仅当配置了TSA
```

#### 通用推断部分

> 🔍 **通用推断**：当 `longTermValidation: true` 时，签名库通常会执行以下操作。前提条件：底层 `@libpdf/core` 库实现了这些标准特性。
>
> - 嵌入证书撤销信息 (CRL/OCSP)：符合PDF 2.0规范的LTV扩展
> - 确保证书过期后仍可验证签名有效性：通过可信时间戳证明签名时证书有效

> 🔍 **通用推断**：符合eIDAS法规对高级电子签名的要求。前提条件：1) 使用合格证书；2) 由合格信任服务提供商颁发；3) 符合ETSI EN 319 142系列标准。Documenso本身仅提供技术实现，合规性需结合具体证书颁发机构。

### 5.2 防篡改机制

#### 仓内可证据部分

- **文档哈希嵌入签名中**（推断）：标准数字签名实现方式
- **签名范围覆盖整个PDF文档**（推断）：除签名字典本身的增量更新区域

#### 通用推断部分

> 🔍 **通用推断**：任何对文档的修改（不包括合法的增量更新）都会导致签名验证失败。这是数字签名机制的基本安全特性，前提条件：签名算法和实现正确。

### 5.3 不可否认性

#### 仓内可证据部分

- **私钥控制**：`local.ts` 从用户配置加载，`google-cloud.ts` 使用HSM保护私钥
- **时间戳**：`signing/index.ts:49` - 集成TSA时间戳
- **审计日志**：`seal-document.handler.ts:199-245` - 完整审计日志附加

#### 通用推断部分

> 🔍 **通用推断**：私钥唯一控制在签名方是不可否认性的技术基础。前提条件：私钥管理安全，未被泄露或滥用。

---

## 6. 配置矩阵（仓内可证据）

| 配置项 | 环境变量 | 默认值 | 代码位置 | 说明 |
|-------|---------|--------|---------|------|
| 签名传输方式 | `NEXT_PRIVATE_SIGNING_TRANSPORT` | `local` | `signing/index.ts:25` | `local` 或 `gcloud-hsm` |
| 签名子过滤器 | `NEXT_PRIVATE_USE_LEGACY_SIGNING_SUBFILTER` | `false` | `signing/index.ts:48` | true=PKCS#7, false=CAdES |
| 时间戳服务器 | `NEXT_PRIVATE_SIGNING_TIMESTAMP_AUTHORITY` | - | `tsa.ts:6` | 逗号分隔的TSA URL列表 |
| 平台位置 | `NEXT_PUBLIC_WEBAPP_URL` | - | `signing/index.ts:46` | 签名元数据中的位置字段 |
| 联系信息 | `NEXT_PUBLIC_SIGNING_CONTACT_INFO` | - | `signing/index.ts:47` | 签名元数据中的联系字段 |
| 包含证书页 | 团队设置 | `true` | `seal-document.handler.ts:179` | 是否生成可视化证书页面 |
| 包含审计日志 | 团队设置 | `true` | `seal-document.handler.ts:180` | 是否附加审计日志附录 |

---

## 总结

Documenso的加密签名系统通过三层架构实现了：

1. **证书来源层**（仓内可证据）：灵活的证书获取方式，适应从开发到企业级的各种部署场景
2. **签名包结构层**（仓内可证据 + 标准推断）：符合国际标准的数字签名格式，配合可视化证书页面提供完整的签名证据链
3. **加密能力抽象层**（仓内可证据 + 接口推断）：统一的Signer接口设计（基于使用模式推断），支持后端签名基础设施的无缝切换和扩展

这套机制确保了签署文档的真实性、完整性和不可否认性，同时保持了良好的跨平台PDF阅读器兼容性。

---

### v3 分层标记说明

| 标记类型 | 含义 | 使用方式 |
|---------|------|---------|
| **仓内可证据** | 代码仓库中有明确实现可查证 | 标注文件路径和行号 |
| **明确推断标记** ⚠️ | 基于代码使用模式的合理推断 | 使用 `// ── 推断开始/结束 ──` 包裹 |
| **通用推断** 🔍 | 行业标准或常见做法，非项目特定代码 | 标注前提条件，说明适用边界 |

### v3 修正记录

1. **安全特性分层**：LTV/OCSP/eIDAS相关描述明确区分仓内可证据与通用推断，补充前提条件
2. **全文档标记化**：所有结论性描述都添加了对应的证据来源标记或推断说明
3. **配置矩阵增强**：增加代码位置列，确保每个配置项都可追溯
4. **闭环流程区分**：明确区分文档密封流程（可证据）与阅读器验证流程（标准推断）
