# 数据获取模式

使用 TanStack Query 进行现代数据获取，包括 Suspense 边界、缓存优先策略和集中式 API 服务。

---

## 主要模式：useSuspenseQuery

### 为什么使用 useSuspenseQuery？

对于**所有新组件**，使用 `useSuspenseQuery` 而不是常规的 `useQuery`：

**好处：**
- 不需要 `isLoading` 检查
- 与 Suspense 边界集成
- 更清晰的组件代码
- 一致的加载用户体验
- 通过错误边界更好地处理错误

### 基本模式

```typescript
import { useSuspenseQuery } from '@tanstack/react-query';
import { myFeatureApi } from '../api/myFeatureApi';

export const MyComponent: React.FC<Props> = ({ id }) => {
    // 没有 isLoading - Suspense 处理它！
    const { data } = useSuspenseQuery({
        queryKey: ['myEntity', id],
        queryFn: () => myFeatureApi.getEntity(id),
    });

    // 在这里 data 总是已定义（不是 undefined | Data）
    return <div>{data.name}</div>;
};

// 在 Suspense 边界中包装
<SuspenseLoader>
    <MyComponent id={123} />
</SuspenseLoader>
```

### useSuspenseQuery vs useQuery

| 功能 | useSuspenseQuery | useQuery |
|---------|------------------|----------|
| 加载状态 | 由 Suspense 处理 | 手动 `isLoading` 检查 |
| 数据类型 | 总是已定义 | `Data \| undefined` |
| 与之配合使用 | Suspense 边界 | 传统组件 |
| 推荐用于 | **新组件** | 仅遗留代码 |
| 错误处理 | 错误边界 | 手动错误状态 |

**何时使用常规 useQuery：**
- 维护遗留代码
- 没有 Suspense 的非常简单的情况
- 具有后台更新的轮询

**对于新组件：始终首选 useSuspenseQuery**

---

## 缓存优先策略

### 缓存优先模式示例

**智能缓存**通过首先检查 React Query 缓存来减少 API 调用：

```typescript
import { useSuspenseQuery, useQueryClient } from '@tanstack/react-query';
import { postApi } from '../api/postApi';

export function useSuspensePost(postId: number) {
    const queryClient = useQueryClient();

    return useSuspenseQuery({
        queryKey: ['post', postId],
        queryFn: async () => {
            // 策略 1：首先尝试从列表缓存中获取
            const cachedListData = queryClient.getQueryData<{ posts: Post[] }>([
                'posts',
                'list'
            ]);

            if (cachedListData?.posts) {
                const cachedPost = cachedListData.posts.find(
                    (post) => post.id === postId
                );

                if (cachedPost) {
                    return cachedPost;  // 从缓存返回！
                }
            }

            // 策略 2：不在缓存中，从 API 获取
            return postApi.getPost(postId);
        },
        staleTime: 5 * 60 * 1000,      // 在 5 分钟内视为新鲜
        gcTime: 10 * 60 * 1000,         // 在缓存中保留 10 分钟
        refetchOnWindowFocus: false,    // 不在聚焦时重新获取
    });
}
```

**关键点：**
- 在 API 调用之前检查网格/列表缓存
- 避免冗余请求
- `staleTime`：数据被视为新鲜的时长
- `gcTime`：未使用的数据在缓存中保留的时长
- `refetchOnWindowFocus: false`：用户偏好

---

## 并行数据获取

### useSuspenseQueries

当获取多个独立资源时：

```typescript
import { useSuspenseQueries } from '@tanstack/react-query';

export const MyComponent: React.FC = () => {
    const [userQuery, settingsQuery, preferencesQuery] = useSuspenseQueries({
        queries: [
            {
                queryKey: ['user'],
                queryFn: () => userApi.getCurrentUser(),
            },
            {
                queryKey: ['settings'],
                queryFn: () => settingsApi.getSettings(),
            },
            {
                queryKey: ['preferences'],
                queryFn: () => preferencesApi.getPreferences(),
            },
        ],
    });

    // 所有数据可用，Suspense 处理加载
    const user = userQuery.data;
    const settings = settingsQuery.data;
    const preferences = preferencesQuery.data;

    return <Display user={user} settings={settings} prefs={preferences} />;
};
```

