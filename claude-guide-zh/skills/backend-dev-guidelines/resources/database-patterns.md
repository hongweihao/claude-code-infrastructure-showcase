# 数据库模式 - Prisma 最佳实践

在后端微服务中使用 Prisma 进行数据库访问模式的完整指南。

## 目录

- [PrismaService 使用](#prismaservice-使用)
- [仓库模式](#仓库模式)
- [事务模式](#事务模式)
- [查询优化](#查询优化)
- [N+1 查询防止](#n1-查询防止)
- [错误处理](#错误处理)

---

## PrismaService 使用

### 基本模式

```typescript
import { PrismaService } from '@project-lifecycle-portal/database';

// 始终使用 PrismaService.main
const users = await PrismaService.main.user.findMany();
```

### 检查可用性

```typescript
if (!PrismaService.isAvailable) {
    throw new Error('Prisma client not initialized');
}

const user = await PrismaService.main.user.findUnique({ where: { id } });
```

---

## 仓库模式

### 为什么使用仓库

✅ **在以下情况使用仓库：**
- 带有 join/include 的复杂查询
- 查询在多个地方使用
- 需要缓存层
- 想要模拟测试

❌ **在以下情况跳过仓库：**
- 简单的一次性查询
- 原型开发（可以稍后重构）

### 仓库模板

```typescript
export class UserRepository {
    async findById(id: string): Promise<User | null> {
        return PrismaService.main.user.findUnique({
            where: { id },
            include: { profile: true },
        });
    }

    async findActive(): Promise<User[]> {
        return PrismaService.main.user.findMany({
            where: { isActive: true },
            orderBy: { createdAt: 'desc' },
        });
    }

    async create(data: Prisma.UserCreateInput): Promise<User> {
        return PrismaService.main.user.create({ data });
    }
}
```

---

## 事务模式

### 简单事务

```typescript
const result = await PrismaService.main.$transaction(async (tx) => {
    const user = await tx.user.create({ data: userData });
    const profile = await tx.userProfile.create({ data: { userId: user.id } });
    return { user, profile };
});
```

### 交互式事务

```typescript
const result = await PrismaService.main.$transaction(
    async (tx) => {
        const user = await tx.user.findUnique({ where: { id } });
        if (!user) throw new Error('User not found');

        return await tx.user.update({
            where: { id },
            data: { lastLogin: new Date() },
        });
    },
    {
        maxWait: 5000,
        timeout: 10000,
    }
);
```

---

## 查询优化

### 使用 select 限制字段

```typescript
// ❌ 获取所有字段
const users = await PrismaService.main.user.findMany();

// ✅ 只获取需要的字段
const users = await PrismaService.main.user.findMany({
    select: {
        id: true,
        email: true,
        profile: { select: { firstName: true, lastName: true } },
    },
});
```

### 小心使用 include

```typescript
// ❌ 过度 include
const user = await PrismaService.main.user.findUnique({
    where: { id },
    include: {
        profile: true,
        posts: { include: { comments: true } },
        workflows: { include: { steps: { include: { actions: true } } } },
    },
});

// ✅ 只 include 你需要的
const user = await PrismaService.main.user.findUnique({
    where: { id },
    include: { profile: true },
});
```

---

## N+1 查询防止

### 问题：N+1 查询

```typescript
// ❌ N+1 查询问题
const users = await PrismaService.main.user.findMany(); // 1 个查询

for (const user of users) {
    // N 个查询（每个用户一个）
    const profile = await PrismaService.main.userProfile.findUnique({
        where: { userId: user.id },
    });
}
```

### 解决方案：使用 include 或批处理

```typescript
// ✅ 使用 include 的单个查询
const users = await PrismaService.main.user.findMany({
    include: { profile: true },
});

// ✅ 或批量查询
const userIds = users.map(u => u.id);
const profiles = await PrismaService.main.userProfile.findMany({
    where: { userId: { in: userIds } },
});
```

---

## 错误处理

### Prisma 错误类型

```typescript
import { Prisma } from '@prisma/client';

try {
    await PrismaService.main.user.create({ data });
} catch (error) {
    if (error instanceof Prisma.PrismaClientKnownRequestError) {
        // 唯一约束违反
        if (error.code === 'P2002') {
            throw new ConflictError('Email already exists');
        }

        // 外键约束
        if (error.code === 'P2003') {
            throw new ValidationError('Invalid reference');
        }

        // 记录未找到
        if (error.code === 'P2025') {
            throw new NotFoundError('Record not found');
        }
    }

    // 未知错误
    Sentry.captureException(error);
    throw error;
}
```

---

**相关文件：**
- [SKILL.md](SKILL.md)
- [services-and-repositories.md](services-and-repositories.md)
- [async-and-errors.md](async-and-errors.md)
