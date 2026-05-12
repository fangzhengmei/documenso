# Documenso 数字证书签名加密系统架构 (v2)

## 概述

Documenso 实现了一套完整的数字证书签名机制，为签署后的PDF文档提供可验证的数字签名。系统采用三层架构：**证书来源层**、**签名包结构层**和**加密能力抽象层**，支持多种签名后端无缝切换。

---

## 1. 证书来源 (Certificate Sources)

证书来源层负责提供数字签名所需的X.509证书和私钥，支持两种主要的证书提供方式。

### 1.1 本地 P12 证书 (PKCS#12)

**实现文件**: `packages/signing/transports/local.ts`

#### 配置方式

| 环境变量 | 说明 |
|---------|------|
| `NEXT_PRIVATE_SIGNING_LOCAL_FILE_PATH` | P12证书文件路径 |
| `NEXT_PRIVATE_SIGNING_LOCAL_FILE_CONTENTS` | Base64编码的P12证书内容 |
| `NEXT_PRIVATE_SIGNING_PASSPHRASE` | 证书密码 |

#### 加载优先级

1. **Base64编码内容**（优先级最高）：适用于无文件系统的Serverless环境
2. **文件路径**：传统文件系统部署
3. **开发默认值**：非生产环境自动加载 `./example/cert.p12`

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

#### 配置方式

| 环境变量 | 说明 |
|---------|------|
| `NEXT_PRIVATE_SIGNING_GCLOUD_HSM_KEY_PATH` | HSM密钥路径 |
| `NEXT_PRIVATE_SIGNING_GCLOUD_HSM_CERT_CHAIN_CONTENTS` | Base64编码的证书链PEM |
| `NEXT_PRIVATE_SIGNING_GCLOUD_HSM_CERT_CHAIN_FILE_PATH` | 证书链文件路径 |
| `NEXT_PRIVATE_SIGNING_GCLOUD_HSM_PUBLIC_CRT_FILE_CONTENTS` | 单证书Base64内容 |
| `NEXT_PRIVATE_SIGNING_GCLOUD_HSM_PUBLIC_CRT_FILE_PATH` | 单证书文件路径 |
| `NEXT_PRIVATE_SIGNING_GCLOUD_HSM_SECRET_MANAGER_CERT_PATH` | Secret Manager证书路径 |
| `GOOGLE_APPLICATION_CREDENTIALS` | Google认证凭据路径 |

#### 证书加载策略（仓内可证据）

```
证书链PEM (优先级最高)
    ↓
单证书PEM
    ↓
Google Secret Manager
    ↓
抛出异常
```

#### 核心特性

- **证书链支持**：完整的CA证书链嵌入，提升跨平台验证兼容性
- **无状态部署**：支持Serverless环境通过Base64编码注入凭据
- **自动凭据写入**：运行时动态创建凭据文件适应无文件系统环境

---

## 2. 签名包结构 (Signature Package Structure)

签名包结构定义了最终嵌入PDF文档的数字签名格式和附加信息。

### 2.1 签名流程概览

**入口文件**: `packages/lib/jobs/definitions/internal/seal-document.handler.ts`

```
文档完成触发
    ↓
PDF规范化 (flatten + 1.7升级)
    ↓
字段插入 (V1/V2两种模式)
    ↓
证书页面生成 (可选)
    ↓
审计日志附录 (可选)
    ↓
数字签名
    ↓
时间戳授权 (可选)
    ↓
持久化存储
```

### 2.2 证书页面结构

**实现文件**: `packages/lib/server-only/pdf/render-certificate.ts`

证书页面是一个可视化的签名凭证，包含三列布局：

| 列名 | 内容 |
|------|------|
| **签名者信息** | 姓名、邮箱、角色、认证级别（密码/2FA/账户/邮件） |
| **签名详情** | 签名图像/手写签名、Signature ID、IP地址、设备信息（UA解析） |
| **事件时间线** | 发送时间、查看时间、签名/拒绝时间、签名原因 |

#### 附加元素

- **QR验证码**：链接到在线验证页面的QR码，ecc级别Q
- **品牌标识**：Documenso品牌水印（可配置隐藏）
- **信封ID**：每页底部显示的唯一标识符

### 2.3 PDF签名规范

**实现文件**: `packages/signing/index.ts`

#### 签名算法配置

| 子过滤器 | 标准 | 说明 |
|---------|------|------|
| `ETSI.CAdES.detached` | ETSI EN 319 142 | 默认，高级电子签名，兼容Adobe Reader |
| `adbe.pkcs7.detached` | PKCS#7 | 传统格式，兼容旧版阅读器 |

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

#### 时间戳授权 (TSA)

**实现文件**: `packages/signing/helpers/tsa.ts`

- **标准**: RFC 3161 时间戳协议
- **配置**: `NEXT_PRIVATE_SIGNING_TIMESTAMP_AUTHORITY`，逗号分隔支持多个TSA服务器
- **策略**: 随机选择TSA服务器实现负载均衡和故障转移

### 2.4 字段插入机制（修正版）

支持两种字段插入版本：

