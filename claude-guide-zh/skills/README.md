# 技能

经过生产环境测试的 Claude Code 技能，基于上下文自动激活。

---

## 什么是技能？

技能是 Claude 在需要时加载的模块化知识库。它们提供：
- 特定领域的指南
- 最佳实践
- 代码示例
- 要避免的反模式

**问题：** 技能默认不会自动激活。

**解决方案：** 本展示库包含使它们激活的钩子 + 配置。

---

## 可用技能

### skill-developer（元技能）
**用途：** 创建和管理 Claude Code 技能

**文件：** 7 个资源文件（总计 426 行）

**使用场景：**
- 创建新技能
- 了解技能结构
- 使用 skill-rules.json
- 调试技能激活

**自定义：** ✅ 无 - 直接复制

**[查看技能 →](skill-developer/)**

---

### backend-dev-guidelines
**用途：** Node.js/Express/TypeScript 开发模式

**文件：** 12 个资源文件（主文件 304 行 + 资源）

**涵盖：**
- 分层架构（路由 → 控制器 → 服务 → 仓储）
- BaseController 模式
- Prisma 数据库访问
- Sentry 错误跟踪
- Zod 验证
- UnifiedConfig 模式
- 依赖注入
- 测试策略

**使用场景：**
- 创建/修改 API 路由
- 构建控制器或服务
- 使用 Prisma 的数据库操作
- 设置错误跟踪

**自定义：** ⚠️ 更新 skill-rules.json 中的 `pathPatterns` 以匹配你的后端目录

**示例 pathPatterns：**
```json
{
  "pathPatterns": [
    "src/api/**/*.ts",       // 单应用，带 src/api
    "backend/**/*.ts",       // 后端目录
    "services/*/src/**/*.ts" // 多服务 monorepo
  ]
}
```

**[查看技能 →](backend-dev-guidelines/)**

---

### frontend-dev-guidelines
**用途：** React/TypeScript/MUI v7 开发模式

**文件：** 11 个资源文件（主文件 398 行 + 资源）

**涵盖：**
- 现代 React 模式（Suspense、懒加载）
- useSuspenseQuery 用于数据获取
- MUI v7 样式（Grid 使用 `size={{}}` 属性）
- TanStack Router
- 文件组织（features/ 模式）
- 性能优化
- TypeScript 最佳实践

**使用场景：**
- 创建 React 组件
- 使用 TanStack Query 获取数据
- 使用 MUI v7 样式
- 设置路由

**自定义：** ⚠️ 更新 `pathPatterns` + 验证你使用 React/MUI

**示例 pathPatterns：**
```json
{
  "pathPatterns": [
    "src/**/*.tsx",          // 单 React 应用
    "frontend/src/**/*.tsx", // 前端目录
    "apps/web/**/*.tsx"      // Monorepo web 应用
  ]
}
```

**注意：** 此技能配置为**防护栏**（enforcement: "block"）以防止 MUI v6→v7 不兼容性。

**[查看技能 →](frontend-dev-guidelines/)**

---

### route-tester
**用途：** 使用 JWT cookie 认证测试已认证的 API 路由

**文件：** 1 个主文件（389 行）

**涵盖：**
- 基于 JWT cookie 的认证测试
- test-auth-route.js 脚本模式
- 使用 cookie 认证的 cURL
- 调试认证问题
- 测试 POST/PUT/DELETE 操作

**使用场景：**
- 测试 API 端点
- 调试认证
- 验证路由功能

**自定义：** ⚠️ 需要 JWT cookie 认证设置

**首先询问：** "你使用基于 JWT cookie 的认证吗？"
- 如果是：复制并自定义服务 URL
- 如果否：跳过或为你的认证方法调整

**[查看技能 →](route-tester/)**

---

### error-tracking
**用途：** Sentry 错误跟踪和监控模式

**文件：** 1 个主文件（~250 行）

**涵盖：**
- Sentry v8 初始化
- 错误捕获模式
- 面包屑和用户上下文
- 性能监控
- 与 Express 和 React 集成

**使用场景：**
- 设置错误跟踪
- 捕获异常
- 添加错误上下文
- 调试生产问题

**自定义：** ⚠️ 为你的后端更新 `pathPatterns`

**[查看技能 →](error-tracking/)**

---

## 如何将技能添加到你的项目

### 快速集成

**对于 Claude Code：**
```
用户："将 backend-dev-guidelines 技能添加到我的项目"

Claude 应该：
1. 询问项目结构
2. 复制技能目录
3. 使用他们的路径更新 skill-rules.json
4. 验证集成
```

