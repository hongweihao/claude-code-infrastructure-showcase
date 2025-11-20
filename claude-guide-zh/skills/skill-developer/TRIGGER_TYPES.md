# 触发类型 - 完整指南

在 Claude Code 技能自动激活系统中配置技能触发器的完整参考。

## 目录

- [关键词触发器（显式）](#关键词触发器显式)
- [意图模式触发器（隐式）](#意图模式触发器隐式)
- [文件路径触发器](#文件路径触发器)
- [内容模式触发器](#内容模式触发器)
- [最佳实践摘要](#最佳实践摘要)

---

## 关键词触发器（显式）

### 工作原理

用户提示中不区分大小写的子字符串匹配。

### 用于

用户明确提及主题时基于主题的激活。

### 配置

```json
"promptTriggers": {
  "keywords": ["layout", "grid", "toolbar", "submission"]
}
```

### 示例

- 用户提示："how does the **layout** system work?"
- 匹配："layout" 关键词
- 激活：`project-catalog-developer`

### 最佳实践

- 使用具体、明确的术语
- 包括常见变体（"layout"、"layout system"、"grid layout"）
- 避免过于通用的词（"system"、"work"、"create"）
- 使用真实提示测试

---

## 意图模式触发器（隐式）

### 工作原理

正则表达式模式匹配以检测用户的意图，即使他们没有明确提及主题。

### 用于

基于动作的激活，其中用户描述他们想做什么而不是具体主题。

### 配置

```json
"promptTriggers": {
  "intentPatterns": [
    "(create|add|implement).*?(feature|endpoint)",
    "(how does|explain).*?(layout|workflow)"
  ]
}
```

### 示例

**数据库工作：**
- 用户提示："add user tracking feature"
- 匹配：`(add).*?(feature)`
- 激活：`database-verification`、`error-tracking`

**组件创建：**
- 用户提示："create a dashboard widget"
- 匹配：`(create).*?(component)`（如果模式中有 component）
- 激活：`frontend-dev-guidelines`

### 最佳实践

- 捕获常见动作动词：`(create|add|modify|build|implement)`
- 包括特定领域名词：`(feature|endpoint|component|workflow)`
- 使用非贪婪匹配：`.*?` 而不是 `.*`
- 使用正则表达式测试器彻底测试模式（https://regex101.com/）
- 不要使模式过于宽泛（导致误报）
- 不要使模式过于具体（导致漏报）

### 常见模式示例

```regex
# 数据库工作
(add|create|implement).*?(user|login|auth|feature)

# 解释
(how does|explain|what is|describe).*?

# 前端工作
(create|add|make|build).*?(component|UI|page|modal|dialog)

# 错误处理
(fix|handle|catch|debug).*?(error|exception|bug)

# 工作流操作
(create|add|modify).*?(workflow|step|branch|condition)
```

---

## 文件路径触发器

### 工作原理

对正在编辑的文件路径进行 Glob 模式匹配。

### 用于

基于项目中文件位置的特定领域/区域激活。

### 配置

```json
"fileTriggers": {
  "pathPatterns": [
    "frontend/src/**/*.tsx",
    "form/src/**/*.ts"
  ],
  "pathExclusions": [
    "**/*.test.ts",
    "**/*.spec.ts"
  ]
}
```

### Glob 模式语法

- `**` = 任意数量的目录（包括零）
- `*` = 目录名称中的任意字符
- 示例：
  - `frontend/src/**/*.tsx` = frontend/src 及子目录中的所有 .tsx 文件
  - `**/schema.prisma` = 项目中任何位置的 schema.prisma
  - `form/src/**/*.ts` = form/src 子目录中的所有 .ts 文件

### 示例

- 正在编辑的文件：`frontend/src/components/Dashboard.tsx`
- 匹配：`frontend/src/**/*.tsx`
- 激活：`frontend-dev-guidelines`

### 最佳实践

- 具体以避免误报
- 对测试文件使用排除：`**/*.test.ts`
- 考虑子目录结构
- 使用实际文件路径测试模式
- 尽可能使用更窄的模式：`form/src/services/**` 而不是 `form/**`

### 常见路径模式

```glob
# 前端
frontend/src/**/*.tsx        # 所有 React 组件
frontend/src/**/*.ts         # 所有 TypeScript 文件
frontend/src/components/**   # 仅组件目录

# 后端服务
form/src/**/*.ts            # Form 服务
email/src/**/*.ts           # Email 服务
users/src/**/*.ts           # Users 服务

# 数据库
**/schema.prisma            # Prisma schema（任何位置）
**/migrations/**/*.sql      # 迁移文件
database/src/**/*.ts        # 数据库脚本

# 工作流
form/src/workflow/**/*.ts              # 工作流引擎
form/src/workflow-definitions/**/*.json # 工作流定义

# 测试排除
**/*.test.ts                # TypeScript 测试
**/*.test.tsx               # React 组件测试
**/*.spec.ts                # Spec 文件
```

---

## 内容模式触发器

### 工作原理

对文件的实际内容（文件内部）进行正则表达式模式匹配。

### 用于

基于代码导入或使用的内容（Prisma、控制器、特定库）的特定技术激活。

### 配置

```json
"fileTriggers": {
  "contentPatterns": [
    "import.*[Pp]risma",
    "PrismaService",
    "\\.findMany\\(",
    "\\.create\\("
  ]
}
```

### 示例

**Prisma 检测：**
- 文件包含：`import { PrismaService } from '@project/database'`
- 匹配：`import.*[Pp]risma`
- 激活：`database-verification`

**控制器检测：**
- 文件包含：`export class UserController {`
- 匹配：`export class.*Controller`
- 激活：`error-tracking`

### 最佳实践

- 匹配导入：`import.*[Pp]risma`（使用 [Pp] 不区分大小写）
- 转义特殊正则表达式字符：`\\.findMany\\(` 而不是 `.findMany(`
- 模式使用不区分大小写标志
- 针对真实文件内容测试
- 使模式足够具体以避免错误匹配

### 常见内容模式

```regex
# Prisma/数据库
import.*[Pp]risma                # Prisma 导入
PrismaService                    # PrismaService 使用
prisma\.                         # prisma.something
\.findMany\(                     # Prisma 查询方法
\.create\(
\.update\(
\.delete\(

# 控制器/路由
export class.*Controller         # 控制器类
router\.                         # Express 路由器
app\.(get|post|put|delete|patch) # Express app 路由

# 错误处理
try\s*\{                        # Try 块
catch\s*\(                      # Catch 块
throw new                        # Throw 语句

# React/组件
export.*React\.FC               # React 函数组件
export default function.*       # 默认函数导出
useState|useEffect              # React 钩子
```

---

## 最佳实践摘要

### 要做：
✅ 使用具体、明确的关键词
✅ 使用真实示例测试所有模式
✅ 包括常见变体
✅ 使用非贪婪正则表达式：`.*?`
✅ 在内容模式中转义特殊字符
✅ 为测试文件添加排除
✅ 使文件路径模式窄且具体

### 不要做：
❌ 使用过于通用的关键词（"system"、"work"）
❌ 使意图模式过于宽泛（误报）
❌ 使模式过于具体（漏报）
❌ 忘记使用正则表达式测试器测试（https://regex101.com/）
❌ 使用贪婪正则表达式：`.*` 而不是 `.*?`
❌ 在文件路径中匹配过于宽泛

### 测试你的触发器

**测试关键词/意图触发器：**
```bash
echo '{"session_id":"test","prompt":"your test prompt"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts
```

**测试文件路径/内容触发器：**
```bash
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{
  "session_id": "test",
  "tool_name": "Edit",
  "tool_input": {"file_path": "/path/to/test/file.ts"}
}
EOF
```

---

**相关文件：**
- [SKILL.md](SKILL.md) - 主技能指南
- [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) - 完整的 skill-rules.json 架构
- [PATTERNS_LIBRARY.md](PATTERNS_LIBRARY.md) - 即用模式库
