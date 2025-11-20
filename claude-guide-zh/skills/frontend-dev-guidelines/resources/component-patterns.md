# 组件模式

应用程序的现代 React 组件架构，强调类型安全、延迟加载和 Suspense 边界。

---

## React.FC 模式（首选）

### 为什么使用 React.FC

所有组件使用 `React.FC<Props>` 模式以实现：
- Props 的显式类型安全
- 一致的组件签名
- 清晰的 prop 接口文档
- 更好的 IDE 自动完成

### 基本模式

```typescript
import React from 'react';

interface MyComponentProps {
    /** 要显示的用户 ID */
    userId: number;
    /** 操作发生时的可选回调 */
    onAction?: () => void;
}

export const MyComponent: React.FC<MyComponentProps> = ({ userId, onAction }) => {
    return (
        <div>
            User: {userId}
        </div>
    );
};

export default MyComponent;
```

**关键点：**
- 单独定义 Props 接口并带有 JSDoc 注释
- `React.FC<Props>` 提供类型安全
- 在参数中解构 props
- 底部默认导出

---

## 延迟加载模式

### 何时延迟加载

延迟加载以下组件：
- 重型组件（DataGrid、图表、富文本编辑器）
- 路由级组件
- 模态框/对话框内容（最初不显示）
- 折叠下方的内容

### 如何延迟加载

```typescript
import React from 'react';

// 延迟加载重型组件
const PostDataGrid = React.lazy(() =>
    import('./grids/PostDataGrid')
);

// 对于命名导出
const MyComponent = React.lazy(() =>
    import('./MyComponent').then(module => ({
        default: module.MyComponent
    }))
);
```

**来自 PostTable.tsx 的示例：**

```typescript
/**
 * 主帖子表格容器组件
 */
import React, { useState, useCallback } from 'react';
import { Box, Paper } from '@mui/material';

// 延迟加载 PostDataGrid 以优化打包大小
const PostDataGrid = React.lazy(() => import('./grids/PostDataGrid'));

import { SuspenseLoader } from '~components/SuspenseLoader';

export const PostTable: React.FC<PostTableProps> = ({ formId }) => {
    return (
        <Box>
            <SuspenseLoader>
                <PostDataGrid formId={formId} />
            </SuspenseLoader>
        </Box>
    );
};

export default PostTable;
```

---

## Suspense 边界

### SuspenseLoader 组件

**导入：**
```typescript
import { SuspenseLoader } from '~components/SuspenseLoader';
// 或
import { SuspenseLoader } from '@/components/SuspenseLoader';
```

**使用：**
```typescript
<SuspenseLoader>
    <LazyLoadedComponent />
</SuspenseLoader>
```

**它的作用：**
- 在延迟组件加载时显示加载指示器
- 平滑淡入动画
- 一致的加载体验
- 防止布局偏移

### 在哪里放置 Suspense 边界

**路由级：**
```typescript
// routes/my-route/index.tsx
const MyPage = lazy(() => import('@/features/my-feature/components/MyPage'));

function Route() {
    return (
        <SuspenseLoader>
            <MyPage />
        </SuspenseLoader>
    );
}
```

**组件级：**
```typescript
function ParentComponent() {
    return (
        <Box>
            <Header />
            <SuspenseLoader>
                <HeavyDataGrid />
            </SuspenseLoader>
        </Box>
    );
}
```

**多个边界：**
```typescript
function Page() {
    return (
        <Box>
            <SuspenseLoader>
                <HeaderSection />
            </SuspenseLoader>

            <SuspenseLoader>
                <MainContent />
            </SuspenseLoader>

            <SuspenseLoader>
                <Sidebar />
            </SuspenseLoader>
        </Box>
    );
}
```

每个部分独立加载，更好的用户体验。

---

## 组件结构模板

### 推荐顺序

```typescript
/**
 * 组件描述
 * 它做什么，何时使用它
 */
import React, { useState, useCallback, useMemo, useEffect } from 'react';
import { Box, Paper, Button } from '@mui/material';
import type { SxProps, Theme } from '@mui/material';
import { useSuspenseQuery } from '@tanstack/react-query';

// 功能导入
import { myFeatureApi } from '../api/myFeatureApi';
import type { MyData } from '~types/myData';

// 组件导入
import { SuspenseLoader } from '~components/SuspenseLoader';

// Hooks
import { useAuth } from '@/hooks/useAuth';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

// 1. PROPS 接口（带 JSDoc）
interface MyComponentProps {
    /** 要显示的实体 ID */
    entityId: number;
    /** 操作完成时的可选回调 */
    onComplete?: () => void;
    /** 显示模式 */
    mode?: 'view' | 'edit';
}

// 2. 样式（如果内联且 <100 行）
const componentStyles: Record<string, SxProps<Theme>> = {
    container: {
        p: 2,
        display: 'flex',
        flexDirection: 'column',
    },
    header: {
        mb: 2,
        display: 'flex',
        justifyContent: 'space-between',
    },
};

// 3. 组件定义
export const MyComponent: React.FC<MyComponentProps> = ({
    entityId,
    onComplete,
    mode = 'view',
}) => {
    // 4. HOOKS（按此顺序）
    // - 首先是 Context hooks
    const { user } = useAuth();
    const { showSuccess, showError } = useMuiSnackbar();

    // - 数据获取
    const { data } = useSuspenseQuery({
        queryKey: ['myEntity', entityId],
        queryFn: () => myFeatureApi.getEntity(entityId),
    });

    // - 本地状态
    const [selectedItem, setSelectedItem] = useState<string | null>(null);
    const [isEditing, setIsEditing] = useState(mode === 'edit');

    // - 记忆化的值
    const filteredData = useMemo(() => {
        return data.filter(item => item.active);
    }, [data]);

    // - Effects
    useEffect(() => {
        // 设置
        return () => {
            // 清理
        };
    }, []);

    // 5. 事件处理程序（使用 useCallback）
    const handleItemSelect = useCallback((itemId: string) => {
        setSelectedItem(itemId);
    }, []);

    const handleSave = useCallback(async () => {
        try {
            await myFeatureApi.updateEntity(entityId, { /* data */ });
            showSuccess('Entity updated successfully');
            onComplete?.();
        } catch (error) {
            showError('Failed to update entity');
        }
    }, [entityId, onComplete, showSuccess, showError]);

    // 6. 渲染
    return (
        <Box sx={componentStyles.container}>
            <Box sx={componentStyles.header}>
                <h2>My Component</h2>
                <Button onClick={handleSave}>Save</Button>
            </Box>

            <Paper sx={{ p: 2 }}>
                {filteredData.map(item => (
                    <div key={item.id}>{item.name}</div>
                ))}
            </Paper>
        </Box>
    );
};

// 7. 导出（底部默认导出）
export default MyComponent;
```

