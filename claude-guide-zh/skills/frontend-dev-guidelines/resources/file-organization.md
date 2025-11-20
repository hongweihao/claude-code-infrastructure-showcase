# 文件组织

应用程序中可维护、可扩展的前端代码的正确文件和目录结构。

---

## features/ vs components/ 区分

### features/ 目录

**目的**：具有自己的逻辑、API 和组件的特定领域功能

**何时使用：**
- 功能有多个相关组件
- 功能有自己的 API 端点
- 功能有特定领域的逻辑
- 功能有自定义 hooks/工具函数

**示例：**
- `features/posts/` - 项目目录/帖子管理
- `features/blogs/` - 博客构建器和渲染
- `features/auth/` - 认证流程

**结构：**
```
features/
  my-feature/
    api/
      myFeatureApi.ts         # API 服务层
    components/
      MyFeatureMain.tsx       # 主组件
      SubComponents/          # 相关组件
    hooks/
      useMyFeature.ts         # 自定义 hooks
      useSuspenseMyFeature.ts # Suspense hooks
    helpers/
      myFeatureHelpers.ts     # 工具函数
    types/
      index.ts                # TypeScript 类型
    index.ts                  # 公共导出
```

### components/ 目录

**目的**：跨多个功能使用的真正可重用组件

**何时使用：**
- 组件在 3+ 个地方使用
- 组件是通用的（没有功能特定的逻辑）
- 组件是 UI 基元或模式

**示例：**
- `components/SuspenseLoader/` - 加载包装器
- `components/CustomAppBar/` - 应用程序头部
- `components/ErrorBoundary/` - 错误处理
- `components/LoadingOverlay/` - 加载覆盖层

**结构：**
```
components/
  SuspenseLoader/
    SuspenseLoader.tsx
    SuspenseLoader.test.tsx
  CustomAppBar/
    CustomAppBar.tsx
    CustomAppBar.test.tsx
```

---

## 功能目录结构（详细）

### 完整功能示例

基于 `features/posts/` 结构：

```
features/
  posts/
    api/
      postApi.ts              # API 服务层（GET、POST、PUT、DELETE）

    components/
      PostTable.tsx           # 主容器组件
      grids/
        PostDataGrid/
          PostDataGrid.tsx
      drawers/
        ProjectPostDrawer/
          ProjectPostDrawer.tsx
      cells/
        editors/
          TextEditCell.tsx
        renderers/
          DateCell.tsx
      toolbar/
        CustomToolbar.tsx

    hooks/
      usePostQueries.ts       # 常规查询
      useSuspensePost.ts      # Suspense 查询
      usePostMutations.ts     # Mutations
      useGridLayout.ts              # 功能特定的 hooks

    helpers/
      postHelpers.ts          # 工具函数
      validation.ts                 # 验证逻辑

    types/
      index.ts                      # TypeScript 类型/接口

    queries/
      postQueries.ts          # 查询键工厂（可选）

    context/
      PostContext.tsx         # React context（如果需要）

    index.ts                        # 公共 API 导出
```

### 子目录指南

#### api/ 目录

**目的**：功能的集中式 API 调用

**文件：**
- `{feature}Api.ts` - 主 API 服务

**模式：**
```typescript
// features/my-feature/api/myFeatureApi.ts
import apiClient from '@/lib/apiClient';

export const myFeatureApi = {
    getItem: async (id: number) => {
        const { data } = await apiClient.get(`/blog/items/${id}`);
        return data;
    },
    createItem: async (payload) => {
        const { data } = await apiClient.post('/blog/items', payload);
        return data;
    },
};
```

#### components/ 目录

**目的**：功能特定的组件

**组织：**
- 如果 <5 个组件则采用扁平结构
- 如果 >5 个组件则按职责划分子目录

**示例：**
```
components/
  MyFeatureMain.tsx           # 主组件
  MyFeatureHeader.tsx         # 支持组件
  MyFeatureFooter.tsx

  # 或者使用子目录：
  containers/
    MyFeatureContainer.tsx
  presentational/
    MyFeatureDisplay.tsx
  blogs/
    MyFeatureBlog.tsx
```

