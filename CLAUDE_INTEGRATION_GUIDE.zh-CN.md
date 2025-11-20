# Claude 集成指南

**给 CLAUDE CODE：** 当用户要求你将此展示仓库中的组件集成到他们的项目时，请仔细遵循这些说明。

---

## 概览

这个仓库是 Claude Code 基础设施组件的**参考库**。用户会要求你帮助将特定部分集成到他们的项目中。你的角色是：

1. **询问澄清性问题**关于他们的项目结构
2. **复制适当的文件**
3. **为他们的设置自定义配置**
4. **验证集成**是否正常工作

**关键原则：** 在假设项目结构之前务必先询问。适用于一个项目的方法不一定适用于另一个项目。

---

## 技术栈兼容性检查

**关键：** 在集成技能之前，验证用户的技术栈是否符合技能要求。

### 前端技能

**frontend-dev-guidelines 需要：**
- React (18+)
- MUI v7
- TanStack Query
- TanStack Router
- TypeScript

**集成前询问：**
"你使用 React 和 MUI v7 吗？"

**如果否：**
```
frontend-dev-guidelines 技能专门为 React + MUI v7 设计。我可以：
1. 帮你创建一个针对 [他们的技术栈] 调整的类似技能，使用这个作为模板
2. 提取框架无关的模式（文件组织、性能等）
3. 如果不相关则跳过此技能

你更喜欢哪个？
```

### 后端技能

**backend-dev-guidelines 需要：**
- Node.js/Express
- TypeScript
- Prisma ORM
- Sentry

**集成前询问：**
"你使用 Node.js、Express 和 Prisma 吗？"

**如果否：**
```
backend-dev-guidelines 技能是为 Express/Prisma 设计的。我可以：
1. 帮你创建针对 [他们的技术栈] 调整的类似指南，使用这个作为模板
2. 提取架构模式（分层架构适用于任何框架）
3. 跳过此技能

你更喜欢哪个？
```

### 技术栈无关的技能

这些适用于任何技术栈：
- ✅ **skill-developer** - 元技能，无技术要求
- ✅ **route-tester** - 仅需要 JWT cookie 认证（框架无关）
- ✅ **error-tracking** - Sentry 适用于大多数技术栈

---

## 通用集成模式

当用户说：**"将 [组件] 添加到我的项目"**

1. 识别组件类型（技能/钩子/Agent/命令）
2. **检查技术栈兼容性**（对于前端/后端技能）
3. 询问他们的项目结构
4. 复制文件或为他们的技术栈进行调整
5. 为他们的设置进行自定义
6. 验证集成
7. 提供后续步骤

---

## 集成技能

### 分步过程

**当用户请求技能时**（例如，"添加 backend-dev-guidelines"）：

#### 1. 了解他们的项目

**询问这些问题：**
- "你的项目结构是什么？单应用、monorepo 还是多服务？"
- "你的 [后端/前端] 代码位于哪里？"
- "你使用什么框架/技术？"

#### 2. 复制技能

```bash
cp -r /path/to/showcase/.claude/skills/[skill-name] \\
      $CLAUDE_PROJECT_DIR/.claude/skills/
```

#### 3. 处理 skill-rules.json

**检查它是否存在：**
```bash
ls $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json
```

**如果不存在（不存在）：**
- 从展示库复制模板
- 删除用户不需要的技能
- 为他们的项目自定义

**如果存在（已存在）：**
- 读取他们当前的 skill-rules.json
- 添加新的技能条目
- 仔细合并以避免破坏现有技能

#### 4. 自定义路径模式

**关键：** 更新 skill-rules.json 中的 `pathPatterns` 以匹配他们的结构：

**示例 - 用户有 monorepo：**
```json
{
  "backend-dev-guidelines": {
    "fileTriggers": {
      "pathPatterns": [
        "packages/api/src/**/*.ts",
        "packages/server/src/**/*.ts",
        "apps/backend/**/*.ts"
      ]
    }
  }
}
```

**示例 - 用户有单一后端：**
```json
{
  "backend-dev-guidelines": {
    "fileTriggers": {
      "pathPatterns": [
        "src/**/*.ts",
        "backend/**/*.ts"
      ]
    }
  }
}
```

