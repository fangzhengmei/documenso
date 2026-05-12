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

### 3.5 Inngest 托管队列的失败处理机制

#### 3.5.1 执行流程与状态管理

```typescript
// packages/lib/jobs/client/inngest.ts:48-61
const fn = this._client.createFunction(
  {
    id: job.id,
    name: job.name,
    optimizeParallelism: job.optimizeParallelism ?? false,
  },
  triggerConfig,
  async (ctx) => {
    const io = this.convertInngestIoToJobRunIo(ctx);
    
    let payload = ctx.event.data as any;
    if (job.trigger.schema) {
      payload = job.trigger.schema.parse(payload);
    }

    await job.handler({ payload, io });  // 异常直接抛出给 Inngest
  },
);
```

**关键特征**：
- **无本地状态管理**：Inngest 驱动不创建 `BackgroundJob` 数据库记录，完全依赖 Inngest 云端状态
- **异常透传**：Handler 抛出的异常直接由 Inngest 平台接管，不经过本地重试逻辑
- **云端持久化**：任务事件、执行历史、重试记录全部存储在 Inngest 云端

---

### 3.6 三种驱动失败处理对比表

| 对比维度 | LocalJobProvider | BullMQJobProvider | InngestJobProvider |
|---------|-----------------|-------------------|--------------------|
| **重试决策方** | 本地代码逻辑判断 | BullMQ Worker + 本地代码 | Inngest 云端引擎 |
| **重试配置** | `BackgroundJob.maxRetries` (数据库) | `DEFAULT_MAX_RETRIES = 3` (硬编码) | Inngest Function 配置 |
| **重试间隔** | 立即重试 (HTTP 回调) | 指数退避 (Redis 延迟队列) | Inngest 托管退避策略 |
| **死信判定** | 本地 catch 后更新数据库 | 本地 catch + BullMQ attempts 计数 | Inngest 云端自动判定 |
| **失败后状态** | `BackgroundJobStatus.FAILED` | `BackgroundJobStatus.FAILED` + Redis 死信队列 | Inngest Failed Run |
| **状态落库** | ✅ 完整落库 (PENDING → PROCESSING → COMPLETED/FAILED) | ✅ 完整落库 (同 Local) | ❌ 不落库，仅 Inngest 云端 |
| **子任务重试** | ✅ 本地数据库 `backgroundJobTask` 表管理 | ✅ 本地数据库管理 (同 Local) | ✅ Inngest `step.run` 托管 |
| **子任务幂等** | ✅ SHA256 生成确定性 task ID | ✅ SHA256 生成确定性 task ID | ✅ Inngest Step ID 机制 |
| **失败日志存储** | 本地 console + 数据库 | 本地 console + 数据库 + Redis | Inngest 云端 Dashboard |
| **死信可视化** | ❌ 无 UI，需查数据库 | ✅ Bull Board UI | ✅ Inngest Dashboard |
| **手动重入** | 重置数据库状态为 PENDING | 重置数据库 + BullMQ 重入队 | Inngest Dashboard 点击重试 |

---

### 3.7 失败归档差异分析

#### Local 驱动归档
```
归档位置：PostgreSQL `BackgroundJob` 表
归档内容：
  - jobId / name / version
  - payload JSON
  - retried 计数
  - lastRetriedAt 时间戳
  - completedAt 失败时间
  - 关联 backgroundJobTask 子任务记录
查询方式：直接 SQL 查询
保留策略：无限期，需手动清理
```

#### BullMQ 驱动归档
```
双归档模式：
1. PostgreSQL：同 Local 驱动完整记录
2. Redis：BullMQ 原生死信队列 (dead letter queue)
   - Job 完整数据 (name, data, opts)
   - attemptsMade / failedReason
   - stacktrace 快照
查询方式：Bull Board UI + 数据库查询
保留策略：Redis 按配置过期，数据库无限期
```

#### Inngest 驱动归档
```
归档位置：Inngest 云端 (AWS/GCP 存储)
归档内容：
  - 完整 Event 数据
  - Function 执行 Trace
  - 每一步 Step 的输入输出
  - 异常栈追踪
  - 重试历史时间线
查询方式：Inngest Dashboard / API
保留策略：按 Inngest 订阅计划 (默认 30 天)
```

---

### 3.8 JobRunIO 三种驱动实现差异

虽然接口签名完全一致，但三种驱动的 `JobRunIO` 内部实现存在本质差异：

| JobRunIO 方法 | Local / BullMQ 实现 | Inngest 实现 |
|--------------|---------------------|-------------|
| **`runTask`** | ```typescript// 本地数据库幂等实现const hashedKey = sha256(cacheKey);const taskId = `task-${hashedKey}--${jobId}`;// 查询 backgroundJobTask 表// 状态机: PENDING → COMPLETED / FAILED// 重试计数存在数据库``` | ```typescript// Inngest Step 托管await step.run(cacheKey, callback);// 幂等性由 Inngest 保证// Step 状态存在 Inngest 云端// 自动 checkpoint，失败从断点恢复``` |
| **`triggerJob`** | ```typescript// 直接调用当前 provider 的 triggerJobawait this._provider.triggerJob(payload);// 立即创建 BackgroundJob 记录``` | ```typescript// 调用 Inngest SDK sendEventawait step.sendEvent(cacheKey, payload);// 事件异步持久化到 Inngest// 不创建本地数据库记录``` |
| **`wait`** | ```typescript// ❌ 未实现，直接抛出错误throw new Error('Not implemented');// 本地队列不支持等待原语``` | ```typescript// ✅ Inngest 原生支持await step.sleep(ms);// 精确时间控制，由云端调度// 不占用进程资源``` |
| **`logger`** | ```typescript// Node.js console 输出{  info: console.info,  debug: console.debug,  error: console.error,  warn: console.warn,  log: console.log}// 日志仅本地可见``` | ```typescript// Inngest 采集的 Logger{  info: ctx.logger.info,  debug: ctx.logger.debug,  error: ctx.logger.error,  warn: ctx.logger.warn,  log: ctx.logger.info}// 日志同步到云端 Dashboard``` |

#### 差异总结

| 特性 | Local / BullMQ | Inngest |
|------|---------------|---------|
| **状态存储** | 本地 PostgreSQL | Inngest 云端 |
| **断点恢复** | 子任务级别 (数据库) | Step 级别 (云端 checkpoint) |
| **等待原语** | ❌ 不支持 | ✅ 原生 `step.sleep` |
| **可观测性** | 本地日志 + 数据库 | 云端 Dashboard + 完整 Trace |
| **执行原子性** | 任务整体重试 | Step 粒度重试 |
| **网络依赖** | 数据库连接 | 与 Inngest 的 HTTPS 连接 |

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