#### hooks/ 目录

**目的**：功能的自定义 hooks

**命名：**
- `use` 前缀（camelCase）
- 描述它们的作用

**示例：**
```
hooks/
  useMyFeature.ts               # 主 hook
  useSuspenseMyFeature.ts       # Suspense 版本
  useMyFeatureMutations.ts      # Mutations
  useMyFeatureFilters.ts        # 过滤器/搜索
```

#### helpers/ 目录

**目的**：功能特定的工具函数

**示例：**
```
helpers/
  myFeatureHelpers.ts           # 通用工具
  validation.ts                 # 验证逻辑
  transblogers.ts               # 数据转换
  constants.ts                  # 常量
```

#### types/ 目录

**目的**：TypeScript 类型和接口

**文件：**
```
types/
  index.ts                      # 主类型，已导出
  internal.ts                   # 内部类型（未导出）
```

---

## 导入别名（Vite 配置）

### 可用别名

来自 `vite.config.ts` 第 180-185 行：

| 别名 | 解析为 | 用于 |
|-------|-------------|---------|
| `@/` | `src/` | 从 src 根目录的绝对导入 |
| `~types` | `src/types` | 共享的 TypeScript 类型 |
| `~components` | `src/components` | 可重用组件 |
| `~features` | `src/features` | 功能导入 |

### 使用示例

```typescript
// ✅ 首选 - 对绝对导入使用别名
import { apiClient } from '@/lib/apiClient';
import { SuspenseLoader } from '~components/SuspenseLoader';
import { postApi } from '~features/posts/api/postApi';
import type { User } from '~types/user';

// ❌ 避免 - 来自深层嵌套的相对路径
import { apiClient } from '../../../lib/apiClient';
import { SuspenseLoader } from '../../../components/SuspenseLoader';
```

### 何时使用哪个别名

**@/（通用）**：
- Lib 工具：`@/lib/apiClient`
- Hooks：`@/hooks/useAuth`
- 配置：`@/config/theme`
- 共享服务：`@/services/authService`

**~types（类型导入）**：
```typescript
import type { Post } from '~types/post';
import type { User, UserRole } from '~types/user';
```

**~components（可重用组件）**：
```typescript
import { SuspenseLoader } from '~components/SuspenseLoader';
import { CustomAppBar } from '~components/CustomAppBar';
import { ErrorBoundary } from '~components/ErrorBoundary';
```

**~features（功能导入）**：
```typescript
import { postApi } from '~features/posts/api/postApi';
import { useAuth } from '~features/auth/hooks/useAuth';
```

---

## 文件命名约定

### 组件

**模式**：PascalCase，`.tsx` 扩展名

```
MyComponent.tsx
PostDataGrid.tsx
CustomAppBar.tsx
```

**避免：**
- camelCase：`myComponent.tsx` ❌
- kebab-case：`my-component.tsx` ❌
- 全大写：`MYCOMPONENT.tsx` ❌

### Hooks

**模式**：camelCase，`use` 前缀，`.ts` 扩展名

```
useMyFeature.ts
useSuspensePost.ts
useAuth.ts
useGridLayout.ts
```

### API 服务

**模式**：camelCase，`Api` 后缀，`.ts` 扩展名

```
myFeatureApi.ts
postApi.ts
userApi.ts
```

### Helpers/工具函数

**模式**：camelCase，描述性名称，`.ts` 扩展名

```
myFeatureHelpers.ts
validation.ts
transblogers.ts
constants.ts
```

### 类型

**模式**：camelCase，`index.ts` 或描述性名称

```
types/index.ts
types/post.ts
types/user.ts
```

---

## 何时创建新功能

### 在以下情况下创建新功能：

- 多个相关组件（>3）
- 有自己的 API 端点
- 特定领域的逻辑
- 随时间增长
- 跨多个路由重用

**示例：** `features/posts/`
- 20+ 个组件
- 自己的 API 服务
- 复杂的状态管理
- 在多个路由中使用

### 在以下情况下添加到现有功能：

- 与现有功能相关
- 共享相同的 API
- 逻辑分组
- 扩展现有功能

