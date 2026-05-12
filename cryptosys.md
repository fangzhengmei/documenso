# Documenso 数字证书签名加密系统架构

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

#### 核心实现

```typescript
const loadP12 = (): Uint8Array => {
  // 1. 尝试从环境变量加载Base64编码内容
  // 2. 尝试从文件路径加载
  // 3. 开发环境使用默认证书
  // 4. 生产环境抛出异常
};

export const createLocalSigner = async () => {
  const p12 = loadP12();
  return await P12Signer.create(p12, passphrase, {
    buildChain: true,  // 自动构建证书链
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

#### 证书加载策略

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

#### 签名元数据

```typescript
{
  reason: 'Signed by Documenso',
  location: NEXT_PUBLIC_WEBAPP_URL(),     // 平台域名
  contactInfo: NEXT_PUBLIC_SIGNING_CONTACT_INFO(),
  subFilter: 'ETSI.CAdES.detached',       // 或 adbe.pkcs7.detached
  timestampAuthority: tsa,                // 可选时间戳
  longTermValidation: !!tsa,              // 启用LTV
  archivalTimestamp: !!tsa,               // 归档时间戳
}
```

#### 时间戳授权 (TSA)

**实现文件**: `packages/signing/helpers/tsa.ts`

- **标准**: RFC 3161 时间戳协议
- **配置**: `NEXT_PRIVATE_SIGNING_TIMESTAMP_AUTHORITY`，逗号分隔支持多个TSA服务器
- **策略**: 随机选择TSA服务器实现负载均衡和故障转移

### 2.4 字段插入机制

支持两种字段插入版本：

| 版本 | 实现文件 | 技术方案 |
|------|---------|----------|
| V1 | `insert-field-in-pdf-v1.ts` | pdf-lib原生表单字段，支持扁平化 |
| V2 | `insert-field-in-pdf-v2.ts` | Konva画布渲染 → 嵌入为PDF页面层 |

**V2 渲染管道**:
```
字段元数据
    ↓
Konva Stage 渲染 (Canvas/Skia后端)
    ↓
PNG导出
    ↓
PDF页面嵌入
    ↓
旋转适配原始页面方向
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

### 3.2 核心接口定义

**入口文件**: `packages/signing/index.ts`

```typescript
// 签名器工厂 - 单例模式
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

// 对外统一签名接口
export const signPdf = async ({ pdf }: SignOptions) => {
  const signer = await getSigner();
  const tsa = getTimestampAuthority();

  const { bytes } = await pdf.sign({
    signer,
    reason: 'Signed by Documenso',
    location: NEXT_PUBLIC_WEBAPP_URL(),
    contactInfo: NEXT_PUBLIC_SIGNING_CONTACT_INFO(),
    subFilter: NEXT_PRIVATE_USE_LEGACY_SIGNING_SUBFILTER() 
      ? 'adbe.pkcs7.detached' 
      : 'ETSI.CAdES.detached',
    timestampAuthority: tsa ?? undefined,
    longTermValidation: !!tsa,
    archivalTimestamp: !!tsa,
  });

  return bytes;
};
```

### 3.3 Signer 接口契约

所有签名后端必须实现统一的 `Signer` 接口：

```typescript
interface Signer {
  // 签名核心方法
  sign(digest: Uint8Array): Promise<Uint8Array>;
  
  // 获取签名证书
  getCertificate(): Uint8Array;
  
  // 获取证书链（可选）
  getCertificateChain?(): Uint8Array[];
  
  // 签名算法标识
  getSignatureAlgorithm(): string;
}
```

### 3.4 传输层扩展点

系统设计支持添加新的签名传输层，只需：

1. 在 `packages/signing/transports/` 下创建新的实现文件
2. 导出 `createXxxSigner()` 工厂函数
3. 在 `index.ts` 的 `match` 表达式中添加新的分支

**潜在扩展方向**:
- AWS KMS 集成
- Azure Key Vault 集成
- HashiCorp Vault 集成
- 自建PKI服务集成

---

## 4. 完整闭环流程

### 4.1 文档密封流程

```
触发: 所有收件人完成签名/拒绝
    ↓
1. 状态验证: 检查文档完整性和所有字段签名
    ↓
2. 生成QR令牌: 为在线验证创建唯一标识
    ↓
3. 预取PDF数据: 加载原始文档二进制
    ↓
4. 证书页面生成: 签名者信息表格渲染
    ↓
5. 审计日志生成: 完整事件链附录
    ↓
6. PDF规范化:
   - 扁平化所有注释和表单
   - 升级到PDF 1.7规范
   - 插入签名字段可视化
    ↓
7. 附加页面合并: 证书页 + 审计日志页
    ↓
8. 数字签名: 通过Signer接口执行签名
    ↓
9. 时间戳: 从TSA服务器获取RFC 3161时间戳
    ↓
10. 持久化: 上传到文件存储系统
    ↓
11. 通知: 发送完成邮件 + Webhook回调
```

### 4.2 外部验证流程

当用户在Adobe Reader等PDF阅读器中打开签名文档时：

```
1. 阅读器检测PDF签名字典
    ↓
2. 提取SignerInfo和证书链
    ↓
3. 验证文档摘要完整性
    ↓
4. 验证证书链信任锚
    ↓
5. 检查CRL/OCSP撤销状态 (LTV启用时)
    ↓
6. 验证时间戳 token (TSA启用时)
    ↓
7. 显示签名验证结果给用户
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
3. **加密能力抽象层**：统一的Signer接口设计，支持后端签名基础设施的无缝切换和扩展

这套机制确保了签署文档的真实性、完整性和不可否认性，同时保持了良好的跨平台PDF阅读器兼容性。
