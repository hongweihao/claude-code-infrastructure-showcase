# 路由和控制器 - 最佳实践

清晰路由定义和控制器模式的完整指南。

## 目录

- [路由：仅路由](#路由仅路由)
- [BaseController 模式](#basecontroller-模式)
- [良好示例](#良好示例)
- [反模式](#反模式)
- [重构指南](#重构指南)
- [错误处理](#错误处理)
- [HTTP 状态码](#http-状态码)

---

## 路由：仅路由

### 黄金规则

**路由应该只：**
- ✅ 定义路由路径
- ✅ 注册中间件
- ✅ 委托给控制器

**路由永远不应该：**
- ❌ 包含业务逻辑
- ❌ 直接访问数据库
- ❌ 实现验证逻辑（使用 Zod + 控制器）
- ❌ 格式化复杂响应
- ❌ 处理复杂错误场景

### 清晰路由模式

```typescript
// routes/userRoutes.ts
import { Router } from 'express';
import { UserController } from '../controllers/UserController';
import { SSOMiddlewareClient } from '../middleware/SSOMiddleware';
import { auditMiddleware } from '../middleware/auditMiddleware';

const router = Router();
const controller = new UserController();

// ✅ 清晰：仅路由定义
router.get('/:id',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.getUser(req, res)
);

router.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.createUser(req, res)
);

router.put('/:id',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.updateUser(req, res)
);

export default router;
```

**关键点：**
- 每个路由：方法、路径、中间件链、控制器委托
- 不需要 try-catch（控制器处理错误）
- 清晰、可读、可维护
- 易于一目了然地查看所有端点

---

## BaseController 模式

### 为什么使用 BaseController？

**好处：**
- 所有控制器的一致错误处理
- 自动 Sentry 集成
- 标准化响应格式
- 可重用的辅助方法
- 性能跟踪工具
- 日志和面包屑辅助函数

### BaseController 模式（模板）

**文件：** `/email/src/controllers/BaseController.ts`

```typescript
import * as Sentry from '@sentry/node';
import { Response } from 'express';

export abstract class BaseController {
    /**
     * 使用 Sentry 集成处理错误
     */
    protected handleError(
        error: unknown,
        res: Response,
        context: string,
        statusCode = 500
    ): void {
        Sentry.withScope((scope) => {
            scope.setTag('controller', this.constructor.name);
            scope.setTag('operation', context);
            scope.setUser({ id: res.locals?.claims?.userId });

            if (error instanceof Error) {
                scope.setContext('error_details', {
                    message: error.message,
                    stack: error.stack,
                });
            }

            Sentry.captureException(error);
        });

        res.status(statusCode).json({
            success: false,
            error: {
                message: error instanceof Error ? error.message : 'An error occurred',
                code: statusCode,
            },
        });
    }

    /**
     * 处理成功响应
     */
    protected handleSuccess<T>(
        res: Response,
        data: T,
        message?: string,
        statusCode = 200
    ): void {
        res.status(statusCode).json({
            success: true,
            message,
            data,
        });
    }

    /**
     * 性能跟踪包装器
     */
    protected async withTransaction<T>(
        name: string,
        operation: string,
        callback: () => Promise<T>
    ): Promise<T> {
        return await Sentry.startSpan(
            { name, op: operation },
            callback
        );
    }

    /**
     * 验证必填字段
     */
    protected validateRequest(
        required: string[],
        actual: Record<string, any>,
        res: Response
    ): boolean {
        const missing = required.filter((field) => !actual[field]);

        if (missing.length > 0) {
            Sentry.captureMessage(
                `Missing required fields: ${missing.join(', ')}`,
                'warning'
            );

            res.status(400).json({
                success: false,
                error: {
                    message: 'Missing required fields',
                    code: 'VALIDATION_ERROR',
                    details: { missing },
                },
            });
            return false;
        }
        return true;
    }

    /**
     * 日志辅助函数
     */
    protected logInfo(message: string, context?: Record<string, any>): void {
        Sentry.addBreadcrumb({
            category: this.constructor.name,
            message,
            level: 'info',
            data: context,
        });
    }

    protected logWarning(message: string, context?: Record<string, any>): void {
        Sentry.captureMessage(message, {
            level: 'warning',
            tags: { controller: this.constructor.name },
            extra: context,
        });
    }

    /**
     * 添加 Sentry 面包屑
     */
    protected addBreadcrumb(
        message: string,
        category: string,
        data?: Record<string, any>
    ): void {
        Sentry.addBreadcrumb({ message, category, level: 'info', data });
    }

    /**
     * 捕获自定义指标
     */
    protected captureMetric(name: string, value: number, unit: string): void {
        Sentry.metrics.gauge(name, value, { unit });
    }
}
```

### 使用 BaseController

```typescript
// controllers/UserController.ts
import { Request, Response } from 'express';
import { BaseController } from './BaseController';
import { UserService } from '../services/userService';
import { createUserSchema } from '../validators/userSchemas';

export class UserController extends BaseController {
    private userService: UserService;

    constructor() {
        super();
        this.userService = new UserService();
    }

    async getUser(req: Request, res: Response): Promise<void> {
        try {
            this.addBreadcrumb('Fetching user', 'user_controller', { userId: req.params.id });

            const user = await this.userService.findById(req.params.id);

            if (!user) {
                return this.handleError(
                    new Error('User not found'),
                    res,
                    'getUser',
                    404
                );
            }

            this.handleSuccess(res, user);
        } catch (error) {
            this.handleError(error, res, 'getUser');
        }
    }

    async createUser(req: Request, res: Response): Promise<void> {
        try {
            // 验证输入
            const validated = createUserSchema.parse(req.body);

            // 跟踪性能
            const user = await this.withTransaction(
                'user.create',
                'db.query',
                () => this.userService.create(validated)
            );

            this.handleSuccess(res, user, 'User created successfully', 201);
        } catch (error) {
            this.handleError(error, res, 'createUser');
        }
    }

    async updateUser(req: Request, res: Response): Promise<void> {
        try {
            const validated = updateUserSchema.parse(req.body);
            const user = await this.userService.update(req.params.id, validated);
            this.handleSuccess(res, user, 'User updated');
        } catch (error) {
            this.handleError(error, res, 'updateUser');
        }
    }
}
```

**好处：**
- 一致的错误处理
- 自动 Sentry 集成
- 性能跟踪
- 清晰、可读的代码
- 易于测试

---

## 良好示例

### 示例 1：Email Notification 路由（优秀 ✅）

**文件：** `/email/src/routes/notificationRoutes.ts`

```typescript
import { Router } from 'express';
import { NotificationController } from '../controllers/NotificationController';
import { SSOMiddlewareClient } from '../middleware/SSOMiddleware';

const router = Router();
const controller = new NotificationController();

// ✅ 优秀：清晰委托
router.get('/',
    SSOMiddlewareClient.verifyLoginStatus,
    async (req, res) => controller.getNotifications(req, res)
);

router.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    async (req, res) => controller.createNotification(req, res)
);

router.put('/:id/read',
    SSOMiddlewareClient.verifyLoginStatus,
    async (req, res) => controller.markAsRead(req, res)
);

export default router;
```

**为什么这很优秀：**
- 路由中零业务逻辑
- 清晰的中间件链
- 一致的模式
- 易于理解

### 示例 2：带验证的 Proxy 路由（良好 ✅）

**文件：** `/form/src/routes/proxyRoutes.ts`

```typescript
import { z } from 'zod';

const createProxySchema = z.object({
    originalUserID: z.string().min(1),
    proxyUserID: z.string().min(1),
    startsAt: z.string().datetime(),
    expiresAt: z.string().datetime(),
});

router.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    async (req, res) => {
        try {
            const validated = createProxySchema.parse(req.body);
            const proxy = await proxyService.createProxyRelationship(validated);
            res.status(201).json({ success: true, data: proxy });
        } catch (error) {
            handler.handleException(res, error);
        }
    }
);
```

**为什么这很好：**
- Zod 验证
- 委托给服务
- 正确的 HTTP 状态码
- 错误处理

**可以更好：**
- 将验证移到控制器
- 使用 BaseController

---

## 反模式

### 反模式 1：路由中的业务逻辑（不好 ❌）

**文件：** `/form/src/routes/responseRoutes.ts`（实际生产代码）

```typescript
// ❌ 反模式：路由中 200+ 行业务逻辑
router.post('/:formID/submit', async (req: Request, res: Response) => {
    try {
        const username = res.locals.claims.preferred_username;
        const responses = req.body.responses;
        const stepInstanceId = req.body.stepInstanceId;

        // ❌ 路由中的权限检查
        const userId = await userProfileService.getProfileByEmail(username).then(p => p.id);
        const canComplete = await permissionService.canCompleteStep(userId, stepInstanceId);
        if (!canComplete) {
            return res.status(403).json({ error: 'No permission' });
        }

        // ❌ 路由中的工作流逻辑
        const { createWorkflowEngine, CompleteStepCommand } = require('../workflow/core/WorkflowEngineV3');
        const engine = await createWorkflowEngine();
        const command = new CompleteStepCommand(
            stepInstanceId,
            userId,
            responses,
            additionalContext
        );
        const events = await engine.executeCommand(command);

        // ❌ 路由中的伪装处理
        if (res.locals.isImpersonating) {
            impersonationContextStore.storeContext(stepInstanceId, {
                originalUserId: res.locals.originalUserId,
                effectiveUserId: userId,
            });
        }

        // ❌ 路由中的响应处理
        const post = await PrismaService.main.post.findUnique({
            where: { id: postData.id },
            include: { comments: true },
        });

        // ❌ 路由中的权限检查
        await checkPostPermissions(post, userId);

        // ... 还有 100+ 行业务逻辑

        res.json({ success: true, data: result });
    } catch (e) {
        handler.handleException(res, e);
    }
});
```

**为什么这很糟糕：**
- 200+ 行业务逻辑
- 难以测试（需要 HTTP 模拟）
- 难以重用（绑定到路由）
- 混合职责
- 难以调试
- 性能跟踪困难

### 如何重构（逐步）

**步骤 1：创建控制器**

```typescript
// controllers/PostController.ts
export class PostController extends BaseController {
    private postService: PostService;

    constructor() {
        super();
        this.postService = new PostService();
    }

    async createPost(req: Request, res: Response): Promise<void> {
        try {
            const validated = createPostSchema.parse({
                ...req.body,
            });

            const result = await this.postService.createPost(
                validated,
                res.locals.userId
            );

            this.handleSuccess(res, result, 'Post created successfully');
        } catch (error) {
            this.handleError(error, res, 'createPost');
        }
    }
}
```

**步骤 2：创建服务**

```typescript
// services/postService.ts
export class PostService {
    async createPost(
        data: CreatePostDTO,
        userId: string
    ): Promise<PostResult> {
        // 权限检查
        const canCreate = await permissionService.canCreatePost(userId);
        if (!canCreate) {
            throw new ForbiddenError('No permission to create post');
        }

        // 执行工作流
        const engine = await createWorkflowEngine();
        const command = new CompleteStepCommand(/* ... */);
        const events = await engine.executeCommand(command);

        // 如需要处理伪装
        if (context.isImpersonating) {
            await this.handleImpersonation(data.stepInstanceId, context);
        }

        // 同步角色
        await this.synchronizeRoles(events, userId);

        return { events, success: true };
    }

    private async handleImpersonation(stepInstanceId: number, context: any) {
        impersonationContextStore.storeContext(stepInstanceId, {
            originalUserId: context.originalUserId,
            effectiveUserId: context.effectiveUserId,
        });
    }

    private async synchronizeRoles(events: WorkflowEvent[], userId: string) {
        // 角色同步逻辑
    }
}
```

**步骤 3：更新路由**

```typescript
// routes/postRoutes.ts
import { PostController } from '../controllers/PostController';

const router = Router();
const controller = new PostController();

// ✅ 清晰：只是路由
router.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.createPost(req, res)
);
```

**结果：**
- 路由：8 行（原来 200+）
- 控制器：25 行（请求处理）
- 服务：50 行（业务逻辑）
- 可测试、可重用、可维护！

---

## 错误处理

### 控制器错误处理

```typescript
async createUser(req: Request, res: Response): Promise<void> {
    try {
        const result = await this.userService.create(req.body);
        this.handleSuccess(res, result, 'User created', 201);
    } catch (error) {
        // BaseController.handleError 自动：
        // - 使用上下文捕获到 Sentry
        // - 设置适当的状态码
        // - 返回格式化错误响应
        this.handleError(error, res, 'createUser');
    }
}
```

### 自定义错误状态码

```typescript
async getUser(req: Request, res: Response): Promise<void> {
    try {
        const user = await this.userService.findById(req.params.id);

        if (!user) {
            // 自定义 404 状态
            return this.handleError(
                new Error('User not found'),
                res,
                'getUser',
                404  // 自定义状态码
            );
        }

        this.handleSuccess(res, user);
    } catch (error) {
        this.handleError(error, res, 'getUser');
    }
}
```

### 验证错误

```typescript
async createUser(req: Request, res: Response): Promise<void> {
    try {
        const validated = createUserSchema.parse(req.body);
        const user = await this.userService.create(validated);
        this.handleSuccess(res, user, 'User created', 201);
    } catch (error) {
        // Zod 错误获取 400 状态
        if (error instanceof z.ZodError) {
            return this.handleError(error, res, 'createUser', 400);
        }
        this.handleError(error, res, 'createUser');
    }
}
```

---

## HTTP 状态码

### 标准代码

| 代码 | 用例 | 示例 |
|------|------|------|
| 200 | 成功（GET、PUT）| 检索用户、已更新 |
| 201 | 已创建（POST）| 用户已创建 |
| 204 | 无内容（DELETE）| 用户已删除 |
| 400 | 错误请求 | 无效输入数据 |
| 401 | 未授权 | 未认证 |
| 403 | 禁止访问 | 无权限 |
| 404 | 未找到 | 资源不存在 |
| 409 | 冲突 | 重复资源 |
| 422 | 无法处理的实体 | 验证失败 |
| 500 | 内部服务器错误 | 意外错误 |

### 使用示例

```typescript
// 200 - 成功（默认）
this.handleSuccess(res, user);

// 201 - 已创建
this.handleSuccess(res, user, 'Created', 201);

// 400 - 错误请求
this.handleError(error, res, 'operation', 400);

// 404 - 未找到
this.handleError(new Error('Not found'), res, 'operation', 404);

// 403 - 禁止访问
this.handleError(new ForbiddenError('No permission'), res, 'operation', 403);
```

---

## 重构指南

### 识别需要重构的路由

**危险信号：**
- 路由文件 > 100 行
- 一个路由中有多个 try-catch 块
- 直接数据库访问（Prisma 调用）
- 复杂的业务逻辑（if 语句、循环）
- 路由中的权限检查

**检查你的路由：**
```bash
# 查找大型路由文件
wc -l form/src/routes/*.ts | sort -n

# 查找使用 Prisma 的路由
grep -r "PrismaService" form/src/routes/
```

### 重构过程

**1. 提取到控制器：**
```typescript
// 之前：带逻辑的路由
router.post('/action', async (req, res) => {
    try {
        // 50 行逻辑
    } catch (e) {
        handler.handleException(res, e);
    }
});

// 之后：清晰路由
router.post('/action', (req, res) => controller.performAction(req, res));

// 新控制器方法
async performAction(req: Request, res: Response): Promise<void> {
    try {
        const result = await this.service.performAction(req.body);
        this.handleSuccess(res, result);
    } catch (error) {
        this.handleError(error, res, 'performAction');
    }
}
```

**2. 提取到服务：**
```typescript
// 控制器保持精简
async performAction(req: Request, res: Response): Promise<void> {
    try {
        const validated = actionSchema.parse(req.body);
        const result = await this.actionService.execute(validated);
        this.handleSuccess(res, result);
    } catch (error) {
        this.handleError(error, res, 'performAction');
    }
}

// 服务包含业务逻辑
export class ActionService {
    async execute(data: ActionDTO): Promise<Result> {
        // 所有业务逻辑在这里
        // 权限检查
        // 数据库操作
        // 复杂转换
        return result;
    }
}
```

**3. 添加仓库（如需要）：**
```typescript
// 服务调用仓库
export class ActionService {
    constructor(private actionRepository: ActionRepository) {}

    async execute(data: ActionDTO): Promise<Result> {
        // 业务逻辑
        const entity = await this.actionRepository.findById(data.id);
        // 更多逻辑
        return await this.actionRepository.update(data.id, changes);
    }
}

// 仓库处理数据访问
export class ActionRepository {
    async findById(id: number): Promise<Entity | null> {
        return PrismaService.main.entity.findUnique({ where: { id } });
    }

    async update(id: number, data: Partial<Entity>): Promise<Entity> {
        return PrismaService.main.entity.update({ where: { id }, data });
    }
}
```

---

**相关文件：**
- [SKILL.md](SKILL.md) - 主指南
- [services-and-repositories.md](services-and-repositories.md) - 服务层详细信息
- [complete-examples.md](complete-examples.md) - 完整重构示例