**安全的通用模式**（当不确定时）：
```json
{
  "pathPatterns": [
    "**/*.ts",          // 所有 TypeScript 文件
    "src/**/*.ts",      // 常见 src 目录
    "backend/**/*.ts"   // 常见后端目录
  ]
}
```

#### 5. 验证集成

```bash
# 检查技能是否已复制
ls -la $CLAUDE_PROJECT_DIR/.claude/skills/[skill-name]

# 验证 skill-rules.json 语法
cat $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json | jq .
```

**告诉用户：** "尝试编辑 [他们的后端路径] 中的文件，技能应该会激活。"

---

### 特定技能说明

#### backend-dev-guidelines
- **技术要求：** Node.js/Express、Prisma、TypeScript、Sentry
- **询问：** "你使用 Express 和 Prisma 吗？" "你的后端代码在哪里？"
- **如果技术栈不同：** 提供使用此作为模板进行调整
- **自定义：** pathPatterns
- **示例路径：** `api/`、`server/`、`backend/`、`services/*/src/`
- **调整提示：** 架构模式（路由→控制器→服务）可以转移到大多数框架

#### frontend-dev-guidelines
- **技术要求：** React 18+、MUI v7、TanStack Query/Router、TypeScript
- **询问：** "你使用 React 和 MUI v7 吗？" "你的前端代码在哪里？"
- **如果技术栈不同：** 提供创建调整版本（Vue、Angular 等）
- **自定义：** pathPatterns + 所有框架特定示例
- **示例路径：** `frontend/`、`client/`、`web/`、`apps/web/src/`
- **调整提示：** 文件组织和性能模式可以转移，组件代码不行

#### route-tester
- **技术要求：** 基于 JWT cookie 的认证（框架无关）
- **询问：** "你使用基于 JWT cookie 的认证吗？"
- **如果否：** "此技能是为 JWT cookie 设计的。想让我为 [他们的认证类型] 调整它还是跳过？"
- **自定义：** 服务 URL、认证模式
- **适用于：** 任何使用 JWT cookie 的后端框架

#### error-tracking
- **技术要求：** Sentry（适用于大多数后端）
- **询问：** "你使用 Sentry 吗？" "你的后端代码在哪里？"
- **如果没有 Sentry：** "想将此用作 [他们的错误跟踪] 的模板吗？"
- **自定义：** pathPatterns
- **调整提示：** 错误跟踪理念可以转移到其他工具（Rollbar、Bugsnag 等）

#### skill-developer
- **技术要求：** 无！
- **直接复制** - 元技能，完全通用，教授为任何技术栈创建技能

---

## 为不同技术栈调整技能

当用户的技术栈与技能要求不同时，你有几个选择：

### 选项 1：调整现有技能（推荐）

**何时使用：** 用户想要类似的指南但针对不同技术

**过程：**
1. **复制技能作为起点：**
   ```bash
   cp -r showcase/.claude/skills/frontend-dev-guidelines \\
         $CLAUDE_PROJECT_DIR/.claude/skills/vue-dev-guidelines
   ```

2. **识别需要更改的内容：**
   - 框架特定的代码示例（React → Vue）
   - 库 API（MUI → Vuetify/PrimeVue）
   - 导入语句
   - 组件模式

3. **保留可转移的内容：**
   - 文件组织原则
   - 性能优化策略
   - TypeScript 标准
   - 一般最佳实践

4. **系统地替换示例：**
   - 询问用户在他们的技术栈中的等效模式
   - 将代码示例更新为他们的框架
   - 保持整体结构和章节

5. **更新技能名称和触发器：**
   - 适当地重命名技能
   - 为他们的技术栈更新 skill-rules.json 触发器
   - 测试激活

**示例 - 为 Vue 调整 frontend-dev-guidelines：**
```
我将基于 React 技能结构创建 vue-dev-guidelines：
- 替换 React.FC → Vue defineComponent
- 替换 useSuspenseQuery → Vue 组合式函数
- 替换 MUI 组件 → [他们的组件库]
- 保留：文件组织、性能模式、TypeScript 指南

这需要几分钟。听起来可以吗？
```

### 选项 2：提取框架无关的模式

**何时使用：** 技术栈非常不同，但核心原则适用

**过程：**
1. 通读现有技能
2. 识别框架无关的模式：
   - 分层架构（后端）
   - 文件组织策略
   - 性能优化原则
   - 测试策略
   - 错误处理理念

