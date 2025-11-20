---
name: skill-developer
description: 按照 Anthropic 最佳实践创建和管理 Claude Code 技能。在创建新技能、修改 skill-rules.json、理解触发模式、使用钩子、调试技能激活或实现渐进式披露时使用。涵盖技能结构、YAML 前言、触发类型（关键词、意图模式、文件路径、内容模式）、执行级别（block、suggest、warn）、钩子机制（UserPromptSubmit、PreToolUse）、会话跟踪和 500 行规则。
---

# 技能开发者指南

## 目的

使用自动激活系统在 Claude Code 中创建和管理技能的综合指南，遵循 Anthropic 的官方最佳实践，包括 500 行规则和渐进式披露模式。

## 何时使用此技能

当你提到以下内容时自动激活：
- 创建或添加技能
- 修改技能触发器或规则
- 理解技能激活的工作原理
- 调试技能激活问题
- 使用 skill-rules.json
- 钩子系统机制
- Claude Code 最佳实践
- 渐进式披露
- YAML 前言
- 500 行规则

---

## 系统概览

### 双钩子架构

**1. UserPromptSubmit 钩子**（主动建议）
- **文件**：`.claude/hooks/skill-activation-prompt.ts`
- **触发**：在 Claude 看到用户提示之前
- **目的**：根据关键词 + 意图模式建议相关技能
- **方法**：将格式化的提醒作为上下文注入（stdout → Claude 的输入）
- **用例**：基于主题的技能、隐式工作检测

**2. Stop 钩子 - 错误处理提醒**（温和提醒）
- **文件**：`.claude/hooks/error-handling-reminder.ts`
- **触发**：在 Claude 完成响应之后
- **目的**：温和提醒自我评估编写代码中的错误处理
- **方法**：分析已编辑文件的风险模式，如需要则显示提醒
- **用例**：错误处理意识，不阻碍工作流程

**理念变化（2025-10-27）：** 我们不再为 Sentry/错误处理使用阻塞的 PreToolUse。相反，使用温和的响应后提醒，不阻塞工作流程但保持代码质量意识。

### 配置文件

**位置**：`.claude/skills/skill-rules.json`

定义：
- 所有技能及其触发条件
- 执行级别（block、suggest、warn）
- 文件路径模式（glob）
- 内容检测模式（正则表达式）
- 跳过条件（会话跟踪、文件标记、环境变量）

---

## 技能类型

### 1. 防护栏技能

**目的：** 执行防止错误的关键最佳实践

**特征：**
- 类型：`"guardrail"`
- 执行：`"block"`
- 优先级：`"critical"` 或 `"high"`
- 阻止文件编辑直到使用技能
- 防止常见错误（列名、关键错误）
- 会话感知（在同一会话中不重复提醒）

**示例：**
- `database-verification` - 在 Prisma 查询之前验证表/列名
- `frontend-dev-guidelines` - 执行 React/TypeScript 模式

**何时使用：**
- 导致运行时错误的错误
- 数据完整性关注
- 关键兼容性问题

### 2. 领域技能

**目的：** 为特定领域提供全面指导

**特征：**
- 类型：`"domain"`
- 执行：`"suggest"`
- 优先级：`"high"` 或 `"medium"`
- 建议性，非强制性
- 特定主题或领域
- 全面文档

**示例：**
- `backend-dev-guidelines` - Node.js/Express/TypeScript 模式
- `frontend-dev-guidelines` - React/TypeScript 最佳实践
- `error-tracking` - Sentry 集成指导

**何时使用：**
- 需要深入知识的复杂系统
- 最佳实践文档
- 架构模式
- 操作指南

---

## 快速开始：创建新技能

### 步骤 1：创建技能文件

**位置：** `.claude/skills/{skill-name}/SKILL.md`

**模板：**
```markdown
---
name: my-new-skill
description: 简要描述，包括触发此技能的关键词。提及主题、文件类型和用例。明确说明触发术语。
---

# 我的新技能

## 目的
此技能帮助什么

## 何时使用
具体场景和条件

## 关键信息
实际指导、文档、模式、示例
```

