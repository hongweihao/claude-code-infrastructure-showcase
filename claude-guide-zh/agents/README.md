# Agent

用于复杂多步骤任务的专业 Agent。

---

## 什么是 Agent？

Agent 是处理特定复杂任务的自主 Claude 实例。与技能（提供内联指导）不同，Agent：
- 作为独立子任务运行
- 以最少的监督自主工作
- 具有专门的工具访问权限
- 完成时返回综合报告

**关键优势：** Agent 是**独立的** - 只需复制 `.md` 文件即可立即使用！

---

## 可用 Agent（10 个）

### code-architecture-reviewer
**用途：** 审查代码的架构一致性和最佳实践

**何时使用：**
- 实现新功能后
- 合并重大更改之前
- 重构代码时
- 验证架构决策时

**集成：** ✅ 直接复制

---

### code-refactor-master
**用途：** 规划和执行全面重构

**何时使用：**
- 重组文件结构
- 拆分大型组件
- 移动后更新导入路径
- 提高代码可维护性

**集成：** ✅ 直接复制

---

### documentation-architect
**用途：** 创建综合文档

**何时使用：**
- 记录新功能
- 创建 API 文档
- 编写开发者指南
- 生成架构概览

**集成：** ✅ 直接复制

---

### frontend-error-fixer
**用途：** 调试和修复前端错误

**何时使用：**
- 浏览器控制台错误
- 前端 TypeScript 编译错误
- React 错误
- 构建失败

**集成：** ⚠️ 可能引用截图路径 - 如需要请更新

---

### plan-reviewer
**用途：** 在实现之前审查开发计划

**何时使用：**
- 开始复杂功能之前
- 验证架构计划
- 尽早识别潜在问题
- 获取方法的第二意见

**集成：** ✅ 直接复制

---

### refactor-planner
**用途：** 创建全面的重构策略

**何时使用：**
- 规划代码重组
- 现代化遗留代码
- 拆分大型文件
- 改进代码结构

**集成：** ✅ 直接复制

---

### web-research-specialist
**用途：** 在线研究技术问题

**何时使用：**
- 调试晦涩错误
- 寻找问题解决方案
- 研究最佳实践
- 比较实现方法

**集成：** ✅ 直接复制

---

### auth-route-tester
**用途：** 测试已认证的 API 端点

**何时使用：**
- 使用 JWT cookie 认证测试路由
- 验证端点功能
- 调试认证问题

**集成：** ⚠️ 需要基于 JWT cookie 的认证

---

### auth-route-debugger
**用途：** 调试认证问题

**何时使用：**
- 认证失败
- 令牌问题
- Cookie 问题
- 权限错误

**集成：** ⚠️ 需要基于 JWT cookie 的认证

---

### auto-error-resolver
**用途：** 自动修复 TypeScript 编译错误

**何时使用：**
- TypeScript 错误导致的构建失败
- 破坏类型的重构后
- 需要系统性错误解决

**集成：** ⚠️ 可能需要路径更新

---

## 如何集成 Agent

### 标准集成（大多数 Agent）

**步骤 1：复制文件**
```bash
cp showcase/.claude/agents/agent-name.md \\
   your-project/.claude/agents/
```

**步骤 2：验证（可选）**
```bash
# 检查硬编码路径
grep -n "~/git/\\|/root/git/\\|/Users/" your-project/.claude/agents/agent-name.md
```

**步骤 3：使用它**
询问 Claude："使用 [agent-name] agent 来 [任务]"

就是这样！Agent 立即工作。

---

### 需要自定义的 Agent

**frontend-error-fixer：**
- 可能引用截图路径
- 询问用户："截图应该保存在哪里？"
- 更新 agent 文件中的路径

**auth-route-tester / auth-route-debugger：**
- 需要 JWT cookie 认证
- 从示例更新服务 URL
- 为用户的认证设置进行自定义

**auto-error-resolver：**
- 可能有硬编码的项目路径
- 更新为使用 `$CLAUDE_PROJECT_DIR` 或相对路径

---

## 何时使用 Agent vs 技能

| 使用 Agent 当... | 使用技能当... |
|-------------------|-------------------|
| 任务需要多个步骤 | 需要内联指导 |
| 需要复杂分析 | 检查最佳实践 |
| 首选自主工作 | 想要保持控制 |
| 任务有明确的最终目标 | 持续开发工作 |
| 示例："审查所有控制器" | 示例："创建新路由" |

**两者可以协同工作：**
- 技能在开发期间提供模式
- Agent 在完成时审查结果

---

## Agent 快速参考

| Agent | 复杂度 | 自定义 | 需要认证 |
|-------|-----------|---------------|---------------|
| code-architecture-reviewer | 中等 | ✅ 无 | 否 |
| code-refactor-master | 高 | ✅ 无 | 否 |
| documentation-architect | 中等 | ✅ 无 | 否 |
| frontend-error-fixer | 中等 | ⚠️ 截图路径 | 否 |
| plan-reviewer | 低 | ✅ 无 | 否 |
| refactor-planner | 中等 | ✅ 无 | 否 |
| web-research-specialist | 低 | ✅ 无 | 否 |
| auth-route-tester | 中等 | ⚠️ 认证设置 | JWT cookies |
| auth-route-debugger | 中等 | ⚠️ 认证设置 | JWT cookies |
| auto-error-resolver | 低 | ⚠️ 路径 | 否 |

---

## 给 Claude Code

**为用户集成 agent 时：**

1. **首先阅读 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)**
2. **只需复制 .md 文件** - agent 是独立的
3. **检查硬编码路径：**
   ```bash
   grep "~/git/\\|/root/" agent-name.md
   ```
4. **如果找到则更新路径**为 `$CLAUDE_PROJECT_DIR` 或 `.`
5. **对于认证 agent：** 首先询问他们是否使用 JWT cookie 认证

**就是这样！** Agent 是最容易集成的组件。

---

## 创建你自己的 Agent

Agent 是带有可选 YAML 前言的 markdown 文件：

```markdown
# Agent 名称

## 用途
此 agent 做什么

## 说明
自主执行的分步说明

## 可用工具
此 agent 可以使用的工具列表

## 预期输出
返回结果的格式
```

**提示：**
- 在说明中要非常具体
- 将复杂任务分解为编号步骤
- 明确指定要返回什么
- 包括良好输出的示例
- 明确列出可用工具

---

## 故障排除

### 找不到 Agent

**检查：**
```bash
# agent 文件存在吗？
ls -la .claude/agents/[agent-name].md
```

### Agent 因路径错误而失败

**检查硬编码路径：**
```bash
grep "~/\\|/root/\\|/Users/" .claude/agents/[agent-name].md
```

**修复：**
```bash
sed -i 's|~/git/.*project|$CLAUDE_PROJECT_DIR|g' .claude/agents/[agent-name].md
```

---

## 后续步骤

1. **浏览上面的 agent** - 找到对你工作有用的
2. **复制你需要的** - 只需 .md 文件
3. **让 Claude 使用它们** - "使用 [agent] 来 [任务]"
4. **创建你自己的** - 按照模式满足你的特定需求

**有问题？** 查看 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)