3. 仅使用这些模式创建新技能
4. 用户稍后可以添加框架特定的示例

**示例：**
```
backend-dev-guidelines 使用 Express，但分层架构
（路由 → 控制器 → 服务 → 仓储）也适用于 Django。

我可以创建一个包含以下内容的技能：
- 分层架构模式
- 关注点分离原则
- 错误处理最佳实践
- 测试策略

然后你可以在建立模式时添加 Django 特定的示例。
```

### 选项 3：仅作为参考使用

**何时使用：** 差异太大无法调整，但用户想要灵感

**过程：**
1. 用户浏览现有技能
2. 你帮助从头创建新技能
3. 使用现有技能的结构作为模板
4. 遵循模块化模式（主文件 + 资源文件）

### 通常可跨技术栈转移的内容

**架构与组织：**
- ✅ 分层架构（路由/控制器/服务模式）
- ✅ 关注点分离
- ✅ 文件组织策略（features/ 模式）
- ✅ 渐进式披露（主文件 + 资源文件）
- ✅ 数据访问的仓储模式

**开发实践：**
- ✅ 错误处理理念
- ✅ 输入验证的重要性
- ✅ 测试策略
- ✅ 性能优化原则
- ✅ TypeScript 最佳实践

**框架特定的代码：**
- ❌ React hooks → 不能转移到 Vue/Angular
- ❌ MUI 组件 → 不同的组件库
- ❌ Prisma 查询 → 不同的 ORM 语法
- ❌ Express 中间件 → 不同的框架模式
- ❌ 路由实现 → 框架特定

### 何时推荐调整 vs 跳过

**在以下情况推荐调整：**
- 用户想要为他们的技术栈提供类似的指南
- 核心模式适用（分层架构等）
- 用户有时间帮助提供框架特定示例

**在以下情况推荐跳过：**
- 技术栈完全不同
- 用户不需要那些模式
- 调整时间太长
- 用户更喜欢从头创建

---

## 集成钩子

### 必备钩子（总是安全复制）

#### skill-activation-prompt（UserPromptSubmit）

**用途：** 根据用户提示自动建议技能

**集成（无需自定义）：**

```bash
# 复制两个文件
cp showcase/.claude/hooks/skill-activation-prompt.sh \\
   $CLAUDE_PROJECT_DIR/.claude/hooks/
cp showcase/.claude/hooks/skill-activation-prompt.ts \\
   $CLAUDE_PROJECT_DIR/.claude/hooks/

# 设置为可执行
chmod +x $CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh

# 如果需要安装依赖
if [ -f "showcase/.claude/hooks/package.json" ]; then
  cp showcase/.claude/hooks/package.json \\
     $CLAUDE_PROJECT_DIR/.claude/hooks/
  cd $CLAUDE_PROJECT_DIR/.claude/hooks && npm install
fi
```

**添加到 settings.json：**
```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
          }
        ]
      }
    ]
  }
}
```

**此钩子完全通用** - 在任何地方都能工作，无需自定义！

#### post-tool-use-tracker（PostToolUse）

**用途：** 跟踪文件更改以进行上下文管理

**集成（无需自定义）：**

```bash
# 复制文件
cp showcase/.claude/hooks/post-tool-use-tracker.sh \\
   $CLAUDE_PROJECT_DIR/.claude/hooks/

# 设置为可执行
chmod +x $CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh
```

**添加到 settings.json：**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|MultiEdit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh"
          }
        ]
      }
    ]
  }
}
```

**此钩子完全通用** - 自动检测项目结构！

---

### 可选钩子（需要大量自定义）

#### tsc-check.sh 和 trigger-build-resolver.sh（Stop 钩子）

⚠️ **警告：** 这些钩子是为特定的多服务 monorepo 结构配置的。

**集成前询问：**
1. "你有多个 TypeScript 服务的 monorepo 吗？"
2. "你的服务目录名称是什么？"
3. "你的 tsconfig.json 文件位于哪里？"

**对于简单项目（单服务）：**
- **推荐跳过**这些钩子
- 对于单服务项目来说过于复杂
- 用户可以手动运行 `tsc --noEmit` 代替

**对于复杂项目（多服务 monorepo）：**

1. 复制文件
2. **必须编辑** tsc-check.sh - 找到此部分：
```bash
case "$repo" in
    email|exports|form|frontend|projects|uploads|users|utilities|events|database)
        echo "$repo"
        return 0
        ;;
