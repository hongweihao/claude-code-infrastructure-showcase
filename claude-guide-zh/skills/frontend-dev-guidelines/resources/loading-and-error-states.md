# 加载和错误状态

**关键**：正确的加载和错误状态处理可防止布局偏移并提供更好的用户体验。

---

## ⚠️ 关键规则：永远不要使用提前返回

### 问题

```typescript
// ❌ 永远不要这样做 - 带加载旋转器的提前返回
const Component = () => {
    const { data, isLoading } = useQuery();

    // 错误：这会导致布局偏移和糟糕的用户体验
    if (isLoading) {
        return <LoadingSpinner />;
    }

    return <Content data={data} />;
};
```

**为什么这很糟糕：**
1. **布局偏移**：加载完成时内容位置跳跃
2. **CLS（累积布局偏移）**：核心 Web Vitals 分数差
3. **刺眼的用户体验**：页面结构突然改变
4. **丢失滚动位置**：用户失去在页面上的位置

### 解决方案

**选项 1：SuspenseLoader（新组件的首选）**

```typescript
import { SuspenseLoader } from '~components/SuspenseLoader';

const HeavyComponent = React.lazy(() => import('./HeavyComponent'));

export const MyComponent: React.FC = () => {
    return (
        <SuspenseLoader>
            <HeavyComponent />
        </SuspenseLoader>
    );
};
```

**选项 2：LoadingOverlay（用于遗留 useQuery 模式）**

```typescript
import { LoadingOverlay } from '~components/LoadingOverlay';

export const MyComponent: React.FC = () => {
    const { data, isLoading } = useQuery({ ... });

    return (
        <LoadingOverlay loading={isLoading}>
            <Content data={data} />
        </LoadingOverlay>
    );
};
```

---

## SuspenseLoader 组件

### 它的作用

- 在延迟组件加载时显示加载指示器
- 平滑淡入动画
- 防止布局偏移
- 整个应用的一致加载体验

### 导入

```typescript
import { SuspenseLoader } from '~components/SuspenseLoader';
// 或
import { SuspenseLoader } from '@/components/SuspenseLoader';
```

### 基本使用

```typescript
<SuspenseLoader>
    <LazyLoadedComponent />
</SuspenseLoader>
```

### 与 useSuspenseQuery 一起使用

```typescript
import { useSuspenseQuery } from '@tanstack/react-query';
import { SuspenseLoader } from '~components/SuspenseLoader';

const Inner: React.FC = () => {
    // 不需要 isLoading！
    const { data } = useSuspenseQuery({
        queryKey: ['data'],
        queryFn: () => api.getData(),
    });

    return <Display data={data} />;
};

// 外部组件在 Suspense 中包装
export const Outer: React.FC = () => {
    return (
        <SuspenseLoader>
            <Inner />
        </SuspenseLoader>
    );
};
```

### 多个 Suspense 边界

**模式**：独立部分的单独加载

```typescript
export const Dashboard: React.FC = () => {
    return (
        <Box>
            <SuspenseLoader>
                <Header />
            </SuspenseLoader>

            <SuspenseLoader>
                <MainContent />
            </SuspenseLoader>

            <SuspenseLoader>
                <Sidebar />
            </SuspenseLoader>
        </Box>
    );
};
```

**好处：**
- 每个部分独立加载
- 用户更快看到部分内容
- 更好的感知性能

### 嵌套 Suspense

```typescript
export const ParentComponent: React.FC = () => {
    return (
        <SuspenseLoader>
            {/* 加载时父组件挂起 */}
            <ParentContent>
                <SuspenseLoader>
                    {/* 子组件的嵌套 suspense */}
                    <ChildComponent />
                </SuspenseLoader>
            </ParentContent>
        </SuspenseLoader>
    );
};
```

---

## LoadingOverlay 组件

### 何时使用

- 带有 `useQuery` 的遗留组件（尚未重构为 Suspense）
- 需要覆盖加载状态
- 无法使用 Suspense 边界

### 使用

