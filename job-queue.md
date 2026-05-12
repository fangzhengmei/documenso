# 任务队列系统设计文档

## 概述

Documenso 任务队列系统采用**驱动抽象模式**，支持三种不同的任务执行引擎：本地自建队列、BullMQ（Redis 队列）和 Inngest（第三方托管队列）。系统通过统一的接口层隔离不同驱动的实现细节，实现无缝切换。

---

## 一、任务定义机制

### 1.1 核心数据结构

任务定义位于 `packages/lib/jobs/client/_internal/job.ts`，采用 Zod 进行类型安全验证：

```typescript
type JobDefinition<Name extends string = string, Schema = any> = {
  id: string;           // 任务唯一标识
  name: string;         // 任务显示名称
  version: string;      // 版本号（用于兼容处理）
  enabled?: boolean;    // 是否启用
  optimizeParallelism?: boolean;
  
  trigger: {
    name: Name;         // 触发器名称
    schema?: z.ZodType<Schema>;  // payload 验证 schema
    cron?: string;      // 可选 cron 表达式（定时任务）
  };
  
  // 任务处理函数
  handler: (options: { 
    payload: Schema; 
    io: JobRunIO        // 任务执行上下文 IO
  }) => Promise<Json | void>;
};
```

### 1.2 JobRunIO 执行上下文

`JobRunIO` 为任务处理提供统一的运行时环境，跨驱动保持一致：

```typescript
interface JobRunIO {
  // 执行子任务，支持幂等性（通过 cacheKey 去重）
  runTask<T>(cacheKey: string, callback: () => Promise<T>): Promise<T>;
  
  // 触发其他任务
  triggerJob(cacheKey: string, options: SimpleTriggerJobOptions): Promise<unknown>;
  
  // 等待指定时间
  wait(cacheKey: string, ms: number): Promise<void>;
  
  // 统一日志接口
  logger: {
    info(...args: unknown[]): void;
    error(...args: unknown[]): void;
    debug(...args: unknown[]): void;
    warn(...args: unknown[]): void;
    log(...args: unknown[]): void;
  };
}
```

### 1.3 任务定义示例

以发送签名邮件任务为例（`packages/lib/jobs/definitions/emails/send-signing-email.ts`）：

```typescript
const SEND_SIGNING_EMAIL_JOB_DEFINITION_ID = 'send.signing.requested.email';

const SEND_SIGNING_EMAIL_JOB_DEFINITION_SCHEMA = z.object({
  userId: z.number(),
  documentId: z.number(),
  recipientId: z.number(),
  requestMetadata: ZRequestMetadataSchema.optional(),
});

export const SEND_SIGNING_EMAIL_JOB_DEFINITION = {
  id: SEND_SIGNING_EMAIL_JOB_DEFINITION_ID,
  name: 'Send Signing Email',
  version: '1.0.0',
  trigger: {
    name: SEND_SIGNING_EMAIL_JOB_DEFINITION_ID,
    schema: SEND_SIGNING_EMAIL_JOB_DEFINITION_SCHEMA,
  },
  handler: async ({ payload, io }) => {
    const handler = await import('./send-signing-email.handler');
    await handler.run({ payload, io });
  },
} as const satisfies JobDefinition;
```

### 1.4 任务注册流程

所有任务在 `packages/lib/jobs/client.ts` 中统一注册：

```typescript
export const jobsClient = new JobClient([
  SEND_SIGNING_EMAIL_JOB_DEFINITION,
  SEND_CONFIRMATION_EMAIL_JOB_DEFINITION,
  SEAL_DOCUMENT_JOB_DEFINITION,
  // ... 其他 20+ 个任务定义
] as const);
```

---

## 二、驱动切换机制

### 2.1 抽象基类设计

位于 `packages/lib/jobs/client/base.ts`，定义所有驱动必须实现的契约：

```typescript
export abstract class BaseJobProvider {
  // 触发任务执行
  public async triggerJob(_options: SimpleTriggerJobOptions): Promise<void> {
    throw new Error('Not implemented');
  }

  // 注册任务定义
  public defineJob<N extends string, T>(_job: JobDefinition<N, T>): void {
    throw new Error('Not implemented');
  }

  // 获取 API 处理器（用于 Webhook/回调）
  public getApiHandler(): (req: HonoContext) => Promise<Response | void> {
    throw new Error('Not implemented');
  }

  // 启动 cron 调度器（对于外部调度的驱动为 No-op）
  public startCron(): void {
    // No-op by default
  }
}
```

### 2.2 驱动选择逻辑