esac
```

3. 替换为用户实际的服务名称：
```bash
case "$repo" in
    api|web|auth|payments|notifications)  # ← 用户的服务
        echo "$repo"
        return 0
        ;;
esac
```

4. 在添加到 settings.json 之前手动测试：
```bash
./.claude/hooks/tsc-check.sh
```

**重要：** 如果此钩子失败，它将阻止 Stop 事件。只有在确定它适用于他们的设置时才添加。

---

## 集成 Agent

**Agent 是独立的** - 最容易集成！

### 标准 Agent 集成

```bash
# 复制 agent 文件
cp showcase/.claude/agents/[agent-name].md \\
   $CLAUDE_PROJECT_DIR/.claude/agents/
```

**就是这样！** Agent 立即工作，无需配置。

### 检查硬编码路径

一些 Agent 可能引用路径。**复制前，阅读 agent 文件并检查：**

- `~/git/old-project/` → 应该是 `$CLAUDE_PROJECT_DIR` 或 `.`
- `/root/git/project/` → 应该是 `$CLAUDE_PROJECT_DIR` 或 `.`
- 硬编码的截图路径 → 询问用户他们希望截图保存在哪里

**如果找到，更新它们：**
```bash
sed -i 's|~/git/old-project/|.|g' $CLAUDE_PROJECT_DIR/.claude/agents/[agent].md
sed -i 's|/root/git/.*PROJECT.*DIR|$CLAUDE_PROJECT_DIR|g' \\
    $CLAUDE_PROJECT_DIR/.claude/agents/[agent].md
```

### 特定 Agent 说明

**auth-route-tester / auth-route-debugger:**
- 需要用户项目中基于 JWT cookie 的认证
- 询问："你使用 JWT cookie 进行认证吗？"
- 如果否："这些 Agent 用于 JWT cookie 认证。跳过它们还是想让我调整？"

**frontend-error-fixer:**
- 可能引用截图路径
- 询问："截图应该保存在哪里？"

**所有其他 Agent：**
- 直接复制，它们完全通用

---

## 集成斜杠命令

```bash
# 复制命令文件
cp showcase/.claude/commands/[command].md \\
   $CLAUDE_PROJECT_DIR/.claude/commands/
