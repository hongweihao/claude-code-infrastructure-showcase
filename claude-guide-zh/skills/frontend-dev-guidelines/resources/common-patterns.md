# 常见模式

表单、身份验证、DataGrid、对话框和其他常见 UI 元素的常用模式。

---

## 使用 useAuth 进行身份验证

### 获取当前用户

```typescript
import { useAuth } from '@/hooks/useAuth';

export const MyComponent: React.FC = () => {
    const { user } = useAuth();

    // 可用属性：
    // - user.id: string
    // - user.email: string
    // - user.username: string
    // - user.roles: string[]

    return (
        <div>
            <p>Logged in as: {user.email}</p>
            <p>Username: {user.username}</p>
            <p>Roles: {user.roles.join(', ')}</p>
        </div>
    );
};
```

**永远不要直接调用 API 进行身份验证** - 始终使用 `useAuth` hook。

---

## 使用 React Hook Form 的表单

### 基本表单

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { TextField, Button } from '@mui/material';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

// Zod 验证 schema
const formSchema = z.object({
    username: z.string().min(3, 'Username must be at least 3 characters'),
    email: z.string().email('Invalid email address'),
    age: z.number().min(18, 'Must be 18 or older'),
});

type FormData = z.infer<typeof formSchema>;

export const MyForm: React.FC = () => {
    const { showSuccess, showError } = useMuiSnackbar();

    const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
        resolver: zodResolver(formSchema),
        defaultValues: {
            username: '',
            email: '',
            age: 18,
        },
    });

    const onSubmit = async (data: FormData) => {
        try {
            await api.submitForm(data);
            showSuccess('Form submitted successfully');
        } catch (error) {
            showError('Failed to submit form');
        }
    };

    return (
        <form onSubmit={handleSubmit(onSubmit)}>
            <TextField
                {...register('username')}
                label='Username'
                error={!!errors.username}
                helperText={errors.username?.message}
            />

            <TextField
                {...register('email')}
                label='Email'
                error={!!errors.email}
                helperText={errors.email?.message}
                type='email'
            />

            <TextField
                {...register('age', { valueAsNumber: true })}
                label='Age'
                error={!!errors.age}
                helperText={errors.age?.message}
                type='number'
            />

            <Button type='submit' variant='contained'>
                Submit
            </Button>
        </form>
    );
};
```

---

## 对话框组件模式

### 标准对话框结构

来自 BEST_PRACTICES.md - 所有对话框应该包含：
- 标题中的图标
- 关闭按钮 (X)
- 底部的操作按钮

```typescript
import { Dialog, DialogTitle, DialogContent, DialogActions, Button, IconButton } from '@mui/material';
import { Close, Info } from '@mui/icons-material';

interface MyDialogProps {
    open: boolean;
    onClose: () => void;
    onConfirm: () => void;
}

export const MyDialog: React.FC<MyDialogProps> = ({ open, onClose, onConfirm }) => {
    return (
        <Dialog open={open} onClose={onClose} maxWidth='sm' fullWidth>
            <DialogTitle>
                <Box sx={{ display: 'flex', alignItems: 'center', justifyContent: 'space-between' }}>
                    <Box sx={{ display: 'flex', alignItems: 'center', gap: 1 }}>
                        <Info color='primary' />
                        Dialog Title
                    </Box>
                    <IconButton onClick={onClose} size='small'>
                        <Close />
                    </IconButton>
                </Box>
            </DialogTitle>

            <DialogContent>
                {/* Content here */}
            </DialogContent>

            <DialogActions>
                <Button onClick={onClose}>Cancel</Button>
                <Button onClick={onConfirm} variant='contained'>
                    Confirm
                </Button>
            </DialogActions>
        </Dialog>
    );
};
```

---

## DataGrid 包装器模式

### 包装器组件契约

来自 BEST_PRACTICES.md - DataGrid 包装器应该接受：

**必需 Props：**
- `rows`：数据数组
- `columns`：列定义
- 加载/错误状态

**可选 Props：**
- 工具栏组件
- 自定义操作
- 初始状态

```typescript
import { DataGridPro } from '@mui/x-data-grid-pro';
import type { GridColDef } from '@mui/x-data-grid-pro';