**最佳实践：**
- ✅ **名称**：小写、连字符、优先使用动名词形式（动词 + -ing）
- ✅ **描述**：包含所有触发关键词/短语（最多 1024 个字符）
- ✅ **内容**：少于 500 行 - 使用参考文件提供详细信息
- ✅ **示例**：真实代码示例
- ✅ **结构**：清晰的标题、列表、代码块

### 步骤 2：添加到 skill-rules.json

完整架构请参见 [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md)。

**基本模板：**
```json
{
  "my-new-skill": {
    "type": "domain",
    "enforcement": "suggest",
    "priority": "medium",
    "promptTriggers": {
      "keywords": ["keyword1", "keyword2"],
      "intentPatterns": ["(create|add).*?something"]
    }
  }
}
```

### 步骤 3：测试触发器

**测试 UserPromptSubmit：**
```bash
echo '{"session_id":"test","prompt":"your test prompt"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts
```

**测试 PreToolUse：**
```bash
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{"session_id":"test","tool_name":"Edit","tool_input":{"file_path":"test.ts"}}
EOF
```

### 步骤 4：优化模式

基于测试：
- 添加缺少的关键词
- 优化意图模式以减少误报
- 调整文件路径模式
- 针对实际文件测试内容模式

### 步骤 5：遵循 Anthropic 最佳实践

✅ 保持 SKILL.md 少于 500 行
✅ 使用参考文件进行渐进式披露
✅ 向超过 100 行的参考文件添加目录
✅ 编写包含触发关键词的详细描述
✅ 在记录之前使用 3+ 个真实场景进行测试
✅ 根据实际使用进行迭代

---

## 执行级别

### BLOCK（关键防护栏）

- 物理阻止 Edit/Write 工具执行
- 钩子的退出代码 2，stderr → Claude
- Claude 看到消息并必须使用技能才能继续
- **用于**：关键错误、数据完整性、安全问题

**示例：** 数据库列名验证

### SUGGEST（推荐）

- 在 Claude 看到提示之前注入提醒
- Claude 知道相关技能
- 不强制执行，仅建议
- **用于**：领域指导、最佳实践、操作指南

**示例：** 前端开发指南

### WARN（可选）

- 低优先级建议
- 仅建议，最小执行
- **用于**：最好有的建议、信息提醒

**很少使用** - 大多数技能要么是 BLOCK 要么是 SUGGEST。

---

## 跳过条件与用户控制

### 1. 会话跟踪

**目的：** 在同一会话中不要重复提醒

**工作原理：**
- 第一次编辑 → 钩子阻止，更新会话状态
- 第二次编辑（同一会话）→ 钩子允许
- 不同会话 → 再次阻止

**状态文件：** `.claude/hooks/state/skills-used-{session_id}.json`

### 2. 文件标记

**目的：** 对已验证文件永久跳过

**标记：** `// @skip-validation`

**用法：**
```typescript
// @skip-validation
import { PrismaService } from './prisma';
// 此文件已手动验证
```

**注意：** 谨慎使用 - 过度使用会失去目的

### 3. 环境变量

**目的：** 紧急禁用、临时覆盖

**全局禁用：**
```bash
export SKIP_SKILL_GUARDRAILS=true  # 禁用所有 PreToolUse 阻止
```

**特定技能：**
```bash
export SKIP_DB_VERIFICATION=true
export SKIP_ERROR_REMINDER=true
```

---

## 测试清单

创建新技能时，验证：

- [ ] 技能文件在 `.claude/skills/{name}/SKILL.md` 中创建
- [ ] 带有名称和描述的适当前言
- [ ] 条目添加到 `skill-rules.json`
- [ ] 使用真实提示测试关键词
- [ ] 使用变体测试意图模式
- [ ] 使用实际文件测试文件路径模式
- [ ] 针对文件内容测试内容模式
- [ ] 阻止消息清晰且可操作（如果是防护栏）
- [ ] 跳过条件配置适当
- [ ] 优先级与重要性匹配
- [ ] 测试中无误报
- [ ] 测试中无漏报
- [ ] 性能可接受（<100ms 或 <200ms）
- [ ] JSON 语法已验证：`jq . skill-rules.json`
- [ ] **SKILL.md 少于 500 行** ⭐
- [ ] 如需要创建参考文件
- [ ] 向超过 100 行的文件添加目录

