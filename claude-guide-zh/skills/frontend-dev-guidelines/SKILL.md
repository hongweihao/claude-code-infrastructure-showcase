---
name: frontend-dev-guidelines
description: React/TypeScript 应用的前端开发指南。包括 Suspense、延迟加载、useSuspenseQuery、features 目录文件组织、MUI v7 样式、TanStack Router、性能优化和 TypeScript 最佳实践的现代模式。在创建组件、页面、功能、获取数据、样式设置、路由或处理前端代码时使用。
---

# 前端开发指南

## 目的

现代 React 开发的综合指南,强调基于 Suspense 的数据获取、延迟加载、正确的文件组织和性能优化。

## 何时使用此技能

- 创建新组件或页面
- 构建新功能
- 使用 TanStack Query 获取数据
- 使用 TanStack Router 设置路由
- 使用 MUI v7 设置组件样式
- 性能优化
- 组织前端代码
- TypeScript 最佳实践

---

## 快速入门

### 新组件检查清单

创建组件？遵循此检查清单：

- [ ] 使用带 TypeScript 的 `React.FC<Props>` 模式
- [ ] 如果是重型组件则延迟加载：`React.lazy(() => import())`
- [ ] 用 `<SuspenseLoader>` 包装以处理加载状态
- [ ] 使用 `useSuspenseQuery` 获取数据
- [ ] 导入别名：`@/`、`~types`、`~components`、`~features`
- [ ] 样式：<100 行内联，>100 行单独文件
- [ ] 对传递给子组件的事件处理程序使用 `useCallback`
- [ ] 底部默认导出
- [ ] 不使用带加载旋转器的提前返回
- [ ] 使用 `useMuiSnackbar` 进行用户通知

### 新功能检查清单

创建功能？设置此结构：

- [ ] 创建 `features/{feature-name}/` 目录
- [ ] 创建子目录：`api/`、`components/`、`hooks/`、`helpers/`、`types/`
- [ ] 创建 API 服务文件：`api/{feature}Api.ts`
- [ ] 在 `types/` 中设置 TypeScript 类型
- [ ] 在 `routes/{feature-name}/index.tsx` 中创建路由
- [ ] 延迟加载功能组件
- [ ] 使用 Suspense 边界
- [ ] 从功能的 `index.ts` 导出公共 API

---

## 导入别名快速参考

| 别名 | 解析为 | 示例 |
|-------|-------------|---------|
| `@/` | `src/` | `import { apiClient } from '@/lib/apiClient'` |
| `~types` | `src/types` | `import type { User } from '~types/user'` |
| `~components` | `src/components` | `import { SuspenseLoader } from '~components/SuspenseLoader'` |
| `~features` | `src/features` | `import { authApi } from '~features/auth'` |

定义位置：[vite.config.ts](../../vite.config.ts) 第 180-185 行

---

## 常用导入速查表

```typescript
// React 和延迟加载
import React, { useState, useCallback, useMemo } from 'react';
const Heavy = React.lazy(() => import('./Heavy'));

// MUI 组件
import { Box, Paper, Typography, Button, Grid } from '@mui/material';
import type { SxProps, Theme } from '@mui/material';

// TanStack Query (Suspense)
import { useSuspenseQuery, useQueryClient } from '@tanstack/react-query';

// TanStack Router
import { createFileRoute } from '@tanstack/react-router';

// 项目组件
import { SuspenseLoader } from '~components/SuspenseLoader';

// Hooks
import { useAuth } from '@/hooks/useAuth';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

// 类型
import type { Post } from '~types/post';
```

---

## 主题指南

### 🎨 组件模式

**现代 React 组件使用：**
- `React.FC<Props>` 实现类型安全
- `React.lazy()` 实现代码分割
- `SuspenseLoader` 处理加载状态
- 命名常量 + 默认导出模式

**关键概念：**
- 延迟加载重型组件（DataGrid、图表、编辑器）
- 始终将延迟组件包装在 Suspense 中
- 使用 SuspenseLoader 组件（带淡入动画）
- 组件结构：Props → Hooks → Handlers → Render → Export

