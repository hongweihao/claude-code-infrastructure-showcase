# 常见模式库

用于技能触发器的即用正则表达式和 glob 模式。复制并为你的技能自定义。

---

## 意图模式（正则表达式）

### 功能/端点创建
```regex
(add|create|implement|build).*?(feature|endpoint|route|service|controller)
```

### 组件创建
```regex
(create|add|make|build).*?(component|UI|page|modal|dialog|form)
```

### 数据库工作
```regex
(add|create|modify|update).*?(user|table|column|field|schema|migration)
(database|prisma).*?(change|update|query)
```

### 错误处理
```regex
(fix|handle|catch|debug).*?(error|exception|bug)
(add|implement).*?(try|catch|error.*?handling)
```

### 解释请求
```regex
(how does|how do|explain|what is|describe|tell me about).*?
```

### 工作流操作
```regex
(create|add|modify|update).*?(workflow|step|branch|condition)
(debug|troubleshoot|fix).*?workflow
```

### 测试
```regex
(write|create|add).*?(test|spec|unit.*?test)
```

---

## 文件路径模式（Glob）

### 前端
```glob
frontend/src/**/*.tsx        # 所有 React 组件
frontend/src/**/*.ts         # 所有 TypeScript 文件
frontend/src/components/**   # 仅组件目录
```

### 后端服务
```glob
form/src/**/*.ts            # Form 服务
email/src/**/*.ts           # Email 服务
users/src/**/*.ts           # Users 服务
projects/src/**/*.ts        # Projects 服务
```

### 数据库
```glob
**/schema.prisma            # Prisma schema（任何位置）
**/migrations/**/*.sql      # 迁移文件
database/src/**/*.ts        # 数据库脚本
```

### 工作流
```glob
form/src/workflow/**/*.ts              # 工作流引擎
form/src/workflow-definitions/**/*.json # 工作流定义
```

### 测试排除
```glob
**/*.test.ts                # TypeScript 测试
**/*.test.tsx               # React 组件测试
**/*.spec.ts                # Spec 文件
```

---

## 内容模式（正则表达式）

### Prisma/数据库
```regex
import.*[Pp]risma                # Prisma 导入
PrismaService                    # PrismaService 使用
prisma\.                         # prisma.something
\.findMany\(                     # Prisma 查询方法
\.create\(
\.update\(
\.delete\(
```

### 控制器/路由
```regex
export class.*Controller         # 控制器类
router\.                         # Express 路由器
app\.(get|post|put|delete|patch) # Express app 路由
```

### 错误处理
```regex
try\s*\{                        # Try 块
catch\s*\(                      # Catch 块
throw new                        # Throw 语句
```

### React/组件
```regex
export.*React\.FC               # React 函数组件
export default function.*       # 默认函数导出
useState|useEffect              # React 钩子
```

---

**使用示例：**

```json
{
  "my-skill": {
    "promptTriggers": {
      "intentPatterns": [
        "(create|add|build).*?(component|UI|page)"
      ]
    },
    "fileTriggers": {
      "pathPatterns": [
        "frontend/src/**/*.tsx"
      ],
      "contentPatterns": [
        "export.*React\\.FC",
        "useState|useEffect"
      ]
    }
  }
}
```

---

**相关文件：**
- [SKILL.md](SKILL.md) - 主技能指南
- [TRIGGER_TYPES.md](TRIGGER_TYPES.md) - 详细触发器文档
- [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) - 完整架构
