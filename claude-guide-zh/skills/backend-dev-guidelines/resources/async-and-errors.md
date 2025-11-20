# 异步模式和错误处理

async/await 模式和自定义错误处理的完整指南。

## 目录

- [Async/Await 最佳实践](#asyncawait-最佳实践)
- [Promise 错误处理](#promise-错误处理)
- [自定义错误类型](#自定义错误类型)
- [asyncErrorWrapper 工具](#asyncerrorwrapper-工具)
- [错误传播](#错误传播)
- [常见异步陷阱](#常见异步陷阱)

---

## Async/Await 最佳实践

### 始终使用 Try-Catch

```typescript
// ❌ 永远不要：未处理的异步错误
async function fetchData() {
    const data = await database.query(); // 如果抛出，未处理！
    return data;
}

// ✅ 总是：用 try-catch 包装
async function fetchData() {
    try {
        const data = await database.query();
        return data;
    } catch (error) {
        Sentry.captureException(error);
        throw error;
    }
}
```

### 避免 .then() 链

```typescript
// ❌ 避免：Promise 链
function processData() {
    return fetchData()
        .then(data => transform(data))
        .then(transformed => save(transformed))
        .catch(error => {
            console.error(error);
        });
}

// ✅ 首选：Async/await
async function processData() {
    try {
        const data = await fetchData();
        const transformed = await transform(data);
        return await save(transformed);
    } catch (error) {
        Sentry.captureException(error);
        throw error;
    }
}
```

---

## Promise 错误处理

### 并行操作

```typescript
// ✅ 在 Promise.all 中处理错误
try {
    const [users, profiles, settings] = await Promise.all([
        userService.getAll(),
        profileService.getAll(),
        settingsService.getAll(),
    ]);
} catch (error) {
    // 一个失败全部失败
    Sentry.captureException(error);
    throw error;
}

// ✅ 使用 Promise.allSettled 单独处理错误
const results = await Promise.allSettled([
    userService.getAll(),
    profileService.getAll(),
    settingsService.getAll(),
]);

results.forEach((result, index) => {
    if (result.status === 'rejected') {
        Sentry.captureException(result.reason, {
            tags: { operation: ['users', 'profiles', 'settings'][index] }
        });
    }
});
```

---

## 自定义错误类型

### 定义自定义错误

```typescript
// 基础错误类
export class AppError extends Error {
    constructor(
        message: string,
        public code: string,
        public statusCode: number,
        public isOperational: boolean = true
    ) {
        super(message);
        this.name = this.constructor.name;
        Error.captureStackTrace(this, this.constructor);
    }
}

// 特定错误类型
export class ValidationError extends AppError {
    constructor(message: string) {
        super(message, 'VALIDATION_ERROR', 400);
    }
}

export class NotFoundError extends AppError {
    constructor(message: string) {
        super(message, 'NOT_FOUND', 404);
    }
}

export class ForbiddenError extends AppError {
    constructor(message: string) {
        super(message, 'FORBIDDEN', 403);
    }
}

export class ConflictError extends AppError {
    constructor(message: string) {
        super(message, 'CONFLICT', 409);
    }
}
```

### 用法

```typescript
// 抛出特定错误
if (!user) {
    throw new NotFoundError('User not found');
}

if (user.age < 18) {
    throw new ValidationError('User must be 18+');
}

// 错误边界处理它们
function errorBoundary(error, req, res, next) {
    if (error instanceof AppError) {
        return res.status(error.statusCode).json({
            error: {
                message: error.message,
                code: error.code
            }
        });
    }

    // 未知错误
    Sentry.captureException(error);
    res.status(500).json({ error: { message: 'Internal server error' } });
}
```

---

## asyncErrorWrapper 工具

### 模式

```typescript
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

### 用法

```typescript
// 没有包装器 - 错误可能未处理
router.get('/users', async (req, res) => {
    const users = await userService.getAll(); // 如果抛出，未处理！
    res.json(users);
});

// 有包装器 - 错误被捕获
router.get('/users', asyncErrorWrapper(async (req, res) => {
    const users = await userService.getAll();
    res.json(users);
}));
```

---

## 错误传播

### 正确的错误链

```typescript
// ✅ 向上传播错误
async function repositoryMethod() {
    try {
        return await PrismaService.main.user.findMany();
    } catch (error) {
        Sentry.captureException(error, { tags: { layer: 'repository' } });
        throw error; // 传播到服务
    }
}

async function serviceMethod() {
    try {
        return await repositoryMethod();
    } catch (error) {
        Sentry.captureException(error, { tags: { layer: 'service' } });
        throw error; // 传播到控制器
    }
}

async function controllerMethod(req, res) {
    try {
        const result = await serviceMethod();
        res.json(result);
    } catch (error) {
        this.handleError(error, res, 'controllerMethod'); // 最终处理器
    }
}
```

---

## 常见异步陷阱

### 即发即弃（不好）

```typescript
// ❌ 永远不要：即发即弃
async function processRequest(req, res) {
    sendEmail(user.email); // 异步触发，错误未处理！
    res.json({ success: true });
}

// ✅ 总是：await 或处理
async function processRequest(req, res) {
    try {
        await sendEmail(user.email);
        res.json({ success: true });
    } catch (error) {
        Sentry.captureException(error);
        res.status(500).json({ error: 'Failed to send email' });
    }
}

// ✅ 或者：有意的后台任务
async function processRequest(req, res) {
    sendEmail(user.email).catch(error => {
        Sentry.captureException(error);
    });
    res.json({ success: true });
}
```

### 未处理的拒绝

```typescript
// ✅ 未处理拒绝的全局处理器
process.on('unhandledRejection', (reason, promise) => {
    Sentry.captureException(reason, {
        tags: { type: 'unhandled_rejection' }
    });
    console.error('Unhandled Rejection:', reason);
});

process.on('uncaughtException', (error) => {
    Sentry.captureException(error, {
        tags: { type: 'uncaught_exception' }
    });
    console.error('Uncaught Exception:', error);
    process.exit(1);
});
```

---

**相关文件：**
- [SKILL.md](SKILL.md)
- [sentry-and-monitoring.md](sentry-and-monitoring.md)
- [complete-examples.md](complete-examples.md)
