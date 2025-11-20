# TypeScript 标准

React 前端代码中类型安全和可维护性的 TypeScript 最佳实践。

---

## 严格模式

### 配置

项目中**启用了** TypeScript 严格模式：

```json
// tsconfig.json
{
    "compilerOptions": {
        "strict": true,
        "noImplicitAny": true,
        "strictNullChecks": true
    }
}
```

**这意味着：**
- 没有隐式 `any` 类型
- 必须显式处理 Null/undefined
- 强制类型安全

---

## 不使用 `any` 类型

### 规则

```typescript
// ❌ 永远不要使用 any
function handleData(data: any) {
    return data.something;
}

// ✅ 使用特定类型
interface MyData {
    something: string;
}

function handleData(data: MyData) {
    return data.something;
}

// ✅ 或对真正未知的数据使用 unknown
function handleUnknown(data: unknown) {
    if (typeof data === 'object' && data !== null && 'something' in data) {
        return (data as MyData).something;
    }
}
```

**如果您真的不知道类型：**
- 使用 `unknown`（强制类型检查）
- 使用类型守卫进行收窄
- 记录为什么类型未知

---

## 显式返回类型

### 函数返回类型

```typescript
// ✅ 正确 - 显式返回类型
function getUser(id: number): Promise<User> {
    return apiClient.get(`/users/${id}`);
}

function calculateTotal(items: Item[]): number {
    return items.reduce((sum, item) => sum + item.price, 0);
}

// ❌ 避免 - 隐式返回类型（不太清晰）
function getUser(id: number) {
    return apiClient.get(`/users/${id}`);
}
```

### 组件返回类型

```typescript
// React.FC 已经提供返回类型（ReactElement）
export const MyComponent: React.FC<Props> = ({ prop }) => {
    return <div>{prop}</div>;
};

// 对于自定义 hooks
function useMyData(id: number): { data: Data; isLoading: boolean } {
    const [data, setData] = useState<Data | null>(null);
    const [isLoading, setIsLoading] = useState(true);

    return { data: data!, isLoading };
}
```

---

## 类型导入

### 使用 'type' 关键字

```typescript
// ✅ 正确 - 显式标记为类型导入
import type { User } from '~types/user';
import type { Post } from '~types/post';
import type { SxProps, Theme } from '@mui/material';

// ❌ 避免 - 混合值和类型导入
import { User } from '~types/user';  // 不清楚是类型还是值
```

**好处：**
- 清楚地将类型与值分开
- 更好的 tree-shaking
- 防止循环依赖
- TypeScript 编译器优化

---

## 组件 Prop 接口

### 接口模式

```typescript
/**
 * MyComponent 的 Props
 */
interface MyComponentProps {
    /** 要显示的用户 ID */
    userId: number;

    /** 操作完成时的可选回调 */
    onComplete?: () => void;

    /** 组件的显示模式 */
    mode?: 'view' | 'edit';

    /** 附加 CSS 类 */
    className?: string;
}

export const MyComponent: React.FC<MyComponentProps> = ({
    userId,
    onComplete,
    mode = 'view',  // 默认值
    className,
}) => {
    return <div>...</div>;
};
```

**关键点：**
- props 的单独接口
- 每个 prop 的 JSDoc 注释
- 可选 props 使用 `?`
- 在解构中提供默认值

### 带 Children 的 Props

```typescript
interface ContainerProps {
    children: React.ReactNode;
    title: string;
}

// React.FC 自动包含 children 类型，但要明确
export const Container: React.FC<ContainerProps> = ({ children, title }) => {
    return (
        <div>
            <h2>{title}</h2>
            {children}
        </div>
    );
};
```

---

## 工具类型

### Partial<T>

```typescript
// 使所有属性可选
type UserUpdate = Partial<User>;

function updateUser(id: number, updates: Partial<User>) {
    // updates 可以有 User 属性的任何子集
}
```

### Pick<T, K>

```typescript
// 选择特定属性
type UserPreview = Pick<User, 'id' | 'name' | 'email'>;

const preview: UserPreview = {
    id: 1,
    name: 'John',
    email: 'john@example.com',
    // 其他 User 属性不允许
};
```

### Omit<T, K>

```typescript
// 排除特定属性
type UserWithoutPassword = Omit<User, 'password' | 'passwordHash'>;

const publicUser: UserWithoutPassword = {
    id: 1,
    name: 'John',
    email: 'john@example.com',
    // password 和 passwordHash 不允许
};
```

