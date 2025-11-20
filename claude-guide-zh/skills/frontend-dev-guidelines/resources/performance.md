# 性能优化

优化 React 组件性能、防止不必要的重新渲染和避免内存泄漏的模式。

---

## 记忆化模式

### useMemo 用于昂贵的计算

```typescript
import { useMemo } from 'react';

export const DataDisplay: React.FC<{ items: Item[], searchTerm: string }> = ({
    items,
    searchTerm,
}) => {
    // ❌ 避免 - 每次渲染时运行
    const filteredItems = items
        .filter(item => item.name.includes(searchTerm))
        .sort((a, b) => a.name.localeCompare(b.name));

    // ✅ 正确 - 记忆化，仅在依赖项更改时重新计算
    const filteredItems = useMemo(() => {
        return items
            .filter(item => item.name.toLowerCase().includes(searchTerm.toLowerCase()))
            .sort((a, b) => a.name.localeCompare(b.name));
    }, [items, searchTerm]);

    return <List items={filteredItems} />;
};
```

**何时使用 useMemo：**
- 过滤/排序大型数组
- 复杂计算
- 转换数据结构
- 昂贵的计算（循环、递归）

**何时不使用 useMemo：**
- 简单的字符串连接
- 基本算术
- 过早优化（先分析！）

---

## useCallback 用于事件处理程序

### 问题

```typescript
// ❌ 避免 - 每次渲染时创建新函数
export const Parent: React.FC = () => {
    const handleClick = (id: string) => {
        console.log('Clicked:', id);
    };

    // 每次 Parent 渲染时 Child 都会重新渲染
    // 因为 handleClick 每次都是新的函数引用
    return <Child onClick={handleClick} />;
};
```

### 解决方案

```typescript
import { useCallback } from 'react';

export const Parent: React.FC = () => {
    // ✅ 正确 - 稳定的函数引用
    const handleClick = useCallback((id: string) => {
        console.log('Clicked:', id);
    }, []); // 空依赖 = 函数永不改变

    // 仅当 props 实际更改时 Child 才重新渲染
    return <Child onClick={handleClick} />;
};
```

**何时使用 useCallback：**
- 作为 props 传递给子组件的函数
- 在 useEffect 中用作依赖项的函数
- 传递给记忆化组件的函数
- 列表中的事件处理程序

**何时不使用 useCallback：**
- 不传递给子组件的事件处理程序
- 简单的内联处理程序：`onClick={() => doSomething()}`

---

## React.memo 用于组件记忆化

### 基本使用

```typescript
import React from 'react';

interface ExpensiveComponentProps {
    data: ComplexData;
    onAction: () => void;
}

// ✅ 将昂贵的组件包装在 React.memo 中
export const ExpensiveComponent = React.memo<ExpensiveComponentProps>(
    function ExpensiveComponent({ data, onAction }) {
        // 复杂的渲染逻辑
        return <ComplexVisualization data={data} />;
    }
);
```

**何时使用 React.memo：**
- 组件频繁渲染
- 组件有昂贵的渲染
- Props 不经常改变
- 组件是列表项
- DataGrid 单元格/渲染器

**何时不使用 React.memo：**
- Props 无论如何都会频繁改变
- 渲染已经很快
- 过早优化

---

## 防抖搜索

### 使用 use-debounce Hook

```typescript
import { useState } from 'react';
import { useDebounce } from 'use-debounce';
import { useSuspenseQuery } from '@tanstack/react-query';

export const SearchComponent: React.FC = () => {
    const [searchTerm, setSearchTerm] = useState('');

    // 防抖 300ms
    const [debouncedSearchTerm] = useDebounce(searchTerm, 300);

    // 查询使用防抖值
    const { data } = useSuspenseQuery({
        queryKey: ['search', debouncedSearchTerm],
        queryFn: () => api.search(debouncedSearchTerm),
        enabled: debouncedSearchTerm.length > 0,
    });

    return (
        <input
            value={searchTerm}
            onChange={(e) => setSearchTerm(e.target.value)}
            placeholder='Search...'
        />
    );
};
```

**最佳防抖时间：**
- **300-500ms**：搜索/过滤
- **1000ms**：自动保存
- **100-200ms**：实时验证

---

## 内存泄漏预防

### 清理 Timeouts/Intervals

```typescript
import { useEffect, useState } from 'react';

export const MyComponent: React.FC = () => {
    const [count, setCount] = useState(0);

    useEffect(() => {
        // ✅ 正确 - 清理 interval
        const intervalId = setInterval(() => {
            setCount(c => c + 1);
        }, 1000);

        return () => {
            clearInterval(intervalId);  // 清理！
        };
    }, []);

    useEffect(() => {
        // ✅ 正确 - 清理 timeout
        const timeoutId = setTimeout(() => {
            console.log('Delayed action');
        }, 5000);

        return () => {
            clearTimeout(timeoutId);  // 清理！
        };
    }, []);

    return <div>{count}</div>;
};
```