interface DataGridWrapperProps {
    rows: any[];
    columns: GridColDef[];
    loading?: boolean;
    toolbar?: React.ReactNode;
    onRowClick?: (row: any) => void;
}

export const DataGridWrapper: React.FC<DataGridWrapperProps> = ({
    rows,
    columns,
    loading = false,
    toolbar,
    onRowClick,
}) => {
    return (
        <DataGridPro
            rows={rows}
            columns={columns}
            loading={loading}
            slots={{ toolbar: toolbar ? () => toolbar : undefined }}
            onRowClick={(params) => onRowClick?.(params.row)}
            // Standard configuration
            pagination
            pageSizeOptions={[25, 50, 100]}
            initialState={{
                pagination: { paginationModel: { pageSize: 25 } },
            }}
        />
    );
};
```

---

## Mutation 模式

### 更新与缓存失效

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

export const useUpdateEntity = () => {
    const queryClient = useQueryClient();
    const { showSuccess, showError } = useMuiSnackbar();

    return useMutation({
        mutationFn: ({ id, data }: { id: number; data: any }) =>
            api.updateEntity(id, data),

        onSuccess: (result, variables) => {
            // 使受影响的查询失效
            queryClient.invalidateQueries({ queryKey: ['entity', variables.id] });
            queryClient.invalidateQueries({ queryKey: ['entities'] });

            showSuccess('Entity updated');
        },

        onError: () => {
            showError('Failed to update entity');
        },
    });
};

// 使用
const updateEntity = useUpdateEntity();

const handleSave = () => {
    updateEntity.mutate({ id: 123, data: { name: 'New Name' } });
};
```

---

## 状态管理模式

### 用于服务器状态的 TanStack Query（主要）

对**所有服务器数据**使用 TanStack Query：
- 获取：useSuspenseQuery
- Mutations：useMutation
- 缓存：自动
- 同步：内置

```typescript
// ✅ 正确 - 对服务器数据使用 TanStack Query
const { data: users } = useSuspenseQuery({
    queryKey: ['users'],
    queryFn: () => userApi.getUsers(),
});
```

### 用于 UI 状态的 useState

对**本地 UI 状态**仅使用 `useState`：
- 表单输入（非受控）
- 模态框打开/关闭
- 选中的标签
- 临时 UI 标志

```typescript
// ✅ 正确 - 对 UI 状态使用 useState
const [modalOpen, setModalOpen] = useState(false);
const [selectedTab, setSelectedTab] = useState(0);
```

### 用于全局客户端状态的 Zustand（最小化）

仅对**全局客户端状态**使用 Zustand：
- 主题偏好
- 侧边栏折叠状态
- 用户偏好（不来自服务器）

```typescript
import { create } from 'zustand';

interface AppState {
    sidebarOpen: boolean;
    toggleSidebar: () => void;
}

export const useAppState = create<AppState>((set) => ({
    sidebarOpen: true,
    toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
}));
```

**避免 prop drilling** - 改用 context 或 Zustand。

---

## 总结

**常见模式：**
- ✅ useAuth hook 用于当前用户（id、email、roles、username）
- ✅ React Hook Form + Zod 用于表单
- ✅ 带图标 + 关闭按钮的对话框
- ✅ DataGrid 包装器契约
- ✅ 带缓存失效的 Mutations
- ✅ TanStack Query 用于服务器状态
- ✅ useState 用于 UI 状态
- ✅ Zustand 用于全局客户端状态（最小化）

**另请参阅：**
- [data-fetching.md](data-fetching.md) - TanStack Query 模式
- [component-patterns.md](component-patterns.md) - 组件结构
- [loading-and-error-states.md](loading-and-error-states.md) - 错误处理