**好处：**
- 所有查询并行执行
- 单个 Suspense 边界
- 类型安全的结果

---

## 查询键组织

### 命名约定

```typescript
// 实体列表
['entities', blogId]
['entities', blogId, 'summary']    // 带视图模式
['entities', blogId, 'flat']

// 单个实体
['entity', blogId, entityId]

// 相关数据
['entity', entityId, 'history']
['entity', entityId, 'comments']

// 用户特定
['user', userId, 'profile']
['user', userId, 'permissions']
```

**规则：**
- 以实体名称开始（列表用复数，单个用单数）
- 包含 ID 以提高特异性
- 在末尾添加视图模式/关系
- 在整个应用中保持一致

### 查询键示例

```typescript
// 来自 useSuspensePost.ts
queryKey: ['post', blogId, postId]
queryKey: ['posts-v2', blogId, 'summary']

// 失效模式
queryClient.invalidateQueries({ queryKey: ['post', blogId] });  // 表单的所有帖子
queryClient.invalidateQueries({ queryKey: ['post'] });          // 所有帖子
```

---

## API 服务层模式

### 文件结构

为每个功能创建集中的 API 服务：

```
features/
  my-feature/
    api/
      myFeatureApi.ts    # 服务层
```

### 服务模式（来自 postApi.ts）

```typescript
/**
 * 用于 my-feature 操作的集中式 API 服务
 * 使用 apiClient 进行一致的错误处理
 */
import apiClient from '@/lib/apiClient';
import type { MyEntity, UpdatePayload } from '../types';

export const myFeatureApi = {
    /**
     * 获取单个实体
     */
    getEntity: async (blogId: number, entityId: number): Promise<MyEntity> => {
        const { data } = await apiClient.get(
            `/blog/entities/${blogId}/${entityId}`
        );
        return data;
    },

    /**
     * 获取表单的所有实体
     */
    getEntities: async (blogId: number, view: 'summary' | 'flat'): Promise<MyEntity[]> => {
        const { data } = await apiClient.get(
            `/blog/entities/${blogId}`,
            { params: { view } }
        );
        return data.rows;
    },

    /**
     * 更新实体
     */
    updateEntity: async (
        blogId: number,
        entityId: number,
        payload: UpdatePayload
    ): Promise<MyEntity> => {
        const { data } = await apiClient.put(
            `/blog/entities/${blogId}/${entityId}`,
            payload
        );
        return data;
    },

    /**
     * 删除实体
     */
    deleteEntity: async (blogId: number, entityId: number): Promise<void> => {
        await apiClient.delete(`/blog/entities/${blogId}/${entityId}`);
    },
};
```

**关键点：**
- 导出带方法的单个对象
- 使用 `apiClient`（来自 `@/lib/apiClient` 的 axios 实例）
- 类型安全的参数和返回
- 每个方法的 JSDoc 注释
- 集中式错误处理（apiClient 处理它）

---

## 路由格式规则（重要）

### 正确格式

```typescript
// ✅ 正确 - 直接服务路径
await apiClient.get('/blog/posts/123');
await apiClient.post('/projects/create', data);
await apiClient.put('/users/update/456', updates);
await apiClient.get('/email/templates');

// ❌ 错误 - 不要添加 /api/ 前缀
await apiClient.get('/api/blog/posts/123');  // 错误！
await apiClient.post('/api/projects/create', data); // 错误！
```

**微服务路由：**
- Form 服务：`/blog/*`
- Projects 服务：`/projects/*`
- Email 服务：`/email/*`
- Users 服务：`/users/*`

