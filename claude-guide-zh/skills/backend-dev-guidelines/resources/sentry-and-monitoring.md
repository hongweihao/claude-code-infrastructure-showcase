# Sentry 集成和监控

使用 Sentry v8 进行错误跟踪和性能监控的完整指南。

## 目录

- [核心原则](#核心原则)
- [Sentry 初始化](#sentry-初始化)
- [错误捕获模式](#错误捕获模式)
- [性能监控](#性能监控)
- [Cron 作业监控](#cron-作业监控)
- [错误上下文最佳实践](#错误上下文最佳实践)
- [常见错误](#常见错误)

---

## 核心原则

**强制性**：所有错误必须被捕获到 Sentry。无例外。

**所有错误必须被捕获** - 使用 Sentry v8 在所有服务中进行全面的错误跟踪。

---

## Sentry 初始化

### instrument.ts 模式

**位置：** `src/instrument.ts`（必须是 server.ts 和所有 cron 作业的第一个导入）

**微服务模板：**

```typescript
import * as Sentry from '@sentry/node';
import * as fs from 'fs';
import * as path from 'path';
import * as ini from 'ini';

const sentryConfigPath = path.join(__dirname, '../sentry.ini');
const sentryConfig = ini.parse(fs.readFileSync(sentryConfigPath, 'utf-8'));

Sentry.init({
    dsn: sentryConfig.sentry?.dsn,
    environment: process.env.NODE_ENV || 'development',
    tracesSampleRate: parseFloat(sentryConfig.sentry?.tracesSampleRate || '0.1'),
    profilesSampleRate: parseFloat(sentryConfig.sentry?.profilesSampleRate || '0.1'),

    integrations: [
        ...Sentry.getDefaultIntegrations({}),
        Sentry.extraErrorDataIntegration({ depth: 5 }),
        Sentry.localVariablesIntegration(),
        Sentry.requestDataIntegration({
            include: {
                cookies: false,
                data: true,
                headers: true,
                ip: true,
                query_string: true,
                url: true,
                user: { id: true, email: true, username: true },
            },
        }),
        Sentry.consoleIntegration(),
        Sentry.contextLinesIntegration(),
        Sentry.prismaIntegration(),
    ],

    beforeSend(event, hint) {
        // 过滤健康检查
        if (event.request?.url?.includes('/healthcheck')) {
            return null;
        }

        // 清理敏感标头
        if (event.request?.headers) {
            delete event.request.headers['authorization'];
            delete event.request.headers['cookie'];
        }

        // 为 PII 掩码邮箱
        if (event.user?.email) {
            event.user.email = event.user.email.replace(/^(.{2}).*(@.*)$/, '$1***$2');
        }

        return event;
    },

    ignoreErrors: [
        /^Invalid JWT/,
        /^JWT expired/,
        'NetworkError',
    ],
});

// 设置服务上下文
Sentry.setTags({
    service: 'form',
    version: '1.0.1',
});

Sentry.setContext('runtime', {
    node_version: process.version,
    platform: process.platform,
});
```

**关键点：**
- 内置 PII 保护（beforeSend）
- 过滤非关键错误
- 全面集成
- Prisma 插桩
- 服务特定标记

---

## 错误捕获模式

### 1. BaseController 模式

```typescript
// 使用 BaseController.handleError
protected handleError(error: unknown, res: Response, context: string, statusCode = 500): void {
    Sentry.withScope((scope) => {
        scope.setTag('controller', this.constructor.name);
        scope.setTag('operation', context);
        scope.setUser({ id: res.locals?.claims?.userId });
        Sentry.captureException(error);
    });

    res.status(statusCode).json({
        success: false,
        error: { message: error instanceof Error ? error.message : 'Error occurred' }
    });
}
```

### 2. 工作流错误处理

```typescript
import { SentryHelper } from '../utils/sentryHelper';

try {
    await businessOperation();
} catch (error) {
    SentryHelper.captureOperationError(error, {
        operationType: 'POST_CREATION',
        entityId: 123,
        userId: 'user-123',
        operation: 'createPost',
    });
    throw error;
}
```

### 3. 服务层错误处理

```typescript
try {
    await someOperation();
} catch (error) {
    Sentry.captureException(error, {
        tags: {
            service: 'form',
            operation: 'someOperation'
        },
        extra: {
            userId: currentUser.id,
            entityId: 123
        }
    });
    throw error;
}
```

---

## 性能监控

### 数据库性能跟踪

```typescript
import { DatabasePerformanceMonitor } from '../utils/databasePerformance';

const result = await DatabasePerformanceMonitor.withPerformanceTracking(
    'findMany',
    'UserProfile',
    async () => {
        return await PrismaService.main.userProfile.findMany({ take: 5 });
    }
);
```

### API 端点跨度

```typescript
router.post('/operation', async (req, res) => {
    return await Sentry.startSpan({
        name: 'operation.execute',
        op: 'http.server',
        attributes: {
            'http.method': 'POST',
            'http.route': '/operation'
        }
    }, async () => {
        const result = await performOperation();
        res.json(result);
    });
});
```

---

## Cron 作业监控

### 强制模式

```typescript
#!/usr/bin/env node
import '../instrument'; // shebang 后的第一行
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
            // Cron 作业逻辑在这里
        } catch (error) {
            Sentry.captureException(error, {
                tags: {
                    'cron.job': 'job-name',
                    'error.type': 'execution_error'
                }
            });
            console.error('[Cron] Error:', error);
            process.exit(1);
        }
    });
}

main().then(() => {
    console.log('[Cron] Completed successfully');
    process.exit(0);
}).catch((error) => {
    console.error('[Cron] Fatal error:', error);
    process.exit(1);
});
```

---

## 错误上下文最佳实践

### 丰富上下文示例

```typescript
Sentry.withScope((scope) => {
    // 用户上下文
    scope.setUser({
        id: user.id,
        email: user.email,
        username: user.username
    });

    // 用于过滤的标签
    scope.setTag('service', 'form');
    scope.setTag('endpoint', req.path);
    scope.setTag('method', req.method);

    // 结构化上下文
    scope.setContext('operation', {
        type: 'workflow.complete',
        workflowId: 123,
        stepId: 456
    });

    // 时间线面包屑
    scope.addBreadcrumb({
        category: 'workflow',
        message: 'Starting step completion',
        level: 'info',
        data: { stepId: 456 }
    });

    Sentry.captureException(error);
});
```

---

## 常见错误

```typescript
// ❌ 吞下错误
try {
    await riskyOperation();
} catch (error) {
    // 静默失败
}

// ❌ 通用错误消息
throw new Error('Error occurred');

// ❌ 暴露敏感数据
Sentry.captureException(error, {
    extra: { password: user.password } // 永远不要
});

// ❌ 缺少异步错误处理
async function bad() {
    fetchData().then(data => processResult(data)); // 未处理
}

// ✅ 正确的异步处理
async function good() {
    try {
        const data = await fetchData();
        processResult(data);
    } catch (error) {
        Sentry.captureException(error);
        throw error;
    }
}
```

---

**相关文件：**
- [SKILL.md](SKILL.md)
- [routing-and-controllers.md](routing-and-controllers.md)
- [async-and-errors.md](async-and-errors.md)