```typescript
import { LoadingOverlay } from '~components/LoadingOverlay';

export const MyComponent: React.FC = () => {
    const { data, isLoading } = useQuery({
        queryKey: ['data'],
        queryFn: () => api.getData(),
    });

    return (
        <LoadingOverlay loading={isLoading}>
            <Box sx={{ p: 2 }}>
                {data && <Content data={data} />}
            </Box>
        </LoadingOverlay>
    );
};
```

**它的作用：**
- 显示带旋转器的半透明覆盖层
- 内容区域保留（无布局偏移）
- 加载时防止交互

---

## 错误处理

### useMuiSnackbar Hook（必需）

**永远不要使用 react-toastify** - 项目标准是 MUI Snackbar

```typescript
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

export const MyComponent: React.FC = () => {
    const { showSuccess, showError, showInfo, showWarning } = useMuiSnackbar();

    const handleAction = async () => {
        try {
            await api.doSomething();
            showSuccess('Operation completed successfully');
        } catch (error) {
            showError('Operation failed');
        }
    };

    return <Button onClick={handleAction}>Do Action</Button>;
};
```

**可用方法：**
- `showSuccess(message)` - 绿色成功消息
- `showError(message)` - 红色错误消息
- `showWarning(message)` - 橙色警告消息
- `showInfo(message)` - 蓝色信息消息

### TanStack Query 错误回调

```typescript
import { useSuspenseQuery } from '@tanstack/react-query';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

export const MyComponent: React.FC = () => {
    const { showError } = useMuiSnackbar();

    const { data } = useSuspenseQuery({
        queryKey: ['data'],
        queryFn: () => api.getData(),

        // 处理错误
        onError: (error) => {
            showError('Failed to load data');
            console.error('Query error:', error);
        },
    });

    return <Content data={data} />;
};
```

### 错误边界

```typescript
import { ErrorBoundary } from 'react-error-boundary';

function ErrorFallback({ error, resetErrorBoundary }) {
    return (
        <Box sx={{ p: 4, textAlign: 'center' }}>
            <Typography variant='h5' color='error'>
                Something went wrong
            </Typography>
            <Typography>{error.message}</Typography>
            <Button onClick={resetErrorBoundary}>Try Again</Button>
        </Box>
    );
}

export const MyPage: React.FC = () => {
    return (
        <ErrorBoundary
            FallbackComponent={ErrorFallback}
            onError={(error) => console.error('Boundary caught:', error)}
        >
            <SuspenseLoader>
                <ComponentThatMightError />
            </SuspenseLoader>
        </ErrorBoundary>
    );
};
```

---

## 完整示例

### 示例 1：带 Suspense 的现代组件

```typescript
import React from 'react';
import { Box, Paper } from '@mui/material';
import { useSuspenseQuery } from '@tanstack/react-query';
import { SuspenseLoader } from '~components/SuspenseLoader';
import { myFeatureApi } from '../api/myFeatureApi';

// 内部组件使用 useSuspenseQuery
const InnerComponent: React.FC<{ id: number }> = ({ id }) => {
    const { data } = useSuspenseQuery({
        queryKey: ['entity', id],
        queryFn: () => myFeatureApi.getEntity(id),
    });

    // data 总是已定义 - 不需要 isLoading！
    return (
        <Paper sx={{ p: 2 }}>
            <h2>{data.title}</h2>
            <p>{data.description}</p>
        </Paper>
    );
};

// 外部组件提供 Suspense 边界
export const OuterComponent: React.FC<{ id: number }> = ({ id }) => {
    return (
        <Box>
            <SuspenseLoader>
                <InnerComponent id={id} />
            </SuspenseLoader>
        </Box>
    );
};

export default OuterComponent;
```

### 示例 2：带 LoadingOverlay 的遗留模式

```typescript
import React from 'react';
import { Box } from '@mui/material';
import { useQuery } from '@tanstack/react-query';
import { LoadingOverlay } from '~components/LoadingOverlay';
import { myFeatureApi } from '../api/myFeatureApi';

export const LegacyComponent: React.FC<{ id: number }> = ({ id }) => {
    const { data, isLoading, error } = useQuery({
        queryKey: ['entity', id],
        queryFn: () => myFeatureApi.getEntity(id),
    });

    return (
        <LoadingOverlay loading={isLoading}>
            <Box sx={{ p: 2 }}>
                {error && <ErrorDisplay error={error} />}
                {data && <Content data={data} />}
            </Box>
        </LoadingOverlay>
    );
};
```