**原因：** API 路由由代理配置处理，不需要 `/api/` 前缀。

---

## Mutations

### 基本 Mutation 模式

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { myFeatureApi } from '../api/myFeatureApi';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

export const MyComponent: React.FC = () => {
    const queryClient = useQueryClient();
    const { showSuccess, showError } = useMuiSnackbar();

    const updateMutation = useMutation({
        mutationFn: (payload: UpdatePayload) =>
            myFeatureApi.updateEntity(blogId, entityId, payload),

        onSuccess: () => {
            // 使失效并重新获取
            queryClient.invalidateQueries({
                queryKey: ['entity', blogId, entityId]
            });
            showSuccess('Entity updated successfully');
        },

        onError: (error) => {
            showError('Failed to update entity');
            console.error('Update error:', error);
        },
    });

    const handleUpdate = () => {
        updateMutation.mutate({ name: 'New Name' });
    };

    return (
        <Button
            onClick={handleUpdate}
            disabled={updateMutation.isPending}
        >
            {updateMutation.isPending ? 'Updating...' : 'Update'}
        </Button>
    );
};
```

### 乐观更新

```typescript
const updateMutation = useMutation({
    mutationFn: (payload) => myFeatureApi.update(id, payload),

    // 乐观更新
    onMutate: async (newData) => {
        // 取消传出的重新获取
        await queryClient.cancelQueries({ queryKey: ['entity', id] });

        // 快照当前值
        const previousData = queryClient.getQueryData(['entity', id]);

        // 乐观地更新
        queryClient.setQueryData(['entity', id], (old) => ({
            ...old,
            ...newData,
        }));

        // 返回回滚函数
        return { previousData };
    },

    // 错误时回滚
    onError: (err, newData, context) => {
        queryClient.setQueryData(['entity', id], context.previousData);
        showError('Update failed');
    },

    // 成功或错误后重新获取
    onSettled: () => {
        queryClient.invalidateQueries({ queryKey: ['entity', id] });
    },
});
```

---

## 高级查询模式

### 预取

```typescript
export function usePrefetchEntity() {
    const queryClient = useQueryClient();

    return (blogId: number, entityId: number) => {
        return queryClient.prefetchQuery({
            queryKey: ['entity', blogId, entityId],
            queryFn: () => myFeatureApi.getEntity(blogId, entityId),
            staleTime: 5 * 60 * 1000,
        });
    };
}

// 使用：在悬停时预取
<div onMouseEnter={() => prefetch(blogId, id)}>
    <Link to={`/entity/${id}`}>View</Link>
</div>
```

### 不获取的缓存访问

```typescript
export function useEntityFromCache(blogId: number, entityId: number) {
    const queryClient = useQueryClient();

    // 从缓存获取，如果缺失则不获取
    const directCache = queryClient.getQueryData<MyEntity>(['entity', blogId, entityId]);

    if (directCache) return directCache;

    // 尝试网格缓存
    const gridCache = queryClient.getQueryData<{ rows: MyEntity[] }>(['entities-v2', blogId]);

    return gridCache?.rows.find(row => row.id === entityId);
}
```

### 依赖查询

```typescript
// 首先获取用户，然后获取用户的设置
const { data: user } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => userApi.getUser(userId),
});

const { data: settings } = useSuspenseQuery({
    queryKey: ['user', userId, 'settings'],
    queryFn: () => settingsApi.getUserSettings(user.id),
    // 由于 Suspense，自动等待用户加载
});
```

---

## API 客户端配置

### 使用 apiClient

```typescript
import apiClient from '@/lib/apiClient';

// apiClient 是一个配置好的 axios 实例
// 自动包括：
// - 基本 URL 配置
// - 基于 Cookie 的认证
// - 错误拦截器
// - 响应转换器
```

**不要创建新的 axios 实例** - 为了一致性使用 apiClient。

---

## 查询中的错误处理

### onError 回调

```typescript
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

