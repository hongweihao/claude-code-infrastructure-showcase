---
name: backend-dev-guidelines
description: Node.js/Express/TypeScript 微服务的综合后端开发指南。在创建路由、控制器、服务、仓库、中间件或使用 Express API、Prisma 数据库访问、Sentry 错误跟踪、Zod 验证、unifiedConfig、依赖注入或异步模式时使用。涵盖分层架构（routes → controllers → services → repositories）、BaseController 模式、错误处理、性能监控、测试策略以及从遗留模式的迁移。
---

# 后端开发指南

## 目的

在使用现代 Node.js/Express/TypeScript 模式的后端微服务（blog-api、auth-service、notifications-service）中建立一致性和最佳实践。

## 何时使用此技能

在以下工作中自动激活：
- 创建或修改路由、端点、API
- 构建控制器、服务、仓库
- 实现中间件（认证、验证、错误处理）
- 使用 Prisma 进行数据库操作
- 使用 Sentry 进行错误跟踪
- 使用 Zod 进行输入验证
- 配置管理
- 后端测试和重构

---

## 快速开始

### 新后端功能清单

- [ ] **路由**：清晰定义，委托给控制器
- [ ] **控制器**：扩展 BaseController
- [ ] **服务**：带有 DI 的业务逻辑
- [ ] **仓库**：数据库访问（如果复杂）
- [ ] **验证**：Zod schema
- [ ] **Sentry**：错误跟踪
- [ ] **测试**：单元 + 集成测试
- [ ] **配置**：使用 unifiedConfig

### 新微服务清单

- [ ] 目录结构（见 [architecture-overview.md](architecture-overview.md)）
- [ ] Sentry 的 instrument.ts
- [ ] unifiedConfig 设置
- [ ] BaseController 类
- [ ] 中间件栈
- [ ] 错误边界
- [ ] 测试框架

---

## 架构概览

### 分层架构

```
HTTP 请求
    ↓
路由（仅路由）
    ↓
控制器（请求处理）
    ↓
服务（业务逻辑）
    ↓
仓库（数据访问）
    ↓
数据库（Prisma）
```

**关键原则：** 每一层只有一个职责。

详细信息请参见 [architecture-overview.md](architecture-overview.md)。

---

## 目录结构

```
service/src/
├── config/              # UnifiedConfig
├── controllers/         # 请求处理器
├── services/            # 业务逻辑
├── repositories/        # 数据访问
├── routes/              # 路由定义
├── middleware/          # Express 中间件
├── types/               # TypeScript 类型
├── validators/          # Zod schemas
├── utils/               # 工具函数
├── tests/               # 测试
├── instrument.ts        # Sentry（第一个导入）
├── app.ts               # Express 设置
└── server.ts            # HTTP 服务器
```

**命名约定：**
- 控制器：`PascalCase` - `UserController.ts`
- 服务：`camelCase` - `userService.ts`
- 路由：`camelCase + Routes` - `userRoutes.ts`
- 仓库：`PascalCase + Repository` - `UserRepository.ts`

---

## 核心原则（7 条关键规则）

### 1. 路由只负责路由，控制器负责控制

```typescript
// ❌ 永远不要：路由中的业务逻辑
router.post('/submit', async (req, res) => {
    // 200 行逻辑
});

// ✅ 总是：委托给控制器
router.post('/submit', (req, res) => controller.submit(req, res));
```

### 2. 所有控制器都扩展 BaseController

```typescript
export class UserController extends BaseController {
    async getUser(req: Request, res: Response): Promise<void> {
        try {
            const user = await this.userService.findById(req.params.id);
            this.handleSuccess(res, user);
        } catch (error) {
            this.handleError(error, res, 'getUser');
        }
    }
}
```

### 3. 所有错误到 Sentry

```typescript
try {
    await operation();
} catch (error) {
    Sentry.captureException(error);
    throw error;
}
```

### 4. 使用 unifiedConfig，永不使用 process.env

```typescript
// ❌ 永远不要
const timeout = process.env.TIMEOUT_MS;

// ✅ 总是
import { config } from './config/unifiedConfig';
const timeout = config.timeouts.default;
```

