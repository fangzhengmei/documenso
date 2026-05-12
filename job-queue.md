# 任务队列系统设计文档

## 概述

Documenso 任务队列系统采用**驱动抽象模式**，支持三种不同的任务执行引擎：LocalJobProvider（本地自建队列）、BullMQJobProvider（Redis 队列）和 InngestJobProvider（第三方托管队列）。系统通过 `BaseJobProvider` 抽象基类和 `JobClient` 门面隔离不同驱动的实现细节，实现环境变量驱动的无缝切换。

---

## 一、任务定义机制

### 1.1 核心数据结构

任务定义位于 `packages/lib/jobs/client/_internal/job.ts`，采用 Zod 进行类型安全验证：

```typescript
// 代码证据: packages/lib/jobs/client/_internal/job.ts:15-34
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

`JobRunIO` 为任务处理提供统一的运行时环境，跨驱动保持接口签名一致：

```typescript
// 代码证据: packages/lib/jobs/client/_internal/job.ts:42-60
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

以发送签名邮件任务为例：

```typescript
// 代码证据: packages/lib/jobs/definitions/emails/send-signing-email.ts:17-29
export const SEND_SIGNING_EMAIL_JOB_DEFINITION = {
  id: 'send.signing.requested.email',
  name: 'Send Signing Email',
  version: '1.0.0',
  trigger: {
    name: 'send.signing.requested.email',
    schema: z.object({
      userId: z.number(),
      documentId: z.number(),
      recipientId: z.number(),
      requestMetadata: ZRequestMetadataSchema.optional(),
    }),
  },
  handler: async ({ payload, io }) => {
    const handler = await import('./send-signing-email.handler');
    await handler.run({ payload, io });
  },
} as const satisfies JobDefinition;
```

### 1.4 任务注册流程

所有任务在 `packages/lib/jobs/client.ts` 中批量注册到 `JobClient`：

```typescript
// 代码证据: packages/lib/jobs/client.ts:29-52
export const jobsClient = new JobClient([
  SEND_SIGNING_EMAIL_JOB_DEFINITION,
  SEND_CONFIRMATION_EMAIL_JOB_DEFINITION,
  SEAL_DOCUMENT_JOB_DEFINITION,
  SEAL_DOCUMENT_SWEEP_JOB_DEFINITION,
  // ... 共计 20+ 个任务定义
] as const);
```

---

## 二、驱动切换机制

### 2.1 抽象基类设计

`BaseJobProvider` 位于 `packages/lib/jobs/client/base.ts`，定义所有驱动必须实现的契约：