### 清理事件监听器

```typescript
useEffect(() => {
    const handleResize = () => {
        console.log('Resized');
    };

    window.addEventListener('resize', handleResize);

    return () => {
        window.removeEventListener('resize', handleResize);  // 清理！
    };
}, []);
```

### Fetch 的中止控制器

```typescript
useEffect(() => {
    const abortController = new AbortController();

    fetch('/api/data', { signal: abortController.signal })
        .then(response => response.json())
        .then(data => setState(data))
        .catch(error => {
            if (error.name === 'AbortError') {
                console.log('Fetch aborted');
            }
        });

    return () => {
        abortController.abort();  // 清理！
    };
}, []);
```

**注意**：使用 TanStack Query，这会自动处理。

---

## 表单性能

### 监视特定字段（而不是全部）

```typescript
import { useForm } from 'react-hook-form';

export const MyForm: React.FC = () => {
    const { register, watch, handleSubmit } = useForm();

    // ❌ 避免 - 监视所有字段，任何更改时重新渲染
    const formValues = watch();

    // ✅ 正确 - 只监视您需要的
    const username = watch('username');
    const email = watch('email');

    // 或多个特定字段
    const [username, email] = watch(['username', 'email']);

    return (
        <form onSubmit={handleSubmit(onSubmit)}>
            <input {...register('username')} />
            <input {...register('email')} />
            <input {...register('password')} />

            {/* 仅在 username/email 更改时重新渲染 */}
            <p>Username: {username}, Email: {email}</p>
        </form>
    );
};
```

---

## 列表渲染优化

### Key Prop 使用

```typescript
// ✅ 正确 - 稳定的唯一键
{items.map(item => (
    <ListItem key={item.id}>
        {item.name}
    </ListItem>
))}

// ❌ 避免 - 索引作为键（如果列表更改则不稳定）
{items.map((item, index) => (
    <ListItem key={index}>  // 如果列表重新排序则错误
        {item.name}
    </ListItem>
))}
```

### 记忆化列表项

```typescript
const ListItem = React.memo<ListItemProps>(({ item, onAction }) => {
    return (
        <Box onClick={() => onAction(item.id)}>
            {item.name}
        </Box>
    );
});

export const List: React.FC<{ items: Item[] }> = ({ items }) => {
    const handleAction = useCallback((id: string) => {
        console.log('Action:', id);
    }, []);

    return (
        <Box>
            {items.map(item => (
                <ListItem
                    key={item.id}
                    item={item}
                    onAction={handleAction}
                />
            ))}
        </Box>
    );
};
```

---

## 防止组件重新初始化

### 问题

```typescript
// ❌ 避免 - 每次渲染时重新创建组件
export const Parent: React.FC = () => {
    // 每次渲染时都有新的组件定义！
    const ChildComponent = () => <div>Child</div>;

    return <ChildComponent />;  // 每次渲染时卸载和重新挂载
};
```

### 解决方案

```typescript
// ✅ 正确 - 在外部定义或使用 useMemo
const ChildComponent: React.FC = () => <div>Child</div>;

export const Parent: React.FC = () => {
    return <ChildComponent />;  // 稳定的组件
};

// ✅ 或者如果是动态的，使用 useMemo
export const Parent: React.FC<{ config: Config }> = ({ config }) => {
    const DynamicComponent = useMemo(() => {
        return () => <div>{config.title}</div>;
    }, [config.title]);

    return <DynamicComponent />;
};
```

---

## 延迟加载重型依赖

### 代码分割

```typescript
// ❌ 避免 - 在顶层导入重型库
import jsPDF from 'jspdf';  // 立即加载大型库
import * as XLSX from 'xlsx';  // 立即加载大型库

// ✅ 正确 - 需要时动态导入
const handleExportPDF = async () => {
    const { jsPDF } = await import('jspdf');
    const doc = new jsPDF();
    // 使用它
};

const handleExportExcel = async () => {
    const XLSX = await import('xlsx');
    // 使用它
};
```

---

## 总结

**性能检查清单：**
- ✅ 对昂贵的计算使用 `useMemo`（filter、sort、map）
- ✅ 对传递给子组件的函数使用 `useCallback`
- ✅ 对昂贵的组件使用 `React.memo`
- ✅ 防抖搜索/过滤（300-500ms）
- ✅ 在 useEffect 中清理 timeouts/intervals
- ✅ 监视特定表单字段（不是全部）
- ✅ 列表中的稳定键
- ✅ 延迟加载重型库
- ✅ 使用 React.lazy 进行代码分割

**另请参阅：**
- [component-patterns.md](component-patterns.md) - 延迟加载
- [data-fetching.md](data-fetching.md) - TanStack Query 优化
- [complete-examples.md](complete-examples.md) - 上下文中的性能模式