```

### 自定义路径

命令可能引用开发文档路径。**检查并更新：**

**dev-docs 和 dev-docs-update：**
- 查找 `dev/active/` 路径引用
- 询问："你希望开发文档存储在哪里？"
- 更新命令文件中的路径

**route-research-for-testing：**
- 可能引用服务路径
- 询问他们的 API 结构

---

## 常见模式与最佳实践

### 模式：询问项目结构

**不要假设：**
- ❌ "我将为你的 blog-api 服务添加这个"
- ❌ "为你的 frontend 目录配置"

**要询问：**
- ✅ "你的项目结构是什么？Monorepo 还是单应用？"
- ✅ "你的后端代码位于哪里？"
- ✅ "你使用工作区还是有多个服务？"

### 模式：自定义 skill-rules.json

**用户有工作区的 monorepo：**
```json
{
  "pathPatterns": [
    "packages/*/src/**/*.ts",
    "apps/*/src/**/*.tsx"
  ]
}
```

**用户有 Nx monorepo：**
```json
{
  "pathPatterns": [
    "apps/api/src/**/*.ts",
    "libs/*/src/**/*.ts"
  ]
}
```

**用户有简单结构：**
```json
{
  "pathPatterns": [
    "src/**/*.ts",
    "backend/**/*.ts"
  ]
}
```

### 模式：settings.json 集成

**永远不要直接复制展示库的 settings.json！**

相反，**提取并合并**他们需要的部分：

1. 读取他们现有的 settings.json
2. 添加他们需要的钩子配置
3. 保留他们现有的配置

**示例合并：**
```json
{
  // ... 他们现有的配置 ...
  "hooks": {
    // ... 他们现有的钩子 ...
    "UserPromptSubmit": [  // ← 添加此部分
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
          }
        ]
      }
    ]
  }
}
```

---

## 验证清单

集成后，**验证这些项目：**

```bash
# 1. 钩子是可执行的
ls -la $CLAUDE_PROJECT_DIR/.claude/hooks/*.sh
# 应该显示：-rwxr-xr-x

# 2. skill-rules.json 是有效的 JSON
cat $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json | jq .
# 应该解析无错误

# 3. 钩子依赖已安装（如果是 TypeScript 钩子）
ls $CLAUDE_PROJECT_DIR/.claude/hooks/node_modules/
# 如果存在 package.json 应该显示包

# 4. Settings.json 是有效的 JSON
cat $CLAUDE_PROJECT_DIR/.claude/settings.json | jq .
# 应该解析无错误
```

**然后要求用户测试：**
- "尝试编辑 [相关路径] 中的文件 - 技能应该激活"
- "尝试问我一个关于 [主题] 的问题 - 我应该建议该技能"

---

## 要避免的常见错误

### ❌ 不要：直接复制 settings.json
**原因：** Stop 钩子引用不存在的服务
**要做：** 仅提取 UserPromptSubmit 和 PostToolUse 部分

### ❌ 不要：保留示例服务名称
**原因：** 用户没有 blog-api、auth-service 等
**要做：** 询问他们的实际结构并更新

### ❌ 不要：跳过使钩子可执行
**原因：** 没有执行权限钩子将无法运行
**要做：** 复制后始终 `chmod +x`

### ❌ 不要：假设 monorepo 结构
**原因：** 大多数项目是单服务的
**要做：** 首先询问，然后自定义

### ❌ 不要：一次添加所有技能
**原因：** 令人不知所措，可能不是全部相关
**要做：** 询问哪些技能与他们的工作相关

### ❌ 不要：在未测试的情况下复制 Stop 钩子
**原因：** 如果失败，它们可能阻止 Stop 事件
**要做：** 首先手动测试，仅在工作时添加

---

## 集成对话示例

### 示例 1：添加后端技能

**用户：** "将 backend-dev-guidelines 技能添加到我的项目"

**Claude 应该回应：**
```
我将添加 backend-dev-guidelines 技能。首先，我需要了解
你的项目结构：

1. 这是一个有多个服务的 monorepo，还是一个单独的后端应用程序？
2. 你的后端代码位于哪里？（例如，src/、backend/、server/）
3. 你使用 Express.js 吗？
```

**用户：** "单应用，代码在 src/api/，是的我们使用 Express"

**Claude 应该：**
1. 复制技能目录
2. 使用 pathPattern 更新 skill-rules.json：`"src/api/**/*.ts"`
3. 验证已正确复制
4. 回应：
```
✅ 已添加 backend-dev-guidelines 技能！

配置：
- 编辑 src/api/ 中的文件时触发技能
- 通过关键词如 "controller"、"service"、"route"、"API" 激活

试一试：编辑 src/api/ 中的文件，你应该看到技能建议。
```

### 示例 2：设置钩子

**用户：** "设置技能激活钩子"

**Claude 应该回应：**
```
我将设置技能自动激活系统。这需要两个钩子：
1. skill-activation-prompt - 根据你的提示建议技能
2. post-tool-use-tracker - 跟踪文件更改

你已经有 .claude/settings.json 文件了吗，还是我应该创建一个？
```

**用户：** "我已经有 settings.json"

**Claude 应该：**
1. 读取他们的 settings.json
2. 复制两个钩子文件
3. 在 hooks 目录中运行 npm install
4. 将钩子配置添加到他们的 settings.json（保留现有配置）
5. 使钩子可执行
6. 回应：
```
✅ 技能激活钩子已安装！

我做了什么：
- 复制了 skill-activation-prompt 钩子
- 复制了 post-tool-use-tracker 钩子
- 安装了依赖
- 添加到你的 settings.json

测试它：问我"帮助后端开发"，我应该建议
backend-dev-guidelines 技能。
```

### 示例 3：添加 Agent

**用户：** "添加 code-architecture-reviewer agent"

**Claude 应该：**
```
code-architecture-reviewer agent 审查代码的架构问题
和最佳实践。此 agent 独立工作，无需配置。

我现在将它复制到你的 .claude/agents/ 目录。
```

**然后：**
1. 复制 agent 文件
2. 回应：
```
✅ 已添加 code-architecture-reviewer agent！

使用 Task 工具使用它：
- subagent_type："code-architecture-reviewer"
- prompt：要审查的代码描述

该 agent 将审查你的代码并提供架构反馈。
```