---

## 参考文件

有关特定主题的详细信息，请参阅：

### [TRIGGER_TYPES.md](TRIGGER_TYPES.md)
所有触发类型的完整指南：
- 关键词触发器（显式主题匹配）
- 意图模式（隐式动作检测）
- 文件路径触发器（glob 模式）
- 内容模式（文件中的正则表达式）
- 每种类型的最佳实践和示例
- 常见陷阱和测试策略

### [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md)
完整的 skill-rules.json 架构：
- 完整的 TypeScript 接口定义
- 逐字段解释
- 完整的防护栏技能示例
- 完整的领域技能示例
- 验证指南和常见错误

### [HOOK_MECHANISMS.md](HOOK_MECHANISMS.md)
钩子内部深入探讨：
- UserPromptSubmit 流程（详细）
- PreToolUse 流程（详细）
- 退出代码行为表（关键）
- 会话状态管理
- 性能考虑

### [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
综合调试指南：
- 技能未触发（UserPromptSubmit）
- PreToolUse 未阻止
- 误报（太多触发）
- 钩子根本未执行
- 性能问题

### [PATTERNS_LIBRARY.md](PATTERNS_LIBRARY.md)
即用模式集合：
- 意图模式库（正则表达式）
- 文件路径模式库（glob）
- 内容模式库（正则表达式）
- 按用例组织
- 准备复制粘贴

### [ADVANCED.md](ADVANCED.md)
未来增强和想法：
- 动态规则更新
- 技能依赖关系
- 条件执行
- 技能分析
- 技能版本控制

---

## 快速参考摘要

### 创建新技能（5 个步骤）

1. 使用前言创建 `.claude/skills/{name}/SKILL.md`
2. 向 `.claude/skills/skill-rules.json` 添加条目
3. 使用 `npx tsx` 命令测试
4. 根据测试优化模式
5. 保持 SKILL.md 少于 500 行

### 触发类型

- **关键词**：显式主题提及
- **意图**：隐式动作检测
- **文件路径**：基于位置的激活
- **内容**：特定技术检测

完整详细信息请参见 [TRIGGER_TYPES.md](TRIGGER_TYPES.md)。

### 执行

- **BLOCK**：退出代码 2，仅关键
- **SUGGEST**：注入上下文，最常见
- **WARN**：建议，很少使用

### 跳过条件

- **会话跟踪**：自动（防止重复提醒）
- **文件标记**：`// @skip-validation`（永久跳过）
- **环境变量**：`SKIP_SKILL_GUARDRAILS`（紧急禁用）

### Anthropic 最佳实践

✅ **500 行规则**：保持 SKILL.md 少于 500 行
✅ **渐进式披露**：使用参考文件提供详细信息
✅ **目录**：添加到超过 100 行的参考文件
✅ **一级深度**：不要深度嵌套引用
✅ **丰富描述**：包括所有触发关键词（最多 1024 个字符）
✅ **先测试**：在大量文档之前构建 3+ 个评估
✅ **动名词命名**：优先使用动词 + -ing（例如，"processing-pdfs"）

### 故障排除

手动测试钩子：
```bash
# UserPromptSubmit
echo '{"prompt":"test"}' | npx tsx .claude/hooks/skill-activation-prompt.ts

# PreToolUse
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{"tool_name":"Edit","tool_input":{"file_path":"test.ts"}}
EOF
```

完整调试指南请参见 [TROUBLESHOOTING.md](TROUBLESHOOTING.md)。

---

## 相关文件

**配置：**
- `.claude/skills/skill-rules.json` - 主配置
- `.claude/hooks/state/` - 会话跟踪
- `.claude/settings.json` - 钩子注册

**钩子：**
- `.claude/hooks/skill-activation-prompt.ts` - UserPromptSubmit
- `.claude/hooks/error-handling-reminder.ts` - Stop 事件（温和提醒）

**所有技能：**
- `.claude/skills/*/SKILL.md` - 技能内容文件

---

**技能状态**：完成 - 按照 Anthropic 最佳实践重组 ✅
**行数**：< 500（遵循 500 行规则）✅
**渐进式披露**：详细信息的参考文件 ✅

**下一步**：创建更多技能，根据使用情况优化模式
