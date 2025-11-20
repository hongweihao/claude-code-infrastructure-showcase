---
name: error-tracking
description: 将 Sentry v8 错误跟踪和性能监控添加到项目服务。在添加错误处理、创建新控制器、检测 cron 作业或跟踪数据库性能时使用此技能。所有错误必须捕获到 Sentry - 没有例外。
---

# 项目 Sentry 集成技能

## 目的
此技能遵循 Sentry v8 模式，在所有项目服务中强制执行全面的 Sentry 错误跟踪和性能监控。

## 何时使用此技能
- 向任何代码添加错误处理
- 创建新控制器或路由
- 检测 cron 作业
- 跟踪数据库性能
- 添加性能 spans
- 处理工作流错误

## 🚨 关键规则

**所有错误必须捕获到 Sentry** - 没有例外。永远不要单独使用 console.error。

## 当前状态

### Form Service ✅ 完成
- Sentry v8 完全集成
- 跟踪所有工作流错误
- SystemActionQueueProcessor 已检测
- 测试端点可用

### Email Service 🟡 进行中
- 阶段 1-2 完成（6/22 任务）
- 剩余 189 个 ErrorLogger.log() 调用

## Sentry 集成模式

### 1. 控制器错误处理

```typescript
// ✅ 正确 - 使用 BaseController
import { BaseController } from '../controllers/BaseController';

export class MyController extends BaseController {
    async myMethod() {
        try {
            // ... 您的代码
        } catch (error) {
            this.handleError(error, 'myMethod'); // 自动发送到 Sentry
        }
    }
}
```

### 2. 路由错误处理（不使用 BaseController）

```typescript
import * as Sentry from '@sentry/node';

router.get('/route', async (req, res) => {
    try {
        // ... 您的代码
    } catch (error) {
        Sentry.captureException(error, {
            tags: { route: '/route', method: 'GET' },
            extra: { userId: req.user?.id }
        });
        res.status(500).json({ error: 'Internal server error' });
    }
});
```

### 3. 工作流错误处理

```typescript
import { WorkflowSentryHelper } from '../workflow/utils/sentryHelper';

// ✅ 正确 - 使用 WorkflowSentryHelper
WorkflowSentryHelper.captureWorkflowError(error, {
    workflowCode: 'DHS_CLOSEOUT',
    instanceId: 123,
    stepId: 456,
    userId: 'user-123',
    operation: 'stepCompletion',
    metadata: { additionalInfo: 'value' }
});
```

### 4. Cron 作业（强制模式）

```typescript
#!/usr/bin/env node
// shebang 后的第一行 - 关键！
import '../instrument';
import * as Sentry from '@sentry/node';

async function main() {
    return await Sentry.startSpan({
        name: 'cron.job-name',
        op: 'cron',
        attributes: {
            'cron.job': 'job-name',
            'cron.startTime': new Date().toISOString(),
        }
    }, async () => {
        try {
            // 您的 cron 作业逻辑
        } catch (error) {
            Sentry.captureException(error, {
                tags: {
                    'cron.job': 'job-name',
                    'error.type': 'execution_error'
                }
            });
            console.error('[Job] Error:', error);
            process.exit(1);
        }
    });
}

main()
    .then(() => {
        console.log('[Job] Completed successfully');
        process.exit(0);
    })
    .catch((error) => {
        console.error('[Job] Fatal error:', error);
        process.exit(1);
    });
```

### 5. 数据库性能监控

```typescript
import { DatabasePerformanceMonitor } from '../utils/databasePerformance';

// ✅ 正确 - 包装数据库操作
const result = await DatabasePerformanceMonitor.withPerformanceTracking(
    'findMany',
    'UserProfile',
    async () => {
        return await PrismaService.main.userProfile.findMany({
            take: 5,
        });
    }
);
```

### 6. 带 Spans 的异步操作

```typescript
import * as Sentry from '@sentry/node';

const result = await Sentry.startSpan({
    name: 'operation.name',
    op: 'operation.type',
    attributes: {
        'custom.attribute': 'value'
    }
}, async () => {
    // 您的异步操作
    return await someAsyncOperation();
});
```

## 错误级别

使用适当的严重性级别：

- **fatal**：系统不可用（数据库宕机、关键服务故障）
- **error**：操作失败，需要立即关注
- **warning**：可恢复的问题、性能下降
- **info**：信息消息、成功的操作
- **debug**：详细的调试信息（仅开发）

## 必需的上下文

```typescript
import * as Sentry from '@sentry/node';

Sentry.withScope((scope) => {
    // 如果可用，始终包括这些
    scope.setUser({ id: userId });
    scope.setTag('service', 'form'); // 或 'email'、'users' 等
    scope.setTag('environment', process.env.NODE_ENV);

    // 添加特定于操作的上下文
    scope.setContext('operation', {
        type: 'workflow.start',
        workflowCode: 'DHS_CLOSEOUT',
        entityId: 123
    });

    Sentry.captureException(error);
});
```

## 服务特定集成

### Form Service