### 示例 3：技术栈不匹配

**用户：** "将 frontend-dev-guidelines 技能添加到我的 Vue 项目"

**Claude 应该：**
```
我看到你在使用 Vue。frontend-dev-guidelines 技能是专门
为 React + MUI v7 设计的。我有几个选择：

选项 1（推荐）：我可以使用
React 技能作为模板创建 vue-dev-guidelines 技能。我将调整：
- Vue 的组件模式（Composition API、defineComponent）
- 使用你的组件库替换 MUI 示例
- 为 Vue 模式调整数据获取
- 保留：文件组织、性能原则、TypeScript 标准

选项 2：仅提取框架无关的模式（文件组织、
性能、测试）而不包含 React 特定代码。

选项 3：跳过它并帮助你从头创建 Vue 指南。

你更喜欢哪个？
```

**用户：** "选项 1 听起来不错，我使用 Vuetify"

**Claude 应该：**
1. 将 frontend-dev-guidelines 复制为 vue-dev-guidelines
2. 遍历每个资源文件
3. 用 Vue Composition API 替换 React 模式
4. 用 Vuetify 替换 MUI 组件
5. 使用 Vue 特定触发器（.vue 文件）更新 skill-rules.json
6. 回应：
```
✅ 创建了针对 Vue 3 + Vuetify 调整的 vue-dev-guidelines 技能！

我更改的内容：
- React.FC 组件 → Vue defineComponent
- MUI v7 示例 → Vuetify 组件
- TanStack Query → 你的数据获取方法
- 文件模式 → .vue 文件

我保留的内容：
- 文件组织（features/ 模式）
- 性能优化策略
- TypeScript 标准
- 加载/错误状态原则

尝试编辑 .vue 文件 - 技能应该激活。
```

---

## 快速参考表

### 什么需要自定义？

| 组件 | 技术要求 | 自定义 | 询问什么 |
|-----------|------------------|--------------|-------------|
| **skill-developer** | 无 | ✅ 无 | 直接复制 |
| **backend-dev-guidelines** | Express/Prisma/Node | ⚠️ 路径 + 技术检查 | "使用 Express/Prisma？" "后端在哪里？" |
| **frontend-dev-guidelines** | React/MUI v7 | ⚠️⚠️ 路径 + 框架 | "使用 React/MUI v7？" "前端在哪里？" |
| **route-tester** | JWT cookies | ⚠️ 认证 + 路径 | "JWT cookie 认证？" |
| **error-tracking** | Sentry | ⚠️ 路径 | "使用 Sentry？" "后端在哪里？" |
| **skill-activation-prompt** | ✅ 无 | 直接复制 |
| **post-tool-use-tracker** | ✅ 无 | 直接复制 |
| **tsc-check** | ⚠️⚠️⚠️ 大量 | "Monorepo 还是单服务？" |
| **所有 agent** | ✅ 最少 | 检查路径 |
| **所有命令** | ⚠️ 路径 | "开发文档放哪里？" |

### 何时推荐跳过

| 组件 | 如果以下情况跳过... |
|-----------|-----------|
| **tsc-check 钩子** | 单服务项目或不同的构建设置 |
| **route-tester** | 不使用 JWT cookie 认证 |
| **frontend-dev-guidelines** | 不使用 React + MUI |
| **auth agent** | 不使用 JWT cookie 认证 |

---

## 给 Claude 的最终提示

**当用户说"添加所有内容"时：**
- 从必需品开始：技能激活钩子 + 1-2 个相关技能
- 不要用所有 5 个技能 + 10 个 Agent 压倒他们
- 询问他们实际需要什么

**当某些东西不工作时：**
- 检查验证清单
- 验证路径是否匹配他们的结构
- 手动测试钩子
- 检查 JSON 语法错误

**当用户不确定时：**
- 推荐仅从技能激活钩子开始
- 添加后端或前端技能（他们使用的那个）
- 稍后根据需要添加更多

**始终解释你在做什么：**
- 显示你正在运行的命令
- 解释为什么要提问
- 在集成后提供明确的后续步骤

---

**记住：** 这是一个参考库，不是一个工作应用程序。你的工作是帮助用户为他们的特定项目结构挑选和调整组件。