**[📖 完整指南：resources/component-patterns.md](resources/component-patterns.md)**

---

### 📊 数据获取

**主要模式：useSuspenseQuery**
- 与 Suspense 边界一起使用
- 缓存优先策略（在 API 之前检查网格缓存）
- 替代 `isLoading` 检查
- 使用泛型实现类型安全

**API 服务层：**
- 创建 `features/{feature}/api/{feature}Api.ts`
- 使用 `apiClient` axios 实例
- 每个功能的集中方法
- 路由格式：`/form/route`（不是 `/api/form/route`）

**[📖 完整指南：resources/data-fetching.md](resources/data-fetching.md)**

---

### 📁 文件组织

**features/ vs components/：**
- `features/`：特定领域（posts、comments、auth）
- `components/`：真正可重用（SuspenseLoader、CustomAppBar）

**功能子目录：**
```
features/
  my-feature/
    api/          # API 服务层
    components/   # 功能组件
    hooks/        # 自定义 hooks
    helpers/      # 工具函数
    types/        # TypeScript 类型
```

**[📖 完整指南：resources/file-organization.md](resources/file-organization.md)**

---

### 🎨 样式设置

**内联 vs 单独：**
- <100 行：内联 `const styles: Record<string, SxProps<Theme>>`
- >100 行：单独的 `.styles.ts` 文件

**主要方法：**
- 对 MUI 组件使用 `sx` prop
- 使用 `SxProps<Theme>` 实现类型安全
- 主题访问：`(theme) => theme.palette.primary.main`

**MUI v7 Grid：**
```typescript
<Grid size={{ xs: 12, md: 6 }}>  // ✅ v7 语法
<Grid xs={12} md={6}>             // ❌ 旧语法
```

**[📖 完整指南：resources/styling-guide.md](resources/styling-guide.md)**

---

### 🛣️ 路由

**TanStack Router - 基于文件夹：**
- 目录：`routes/my-route/index.tsx`
- 延迟加载组件
- 使用 `createFileRoute`
- 加载器中的面包屑数据

**示例：**
```typescript
import { createFileRoute } from '@tanstack/react-router';
import { lazy } from 'react';

const MyPage = lazy(() => import('@/features/my-feature/components/MyPage'));

export const Route = createFileRoute('/my-route/')({
    component: MyPage,
    loader: () => ({ crumb: 'My Route' }),
});
```

**[📖 完整指南：resources/routing-guide.md](resources/routing-guide.md)**

---

### ⏳ 加载和错误状态

**关键规则：不提前返回**

```typescript
// ❌ 永远不要 - 导致布局偏移
if (isLoading) {
    return <LoadingSpinner />;
}

// ✅ 始终 - 一致的布局
<SuspenseLoader>
    <Content />
</SuspenseLoader>
```

**原因：** 防止累积布局偏移（CLS），更好的用户体验

**错误处理：**
- 使用 `useMuiSnackbar` 进行用户反馈
- 永远不要使用 `react-toastify`
- TanStack Query `onError` 回调

**[📖 完整指南：resources/loading-and-error-states.md](resources/loading-and-error-states.md)**

---

### ⚡ 性能

**优化模式：**
- `useMemo`：昂贵的计算（filter、sort、map）
- `useCallback`：传递给子组件的事件处理程序
- `React.memo`：昂贵的组件
- 防抖搜索（300-500ms）
- 防止内存泄漏（useEffect 中清理）

**[📖 完整指南：resources/performance.md](resources/performance.md)**

---

### 📘 TypeScript

**标准：**
- 严格模式，不使用 `any` 类型
- 函数上的显式返回类型
- 类型导入：`import type { User } from '~types/user'`
- 带 JSDoc 的组件 prop 接口

**[📖 完整指南：resources/typescript-standards.md](resources/typescript-standards.md)**

---

### 🔧 常见模式

**涵盖的主题：**
- React Hook Form 与 Zod 验证
- DataGrid 包装器契约
- Dialog 组件标准
- 用于当前用户的 `useAuth` hook
- 带缓存失效的 Mutation 模式