### 5. 使用 Zod 验证所有输入

```typescript
const schema = z.object({ email: z.string().email() });
const validated = schema.parse(req.body);
```

### 6. 使用仓库模式进行数据访问

```typescript
// 服务 → 仓库 → 数据库
const users = await userRepository.findActive();
```

### 7. 需要全面测试

```typescript
describe('UserService', () => {
    it('should create user', async () => {
        expect(user).toBeDefined();
    });
});
```

---

## 常见导入

```typescript
// Express
import express, { Request, Response, NextFunction, Router } from 'express';

// 验证
import { z } from 'zod';

// 数据库
import { PrismaClient } from '@prisma/client';
import type { Prisma } from '@prisma/client';

// Sentry
import * as Sentry from '@sentry/node';

// 配置
import { config } from './config/unifiedConfig';

// 中间件
import { SSOMiddlewareClient } from './middleware/SSOMiddleware';
import { asyncErrorWrapper } from './middleware/errorBoundary';
```

---

## 快速参考

### HTTP 状态码

| 代码 | 用途 |
|------|------|
| 200 | 成功 |
| 201 | 已创建 |
| 400 | 错误请求 |
| 401 | 未授权 |
| 403 | 禁止访问 |
| 404 | 未找到 |
| 500 | 服务器错误 |

### 服务模板

**Blog API**（✅ 成熟）- 用作 REST API 的模板
**Auth Service**（✅ 成熟）- 用作认证模式的模板

---

## 要避免的反模式

❌ 路由中的业务逻辑
❌ 直接使用 process.env
❌ 缺少错误处理
❌ 无输入验证
❌ 到处直接使用 Prisma
❌ 使用 console.log 而不是 Sentry

---

## 导航指南

| 需要... | 阅读此文档 |
|---------|-----------|
| 理解架构 | [architecture-overview.md](architecture-overview.md) |
| 创建路由/控制器 | [routing-and-controllers.md](routing-and-controllers.md) |
| 组织业务逻辑 | [services-and-repositories.md](services-and-repositories.md) |
| 验证输入 | [validation-patterns.md](validation-patterns.md) |
| 添加错误跟踪 | [sentry-and-monitoring.md](sentry-and-monitoring.md) |
| 创建中间件 | [middleware-guide.md](middleware-guide.md) |
| 数据库访问 | [database-patterns.md](database-patterns.md) |
| 管理配置 | [configuration.md](configuration.md) |
| 处理异步/错误 | [async-and-errors.md](async-and-errors.md) |
| 编写测试 | [testing-guide.md](testing-guide.md) |
| 查看示例 | [complete-examples.md](complete-examples.md) |

---

## 资源文件

### [architecture-overview.md](architecture-overview.md)
分层架构、请求生命周期、关注点分离

### [routing-and-controllers.md](routing-and-controllers.md)
路由定义、BaseController、错误处理、示例

### [services-and-repositories.md](services-and-repositories.md)
服务模式、DI、仓库模式、缓存

### [validation-patterns.md](validation-patterns.md)
Zod schemas、验证、DTO 模式

### [sentry-and-monitoring.md](sentry-and-monitoring.md)
Sentry 初始化、错误捕获、性能监控

### [middleware-guide.md](middleware-guide.md)
认证、审计、错误边界、AsyncLocalStorage

### [database-patterns.md](database-patterns.md)
PrismaService、仓库、事务、优化

### [configuration.md](configuration.md)
UnifiedConfig、环境配置、密钥

### [async-and-errors.md](async-and-errors.md)
异步模式、自定义错误、asyncErrorWrapper

### [testing-guide.md](testing-guide.md)
单元/集成测试、模拟、覆盖率

### [complete-examples.md](complete-examples.md)
完整示例、重构指南

---

## 相关技能

- **database-verification** - 验证列名和模式一致性
- **error-tracking** - Sentry 集成模式
- **skill-developer** - 创建和管理技能的元技能

---

**技能状态**：完成 ✅
**行数**：< 500 ✅
**渐进式披露**：11 个资源文件 ✅
