# 样式指南

使用 MUI v7 sx prop、内联样式和主题集成的现代样式模式。

---

## 内联 vs 单独样式

### 决策阈值

**<100 行：组件顶部的内联样式**

```typescript
import type { SxProps, Theme } from '@mui/material';

const componentStyles: Record<string, SxProps<Theme>> = {
    container: {
        p: 2,
        display: 'flex',
        flexDirection: 'column',
    },
    header: {
        mb: 2,
        borderBottom: '1px solid',
        borderColor: 'divider',
    },
    // ... 更多样式
};

export const MyComponent: React.FC = () => {
    return (
        <Box sx={componentStyles.container}>
            <Box sx={componentStyles.header}>
                <h2>Title</h2>
            </Box>
        </Box>
    );
};
```

**>100 行：单独的 `.styles.ts` 文件**

```typescript
// MyComponent.styles.ts
import type { SxProps, Theme } from '@mui/material';

export const componentStyles: Record<string, SxProps<Theme>> = {
    container: { ... },
    header: { ... },
    // ... 100+ 行样式
};

// MyComponent.tsx
import { componentStyles } from './MyComponent.styles';

export const MyComponent: React.FC = () => {
    return <Box sx={componentStyles.container}>...</Box>;
};
```

### 真实示例：UnifiedForm.tsx

**第 48-126 行**：78 行内联样式（可接受）

```typescript
const formStyles: Record<string, SxProps<Theme>> = {
    gridContainer: {
        height: '100%',
        maxHeight: 'calc(100vh - 220px)',
    },
    section: {
        height: '100%',
        maxHeight: 'calc(100vh - 220px)',
        overflow: 'auto',
        p: 4,
    },
    // ... 还有 15 个样式对象
};
```

**指南**：用户对约 80 行内联感到满意。在 100 行左右使用您的判断。

---

## sx Prop 模式

### 基本使用

```typescript
<Box sx={{ p: 2, mb: 3, display: 'flex' }}>
    Content
</Box>
```

### 带主题访问

```typescript
<Box
    sx={{
        p: 2,
        backgroundColor: (theme) => theme.palette.primary.main,
        color: (theme) => theme.palette.primary.contrastText,
        borderRadius: (theme) => theme.shape.borderRadius,
    }}
>
    Themed Box
</Box>
```

### 响应式样式

```typescript
<Box
    sx={{
        p: { xs: 1, sm: 2, md: 3 },
        width: { xs: '100%', md: '50%' },
        flexDirection: { xs: 'column', md: 'row' },
    }}
>
    Responsive Layout
</Box>
```

### 伪选择器

```typescript
<Box
    sx={{
        p: 2,
        '&:hover': {
            backgroundColor: 'rgba(0,0,0,0.05)',
        },
        '&:active': {
            backgroundColor: 'rgba(0,0,0,0.1)',
        },
        '& .child-class': {
            color: 'primary.main',
        },
    }}
>
    Interactive Box
</Box>
```

---

## MUI v7 模式

### Grid 组件（v7 语法）

```typescript
import { Grid } from '@mui/material';

// ✅ 正确 - v7 语法带 size prop
<Grid container spacing={2}>
    <Grid size={{ xs: 12, md: 6 }}>
        Left Column
    </Grid>
    <Grid size={{ xs: 12, md: 6 }}>
        Right Column
    </Grid>
</Grid>

// ❌ 错误 - 旧的 v6 语法
<Grid container spacing={2}>
    <Grid xs={12} md={6}>  {/* 旧语法 - 不要使用 */}
        Content
    </Grid>
</Grid>
```

**关键变化**：`size={{ xs: 12, md: 6 }}` 而不是 `xs={12} md={6}`

### 响应式 Grid

```typescript
<Grid container spacing={3}>
    <Grid size={{ xs: 12, sm: 6, md: 4, lg: 3 }}>
        Responsive Column
    </Grid>
</Grid>
```

### 嵌套 Grids

```typescript
<Grid container spacing={2}>
    <Grid size={{ xs: 12, md: 8 }}>
        <Grid container spacing={1}>
            <Grid size={{ xs: 12, sm: 6 }}>
                Nested 1
            </Grid>
            <Grid size={{ xs: 12, sm: 6 }}>
                Nested 2
            </Grid>
        </Grid>
    </Grid>

    <Grid size={{ xs: 12, md: 4 }}>
        Sidebar
    </Grid>
</Grid>
```

---

## 类型安全样式

