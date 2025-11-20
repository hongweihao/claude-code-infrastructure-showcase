# 验证模式 - 使用 Zod 进行输入验证

使用 Zod 模式进行类型安全验证的完整指南。

## 目录

- [为什么选择 Zod？](#为什么选择-zod)
- [基本 Zod 模式](#基本-zod-模式)
- [代码库中的模式示例](#代码库中的模式示例)
- [路由级验证](#路由级验证)
- [控制器验证](#控制器验证)
- [DTO 模式](#dto-模式)
- [错误处理](#错误处理)
- [高级模式](#高级模式)

---

## 为什么选择 Zod？

### 相比 Joi/其他库的优势

**类型安全：**
- ✅ 完整的 TypeScript 推断
- ✅ 运行时 + 编译时验证
- ✅ 自动类型生成

**开发者体验：**
- ✅ 直观的 API
- ✅ 可组合的模式
- ✅ 出色的错误消息

**性能：**
- ✅ 快速验证
- ✅ 小打包大小
- ✅ 可摇树优化

### 从 Joi 迁移

现代验证使用 Zod 而不是 Joi：

```typescript
// ❌ 旧 - Joi（正在淘汰）
const schema = Joi.object({
    email: Joi.string().email().required(),
    name: Joi.string().min(3).required(),
});

// ✅ 新 - Zod（首选）
const schema = z.object({
    email: z.string().email(),
    name: z.string().min(3),
});
```

---

## 基本 Zod 模式

### 原始类型

```typescript
import { z } from 'zod';

// 字符串
const nameSchema = z.string();
const emailSchema = z.string().email();
const urlSchema = z.string().url();
const uuidSchema = z.string().uuid();
const minLengthSchema = z.string().min(3);
const maxLengthSchema = z.string().max(100);

// 数字
const ageSchema = z.number().int().positive();
const priceSchema = z.number().positive();
const rangeSchema = z.number().min(0).max(100);

// 布尔值
const activeSchema = z.boolean();

// 日期
const dateSchema = z.string().datetime(); // ISO 8601 字符串
const nativeDateSchema = z.date(); // 原生 Date 对象

// 枚举
const roleSchema = z.enum(['admin', 'operations', 'user']);
const statusSchema = z.enum(['PENDING', 'APPROVED', 'REJECTED']);
```

### 对象

```typescript
// 简单对象
const userSchema = z.object({
    email: z.string().email(),
    name: z.string(),
    age: z.number().int().positive(),
});

// 嵌套对象
const addressSchema = z.object({
    street: z.string(),
    city: z.string(),
    zipCode: z.string().regex(/^\d{5}$/),
});

const userWithAddressSchema = z.object({
    name: z.string(),
    address: addressSchema,
});

// 可选字段
const userSchema = z.object({
    name: z.string(),
    email: z.string().email().optional(),
    phone: z.string().optional(),
});

// 可空字段
const userSchema = z.object({
    name: z.string(),
    middleName: z.string().nullable(),
});
```

### 数组

```typescript
// 原始类型数组
const rolesSchema = z.array(z.string());
const numbersSchema = z.array(z.number());

// 对象数组
const usersSchema = z.array(
    z.object({
        id: z.string(),
        name: z.string(),
    })
);

// 带约束的数组
const tagsSchema = z.array(z.string()).min(1).max(10);
const nonEmptyArray = z.array(z.string()).nonempty();
```

---

## 代码库中的模式示例

### 表单验证模式

**文件：** `/form/src/helpers/zodSchemas.ts`

```typescript
import { z } from 'zod';

// 问题类型枚举
export const questionTypeSchema = z.enum([
    'input',
    'textbox',
    'editor',
    'dropdown',
    'autocomplete',
    'checkbox',
    'radio',
    'upload',
]);

// 上传类型
export const uploadTypeSchema = z.array(
    z.enum(['pdf', 'image', 'excel', 'video', 'powerpoint', 'word']).nullable()
);

// 输入类型
export const inputTypeSchema = z
    .enum(['date', 'number', 'input', 'currency'])
    .nullable();

// 问题选项
export const questionOptionSchema = z.object({
    id: z.number().int().positive().optional(),
    controlTag: z.string().max(150).nullable().optional(),
    label: z.string().max(100).nullable().optional(),
    order: z.number().int().min(0).default(0),
});

// 问题模式
export const questionSchema = z.object({
    id: z.number().int().positive().optional(),
    formID: z.number().int().positive(),
    sectionID: z.number().int().positive().optional(),
    options: z.array(questionOptionSchema).optional(),
    label: z.string().max(500),
    description: z.string().max(5000).optional(),
    type: questionTypeSchema,
    uploadTypes: uploadTypeSchema.optional(),
    inputType: inputTypeSchema.optional(),
    tags: z.array(z.string().max(150)).optional(),
    required: z.boolean(),
    isStandard: z.boolean().optional(),
    deprecatedKey: z.string().nullable().optional(),
    maxLength: z.number().int().positive().nullable().optional(),
    isOptionsSorted: z.boolean().optional(),
});

// 表单部分模式
export const formSectionSchema = z.object({
    id: z.number().int().positive(),
    formID: z.number().int().positive(),
    questions: z.array(questionSchema).optional(),
    label: z.string().max(500),
    description: z.string().max(5000).optional(),
    isStandard: z.boolean(),
});

// 创建表单模式
export const createFormSchema = z.object({
    id: z.number().int().positive(),
    label: z.string().max(150),
    description: z.string().max(6000).nullable().optional(),
    isPhase: z.boolean().optional(),
    username: z.string(),
});

// 更新顺序模式
export const updateOrderSchema = z.object({
    source: z.object({
        index: z.number().int().min(0),
        sectionID: z.number().int().min(0),
    }),
    destination: z.object({
        index: z.number().int().min(0),
        sectionID: z.number().int().min(0),
    }),
});

// 控制器特定验证模式
export const createQuestionValidationSchema = z.object({
    formID: z.number().int().positive(),
    sectionID: z.number().int().positive(),
    question: questionSchema,
    index: z.number().int().min(0).nullable().optional(),
    username: z.string(),
});

export const updateQuestionValidationSchema = z.object({
    questionID: z.number().int().positive(),
    username: z.string(),
    question: questionSchema,
});
```

### Proxy 关系模式

```typescript
// Proxy 关系验证
const createProxySchema = z.object({
    originalUserID: z.string().min(1),
    proxyUserID: z.string().min(1),
    startsAt: z.string().datetime(),
    expiresAt: z.string().datetime(),
});

// 带自定义验证
const createProxySchemaWithValidation = createProxySchema.refine(
    (data) => new Date(data.expiresAt) > new Date(data.startsAt),
    {
        message: 'expiresAt must be after startsAt',
        path: ['expiresAt'],
    }
);
```

### 工作流验证

```typescript
// 工作流启动模式
const startWorkflowSchema = z.object({
    workflowCode: z.string().min(1),
    entityType: z.enum(['Post', 'User', 'Comment']),
    entityID: z.number().int().positive(),
    dryRun: z.boolean().optional().default(false),
});

// 工作流步骤完成模式
const completeStepSchema = z.object({
    stepInstanceID: z.number().int().positive(),
    answers: z.record(z.string(), z.any()),
    dryRun: z.boolean().optional().default(false),
});
```

---

## 路由级验证

### 模式 1：内联验证

```typescript
// routes/proxyRoutes.ts
import { z } from 'zod';

const createProxySchema = z.object({
    originalUserID: z.string().min(1),
    proxyUserID: z.string().min(1),
    startsAt: z.string().datetime(),
    expiresAt: z.string().datetime(),
});

router.post(
    '/',
    SSOMiddlewareClient.verifyLoginStatus,
    async (req, res) => {
        try {
            // 路由级验证
            const validated = createProxySchema.parse(req.body);

            // 委托给服务
            const proxy = await proxyService.createProxyRelationship(validated);

            res.status(201).json({ success: true, data: proxy });
        } catch (error) {
            if (error instanceof z.ZodError) {
                return res.status(400).json({
                    success: false,
                    error: {
                        message: 'Validation failed',
                        details: error.errors,
                    },
                });
            }
            handler.handleException(res, error);
        }
    }
);
```

**优点：**
- 快速简单
- 适合简单路由

**缺点：**
- 路由中的验证逻辑
- 更难测试
- 不可重用

---

## 控制器验证

### 模式 2：控制器验证（推荐）

```typescript
// validators/userSchemas.ts
import { z } from 'zod';

export const createUserSchema = z.object({
    email: z.string().email(),
    name: z.string().min(2).max(100),
    roles: z.array(z.enum(['admin', 'operations', 'user'])),
    isActive: z.boolean().default(true),
});

export const updateUserSchema = z.object({
    email: z.string().email().optional(),
    name: z.string().min(2).max(100).optional(),
    roles: z.array(z.enum(['admin', 'operations', 'user'])).optional(),
    isActive: z.boolean().optional(),
});

export type CreateUserDTO = z.infer<typeof createUserSchema>;
export type UpdateUserDTO = z.infer<typeof updateUserSchema>;
```

```typescript
// controllers/UserController.ts
import { Request, Response } from 'express';
import { BaseController } from './BaseController';
import { UserService } from '../services/userService';
import { createUserSchema, updateUserSchema } from '../validators/userSchemas';
import { z } from 'zod';

export class UserController extends BaseController {
    private userService: UserService;

    constructor() {
        super();
        this.userService = new UserService();
    }

    async createUser(req: Request, res: Response): Promise<void> {
        try {
            // 验证输入
            const validated = createUserSchema.parse(req.body);

            // 调用服务
            const user = await this.userService.createUser(validated);

            this.handleSuccess(res, user, 'User created successfully', 201);
        } catch (error) {
            if (error instanceof z.ZodError) {
                // 用 400 状态处理验证错误
                return this.handleError(error, res, 'createUser', 400);
            }
            this.handleError(error, res, 'createUser');
        }
    }

    async updateUser(req: Request, res: Response): Promise<void> {
        try {
            // 验证参数和正文
            const userId = req.params.id;
            const validated = updateUserSchema.parse(req.body);

            const user = await this.userService.updateUser(userId, validated);

            this.handleSuccess(res, user, 'User updated successfully');
        } catch (error) {
            if (error instanceof z.ZodError) {
                return this.handleError(error, res, 'updateUser', 400);
            }
            this.handleError(error, res, 'updateUser');
        }
    }
}
```

**优点：**
- 清晰的分离
- 可重用的模式
- 易于测试
- 类型安全的 DTO

**缺点：**
- 需要管理更多文件

---

## DTO 模式

### 从模式推断类型

```typescript
import { z } from 'zod';

// 定义模式
const createUserSchema = z.object({
    email: z.string().email(),
    name: z.string(),
    age: z.number().int().positive(),
});

// 从模式推断 TypeScript 类型
type CreateUserDTO = z.infer<typeof createUserSchema>;

// 等同于：
// type CreateUserDTO = {
//     email: string;
//     name: string;
//     age: number;
// }

// 在服务中使用
class UserService {
    async createUser(data: CreateUserDTO): Promise<User> {
        // data 是完全类型化的！
        console.log(data.email); // ✅ TypeScript 知道这存在
        console.log(data.invalid); // ❌ TypeScript 错误！
    }
}
```

### 输入与输出类型

```typescript
// 输入模式（API 接收）
const createUserInputSchema = z.object({
    email: z.string().email(),
    name: z.string(),
    password: z.string().min(8),
});

// 输出模式（API 返回）
const userOutputSchema = z.object({
    id: z.string().uuid(),
    email: z.string().email(),
    name: z.string(),
    createdAt: z.string().datetime(),
    // password 被排除！
});

type CreateUserInput = z.infer<typeof createUserInputSchema>;
type UserOutput = z.infer<typeof userOutputSchema>;
```

---

## 错误处理

### Zod 错误格式

```typescript
try {
    const validated = schema.parse(data);
} catch (error) {
    if (error instanceof z.ZodError) {
        console.log(error.errors);
        // [
        //   {
        //     code: 'invalid_type',
        //     expected: 'string',
        //     received: 'number',
        //     path: ['email'],
        //     message: 'Expected string, received number'
        //   }
        // ]
    }
}
```

### 自定义错误消息

```typescript
const userSchema = z.object({
    email: z.string().email({ message: 'Please provide a valid email address' }),
    name: z.string().min(2, { message: 'Name must be at least 2 characters' }),
    age: z.number().int().positive({ message: 'Age must be a positive number' }),
});
```

### 格式化错误响应

```typescript
// 格式化 Zod 错误的辅助函数
function formatZodError(error: z.ZodError) {
    return {
        message: 'Validation failed',
        errors: error.errors.map((err) => ({
            field: err.path.join('.'),
            message: err.message,
            code: err.code,
        })),
    };
}

// 在控制器中
catch (error) {
    if (error instanceof z.ZodError) {
        return res.status(400).json({
            success: false,
            error: formatZodError(error),
        });
    }
}

// 响应示例：
// {
//   "success": false,
//   "error": {
//     "message": "Validation failed",
//     "errors": [
//       {
//         "field": "email",
//         "message": "Invalid email",
//         "code": "invalid_string"
//       }
//     ]
//   }
// }
```

---

## 高级模式

### 条件验证

```typescript
// 基于其他字段值验证
const submissionSchema = z.object({
    type: z.enum(['NEW', 'UPDATE']),
    postId: z.number().optional(),
}).refine(
    (data) => {
        // 如果类型是 UPDATE，postId 是必需的
        if (data.type === 'UPDATE') {
            return data.postId !== undefined;
        }
        return true;
    },
    {
        message: 'postId is required when type is UPDATE',
        path: ['postId'],
    }
);
```

### 转换数据

```typescript
// 将字符串转换为数字
const userSchema = z.object({
    name: z.string(),
    age: z.string().transform((val) => parseInt(val, 10)),
});

// 转换日期
const eventSchema = z.object({
    name: z.string(),
    date: z.string().transform((str) => new Date(str)),
});
```

### 预处理数据

```typescript
// 验证前修剪字符串
const userSchema = z.object({
    email: z.preprocess(
        (val) => typeof val === 'string' ? val.trim().toLowerCase() : val,
        z.string().email()
    ),
    name: z.preprocess(
        (val) => typeof val === 'string' ? val.trim() : val,
        z.string().min(2)
    ),
});
```

### 联合类型

```typescript
// 多种可能类型
const idSchema = z.union([z.string(), z.number()]);

// 区分联合
const notificationSchema = z.discriminatedUnion('type', [
    z.object({
        type: z.literal('email'),
        recipient: z.string().email(),
        subject: z.string(),
    }),
    z.object({
        type: z.literal('sms'),
        phoneNumber: z.string(),
        message: z.string(),
    }),
]);
```

### 递归模式

```typescript
// 用于树等嵌套结构
type Category = {
    id: number;
    name: string;
    children?: Category[];
};

const categorySchema: z.ZodType<Category> = z.lazy(() =>
    z.object({
        id: z.number(),
        name: z.string(),
        children: z.array(categorySchema).optional(),
    })
);
```

### 模式组合

```typescript
// 基础模式
const timestampsSchema = z.object({
    createdAt: z.string().datetime(),
    updatedAt: z.string().datetime(),
});

const auditSchema = z.object({
    createdBy: z.string(),
    updatedBy: z.string(),
});

// 组合模式
const userSchema = z.object({
    id: z.string(),
    email: z.string().email(),
    name: z.string(),
}).merge(timestampsSchema).merge(auditSchema);

// 扩展模式
const adminUserSchema = userSchema.extend({
    adminLevel: z.number().int().min(1).max(5),
    permissions: z.array(z.string()),
});

// 选择特定字段
const publicUserSchema = userSchema.pick({
    id: true,
    name: true,
    // email 被排除
});

// 省略字段
const userWithoutTimestamps = userSchema.omit({
    createdAt: true,
    updatedAt: true,
});
```

### 验证中间件

```typescript
// 创建可重用的验证中间件
import { Request, Response, NextFunction } from 'express';
import { z } from 'zod';

export function validateBody<T extends z.ZodType>(schema: T) {
    return (req: Request, res: Response, next: NextFunction) => {
        try {
            req.body = schema.parse(req.body);
            next();
        } catch (error) {
            if (error instanceof z.ZodError) {
                return res.status(400).json({
                    success: false,
                    error: {
                        message: 'Validation failed',
                        details: error.errors,
                    },
                });
            }
            next(error);
        }
    };
}

// 使用
router.post('/users',
    validateBody(createUserSchema),
    async (req, res) => {
        // req.body 已验证并类型化！
        const user = await userService.createUser(req.body);
        res.json({ success: true, data: user });
    }
);
```

---

**相关文件：**
- [SKILL.md](SKILL.md) - 主指南
- [routing-and-controllers.md](routing-and-controllers.md) - 在控制器中使用验证
- [services-and-repositories.md](services-and-repositories.md) - 在服务中使用 DTO
- [async-and-errors.md](async-and-errors.md) - 错误处理模式