`JobClient` 类（`packages/lib/jobs/client/client.ts`）通过环境变量动态选择驱动：

```typescript
export class JobClient<T extends ReadonlyArray<JobDefinition> = []> {
  private _provider: JobClientProvider;

  public constructor(definitions: T) {
    // 通过 ts-pattern 的 match 实现驱动分发
    this._provider = match(env('NEXT_PRIVATE_JOBS_PROVIDER'))
      .with('inngest', () => InngestJobProvider.getInstance())
      .with('bullmq', () => BullMQJobProvider.getInstance())
      .otherwise(() => LocalJobProvider.getInstance());  // 默认本地队列

    // 批量注册所有任务定义
    definitions.forEach((definition) => {
      this._provider.defineJob(definition);
    });
  }

  // 统一的任务触发接口
  public async triggerJob(options: TriggerJobOptions<T>) {
    return this._provider.triggerJob(options);
  }

  // 启动 cron 调度
  public startCron() {
    this._provider.startCron();
  }
}
```

### 2.3 三种驱动实现对比

| 特性 | LocalJobProvider（自建） | BullMQJobProvider | InngestJobProvider（第三方托管） |
|------|-------------------------|-------------------|--------------------------------|
| **存储依赖** | PostgreSQL + Prisma | Redis | Inngest 云端 |
| **调度方式** | HTTP 回调 + 30s 轮询 | Worker 进程消费 | Inngest 云端调度 |
| **Cron 实现** | 自建轮询 + 幂等 ID | BullMQ 原生 upsertJobScheduler | Inngest 原生 cron |
| **重试机制** | 数据库状态 + HTTP 重试 | 指数退避 + Redis 持久化 | Inngest 托管重试 |
| **监控面板** | 无 | Bull Board UI | Inngest Dashboard |
| **适用场景** | 开发/小规模部署 | 中大规模自托管 | 企业级/不想运维队列 |

### 2.4 驱动单例模式

所有驱动均采用单例模式确保全局唯一实例：

```typescript
// LocalJobProvider
static getInstance() {
  if (!LocalJobProvider._instance) {
    LocalJobProvider._instance = new LocalJobProvider();
  }
  return LocalJobProvider._instance;
}

// BullMQJobProvider - 使用 globalThis 跨 bundle 共享
static getInstance() {
  if (globalThis.__documenso_bullmq_provider__) {
    return globalThis.__documenso_bullmq_provider__;
  }
  const instance = new BullMQJobProvider();
  globalThis.__documenso_bullmq_provider__ = instance;
  return instance;
}
```

---

## 三、死信处理机制

### 3.1 任务状态模型

系统使用 `BackgroundJobStatus` 枚举跟踪任务生命周期：

```typescript
enum BackgroundJobStatus {
  PENDING,    // 待执行
  PROCESSING, // 执行中
  COMPLETED,  // 已完成
  FAILED,     // 失败（死信）
}
```

### 3.2 Local 驱动的死信处理流程

```typescript
// packages/lib/jobs/client/local.ts:310-346
try {
  await definition.handler({ payload, io });
  // 成功：更新为 COMPLETED
  backgroundJob = await prisma.backgroundJob.update({
    where: { id: jobId },
    data: { status: BackgroundJobStatus.COMPLETED, completedAt: new Date() },
  });
} catch (error) {
  const taskHasExceededRetries = error instanceof BackgroundTaskExceededRetriesError;
  const jobHasExceededRetries = backgroundJob.retried >= backgroundJob.maxRetries;

  if (taskHasExceededRetries || jobHasExceededRetries) {
    // 超过重试次数：标记为 FAILED（死信）
    backgroundJob = await prisma.backgroundJob.update({
      where: { id: jobId },
      data: { status: BackgroundJobStatus.FAILED, completedAt: new Date() },
    });
    return c.text('Task exceeded retries', 500);
  }

  // 未超过重试次数：重置为 PENDING，重新入队
  backgroundJob = await prisma.backgroundJob.update({
    where: { id: jobId },
    data: { status: BackgroundJobStatus.PENDING },
  });

  await this.submitJobToEndpoint({ jobId, jobDefinitionId: backgroundJob.jobId, data: options, isRetry: true });
}
```

### 3.3 BullMQ 驱动的死信处理流程