**示例：** 向 posts 功能添加导出对话框

### 在以下情况下创建可重用组件：

- 跨 3+ 个功能使用
- 通用，无领域逻辑
- 纯展示
- 共享模式

**示例：** `components/SuspenseLoader/`

---

## 导入组织

### 导入顺序（推荐）

```typescript
// 1. React 和 React 相关
import React, { useState, useCallback, useMemo } from 'react';
import { lazy } from 'react';

// 2. 第三方库（按字母顺序）
import { Box, Paper, Button, Grid } from '@mui/material';
import type { SxProps, Theme } from '@mui/material';
import { useSuspenseQuery, useQueryClient } from '@tanstack/react-query';
import { createFileRoute } from '@tanstack/react-router';

// 3. 别名导入（@ 在前，然后是 ~）
import { apiClient } from '@/lib/apiClient';
import { useAuth } from '@/hooks/useAuth';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';
import { SuspenseLoader } from '~components/SuspenseLoader';
import { postApi } from '~features/posts/api/postApi';

// 4. 类型导入（分组）
import type { Post } from '~types/post';
import type { User } from '~types/user';

// 5. 相对导入（同一功能）
import { MySubComponent } from './MySubComponent';
import { useMyFeature } from '../hooks/useMyFeature';
import { myFeatureHelpers } from '../helpers/myFeatureHelpers';
```

**对所有导入使用单引号**（项目标准）

---

## 公共 API 模式

### feature/index.ts

从功能导出公共 API 以实现清晰的导入：

```typescript
// features/my-feature/index.ts

// 导出主组件
export { MyFeatureMain } from './components/MyFeatureMain';
export { MyFeatureHeader } from './components/MyFeatureHeader';

// 导出 hooks
export { useMyFeature } from './hooks/useMyFeature';
export { useSuspenseMyFeature } from './hooks/useSuspenseMyFeature';

// 导出 API
export { myFeatureApi } from './api/myFeatureApi';

// 导出类型
export type { MyFeatureData, MyFeatureConfig } from './types';
```

**使用：**
```typescript
// ✅ 从功能索引清晰导入
import { MyFeatureMain, useMyFeature } from '~features/my-feature';

// ❌ 避免深层导入（但如果需要可以）
import { MyFeatureMain } from '~features/my-feature/components/MyFeatureMain';
```

---

## 目录结构可视化

```
src/
├── features/                    # 特定领域功能
│   ├── posts/
│   │   ├── api/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── helpers/
│   │   ├── types/
│   │   └── index.ts
│   ├── blogs/
│   └── auth/
│
├── components/                  # 可重用组件
│   ├── SuspenseLoader/
│   ├── CustomAppBar/
│   ├── ErrorBoundary/
│   └── LoadingOverlay/
│
├── routes/                      # TanStack Router 路由
│   ├── __root.tsx
│   ├── index.tsx
│   ├── project-catalog/
│   │   ├── index.tsx
│   │   └── create/
│   └── blogs/
│
├── hooks/                       # 共享 hooks
│   ├── useAuth.ts
│   ├── useMuiSnackbar.ts
│   └── useDebounce.ts
│
├── lib/                         # 共享工具
│   ├── apiClient.ts
│   └── utils.ts
│
├── types/                       # 共享 TypeScript 类型
│   ├── user.ts
│   ├── post.ts
│   └── common.ts
│
├── config/                      # 配置
│   └── theme.ts
│
└── App.tsx                      # 根组件
```

---

## 总结

**关键原则：**
1. **features/** 用于特定领域的代码
2. **components/** 用于真正可重用的 UI
3. 使用子目录：api/、components/、hooks/、helpers/、types/
4. 导入别名用于清晰的导入（@/、~types、~components、~features）
5. 一致的命名：PascalCase 组件，camelCase 工具
6. 从功能 index.ts 导出公共 API

**另请参阅：**
- [component-patterns.md](component-patterns.md) - 组件结构
- [data-fetching.md](data-fetching.md) - API 服务模式
- [complete-examples.md](complete-examples.md) - 完整功能示例