### Required<T>

```typescript
// 使所有属性必需
type RequiredConfig = Required<Config>;  // 所有可选 props 变为必需
```

### Record<K, V>

```typescript
// 类型安全的对象/映射
const userMap: Record<string, User> = {
    'user1': { id: 1, name: 'John' },
    'user2': { id: 2, name: 'Jane' },
};

// 对于样式
import type { SxProps, Theme } from '@mui/material';

const styles: Record<string, SxProps<Theme>> = {
    container: { p: 2 },
    header: { mb: 1 },
};
```

---

## 类型守卫

### 基本类型守卫

```typescript
function isUser(data: unknown): data is User {
    return (
        typeof data === 'object' &&
        data !== null &&
        'id' in data &&
        'name' in data
    );
}

// 使用
if (isUser(response)) {
    console.log(response.name);  // TypeScript 知道它是 User
}
```

### 可区分联合

```typescript
type LoadingState =
    | { status: 'idle' }
    | { status: 'loading' }
    | { status: 'success'; data: Data }
    | { status: 'error'; error: Error };

function Component({ state }: { state: LoadingState }) {
    // TypeScript 根据 status 收窄类型
    if (state.status === 'success') {
        return <Display data={state.data} />;  // 这里 data 可用
    }

    if (state.status === 'error') {
        return <Error error={state.error} />;  // 这里 error 可用
    }

    return <Loading />;
}
```

---

## 泛型类型

### 泛型函数

```typescript
function getById<T>(items: T[], id: number): T | undefined {
    return items.find(item => (item as any).id === id);
}

// 使用类型推断
const users: User[] = [...];
const user = getById(users, 123);  // 类型：User | undefined
```

### 泛型组件

```typescript
interface ListProps<T> {
    items: T[];
    renderItem: (item: T) => React.ReactNode;
}

export function List<T>({ items, renderItem }: ListProps<T>): React.ReactElement {
    return (
        <div>
            {items.map((item, index) => (
                <div key={index}>{renderItem(item)}</div>
            ))}
        </div>
    );
}

// 使用
<List<User>
    items={users}
    renderItem={(user) => <UserCard user={user} />}
/>
```

---

## 类型断言（谨慎使用）

### 何时使用

```typescript
// ✅ 可以 - 当您比 TypeScript 知道得更多时
const element = document.getElementById('my-element') as HTMLInputElement;
const value = element.value;

// ✅ 可以 - 您已验证的 API 响应
const response = await api.getData();
const user = response.data as User;  // 您知道形状
```

### 何时不使用

```typescript
// ❌ 避免 - 规避类型安全
const data = getData() as any;  // 错误 - 破坏 TypeScript

// ❌ 避免 - 不安全的断言
const value = unknownValue as string;  // 可能实际上不是字符串
```

---

## Null/Undefined 处理

### 可选链

```typescript
// ✅ 正确
const name = user?.profile?.name;

// 等同于：
const name = user && user.profile && user.profile.name;
```

### 空值合并

```typescript
// ✅ 正确
const displayName = user?.name ?? 'Anonymous';

// 仅在 null 或 undefined 时使用默认值
// （与 || 不同，|| 在 ''、0、false 时触发）
```

### 非空断言（小心使用）

```typescript
// ✅ 可以 - 当您确定值存在时
const data = queryClient.getQueryData<Data>(['data'])!;

// ⚠️ 小心 - 仅在您知道它不为 null 时使用
// 更好的做法是显式检查：
const data = queryClient.getQueryData<Data>(['data']);
if (data) {
    // 使用 data
}
```

---

## 总结

**TypeScript 检查清单：**
- ✅ 启用严格模式
- ✅ 不使用 `any` 类型（如果需要使用 `unknown`）
- ✅ 函数上的显式返回类型
- ✅ 对类型导入使用 `import type`
- ✅ prop 接口上的 JSDoc 注释
- ✅ 工具类型（Partial、Pick、Omit、Required、Record）
- ✅ 用于收窄的类型守卫
- ✅ 可选链和空值合并
- ❌ 除非必要，否则避免类型断言

**另请参阅：**
- [component-patterns.md](component-patterns.md) - 组件类型化
- [data-fetching.md](data-fetching.md) - API 类型化