const { showError } = useMuiSnackbar();

const { data } = useSuspenseQuery({
    queryKey: ['entity', id],
    queryFn: () => myFeatureApi.getEntity(id),

    // 处理错误
    onError: (error) => {
        showError('Failed to load entity');
        console.error('Load error:', error);
    },
});
```

### 错误边界

与错误边界结合以实现全面的错误处理：

```typescript
import { ErrorBoundary } from 'react-error-boundary';

<ErrorBoundary
    fallback={<ErrorDisplay />}
    onError={(error) => console.error(error)}
>
    <SuspenseLoader>
        <ComponentWithSuspenseQuery />
    </SuspenseLoader>
</ErrorBoundary>
```

---

## 完整示例

### 示例 1：简单实体获取

```typescript
import React from 'react';
import { useSuspenseQuery } from '@tanstack/react-query';
import { Box, Typography } from '@mui/material';
import { userApi } from '../api/userApi';

interface UserProfileProps {
    userId: string;
}

export const UserProfile: React.FC<UserProfileProps> = ({ userId }) => {
    const { data: user } = useSuspenseQuery({
        queryKey: ['user', userId],
        queryFn: () => userApi.getUser(userId),
        staleTime: 5 * 60 * 1000,
    });

    return (
        <Box>
            <Typography variant='h5'>{user.name}</Typography>
            <Typography>{user.email}</Typography>
        </Box>
    );
};

// 与 Suspense 一起使用
<SuspenseLoader>
    <UserProfile userId='123' />
</SuspenseLoader>
```

### 示例 2：缓存优先策略

```typescript
import { useSuspenseQuery, useQueryClient } from '@tanstack/react-query';
import { postApi } from '../api/postApi';
import type { Post } from '../types';

/**
 * 带缓存优先策略的 Hook
 * 在 API 调用之前检查网格缓存
 */
export function useSuspensePost(blogId: number, postId: number) {
    const queryClient = useQueryClient();

    return useSuspenseQuery<Post, Error>({
        queryKey: ['post', blogId, postId],
        queryFn: async () => {
            // 1. 首先检查网格缓存
            const gridCache = queryClient.getQueryData<{ rows: Post[] }>([
                'posts-v2',
                blogId,
                'summary'
            ]) || queryClient.getQueryData<{ rows: Post[] }>([
                'posts-v2',
                blogId,
                'flat'
            ]);

            if (gridCache?.rows) {
                const cached = gridCache.rows.find(row => row.S_ID === postId);
                if (cached) {
                    return cached;  // 重用网格数据
                }
            }

            // 2. 不在缓存中，直接获取
            return postApi.getPost(blogId, postId);
        },
        staleTime: 5 * 60 * 1000,
        gcTime: 10 * 60 * 1000,
        refetchOnWindowFocus: false,
    });
}
```

**好处：**
- 避免重复的 API 调用
- 如果已加载则立即获得数据
- 如果未缓存则回退到 API

### 示例 3：并行获取

```typescript
import { useSuspenseQueries } from '@tanstack/react-query';