| 版本 | 实现文件 | 技术方案 |
|------|---------|----------|
| V1 | `insert-field-in-pdf-v1.ts` | pdf-lib原生表单字段，支持扁平化 |
| V2 | `insert-field-in-pdf-v2.ts` | Konva画布直接渲染 → PDF覆层合并 |

**V2 渲染管道（仓内可证据 - 已修正）**:
```
字段元数据
    ↓
Konva Stage + Layer 渲染
    ↓
Skia Canvas 直接导出 PDF 覆层 (canvas.toBuffer('pdf'))
    ↓
PDF.load() 加载覆层文档
    ↓
embedPage 嵌入主文档页面
    ↓
drawPage 绘制到目标页面（处理旋转适配）
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

### 3.1 架构设计

```
┌─────────────────────────────────────────────────────────┐
│                   signPdf() 入口函数                    │
└─────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────┐
│                   Signer 工厂选择器                     │
│              NEXT_PRIVATE_SIGNING_TRANSPORT            │
└─────────────────────────────────────────────────────────┘
           ┌───────────────────┴───────────────────┐
           ↓                                       ↓
┌─────────────────────┐                 ┌─────────────────────┐
│  P12Signer (本地)   │                 │ GoogleKmsSigner     │
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
// - local.ts: return await P12Signer.create(...)
// - google-cloud.ts: return GoogleKmsSigner.create(...)
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

### 4.1 文档密封管道（11步核心流程）

```
[触发] 所有收件人签名/拒绝 → seal-document Job
  ├─ [验证] 文档完整性 + 字段签名检查
  ├─ [标识] 生成QR验证令牌（如无）
  ├─ [预取] 加载原始PDF二进制数据
  ├─ [渲染] 证书页 + 审计日志（按团队设置）
  ├─ [规范化] flatten + PDF 1.7升级 + 字段可视化嵌入
  ├─ [合并] 附加页合并到主文档
  ├─ [签名] 通过 Signer 接口执行数字签名
  ├─ [时间戳] 从TSA获取RFC 3161时间戳（如配置）
  ├─ [持久化] 上传到文件存储，更新 envelopeItem 引用
  └─ [通知] 完成邮件 + Webhook 回调
```

### 4.2 外部阅读器验证流程

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

### 4.3 关键数据流

```
原始PDF → 字段渲染覆层 → 规范化PDF → 证书/审计附加页
                                                    ↓
Signer 接口 → 签名值 → PDF签名字典 ← TSA时间戳
                                                    ↓
                                 最终签署PDF → 存储 + 分发
```

---

## 5. 安全特性

### 5.1 长期验证 (LTV)

启用TSA时自动激活LTV特性：
- 嵌入证书撤销信息 (CRL/OCSP)
- 确保证书过期后仍可验证签名有效性
- 符合eIDAS法规对高级电子签名的要求

### 5.2 防篡改机制

- 文档哈希嵌入签名中
- 任何修改都会导致签名验证失败
- 签名范围覆盖整个PDF文档（除签名字典本身）

### 5.3 不可否认性

- 私钥唯一控制在签名方
- 时间戳提供可信的签名时间证明
- 完整审计日志记录所有操作事件

---

## 6. 配置矩阵

| 配置项 | 环境变量 | 默认值 | 说明 |
|-------|---------|--------|------|
| 签名传输方式 | `NEXT_PRIVATE_SIGNING_TRANSPORT` | `local` | `local` 或 `gcloud-hsm` |
| 签名子过滤器 | `NEXT_PRIVATE_USE_LEGACY_SIGNING_SUBFILTER` | `false` | true=PKCS#7, false=CAdES |
| 时间戳服务器 | `NEXT_PRIVATE_SIGNING_TIMESTAMP_AUTHORITY` | - | 逗号分隔的TSA URL列表 |
| 平台位置 | `NEXT_PUBLIC_WEBAPP_URL` | - | 签名元数据中的位置字段 |
| 联系信息 | `NEXT_PUBLIC_SIGNING_CONTACT_INFO` | - | 签名元数据中的联系字段 |
| 包含证书页 | 团队设置 | `true` | 是否生成可视化证书页面 |
| 包含审计日志 | 团队设置 | `true` | 是否附加审计日志附录 |

---

## 总结

Documenso的加密签名系统通过三层架构实现了：

1. **证书来源层**：灵活的证书获取方式，适应从开发到企业级的各种部署场景
2. **签名包结构层**：符合国际标准的数字签名格式，配合可视化证书页面提供完整的签名证据链
3. **加密能力抽象层**：统一的Signer接口设计（基于使用模式推断），支持后端签名基础设施的无缝切换和扩展

这套机制确保了签署文档的真实性、完整性和不可否认性，同时保持了良好的跨平台PDF阅读器兼容性。

---

### v2 修正记录

1. **字段渲染链路修正**：V2字段插入流程为 Konva 画布直接导出 PDF 覆层 → embedPage 合并，无 PNG 中转环节
2. **接口契约标记化**：Signer 接口和扩展点增加了明确的推断标记，区分仓内可证据代码与合理推断
3. **闭环流程紧凑化**：新增管道式11步核心流程和数据流图，提升可读性
