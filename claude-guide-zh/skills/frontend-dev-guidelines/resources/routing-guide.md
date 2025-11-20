# 路由指南

TanStack Router 实现，包括基于文件夹的路由和延迟加载模式。

---

## TanStack Router 概述

**TanStack Router** 与基于文件的路由：
- 文件夹结构定义路由
- 用于代码分割的延迟加载
- 类型安全的路由
- 面包屑加载器

---

## 基于文件夹的路由

### 目录结构

```
routes/
  __root.tsx                    # 根布局
  index.tsx                     # 主页路由 (/)
  posts/
    index.tsx                   # /posts
    create/
      index.tsx                 # /posts/create
    $postId.tsx                 # /posts/:postId（动态）
  comments/
    index.tsx                   # /comments
```

**模式**：
- `index.tsx` = 该路径的路由
- `$param.tsx` = 动态参数
- 嵌套文件夹 = 嵌套路由

---

## 基本路由模式

### 来自 posts/index.tsx 的示例

```typescript
/**
 * Posts 路由组件
 * 显示主博客文章列表
 */

import { createFileRoute } from '@tanstack/react-router';
import { lazy } from 'react';

// 延迟加载页面组件
const PostsList = lazy(() =>
    import('@/features/posts/components/PostsList').then(
        (module) => ({ default: module.PostsList }),
    ),
);

export const Route = createFileRoute('/posts/')({
    component: PostsPage,
    // 定义面包屑数据
    loader: () => ({
        crumb: 'Posts',
    }),
});

function PostsPage() {
    return (
        <PostsList
            title='All Posts'
            showFilters={true}
        />
    );
}

export default PostsPage;
```

**关键点：**
- 延迟加载重型组件
- 带路由路径的 `createFileRoute`
- 用于面包屑数据的 `loader`
- 页面组件渲染内容
- 导出 Route 和组件

---

## 延迟加载路由

### 命名导出模式

```typescript
import { lazy } from 'react';

// 对于命名导出，使用 .then() 映射到 default
const MyPage = lazy(() =>
    import('@/features/my-feature/components/MyPage').then(
        (module) => ({ default: module.MyPage })
    )
);
```

### 默认导出模式

```typescript
import { lazy } from 'react';

// 对于默认导出，语法更简单
const MyPage = lazy(() => import('@/features/my-feature/components/MyPage'));
```

### 为什么延迟加载路由？

- 代码分割 - 更小的初始打包
- 更快的初始页面加载
- 仅在导航时加载路由代码
- 更好的性能

---

## createFileRoute

### 基本配置

```typescript
export const Route = createFileRoute('/my-route/')({
    component: MyRoutePage,
});

function MyRoutePage() {
    return <div>My Route Content</div>;
}
```

### 带面包屑加载器

```typescript
export const Route = createFileRoute('/my-route/')({
    component: MyRoutePage,
    loader: () => ({
        crumb: 'My Route Title',
    }),
});
```

面包屑自动出现在导航/应用栏中。

### 带数据加载器

```typescript
export const Route = createFileRoute('/my-route/')({
    component: MyRoutePage,
    loader: async () => {
        // 可以在这里预取数据
        const data = await api.getData();
        return { crumb: 'My Route', data };
    },
});
```

### 带搜索参数

```typescript
export const Route = createFileRoute('/search/')({
    component: SearchPage,
    validateSearch: (search: Record<string, unknown>) => {
        return {
            query: (search.query as string) || '',
            page: Number(search.page) || 1,
        };
    },
});

function SearchPage() {
    const { query, page } = Route.useSearch();
    // 使用 query 和 page
}
```

---

## 动态路由

### 参数路由

```typescript
// routes/users/$userId.tsx

export const Route = createFileRoute('/users/$userId')({
    component: UserPage,
});

function UserPage() {
    const { userId } = Route.useParams();

    return <UserProfile userId={userId} />;
}
```

### 多个参数

```typescript
// routes/posts/$postId/comments/$commentId.tsx

export const Route = createFileRoute('/posts/$postId/comments/$commentId')({
    component: CommentPage,
});

function CommentPage() {
    const { postId, commentId } = Route.useParams();

    return <CommentEditor postId={postId} commentId={commentId} />;
}
```

---

## 导航

### 编程式导航

```typescript
import { useNavigate } from '@tanstack/react-router';

export const MyComponent: React.FC = () => {
    const navigate = useNavigate();

    const handleClick = () => {
        navigate({ to: '/posts' });
    };

    return <Button onClick={handleClick}>View Posts</Button>;
};
```

### 带参数

```typescript
const handleNavigate = () => {
    navigate({
        to: '/users/$userId',
        params: { userId: '123' },
    });
};
```

### 带搜索参数

```typescript
const handleSearch = () => {
    navigate({
        to: '/search',
        search: { query: 'test', page: 1 },
    });
};
```

---

## 路由布局模式

### 根布局 (__root.tsx)

```typescript
import { createRootRoute, Outlet } from '@tanstack/react-router';
import { Box } from '@mui/material';
import { CustomAppBar } from '~components/CustomAppBar';

export const Route = createRootRoute({
    component: RootLayout,
});

function RootLayout() {
    return (
        <Box>
            <CustomAppBar />
            <Box sx={{ p: 2 }}>
                <Outlet />  {/* 子路由在这里渲染 */}
            </Box>
        </Box>
    );
}
```

### 嵌套布局

```typescript
// routes/dashboard/index.tsx
export const Route = createFileRoute('/dashboard/')({
    component: DashboardLayout,
});

function DashboardLayout() {
    return (
        <Box>
            <DashboardSidebar />
            <Box sx={{ flex: 1 }}>
                <Outlet />  {/* 嵌套路由 */}
            </Box>
        </Box>
    );
}
```

---

## 完整路由示例

```typescript
/**
 * 用户资料路由
 * 路径：/users/:userId
 */

import { createFileRoute } from '@tanstack/react-router';
import { lazy } from 'react';
import { SuspenseLoader } from '~components/SuspenseLoader';

// 延迟加载重型组件
const UserProfile = lazy(() =>
    import('@/features/users/components/UserProfile').then(
        (module) => ({ default: module.UserProfile })
    )
);

export const Route = createFileRoute('/users/$userId')({
    component: UserPage,
    loader: () => ({
        crumb: 'User Profile',
    }),
});

function UserPage() {
    const { userId } = Route.useParams();

    return (
        <SuspenseLoader>
            <UserProfile userId={userId} />
        </SuspenseLoader>
    );
}

export default UserPage;
```

---

## 总结

**路由检查清单：**
- ✅ 基于文件夹：`routes/my-route/index.tsx`
- ✅ 延迟加载组件：`React.lazy(() => import())`
- ✅ 使用带路由路径的 `createFileRoute`
- ✅ 在 `loader` 函数中添加面包屑
- ✅ 用 `SuspenseLoader` 包装以处理加载状态
- ✅ 对动态参数使用 `Route.useParams()`
- ✅ 对编程式导航使用 `useNavigate()`

**另请参阅：**
- [component-patterns.md](component-patterns.md) - 延迟加载模式
- [loading-and-error-states.md](loading-and-error-states.md) - SuspenseLoader 使用
- [complete-examples.md](complete-examples.md) - 完整路由示例