**[📖 完整指南：resources/common-patterns.md](resources/common-patterns.md)**

---

### 📚 完整示例

**完整工作示例：**
- 包含所有模式的现代组件
- 完整的功能结构
- API 服务层
- 带延迟加载的路由
- Suspense + useSuspenseQuery
- 带验证的表单

**[📖 完整指南：resources/complete-examples.md](resources/complete-examples.md)**

---

## 导航指南

| 需要... | 阅读此资源 |
|------------|-------------------|
| 创建组件 | [component-patterns.md](resources/component-patterns.md) |
| 获取数据 | [data-fetching.md](resources/data-fetching.md) |
| 组织文件/文件夹 | [file-organization.md](resources/file-organization.md) |
| 设置组件样式 | [styling-guide.md](resources/styling-guide.md) |
| 设置路由 | [routing-guide.md](resources/routing-guide.md) |
| 处理加载/错误 | [loading-and-error-states.md](resources/loading-and-error-states.md) |
| 优化性能 | [performance.md](resources/performance.md) |
| TypeScript 类型 | [typescript-standards.md](resources/typescript-standards.md) |
| 表单/认证/DataGrid | [common-patterns.md](resources/common-patterns.md) |
| 查看完整示例 | [complete-examples.md](resources/complete-examples.md) |

---

## 核心原则

1. **延迟加载所有重型内容**：路由、DataGrid、图表、编辑器
2. **使用 Suspense 加载**：使用 SuspenseLoader，而不是提前返回
3. **useSuspenseQuery**：新代码的主要数据获取模式
4. **功能已组织**：api/、components/、hooks/、helpers/ 子目录
5. **基于大小的样式**：<100 内联，>100 单独
6. **导入别名**：使用 @/、~types、~components、~features
7. **不提前返回**：防止布局偏移
8. **useMuiSnackbar**：用于所有用户通知

---

## 快速参考：文件结构

```
src/
  features/
    my-feature/
      api/
        myFeatureApi.ts       # API 服务
      components/
        MyFeature.tsx         # 主组件
        SubComponent.tsx      # 相关组件
      hooks/
        useMyFeature.ts       # 自定义 hooks
        useSuspenseMyFeature.ts  # Suspense hooks
      helpers/
        myFeatureHelpers.ts   # 工具函数
      types/
        index.ts              # TypeScript 类型
      index.ts                # 公共导出

  components/
    SuspenseLoader/
      SuspenseLoader.tsx      # 可重用加载器
    CustomAppBar/
      CustomAppBar.tsx        # 可重用应用栏

  routes/
    my-route/
      index.tsx               # 路由组件
      create/
        index.tsx             # 嵌套路由
```

---

## 现代组件模板（快速复制）

```typescript
import React, { useState, useCallback } from 'react';
import { Box, Paper } from '@mui/material';
import { useSuspenseQuery } from '@tanstack/react-query';
import { featureApi } from '../api/featureApi';
import type { FeatureData } from '~types/feature';

interface MyComponentProps {
    id: number;
    onAction?: () => void;
}

export const MyComponent: React.FC<MyComponentProps> = ({ id, onAction }) => {
    const [state, setState] = useState<string>('');

    const { data } = useSuspenseQuery({
        queryKey: ['feature', id],
        queryFn: () => featureApi.getFeature(id),
    });

    const handleAction = useCallback(() => {
        setState('updated');
        onAction?.();
    }, [onAction]);

    return (
        <Box sx={{ p: 2 }}>
            <Paper sx={{ p: 3 }}>
                {/* 内容 */}
            </Paper>
        </Box>
    );
};

export default MyComponent;
```

有关完整示例，请参阅 [resources/complete-examples.md](resources/complete-examples.md)

---

## 相关技能

- **error-tracking**：使用 Sentry 进行错误跟踪（也适用于前端）
- **backend-dev-guidelines**：前端使用的后端 API 模式

---

**技能状态**：模块化结构，具有渐进式加载，以实现最佳上下文管理