### 示例 3：带 Snackbar 的错误处理

```typescript
import React from 'react';
import { useSuspenseQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { Button } from '@mui/material';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';
import { myFeatureApi } from '../api/myFeatureApi';

export const EntityEditor: React.FC<{ id: number }> = ({ id }) => {
    const queryClient = useQueryClient();
    const { showSuccess, showError } = useMuiSnackbar();

    const { data } = useSuspenseQuery({
        queryKey: ['entity', id],
        queryFn: () => myFeatureApi.getEntity(id),
        onError: () => {
            showError('Failed to load entity');
        },
    });

    const updateMutation = useMutation({
        mutationFn: (updates) => myFeatureApi.update(id, updates),

        onSuccess: () => {
            queryClient.invalidateQueries({ queryKey: ['entity', id] });
            showSuccess('Entity updated successfully');
        },

        onError: () => {
            showError('Failed to update entity');
        },
    });

    return (
        <Button onClick={() => updateMutation.mutate({ name: 'New' })}>
            Update
        </Button>
    );
};
```

---

## 加载状态反模式

### ❌ 不要做什么

```typescript
// ❌ 永远不要 - 提前返回
if (isLoading) {
    return <CircularProgress />;
}

// ❌ 永远不要 - 条件渲染
{isLoading ? <Spinner /> : <Content />}

// ❌ 永远不要 - 布局改变
if (isLoading) {
    return (
        <Box sx={{ height: 100 }}>
            <Spinner />
        </Box>
    );
}
return (
    <Box sx={{ height: 500 }}>  // 不同的高度！
        <Content />
    </Box>
);
```

### ✅ 要做什么

```typescript
// ✅ 最佳 - useSuspenseQuery + SuspenseLoader
<SuspenseLoader>
    <ComponentWithSuspenseQuery />
</SuspenseLoader>

// ✅ 可接受 - LoadingOverlay
<LoadingOverlay loading={isLoading}>
    <Content />
</LoadingOverlay>

// ✅ 可以 - 带相同布局的内联骨架屏
<Box sx={{ height: 500 }}>
    {isLoading ? <Skeleton variant='rectangular' height='100%' /> : <Content />}
</Box>
```

---

## 骨架屏加载（替代方案）

### MUI Skeleton 组件

```typescript
import { Skeleton, Box } from '@mui/material';

export const MyComponent: React.FC = () => {
    const { data, isLoading } = useQuery({ ... });

    return (
        <Box sx={{ p: 2 }}>
            {isLoading ? (
                <>
                    <Skeleton variant='text' width={200} height={40} />
                    <Skeleton variant='rectangular' width='100%' height={200} />
                    <Skeleton variant='text' width='100%' />
                </>
            ) : (
                <>
                    <Typography variant='h5'>{data.title}</Typography>
                    <img src={data.image} />
                    <Typography>{data.description}</Typography>
                </>
            )}
        </Box>
    );
};
```

**关键**：骨架屏必须与实际内容具有**相同的布局**（无偏移）

---

## 总结

**加载状态：**
- ✅ **首选**：SuspenseLoader + useSuspenseQuery（现代模式）
- ✅ **可接受**：LoadingOverlay（遗留模式）
- ✅ **可以**：带相同布局的骨架屏
- ❌ **永远不要**：提前返回或条件布局

**错误处理：**
- ✅ **始终**：useMuiSnackbar 用于用户反馈
- ❌ **永远不要**：react-toastify
- ✅ 在查询/mutations 中使用 onError 回调
- ✅ 组件级错误的错误边界

**另请参阅：**
- [component-patterns.md](component-patterns.md) - Suspense 集成
- [data-fetching.md](data-fetching.md) - useSuspenseQuery 详细信息