### 样式对象类型

```typescript
import type { SxProps, Theme } from '@mui/material';

// 类型安全的样式
const styles: Record<string, SxProps<Theme>> = {
    container: {
        p: 2,
        // 自动完成和类型检查在这里工作
    },
};

// 或单个样式
const containerStyle: SxProps<Theme> = {
    p: 2,
    display: 'flex',
};
```

### 主题感知样式

```typescript
const styles: Record<string, SxProps<Theme>> = {
    primary: {
        color: (theme) => theme.palette.primary.main,
        backgroundColor: (theme) => theme.palette.primary.light,
        '&:hover': {
            backgroundColor: (theme) => theme.palette.primary.dark,
        },
    },
    customSpacing: {
        padding: (theme) => theme.spacing(2),
        margin: (theme) => theme.spacing(1, 2), // top/bottom: 1, left/right: 2
    },
};
```

---

## 不要使用什么

### ❌ makeStyles（MUI v4 模式）

```typescript
// ❌ 避免 - 旧的 Material-UI v4 模式
import { makeStyles } from '@mui/styles';

const useStyles = makeStyles((theme) => ({
    root: {
        padding: theme.spacing(2),
    },
}));
```

**为什么避免**：已弃用，v7 不能很好地支持它

### ❌ styled() 组件

```typescript
// ❌ 避免 - styled-components 模式
import { styled } from '@mui/material/styles';

const StyledBox = styled(Box)(({ theme }) => ({
    padding: theme.spacing(2),
}));
```

**为什么避免**：sx prop 更灵活且不创建新组件

### ✅ 改用 sx Prop

```typescript
// ✅ 首选
<Box
    sx={{
        p: 2,
        backgroundColor: 'primary.main',
    }}
>
    Content
</Box>
```

---

## 代码风格标准

### 缩进

**4 个空格**（不是 2，不是制表符）

```typescript
const styles: Record<string, SxProps<Theme>> = {
    container: {
        p: 2,
        display: 'flex',
        flexDirection: 'column',
    },
};
```

### 引号

**单引号**用于字符串（项目标准）

```typescript
// ✅ 正确
const color = 'primary.main';
import { Box } from '@mui/material';

// ❌ 错误
const color = "primary.main";
import { Box } from "@mui/material";
```

### 尾随逗号

**始终在对象和数组中使用尾随逗号**

```typescript
// ✅ 正确
const styles = {
    container: { p: 2 },
    header: { mb: 1 },  // 尾随逗号
};

const items = [
    'item1',
    'item2',  // 尾随逗号
];

// ❌ 错误 - 没有尾随逗号
const styles = {
    container: { p: 2 },
    header: { mb: 1 }  // 缺少逗号
};
```

---

## 常见样式模式

### Flexbox 布局

```typescript
const styles = {
    flexRow: {
        display: 'flex',
        flexDirection: 'row',
        alignItems: 'center',
        gap: 2,
    },
    flexColumn: {
        display: 'flex',
        flexDirection: 'column',
        gap: 1,
    },
    spaceBetween: {
        display: 'flex',
        justifyContent: 'space-between',
        alignItems: 'center',
    },
};
```

### 间距

```typescript
// 内边距
p: 2           // 所有侧面
px: 2          // 水平（左 + 右）
py: 2          // 垂直（上 + 下）
pt: 2, pr: 1   // 特定侧面

// 外边距
m: 2, mx: 2, my: 2, mt: 2, mr: 1

// 单位：1 = 8px (theme.spacing(1))
p: 2  // = 16px
p: 0.5  // = 4px
```

### 定位

```typescript
const styles = {
    relative: {
        position: 'relative',
    },
    absolute: {
        position: 'absolute',
        top: 0,
        right: 0,
    },
    sticky: {
        position: 'sticky',
        top: 0,
        zIndex: 1000,
    },
};
```

---

## 总结

**样式检查清单：**
- ✅ 对 MUI 样式使用 `sx` prop
- ✅ 使用 `SxProps<Theme>` 实现类型安全
- ✅ <100 行：内联；>100 行：单独文件
- ✅ MUI v7 Grid：`size={{ xs: 12 }}`
- ✅ 4 个空格缩进
- ✅ 单引号
- ✅ 尾随逗号
- ❌ 不使用 makeStyles 或 styled()

**另请参阅：**
- [component-patterns.md](component-patterns.md) - 组件结构
- [complete-examples.md](complete-examples.md) - 完整样式示例