export const Dashboard: React.FC = () => {
    const [statsQuery, projectsQuery, notificationsQuery] = useSuspenseQueries({
        queries: [
            {
                queryKey: ['stats'],
                queryFn: () => statsApi.getStats(),
            },
            {
                queryKey: ['projects', 'active'],
                queryFn: () => projectsApi.getActiveProjects(),
            },
            {
                queryKey: ['notifications', 'unread'],
                queryFn: () => notificationsApi.getUnread(),
            },
        ],
    });

    return (
        <Box>
            <StatsCard data={statsQuery.data} />
            <ProjectsList projects={projectsQuery.data} />
            <Notifications items={notificationsQuery.data} />
        </Box>
    );
};
```

---

## 带缓存失效的 Mutations

### Update Mutation

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { postApi } from '../api/postApi';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

export const useUpdatePost = () => {
    const queryClient = useQueryClient();
    const { showSuccess, showError } = useMuiSnackbar();

    return useMutation({
        mutationFn: ({ blogId, postId, data }: UpdateParams) =>
            postApi.updatePost(blogId, postId, data),

        onSuccess: (data, variables) => {
            // 使特定帖子失效
            queryClient.invalidateQueries({
                queryKey: ['post', variables.blogId, variables.postId]
            });

            // 使列表失效以刷新网格
            queryClient.invalidateQueries({
                queryKey: ['posts-v2', variables.blogId]
            });

            showSuccess('Post updated');
        },

        onError: (error) => {
            showError('Failed to update post');
            console.error('Update error:', error);
        },
    });
};

// 使用
const updatePost = useUpdatePost();

const handleSave = () => {
    updatePost.mutate({
        blogId: 123,
        postId: 456,
        data: { responses: { '101': 'value' } }
    });
};
```

### Delete Mutation

```typescript
export const useDeletePost = () => {
    const queryClient = useQueryClient();
    const { showSuccess, showError } = useMuiSnackbar();

    return useMutation({
        mutationFn: ({ blogId, postId }: DeleteParams) =>
            postApi.deletePost(blogId, postId),

        onSuccess: (data, variables) => {
            // 手动从缓存中删除（乐观）
            queryClient.setQueryData<{ rows: Post[] }>(
                ['posts-v2', variables.blogId],
                (old) => ({
                    ...old,
                    rows: old?.rows.filter(row => row.S_ID !== variables.postId) || []
                })
            );

            showSuccess('Post deleted');
        },

        onError: (error, variables) => {
            // 回滚 - 重新获取以获得准确状态
            queryClient.invalidateQueries({
                queryKey: ['posts-v2', variables.blogId]
            });
            showError('Failed to delete post');
        },
    });
};
```

---

## 查询配置最佳实践

### 默认配置

```typescript
// 在 QueryClientProvider 设置中
const queryClient = new QueryClient({
    defaultOptions: {
        queries: {
            staleTime: 1000 * 60 * 5,        // 5 分钟
            gcTime: 1000 * 60 * 10,           // 10 分钟（以前是 cacheTime）
            refetchOnWindowFocus: false,       // 不在聚焦时重新获取
            refetchOnMount: false,             // 如果新鲜则不在挂载时重新获取
            retry: 1,                          // 重试失败的查询一次
        },
    },
});
```

### 每个查询的覆盖

```typescript
// 频繁变化的数据 - 更短的 staleTime
useSuspenseQuery({
    queryKey: ['notifications', 'unread'],
    queryFn: () => notificationApi.getUnread(),
    staleTime: 30 * 1000,  // 30 秒
});

// 很少变化的数据 - 更长的 staleTime
useSuspenseQuery({
    queryKey: ['form', blogId, 'structure'],
    queryFn: () => formApi.getStructure(blogId),
    staleTime: 30 * 60 * 1000,  // 30 分钟
});
```

---

## 总结

**现代数据获取配方：**

1. **创建 API 服务**：使用 apiClient 的 `features/X/api/XApi.ts`
2. **使用 useSuspenseQuery**：在由 SuspenseLoader 包装的组件中
3. **缓存优先**：在 API 调用之前检查网格缓存
4. **查询键**：一致的命名 ['entity', id]
5. **路由格式**：`/blog/route` 不是 `/api/blog/route`
6. **Mutations**：成功后使 invalidateQueries
7. **错误处理**：onError + useMuiSnackbar
8. **类型安全**：对所有参数和返回进行类型化

**另请参阅：**
- [component-patterns.md](component-patterns.md) - Suspense 集成
- [loading-and-error-states.md](loading-and-error-states.md) - SuspenseLoader 使用
- [complete-examples.md](complete-examples.md) - 完整工作示例
