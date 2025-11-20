# 故障排除 - 技能激活问题

技能激活问题的完整调试指南。

## 目录

- [技能未触发](#技能未触发)
  - [UserPromptSubmit 未建议](#userpromptsubmit-未建议)
  - [PreToolUse 未阻止](#pretooluse-未阻止)
- [误报](#误报)
- [钩子未执行](#钩子未执行)
- [性能问题](#性能问题)

---

## 技能未触发

### UserPromptSubmit 未建议

**症状：** 提问，但输出中没有出现技能建议。

**常见原因：**

#### 1. 关键词不匹配

**检查：**
- 查看 skill-rules.json 中的 `promptTriggers.keywords`
- 关键词实际上在你的提示中吗？
- 记住：不区分大小写的子字符串匹配

**示例：**
```json
"keywords": ["layout", "grid"]
```
- "how does the layout work?" → ✅ 匹配 "layout"
- "how does the grid system work?" → ✅ 匹配 "grid"
- "how do layouts work?" → ✅ 匹配 "layout"
- "how does it work?" → ❌ 不匹配

**修复：** 向 skill-rules.json 添加更多关键词变体

#### 2. 意图模式过于具体

**检查：**
- 查看 `promptTriggers.intentPatterns`
- 在 https://regex101.com/ 测试正则表达式
- 可能需要更宽泛的模式

**示例：**
```json
"intentPatterns": [
  "(create|add).*?(database.*?table)"  // 太具体
]
```
- "create a database table" → ✅ 匹配
- "add new table" → ❌ 不匹配（缺少 "database"）

**修复：** 扩大模式：
```json
"intentPatterns": [
  "(create|add).*?(table|database)"  // 更好
]
```

#### 3. 技能名称拼写错误

**检查：**
- SKILL.md 前言中的技能名称
- skill-rules.json 中的技能名称
- 必须完全匹配

**示例：**
```yaml
# SKILL.md
name: project-catalog-developer
```
```json
// skill-rules.json
"project-catalogue-developer": {  // ❌ 拼写错误：catalogue vs catalog
  ...
}
```

**修复：** 使名称完全匹配

#### 4. JSON 语法错误

**检查：**
```bash
cat .claude/skills/skill-rules.json | jq .
```

如果 JSON 无效，jq 将显示错误。

**常见错误：**
- 尾随逗号
- 缺少引号
- 单引号而不是双引号
- 字符串中未转义的字符

**修复：** 更正 JSON 语法，使用 jq 验证

#### 调试命令

手动测试钩子：

```bash
echo '{"session_id":"debug","prompt":"your test prompt here"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts
```

预期：你的技能应该出现在输出中。

---

### PreToolUse 未阻止

**症状：** 编辑应该触发防护栏的文件，但没有阻止发生。

**常见原因：**

#### 1. 文件路径不匹配模式

**检查：**
- 正在编辑的文件路径
- skill-rules.json 中的 `fileTriggers.pathPatterns`
- Glob 模式语法

**示例：**
```json
"pathPatterns": [
  "frontend/src/**/*.tsx"
]
```
- 编辑：`frontend/src/components/Dashboard.tsx` → ✅ 匹配
- 编辑：`frontend/tests/Dashboard.test.tsx` → ✅ 匹配（添加排除！）
- 编辑：`backend/src/app.ts` → ❌ 不匹配

**修复：** 调整 glob 模式或添加缺少的路径

#### 2. 被 pathExclusions 排除

**检查：**
- 你在编辑测试文件吗？
- 查看 `fileTriggers.pathExclusions`

**示例：**
```json
"pathExclusions": [
  "**/*.test.ts",
  "**/*.spec.ts"
]
```
- 编辑：`services/user.test.ts` → ❌ 已排除
- 编辑：`services/user.ts` → ✅ 未排除

**修复：** 如果测试排除过于宽泛，缩小范围或删除

#### 3. 未找到内容模式

**检查：**
- 文件实际上包含模式吗？
- 查看 `fileTriggers.contentPatterns`
- 正则表达式正确吗？

**示例：**
```json
"contentPatterns": [
  "import.*[Pp]risma"
]
```
- 文件有：`import { PrismaService } from './prisma'` → ✅ 匹配
- 文件有：`import { Database } from './db'` → ❌ 不匹配

**调试：**
```bash
# 检查文件中是否存在模式
grep -i "prisma" path/to/file.ts
```

**修复：** 调整内容模式或添加缺少的导入

#### 4. 会话已使用技能

**检查会话状态：**
```bash
ls .claude/hooks/state/
cat .claude/hooks/state/skills-used-{session-id}.json
```

**示例：**
```json
{
  "skills_used": ["database-verification"],
  "files_verified": []
}
```

如果技能在 `skills_used` 中，它不会在此会话中再次阻止。

**修复：** 删除状态文件以重置：
```bash
rm .claude/hooks/state/skills-used-{session-id}.json
```

#### 5. 存在文件标记

**检查文件中的跳过标记：**
```bash
grep "@skip-validation" path/to/file.ts
```

如果找到，该文件将永久跳过。

**修复：** 如果再次需要验证，删除标记

#### 6. 环境变量覆盖

**检查：**
```bash
echo $SKIP_DB_VERIFICATION
echo $SKIP_SKILL_GUARDRAILS
```

如果已设置，技能将被禁用。

**修复：** 取消设置环境变量：
```bash
unset SKIP_DB_VERIFICATION
```

#### 调试命令

手动测试钩子：

```bash
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts 2>&1
{
  "session_id": "debug",
  "tool_name": "Edit",
  "tool_input": {"file_path": "/root/git/your-project/form/src/services/user.ts"}
}
EOF
echo "Exit code: $?"
```

预期：
- 如果应该阻止，退出代码 2 + stderr 消息
- 如果应该允许，退出代码 0 + 无输出

---

## 误报

**症状：** 技能在不应该触发时触发。

**常见原因与解决方案：**

### 1. 关键词过于通用

**问题：**
```json
"keywords": ["user", "system", "create"]  // 太宽泛
```
- 触发于："user manual"、"file system"、"create directory"

**解决方案：** 使关键词更具体
```json
"keywords": [
  "user authentication",
  "user tracking",
  "create feature"
]
```

### 2. 意图模式过于宽泛

**问题：**
```json
"intentPatterns": [
  "(create)"  // 匹配所有带有 "create" 的内容
]
```
- 触发于："create file"、"create folder"、"create account"

**解决方案：** 向模式添加上下文
```json
"intentPatterns": [
  "(create|add).*?(database|table|feature)"  // 更具体
]
```

**高级：** 使用否定预查来排除
```regex
(create)(?!.*test).*?(feature)  // 如果出现 "test" 则不匹配
```

### 3. 文件路径过于通用

**问题：**
```json
"pathPatterns": [
  "form/**"  // 匹配 form/ 中的所有内容
]
```
- 触发于：测试文件、配置文件、所有内容

**解决方案：** 使用更窄的模式
```json
"pathPatterns": [
  "form/src/services/**/*.ts",  // 仅服务文件
  "form/src/controllers/**/*.ts"
]
```

### 4. 内容模式捕获不相关的代码

**问题：**
```json
"contentPatterns": [
  "Prisma"  // 在注释、字符串等中匹配
]
```
- 触发于：`// Don't use Prisma here`
- 触发于：`const note = "Prisma is cool"`

**解决方案：** 使模式更具体
```json
"contentPatterns": [
  "import.*[Pp]risma",        // 仅导入
  "PrismaService\\.",         // 仅实际使用
  "prisma\\.(findMany|create)" // 特定方法
]
```

### 5. 调整执行级别

**最后手段：** 如果误报频繁：

```json
{
  "enforcement": "block"  // 更改为 "suggest"
}
```

这使其成为建议而不是阻止。

---

## 钩子未执行

**症状：** 钩子根本不运行 - 无建议、无阻止。

**常见原因：**

### 1. 钩子未注册

**检查 `.claude/settings.json`：**
```bash
cat .claude/settings.json | jq '.hooks.UserPromptSubmit'
cat .claude/settings.json | jq '.hooks.PreToolUse'
```

预期：存在钩子条目

**修复：** 添加缺少的钩子注册：
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

### 2. Bash 包装器不可执行

**检查：**
```bash
ls -l .claude/hooks/*.sh
```

预期：`-rwxr-xr-x`（可执行）

**修复：**
```bash
chmod +x .claude/hooks/*.sh
```

### 3. shebang 不正确

**检查：**
```bash
head -1 .claude/hooks/skill-activation-prompt.sh
```

预期：`#!/bin/bash`

**修复：** 在第一行添加正确的 shebang

### 4. npx/tsx 不可用

**检查：**
```bash
npx tsx --version
```

预期：版本号

**修复：** 安装依赖项：
```bash
cd .claude/hooks
npm install
```

### 5. TypeScript 编译错误

**检查：**
```bash
cd .claude/hooks
npx tsc --noEmit skill-activation-prompt.ts
```

预期：无输出（无错误）

**修复：** 更正 TypeScript 语法错误

---

## 性能问题

**症状：** 钩子很慢，提示/编辑前有明显延迟。

**常见原因：**

### 1. 模式过多

**检查：**
- 统计 skill-rules.json 中的模式数量
- 每个模式 = 正则表达式编译 + 匹配

**解决方案：** 减少模式
- 合并相似模式
- 删除冗余模式
- 使用更具体的模式（匹配更快）

### 2. 复杂的正则表达式

**问题：**
```regex
(create|add|modify|update|implement|build).*?(feature|endpoint|route|service|controller|component|UI|page)
```
- 长交替 = 慢

**解决方案：** 简化
```regex
(create|add).*?(feature|endpoint)  // 更少的备选
```

### 3. 检查的文件过多

**问题：**
```json
"pathPatterns": [
  "**/*.ts"  // 检查所有 TypeScript 文件
]
```

**解决方案：** 更具体
```json
"pathPatterns": [
  "form/src/services/**/*.ts",  // 仅特定目录
  "form/src/controllers/**/*.ts"
]
```

### 4. 大文件

内容模式匹配读取整个文件 - 大文件很慢。

**解决方案：**
- 仅在必要时使用内容模式
- 考虑文件大小限制（未来增强）

### 测量性能

```bash
# UserPromptSubmit
time echo '{"prompt":"test"}' | npx tsx .claude/hooks/skill-activation-prompt.ts

# PreToolUse
time cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{"tool_name":"Edit","tool_input":{"file_path":"test.ts"}}
EOF
```

**目标指标：**
- UserPromptSubmit：< 100ms
- PreToolUse：< 200ms

---

**相关文件：**
- [SKILL.md](SKILL.md) - 主技能指南
- [HOOK_MECHANISMS.md](HOOK_MECHANISMS.md) - 钩子工作原理
- [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) - 配置参考
