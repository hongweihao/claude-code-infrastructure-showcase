# 架构概览 - 后端服务

后端微服务中使用的分层架构模式的完整指南。

## 目录

- [分层架构模式](#分层架构模式)
- [请求生命周期](#请求生命周期)
- [服务比较](#服务比较)
- [目录结构基本原理](#目录结构基本原理)
- [模块组织](#模块组织)
- [关注点分离](#关注点分离)

---

## 分层架构模式

### 四个层次

```
┌─────────────────────────────────────┐
│         HTTP 请求                   │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│  第 1 层：路由                      │
│  - 仅路由定义                       │
│  - 中间件注册                       │
│  - 委托给控制器                     │
│  - 无业务逻辑                       │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│  第 2 层：控制器                    │
│  - 请求/响应处理                    │
│  - 输入验证                         │
│  - 调用服务                         │
│  - 格式化响应                       │
│  - 错误处理                         │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│  第 3 层：服务                      │
│  - 业务逻辑                         │
│  - 编排                             │
│  - 调用仓库                         │
│  - 无 HTTP 知识                     │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│  第 4 层：仓库                      │
│  - 数据访问抽象                     │
│  - Prisma 操作                      │
│  - 查询优化                         │
│  - 缓存                             │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│         数据库（MySQL）             │
└─────────────────────────────────────┘
```

### 为什么选择这种架构？

**可测试性：**
- 每一层都可以独立测试
- 易于模拟依赖项
- 清晰的测试边界

**可维护性：**
- 更改隔离到特定层
- 业务逻辑与 HTTP 关注点分离
- 易于定位错误

**可重用性：**
- 服务可被路由、cron 作业、脚本使用
- 仓库隐藏数据库实现
- 业务逻辑不绑定到 HTTP

**可扩展性：**
- 易于添加新端点
- 清晰的模式可遵循
- 一致的结构

---

## 请求生命周期

### 完整流程示例

```typescript
1. HTTP POST /api/users
   ↓
2. Express 在 userRoutes.ts 中匹配路由
   ↓
3. 中间件链执行：
   - SSOMiddleware.verifyLoginStatus（认证）
   - auditMiddleware（上下文跟踪）
   ↓
4. 路由处理器委托给控制器：
   router.post('/users', (req, res) => userController.create(req, res))
   ↓
5. 控制器验证并调用服务：
   - 使用 Zod 验证输入
   - 调用 userService.create(data)
   - 处理成功/错误
   ↓
6. 服务执行业务逻辑：
   - 检查业务规则
   - 调用 userRepository.create(data)
   - 返回结果
   ↓
7. 仓库执行数据库操作：
   - PrismaService.main.user.create({ data })
   - 处理数据库错误
   - 返回创建的用户
   ↓
8. 响应向上流动：
   仓库 → 服务 → 控制器 → Express → 客户端
```

### 中间件执行顺序

**关键：** 中间件按注册顺序执行

```typescript
app.use(Sentry.Handlers.requestHandler());  // 1. Sentry 跟踪（首先）
app.use(express.json());                     // 2. 正文解析
app.use(express.urlencoded({ extended: true })); // 3. URL 编码
app.use(cookieParser());                     // 4. Cookie 解析
app.use(SSOMiddleware.initialize());         // 5. 认证初始化
// ... 在这里注册路由
app.use(auditMiddleware);                    // 6. 审计（如果全局）
app.use(errorBoundary);                      // 7. 错误处理器（最后）
app.use(Sentry.Handlers.errorHandler());     // 8. Sentry 错误（最后）
```

**规则：** 错误处理器必须在路由之后注册！

---

## 服务比较

### Email Service（成熟模式 ✅）

**优势：**
- 具有 Sentry 集成的综合 BaseController
- 清晰的路由委托（路由中无业务逻辑）
- 一致的依赖注入模式
- 良好的中间件组织
- 全类型安全
- 出色的错误处理

**示例结构：**
```
email/src/
├── controllers/
│   ├── BaseController.ts          ✅ 出色的模板
│   ├── NotificationController.ts  ✅ 扩展 BaseController
│   └── EmailController.ts         ✅ 清晰的模式
├── routes/
│   ├── notificationRoutes.ts      ✅ 清晰的委托
│   └── emailRoutes.ts             ✅ 无业务逻辑
├── services/
│   ├── NotificationService.ts     ✅ 依赖注入
│   └── BatchingService.ts         ✅ 清晰的职责
└── middleware/
    ├── errorBoundary.ts           ✅ 全面
    └── DevImpersonationSSOMiddleware.ts
```

**用作**新服务的模板！

### Form Service（过渡中 ⚠️）

**优势：**
- 出色的工作流架构（事件溯源）
- 良好的 Sentry 集成
- 创新的审计中间件（AsyncLocalStorage）
- 全面的权限系统

**弱点：**
- 一些路由有 200+ 行业务逻辑
- 控制器命名不一致
- 直接使用 process.env（60+ 次出现）
- 仓库模式使用最少

**示例：**
```
form/src/
├── routes/
│   ├── responseRoutes.ts          ❌ 路由中的业务逻辑
│   └── proxyRoutes.ts             ✅ 良好的验证模式
├── controllers/
│   ├── formController.ts          ⚠️ 小写命名
│   └── UserProfileController.ts   ✅ PascalCase 命名
├── workflow/                      ✅ 出色的架构！
│   ├── core/
│   │   ├── WorkflowEngineV3.ts   ✅ 事件溯源
│   │   └── DryRunWrapper.ts      ✅ 创新
│   └── services/
└── middleware/
    └── auditMiddleware.ts         ✅ AsyncLocalStorage 模式
```

**学习：** workflow/、middleware/auditMiddleware.ts
**避免：** responseRoutes.ts、直接使用 process.env

---

## 目录结构基本原理

### Controllers 目录

**目的：** 处理 HTTP 请求/响应关注点

**内容：**
- `BaseController.ts` - 带有通用方法的基类
- `{Feature}Controller.ts` - 特定功能的控制器

**命名：** PascalCase + Controller

**职责：**
- 解析请求参数
- 验证输入（Zod）
- 调用适当的服务方法
- 格式化响应
- 处理错误（通过 BaseController）
- 设置 HTTP 状态码

### Services 目录

**目的：** 业务逻辑和编排

**内容：**
- `{feature}Service.ts` - 功能业务逻辑

**命名：** camelCase + Service（或 PascalCase + Service）

**职责：**
- 实现业务规则
- 编排多个仓库
- 事务管理
- 业务验证
- 无 HTTP 知识（Request/Response 类型）

### Repositories 目录

**目的：** 数据访问抽象

**内容：**
- `{Entity}Repository.ts` - 实体的数据库操作

**命名：** PascalCase + Repository

**职责：**
- Prisma 查询操作
- 查询优化
- 数据库错误处理
- 缓存层
- 隐藏 Prisma 实现细节

**当前差距：** 仅存在 1 个仓库（WorkflowRepository）

### Routes 目录

**目的：** 仅路由注册

**内容：**
- `{feature}Routes.ts` - 功能的 Express 路由器

**命名：** camelCase + Routes

**职责：**
- 使用 Express 注册路由
- 应用中间件
- 委托给控制器
- **无业务逻辑！**

### Middleware 目录

**目的：** 横切关注点

**内容：**
- 认证中间件
- 审计中间件
- 错误边界
- 验证中间件
- 自定义中间件

**命名：** camelCase

**类型：**
- 请求处理（处理器之前）
- 响应处理（处理器之后）
- 错误处理（错误边界）

### Config 目录

**目的：** 配置管理

**内容：**
- `unifiedConfig.ts` - 类型安全配置
- 特定环境配置

**模式：** 单一事实来源

### Types 目录

**目的：** TypeScript 类型定义

**内容：**
- `{feature}.types.ts` - 特定功能类型
- DTO（数据传输对象）
- Request/Response 类型
- 领域模型

---

## 模块组织

### 基于功能的组织

对于大型功能，使用子目录：

```
src/workflow/
├── core/              # 核心引擎
├── services/          # 工作流特定服务
├── actions/           # 系统动作
├── models/            # 领域模型
├── validators/        # 工作流验证
└── utils/             # 工作流工具
```

**何时使用：**
- 功能有 5+ 个文件
- 存在清晰的子域
- 逻辑分组提高清晰度

### 扁平组织

对于简单功能：

```
src/
├── controllers/UserController.ts
├── services/userService.ts
├── routes/userRoutes.ts
└── repositories/UserRepository.ts
```

**何时使用：**
- 简单功能（< 5 个文件）
- 无清晰的子域
- 扁平结构更清晰

---

## 关注点分离

### 什么放在哪里

**路由层：**
- ✅ 路由定义
- ✅ 中间件注册
- ✅ 控制器委托
- ❌ 业务逻辑
- ❌ 数据库操作
- ❌ 验证逻辑（应该在验证器或控制器中）

**控制器层：**
- ✅ 请求解析（params、body、query）
- ✅ 输入验证（Zod）
- ✅ 服务调用
- ✅ 响应格式化
- ✅ 错误处理
- ❌ 业务逻辑
- ❌ 数据库操作

**服务层：**
- ✅ 业务逻辑
- ✅ 业务规则执行
- ✅ 编排（多个仓库）
- ✅ 事务管理
- ❌ HTTP 关注点（Request/Response）
- ❌ 直接 Prisma 调用（使用仓库）

**仓库层：**
- ✅ Prisma 操作
- ✅ 查询构建
- ✅ 数据库错误处理
- ✅ 缓存
- ❌ 业务逻辑
- ❌ HTTP 关注点

### 示例：用户创建

**路由：**
```typescript
router.post('/users',
    SSOMiddleware.verifyLoginStatus,
    auditMiddleware,
    (req, res) => userController.create(req, res)
);
```

**控制器：**
```typescript
async create(req: Request, res: Response): Promise<void> {
    try {
        const validated = createUserSchema.parse(req.body);
        const user = await this.userService.create(validated);
        this.handleSuccess(res, user, 'User created');
    } catch (error) {
        this.handleError(error, res, 'create');
    }
}
```

**服务：**
```typescript
async create(data: CreateUserDTO): Promise<User> {
    // 业务规则：检查邮箱是否已存在
    const existing = await this.userRepository.findByEmail(data.email);
    if (existing) throw new ConflictError('Email already exists');

    // 创建用户
    return await this.userRepository.create(data);
}
```

**仓库：**
```typescript
async create(data: CreateUserDTO): Promise<User> {
    return PrismaService.main.user.create({ data });
}

async findByEmail(email: string): Promise<User | null> {
    return PrismaService.main.user.findUnique({ where: { email } });
}
```

**注意：** 每一层都有清晰、独特的职责！

---

**相关文件：**
- [SKILL.md](SKILL.md) - 主指南
- [routing-and-controllers.md](routing-and-controllers.md) - 路由和控制器详细信息
- [services-and-repositories.md](services-and-repositories.md) - 服务和仓库模式