```typescript
// 代码证据: packages/lib/jobs/client/base.ts:5-28
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
// 代码证据: packages/lib/jobs/client/client.ts:10-41
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

### 2.3 三种驱动实现对比（代码可证部分）

| 特性 | LocalJobProvider | BullMQJobProvider | InngestJobProvider |
|------|-----------------|-------------------|--------------------|
| **实现文件** | `packages/lib/jobs/client/local.ts` | `packages/lib/jobs/client/bullmq.ts` | `packages/lib/jobs/client/inngest.ts` |
| **单例实现** | `static _instance` 私有静态变量 | `globalThis.__documenso_bullmq_provider__` | `static _instance` 私有静态变量 |
| **外部依赖** | `@prisma/client` | `bullmq`, `ioredis`, `@bull-board/api` | `inngest` |
| **triggerJob 落库** | ✅ 调用 `prisma.backgroundJob.create()` | ✅ 调用 `prisma.backgroundJob.create()` | ❌ 代码中无 prisma 调用 |
| **cron 调度** | ✅ 30秒轮询 + SHA256 幂等 ID | ✅ 调用 `queue.upsertJobScheduler()` | ✅ Inngest createFunction 触发配置 |
| **监控 UI** | ❌ 无相关代码 | ✅ Bull Board Hono 路由 | ❌ 无 UI 相关代码 |

---

## 三、死信处理机制

### 3.1 任务状态模型

系统使用 `BackgroundJobStatus` 枚举跟踪任务生命周期：

```typescript
// 代码证据: @prisma/client 类型，见各驱动中引用
enum BackgroundJobStatus {
  PENDING,    // 待执行
  PROCESSING, // 执行中
  COMPLETED,  // 已完成
  FAILED,     // 失败（死信）
}
```

### 3.2 Local 驱动的死信处理流程

```typescript
// 代码证据: packages/lib/jobs/client/local.ts:273-347
try {
  await definition.handler({ payload, io });
  // 成功：更新为 COMPLETED
  backgroundJob = await prisma.backgroundJob.update({
    where: { id: jobId },
    data: { status: BackgroundJobStatus.COMPLETED, completedAt: new Date() },
  });
} catch (error) {
  console.log(`[JOBS]: Job ${options.name} failed`, error);

  // 子任务超限：直接判定为死信
  const taskHasExceededRetries = error instanceof BackgroundTaskExceededRetriesError;
  
  // 任务级重试超限 AND 错误不是子任务普通失败（即非 runTask 抛出的 BackgroundTaskFailedError）
  // 注意: !(error instanceof BackgroundTaskFailedError) 这个分支意味着
  // runTask 抛出的普通子任务失败不会触发任务级失败，而是继续重试
  const jobHasExceededRetries =
    backgroundJob.retried >= backgroundJob.maxRetries && 
    !(error instanceof BackgroundTaskFailedError);

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

**代码可证结论：**
1. Local 驱动在 HTTP handler 中 catch 所有异常
2. 两阶段判定逻辑：
   - **子任务超限判定**：如果错误是 `BackgroundTaskExceededRetriesError`，直接标记失败
   - **任务级超限判定**：只有当重试次数达到 `maxRetries` **AND** 错误不是 `BackgroundTaskFailedError` 时才标记失败
3. 关键逻辑 `!(error instanceof BackgroundTaskFailedError)` 意味着：runTask 子任务的普通失败（`BackgroundTaskFailedError`）不会触发任务级死信，而是继续重试
4. 最后一次失败更新状态为 `FAILED`，否则重置为 `PENDING` 并重新调用 HTTP 回调
5. 重试计数存储在 PostgreSQL `BackgroundJob.retried` 字段

### 3.3 BullMQ 驱动的死信处理流程

```typescript
// 代码证据: packages/lib/jobs/client/bullmq.ts:265-298
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
    const isFinalAttempt = job.attemptsMade >= (job.opts.attempts ?? DEFAULT_MAX_RETRIES) - 1;
    
    // 最后一次尝试失败则标记为 FAILED，否则保持 PENDING
    await prisma.backgroundJob.update({
      where: { id: backgroundJobId },
      data: {
        status: isFinalAttempt ? BackgroundJobStatus.FAILED : BackgroundJobStatus.PENDING,
        completedAt: isFinalAttempt ? new Date() : undefined,
      },
    }).catch(() => null);
  }
  throw error; // 抛出异常让 BullMQ 处理重试队列
}
```

**代码可证结论：**
1. BullMQ 驱动 catch 异常但最终重新 `throw error`
2. 使用 BullMQ 原生的 `job.attemptsMade` 和 `job.opts.attempts` 判断是否为最后一次尝试
3. 最后一次失败更新数据库状态为 `FAILED`，否则保持 `PENDING`
4. 重试机制由 BullMQ 本身处理，代码中无显式重试逻辑

### 3.4 Inngest 驱动的失败处理流程

```typescript
// 代码证据: packages/lib/jobs/client/inngest.ts:41-61
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

    await job.handler({ payload, io });  // 无 try-catch，异常直接抛出给 Inngest
  },
);
```

**代码可证结论：**
1. Inngest 驱动代码中**无 try-catch** 包裹 handler 调用
2. 异常直接抛出给 Inngest SDK，由其接管后续处理
3. 代码中**无任何 prisma.backgroundJob 调用**，任务状态不落本地数据库
4. 重试、死信、归档等逻辑完全不在仓内代码控制范围内

### 3.5 子任务（runTask）的重试机制

Local 与 BullMQ 驱动的 `runTask` 实现完全一致（代码相同）：

```typescript
// 代码证据: packages/lib/jobs/client/bullmq.ts:303-379 (local.ts 类似)
runTask: async <T extends void | Json>(cacheKey: string, callback: () => Promise<T>) => {
  const hashedKey = Buffer.from(sha256(cacheKey)).toString('hex');

  let task = await prisma.backgroundJobTask.findFirst({
    where: {
      id: `task-${hashedKey}--${jobId}`,
      jobId,
    },
  });

  if (!task) {
    task = await prisma.backgroundJobTask.create({
      data: {
        id: `task-${hashedKey}--${jobId}`,
        name: cacheKey,
        jobId,
        status: BackgroundJobStatus.PENDING,
      },
    });
  }

  if (task.status === BackgroundJobStatus.COMPLETED) {
    return task.result as T; // 幂等：已完成则直接返回结果
  }

  if (task.retried >= 3) {
    throw new Error('Task exceeded retries'); // Local 驱动抛出自定义异常类
  }

  try {
    const result = await callback();
    await prisma.backgroundJobTask.update({
      where: { id: task.id, jobId },
      data: { status: BackgroundJobStatus.COMPLETED, result, completedAt: new Date() },
    });
    return result;
  } catch (err) {
    await prisma.backgroundJobTask.update({
      where: { id: task.id, jobId },
      data: { status: BackgroundJobStatus.PENDING, retried: { increment: 1 } },
    });
    throw err;
  }
}
```

**代码可证结论：**
1. 子任务 ID 通过 `sha256(cacheKey) + jobId` 确定性生成，保证幂等
2. 子任务状态完整落库到 `backgroundJobTask` 表
3. 最大重试次数硬编码为 3 次
4. Local 驱动抛出 `BackgroundTaskExceededRetriesError` 自定义异常，BullMQ 抛出普通 `Error`

### 3.6 JobRunIO 三种驱动实现差异

虽然接口签名完全一致，但三种驱动的 `JobRunIO` 内部实现存在本质差异：

| JobRunIO 方法 | Local / BullMQ 实现（代码可证） | Inngest 实现（代码可证） |
|--------------|--------------------------------|--------------------------|
| **`runTask`** | ```typescript// 本地数据库幂等实现// SHA256 + jobId 生成确定性 task ID// 查询 backgroundJobTask 表// 状态机: PENDING → COMPLETED / FAILED// 重试计数存在数据库，硬编码 3 次// 代码证据: bullmq.ts:303-379``` | ```typescript// Inngest Step 托管await step.run(cacheKey, callback);// 无数据库操作// 代码证据: inngest.ts:98-103``` |
| **`triggerJob`** | ```typescript// 直接调用当前 provider 的 triggerJob// 立即创建 BackgroundJob 数据库记录// 代码证据: bullmq.ts:368``` | ```typescript// 调用 Inngest SDK sendEventawait step.sendEvent(cacheKey, payload);// 无数据库操作// 代码证据: inngest.ts:105-109``` |
| **`wait`** | ```typescript// ❌ 未实现，直接抛出错误throw new Error('Not implemented');// 代码证据: bullmq.ts:377-379``` | ```typescript// ✅ Inngest 原生支持await step.sleep(ms);// 代码证据: inngest.ts:90``` |
| **`logger`** | ```typescript// Node.js console 直接输出{ info: console.info, debug: console.debug, error: console.error, warn: console.warn, log: console.log }// 代码证据: bullmq.ts:369-375``` | ```typescript// Inngest ctx 提供的 Logger{ info: ctx.logger.info, debug: ctx.logger.debug, error: ctx.logger.error, warn: ctx.logger.warn, log: ctx.logger.info }// 代码证据: inngest.ts:91-97``` |

---

## 四、代码证据边界说明

### 4.1 已确认的代码证据

| 结论 | 代码位置 | 确认程度 |
|------|---------|---------|
| Local/BullMQ 触发任务时创建 BackgroundJob 记录 | `local.ts:200`, `bullmq.ts:139` | ✅ 100% 确认 |
| Inngest 驱动不创建 BackgroundJob 记录 | `inngest.ts` 全文无 prisma 调用 | ✅ 100% 确认 |
| Local 驱动重试逻辑在本地代码中实现 | `local.ts:310-347` | ✅ 100% 确认 |
| BullMQ 重试依赖 throw error + BullMQ 原生机制 | `bullmq.ts:297` | ✅ 100% 确认 |
| runTask 最大重试次数硬编码为 3 次 | `bullmq.ts:329`, `local.ts:416` | ✅ 100% 确认 |
| Local/BullMQ 的 wait 方法未实现抛出错误 | `bullmq.ts:377`, `local.ts:463` | ✅ 100% 确认 |
| BullMQ 最大重试次数 DEFAULT_MAX_RETRIES = 3 | `bullmq.ts:24` | ✅ 100% 确认 |

### 4.2 需平台文档确认的内容

以下内容**无法**从仓内代码直接推导，需查阅对应平台官方文档：

1. **Inngest 重试策略**：重试次数、重试间隔、退避算法、死信处理逻辑
2. **BullMQ 死信队列**：仓内代码未配置死信队列，是否自动创建需 BullMQ 文档确认
3. **BullMQ 指数退避参数**：`backoff: { type: 'exponential', delay: 1000 }` 的具体行为需 BullMQ 文档确认
4. **Inngest 数据保留**：任务历史、日志、Trace 的保留策略需 Inngest 文档确认
5. **Inngest 断点恢复**：Step 级别的 checkpoint 和恢复机制细节需 Inngest 文档确认
6. **BullMQ Worker 失败回调**：`_worker.on('failed')` 仅打日志，无其他处理逻辑可证，但后续行为需 BullMQ 文档确认

### 4.3 代码中未体现的实现

以下内容在当前仓内代码中**不存在**，请勿假设已实现：

1. ❌ BullMQ 死信队列配置（无 `deadLetterQueue` 相关代码）
2. ❌ Local 驱动的重试延迟机制（立即重试无间隔）
3. ❌ 失败任务的自动告警或通知
4. ❌ 死信队列的专门管理 UI
5. ❌ Inngest 的失败回调或 Webhook 配置