有关完整说明，请参阅 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)。

### 手动集成

**步骤 1：复制技能目录**
```bash
cp -r claude-code-infrastructure-showcase/.claude/skills/backend-dev-guidelines \\
      your-project/.claude/skills/
```

**步骤 2：更新 skill-rules.json**

如果你没有，创建它：
```bash
cp claude-code-infrastructure-showcase/.claude/skills/skill-rules.json \\
   your-project/.claude/skills/
```

然后为你的项目自定义 `pathPatterns`：
```json
{
  "skills": {
    "backend-dev-guidelines": {
      "fileTriggers": {
        "pathPatterns": [
          "YOUR_BACKEND_PATH/**/*.ts"  // ← 更新这个！
        ]
      }
    }
  }
}
```

**步骤 3：测试**
- 在你的后端目录中编辑文件
- 技能应该自动激活

---

## skill-rules.json 配置

### 它的作用

定义何时应根据以下内容激活技能：
- 用户提示中的**关键词**（"backend"、"API"、"route"）
- **意图模式**（匹配用户意图的正则表达式）
- **文件路径模式**（编辑后端文件）
- **内容模式**（代码包含 Prisma 查询）

### 配置格式

```json
{
  "skill-name": {
    "type": "domain" | "guardrail",
    "enforcement": "suggest" | "block",
    "priority": "high" | "medium" | "low",
    "promptTriggers": {
      "keywords": ["list", "of", "keywords"],
      "intentPatterns": ["regex patterns"]
    },
    "fileTriggers": {
      "pathPatterns": ["path/to/files/**/*.ts"],
      "contentPatterns": ["import.*Prisma"]
    }
  }
}
```

### 执行级别

- **suggest**：技能显示为建议，不阻止
- **block**：继续之前必须使用技能（防护栏）

**使用 "block" 用于：**
- 防止破坏性更改（MUI v6→v7）
- 关键数据库操作
- 安全敏感代码

**使用 "suggest" 用于：**
- 一般最佳实践
- 领域指导
- 代码组织

---

## 创建你自己的技能

有关以下完整指南，请参阅 **skill-developer** 技能：
- 技能 YAML 前言结构
- 资源文件组织
- 触发模式设计
- 测试技能激活

**快速模板：**
```markdown
---
name: my-skill
description: 此技能的作用
---

# 我的技能标题

## 用途
[为什么此技能存在]

## 何时使用此技能
[自动激活场景]

## 快速参考
[关键模式和示例]

## 资源文件
- [topic-1.md](resources/topic-1.md)
- [topic-2.md](resources/topic-2.md)
```

---

## 故障排除

### 技能没有激活

**检查：**
1. 技能目录在 `.claude/skills/` 中吗？
2. 技能在 `skill-rules.json` 中列出了吗？
3. `pathPatterns` 匹配你的文件吗？
4. 钩子已安装并正常工作吗？
5. settings.json 配置正确吗？

**调试：**
```bash
# 检查技能存在
ls -la .claude/skills/

# 验证 skill-rules.json
cat .claude/skills/skill-rules.json | jq .

# 检查钩子是可执行的
ls -la .claude/hooks/*.sh

# 手动测试钩子
./.claude/hooks/skill-activation-prompt.sh
```

### 技能激活太频繁

更新 skill-rules.json：
- 使关键词更具体
- 缩小 `pathPatterns`
- 增加 `intentPatterns` 的特异性

### 技能从不激活

更新 skill-rules.json：
- 添加更多关键词
- 扩大 `pathPatterns`
- 添加更多 `intentPatterns`

---

## 给 Claude Code

**为用户集成技能时：**

1. **首先阅读 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)**
2. 询问他们的项目结构
3. 在 skill-rules.json 中自定义 `pathPatterns`
4. 验证技能文件没有硬编码路径
5. 集成后测试激活

**常见错误：**
- 保留示例路径（blog-api/、frontend/）
- 不询问 monorepo vs 单应用
- 复制 skill-rules.json 而不进行自定义

---

## 后续步骤

1. **从简单开始：** 添加一个与你工作匹配的技能
2. **验证激活：** 编辑相关文件，技能应该建议
3. **添加更多：** 一旦第一个技能工作，添加其他技能
4. **自定义：** 根据你的工作流调整触发器

**有问题？** 有关综合集成说明，请参阅 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)。