**位置**：`./blog-api/src/instrument.ts`

```typescript
import * as Sentry from '@sentry/node';
import { nodeProfilingIntegration } from '@sentry/profiling-node';

Sentry.init({
    dsn: process.env.SENTRY_DSN,
    environment: process.env.NODE_ENV || 'development',
    integrations: [
        nodeProfilingIntegration(),
    ],
    tracesSampleRate: 0.1,
    profilesSampleRate: 0.1,
});
```

**关键助手**：
- `WorkflowSentryHelper` - 工作流特定错误
- `DatabasePerformanceMonitor` - 数据库查询跟踪
- `BaseController` - 控制器错误处理

### Email Service

**位置**：`./notifications/src/instrument.ts`

```typescript
import * as Sentry from '@sentry/node';
import { nodeProfilingIntegration } from '@sentry/profiling-node';

Sentry.init({
    dsn: process.env.SENTRY_DSN,
    environment: process.env.NODE_ENV || 'development',
    integrations: [
        nodeProfilingIntegration(),
    ],
    tracesSampleRate: 0.1,
    profilesSampleRate: 0.1,
});
```

**关键助手**：
- `EmailSentryHelper` - 邮件特定错误
- `BaseController` - 控制器错误处理

## 配置 (config.ini)

```ini
[sentry]
dsn = your-sentry-dsn
environment = development
tracesSampleRate = 0.1
profilesSampleRate = 0.1

[databaseMonitoring]
enableDbTracing = true
slowQueryThreshold = 100
logDbQueries = false
dbErrorCapture = true
enableN1Detection = true
```

## 测试 Sentry 集成

### Form Service 测试端点

```bash
# 测试基本错误捕获
curl http://localhost:3002/blog-api/api/sentry/test-error

# 测试工作流错误
curl http://localhost:3002/blog-api/api/sentry/test-workflow-error

# 测试数据库性能
curl http://localhost:3002/blog-api/api/sentry/test-database-performance

# 测试错误边界
curl http://localhost:3002/blog-api/api/sentry/test-error-boundary
```

### Email Service 测试端点

```bash
# 测试基本错误捕获
curl http://localhost:3003/notifications/api/sentry/test-error

# 测试邮件特定错误
curl http://localhost:3003/notifications/api/sentry/test-email-error

# 测试性能跟踪
curl http://localhost:3003/notifications/api/sentry/test-performance
```

## 性能监控

### 要求

1. **所有 API 端点**必须有事务跟踪
2. **数据库查询 > 100ms** 自动标记
3. **N+1 查询**被检测并报告
4. **Cron 作业**必须跟踪执行时间

### 事务跟踪

```typescript
import * as Sentry from '@sentry/node';

// Express 路由的自动事务跟踪
app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.tracingHandler());

// 自定义操作的手动事务
const transaction = Sentry.startTransaction({
    op: 'operation.type',
    name: 'Operation Name',
});

try {
    // 您的操作
} finally {
    transaction.finish();
}
```

## 要避免的常见错误

❌ **永远不要**在没有 Sentry 的情况下使用 console.error
❌ **永远不要**静默吞掉错误
❌ **永远不要**在错误上下文中暴露敏感数据
❌ **永远不要**使用没有上下文的通用错误消息
❌ **永远不要**跳过异步操作中的错误处理
❌ **永远不要**忘记在 cron 作业中将 instrument.ts 作为第一行导入

## 实现检查清单

向新代码添加 Sentry 时：

- [ ] 已导入 Sentry 或适当的助手
- [ ] 所有 try/catch 块都捕获到 Sentry
- [ ] 向错误添加了有意义的上下文
- [ ] 使用了适当的错误级别
- [ ] 错误消息中没有敏感数据
- [ ] 为慢操作添加了性能跟踪
- [ ] 测试了错误处理路径
- [ ] 对于 cron 作业：首先导入 instrument.ts

## 关键文件

### Form Service
- `/blog-api/src/instrument.ts` - Sentry 初始化
- `/blog-api/src/workflow/utils/sentryHelper.ts` - 工作流错误
- `/blog-api/src/utils/databasePerformance.ts` - 数据库监控
- `/blog-api/src/controllers/BaseController.ts` - 控制器基类

### Email Service
- `/notifications/src/instrument.ts` - Sentry 初始化
- `/notifications/src/utils/EmailSentryHelper.ts` - 邮件错误
- `/notifications/src/controllers/BaseController.ts` - 控制器基类

### 配置
- `/blog-api/config.ini` - Form 服务配置
- `/notifications/config.ini` - Email 服务配置
- `/sentry.ini` - 共享 Sentry 配置

## 文档

- 完整实现：`/dev/active/email-sentry-integration/`
- Form 服务文档：`/blog-api/docs/sentry-integration.md`
- Email 服务文档：`/notifications/docs/sentry-integration.md`

## 相关技能

- 在数据库操作之前使用 **database-verification**
- 使用 **workflow-builder** 获取工作流错误上下文
- 使用 **database-scripts** 进行数据库错误处理