```typescript
// packages/lib/jobs/client/bullmq.ts:282-298
try {
  await definition.handler({ payload, io });
  if (backgroundJobId) {
    await prisma.backgroundJob.update({
      where: { id: backgroundJobId },
      data: { status: BackgroundJobStatus.COMPLETED, completedAt: new Date() },
    });
  }
} catch (error) {
  if (backgroundJobId) {
    const isFinalAttempt = job.attemptsMade >= DEFAULT_MAX_RETRIES - 1;
    
    // 最后一次尝试失败则标记为 FAILED，否则保持 PENDING
    await prisma.backgroundJob.update({
      where: { id: backgroundJobId },
      data: {
        status: isFinalAttempt ? BackgroundJobStatus.FAILED : BackgroundJobStatus.PENDING,
        completedAt: isFinalAttempt ? new Date() : undefined,
      },
    });
  }
  throw error; // 抛出异常让 BullMQ 处理重试队列
}
```

### 3.4 子任务（runTask）的重试机制

每个 `runTask` 内部独立维护重试计数（默认 3 次）：

```typescript
runTask: async <T>(cacheKey: string, callback: () => Promise<T>) => {
  const hashedKey = Buffer.from(sha256(cacheKey)).toString('hex');
  
  let task = await prisma.backgroundJobTask.findFirst({
    where: { id: `task-${hashedKey}--${jobId}`, jobId },
  });

  if (!task) {
    task = await prisma.backgroundJobTask.create({
      data: { id: `task-${hashedKey}--${jobId}`, name: cacheKey, jobId, status: PENDING },
    });
  }

  if (task.status === COMPLETED) {
    return task.result as T; // 幂等：已完成则直接返回结果
  }

  if (task.retried >= 3) {
    throw new BackgroundTaskExceededRetriesError('Task exceeded retries');
  }

  try {
    const result = await callback();
    await prisma.backgroundJobTask.update({
      where: { id: task.id, jobId },
      data: { status: COMPLETED, result, completedAt: new Date() },
    });
    return result;
  } catch (err) {
    await prisma.backgroundJobTask.update({
      where: { id: task.id, jobId },
      data: { status: PENDING, retried: { increment: 1 } },
    });
    throw err;
  }
}
```

### 3.5 死信的特征与处理

| 维度 | 说明 |
|------|------|
| **死信判定条件** | 1. 任务级重试超过 `maxRetries` <br> 2. 任一子任务重试超过 3 次 |
| **死信存储** | PostgreSQL `BackgroundJob` 表，`status = FAILED` |
| **保留信息** | 任务 ID、payload、重试次数、失败时间、关联子任务状态 |
| **人工干预** | 需查询数据库直接操作，无内置死信队列 UI |
| **恢复方式** | 重置任务状态为 PENDING 或重新触发同名任务 |

---

## 四、架构设计总结

### 4.1 分层架构

```
┌─────────────────────────────────────────────────────────┐
│                   应用层 (Application)                   │
│  jobsClient.triggerJob({ name: '...', payload: {...} })  │
└────────────────────────────┬────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────┐
│                   JobClient 门面层                       │
│  - 驱动分发 (ts-pattern match)                          │
│  - 任务定义批量注册                                      │
│  - 统一接口封装                                          │
└────────────────────────────┬────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────┐
│               BaseJobProvider 抽象基类                   │
│  triggerJob / defineJob / getApiHandler / startCron     │
└────────────────────┬───────────┬───────────────────────┘
                     │           │
        ┌────────────┘           └────────────┐
        │                                     │
┌───────▼────────┐   ┌───────────────┐   ┌──▼───────────┐
│  LocalProvider │   │  BullMQProvider │   │ InngestProvider │
│  (自建队列)    │   │  (Redis队列)   │   │  (第三方托管) │
└────────────────┘   └────────────────┘   └───────────────┘
```

### 4.2 关键设计决策

| 决策 | 理由 |
|------|------|
| **驱动抽象** | 支持按需切换，降低 vendor lock-in |
| **单例模式** | 避免重复初始化队列连接和调度器 |
| **Prisma + SHA256** | 实现跨实例任务幂等性（任务 ID 确定性生成） |
| **Handler 动态 import** | 减少启动时内存占用，支持懒加载 |
| **统一 JobRunIO** | 跨驱动保持任务代码一致，便于迁移 |
| **两级重试** | 任务级重试 + 子任务级重试，提供细粒度容错 |

### 4.3 文件组织结构

```
packages/lib/jobs/
├── client/
│   ├── base.ts           # 抽象基类
│   ├── local.ts          # 本地队列实现
│   ├── bullmq.ts         # BullMQ 实现
│   ├── inngest.ts        # Inngest 实现
│   ├── client.ts         # JobClient 门面
│   └── _internal/
│       └── job.ts        # 核心类型定义
├── definitions/
│   ├── emails/           # 邮件相关任务
│   └── internal/         # 内部系统任务
└── client.ts             # 任务注册与导出
```