---

## 组件分离

### 何时拆分组件

**在以下情况下拆分为多个组件：**
- 组件超过 300 行
- 多个不同的职责
- 可重用的部分
- 复杂的嵌套 JSX

**示例：**

```typescript
// ❌ 避免 - 单体式
function MassiveComponent() {
    // 500+ 行
    // 搜索逻辑
    // 过滤逻辑
    // 网格逻辑
    // 操作面板逻辑
}

// ✅ 首选 - 模块化
function ParentContainer() {
    return (
        <Box>
            <SearchAndFilter onFilter={handleFilter} />
            <DataGrid data={filteredData} />
            <ActionPanel onAction={handleAction} />
        </Box>
    );
}
```

### 何时保持在一起

**在以下情况下保持在同一文件中：**
- 组件 < 200 行
- 紧密耦合的逻辑
- 在其他地方不可重用
- 简单的展示组件

---

## 导出模式

### 命名常量 + 默认导出（首选）

```typescript
export const MyComponent: React.FC<Props> = ({ ... }) => {
    // 组件逻辑
};

export default MyComponent;
```

**原因：**
- 命名导出用于测试/重构
- 默认导出方便延迟加载
- 两种选项都可供使用者使用

### 延迟加载命名导出

```typescript
const MyComponent = React.lazy(() =>
    import('./MyComponent').then(module => ({
        default: module.MyComponent
    }))
);
```

---

## 组件通信

### Props 向下，事件向上

```typescript
// 父组件
function Parent() {
    const [selectedId, setSelectedId] = useState<string | null>(null);

    return (
        <Child
            data={data}                    // Props 向下
            onSelect={setSelectedId}       // 事件向上
        />
    );
}

// 子组件
interface ChildProps {
    data: Data[];
    onSelect: (id: string) => void;
}

export const Child: React.FC<ChildProps> = ({ data, onSelect }) => {
    return (
        <div onClick={() => onSelect(data[0].id)}>
            {/* 内容 */}
        </div>
    );
};
```

### 避免 Prop 钻取

**对于深层嵌套使用 context：**
```typescript
// ❌ 避免 - Prop 钻取 5+ 层
<A prop={x}>
  <B prop={x}>
    <C prop={x}>
      <D prop={x}>
        <E prop={x} />  // 最终在这里使用
      </D>
    </C>
  </B>
</A>

// ✅ 首选 - Context 或 TanStack Query
const MyContext = createContext<MyData | null>(null);

function Provider({ children }) {
    const { data } = useSuspenseQuery({ ... });
    return <MyContext.Provider value={data}>{children}</MyContext.Provider>;
}

function DeepChild() {
    const data = useContext(MyContext);
    // 直接使用数据
}
```

---

## 高级模式

### 复合组件

```typescript
// Card.tsx
export const Card: React.FC<CardProps> & {
    Header: typeof CardHeader;
    Body: typeof CardBody;
    Footer: typeof CardFooter;
} = ({ children }) => {
    return <Paper>{children}</Paper>;
};

Card.Header = CardHeader;
Card.Body = CardBody;
Card.Footer = CardFooter;

// 使用
<Card>
    <Card.Header>Title</Card.Header>
    <Card.Body>Content</Card.Body>
    <Card.Footer>Actions</Card.Footer>
</Card>
```

### Render Props（罕见，但有用）

```typescript
interface DataProviderProps {
    children: (data: Data) => React.ReactNode;
}

export const DataProvider: React.FC<DataProviderProps> = ({ children }) => {
    const { data } = useSuspenseQuery({ ... });
    return <>{children(data)}</>;
};

// 使用
<DataProvider>
    {(data) => <Display data={data} />}
</DataProvider>
```

---

## 总结

**现代组件配方：**
1. 带 TypeScript 的 `React.FC<Props>`
2. 如果是重型组件则延迟加载：`React.lazy(() => import())`
3. 用 `<SuspenseLoader>` 包装以处理加载
4. 使用 `useSuspenseQuery` 获取数据
5. 导入别名（@/、~types、~components）
6. 带 `useCallback` 的事件处理程序
7. 底部默认导出
8. 不对加载状态使用提前返回

**另请参阅：**
- [data-fetching.md](data-fetching.md) - useSuspenseQuery 详细信息
- [loading-and-error-states.md](loading-and-error-states.md) - Suspense 最佳实践
- [complete-examples.md](complete-examples.md) - 完整工作示例
