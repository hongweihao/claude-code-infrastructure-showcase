# 中间件指南 - Express 中间件模式

在后端微服务中创建和使用中间件的完整指南。

## 目录

- [认证中间件](#认证中间件)
- [使用 AsyncLocalStorage 的审计中间件](#使用-asynclocalstorage-的审计中间件)
- [错误边界中间件](#错误边界中间件)
- [验证中间件](#验证中间件)
- [可组合中间件](#可组合中间件)
- [中间件顺序](#中间件顺序)

---

## 认证中间件

### SSOMiddleware 模式

**文件：** `/form/src/middleware/SSOMiddleware.ts`

```typescript
export class SSOMiddlewareClient {
    static verifyLoginStatus(req: Request, res: Response, next: NextFunction): void {
        const token = req.cookies.refresh_token;

        if (!token) {
            return res.status(401).json({ error: 'Not authenticated' });
        }

        try {
            const decoded = jwt.verify(token, config.tokens.jwt);
            res.locals.claims = decoded;
            res.locals.effectiveUserId = decoded.sub;
            next();
        } catch (error) {
            res.status(401).json({ error: 'Invalid token' });
        }
    }
}
```

---

## 使用 AsyncLocalStorage 的审计中间件

### Blog API 的优秀模式

**文件：** `/form/src/middleware/auditMiddleware.ts`

```typescript
import { AsyncLocalStorage } from 'async_hooks';

export interface AuditContext {
    userId: string;
    userName?: string;
    impersonatedBy?: string;
    sessionId?: string;
    timestamp: Date;
    requestId: string;
}

export const auditContextStorage = new AsyncLocalStorage<AuditContext>();

export function auditMiddleware(req: Request, res: Response, next: NextFunction): void {
    const context: AuditContext = {
        userId: res.locals.effectiveUserId || 'anonymous',
        userName: res.locals.claims?.preferred_username,
        impersonatedBy: res.locals.isImpersonating ? res.locals.originalUserId : undefined,
        timestamp: new Date(),
        requestId: req.id || uuidv4(),
    };

    auditContextStorage.run(context, () => {
        next();
    });
}

// 获取当前上下文
export function getAuditContext(): AuditContext | null {
    return auditContextStorage.getStore() || null;
}
```

**好处：**
- 上下文在整个请求中传播
- 无需通过每个函数传递上下文
- 在服务、仓库中自动可用
- 类型安全的上下文访问

**在服务中使用：**
```typescript
import { getAuditContext } from '../middleware/auditMiddleware';

async function someOperation() {
    const context = getAuditContext();
    console.log('Operation by:', context?.userId);
}
```

---

## 错误边界中间件

### 综合错误处理器

**文件：** `/form/src/middleware/errorBoundary.ts`

```typescript
export function errorBoundary(
    error: Error,
    req: Request,
    res: Response,
    next: NextFunction
): void {
    // 确定状态码
    const statusCode = getStatusCodeForError(error);

    // 捕获到 Sentry
    Sentry.withScope((scope) => {
        scope.setLevel(statusCode >= 500 ? 'error' : 'warning');
        scope.setTag('error_type', error.name);
        scope.setContext('error_details', {
            message: error.message,
            stack: error.stack,
        });
        Sentry.captureException(error);
    });

    // 用户友好响应
    res.status(statusCode).json({
        success: false,
        error: {
            message: getUserFriendlyMessage(error),
            code: error.name,
        },
        requestId: Sentry.getCurrentScope().getPropagationContext().traceId,
    });
}

// 异步包装器
export function asyncErrorWrapper(
    handler: (req: Request, res: Response, next: NextFunction) => Promise<any>
) {
    return async (req: Request, res: Response, next: NextFunction) => {
        try {
            await handler(req, res, next);
        } catch (error) {
            next(error);
        }
    };
}
```

---

## 可组合中间件

### withAuthAndAudit 模式

```typescript
export function withAuthAndAudit(...authMiddleware: any[]) {
    return [
        ...authMiddleware,
        auditMiddleware,
    ];
}

// 使用
router.post('/:formID/submit',
    ...withAuthAndAudit(SSOMiddlewareClient.verifyLoginStatus),
    async (req, res) => controller.submit(req, res)
);
```

---

## 中间件顺序

### 关键顺序（必须遵循）

```typescript
// 1. Sentry 请求处理器（首先）
app.use(Sentry.Handlers.requestHandler());

// 2. 正文解析
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// 3. Cookie 解析
app.use(cookieParser());

// 4. 认证初始化
app.use(SSOMiddleware.initialize());

// 5. 在这里注册路由
app.use('/api/users', userRoutes);

// 6. 错误处理器（在路由之后）
app.use(errorBoundary);

// 7. Sentry 错误处理器（最后）
app.use(Sentry.Handlers.errorHandler());
```

**规则：** 错误处理器必须在所有路由之后注册！

---

**相关文件：**
- [SKILL.md](SKILL.md)
- [routing-and-controllers.md](routing-and-controllers.md)
- [async-and-errors.md](async-and-errors.md)
