# 钩子

启用技能自动激活、文件跟踪和验证的 Claude Code 钩子。

---

## 什么是钩子？

钩子是在 Claude 工作流的特定点运行的脚本：
- **UserPromptSubmit**：当用户提交提示时
- **PreToolUse**：在工具执行之前
- **PostToolUse**：在工具完成之后
- **Stop**：当用户请求停止时

**关键洞察：** 钩子可以修改提示、阻止操作和跟踪状态 - 启用 Claude 单独无法完成的功能。

---

## 必备钩子（从这里开始）

### skill-activation-prompt（UserPromptSubmit）

**用途：** 根据用户提示和文件上下文自动建议相关技能

**工作原理：**
1. 读取 `skill-rules.json`
2. 将用户提示与触发模式匹配
3. 检查用户正在使用哪些文件
4. 将技能建议注入到 Claude 的上下文中

**为什么它是必需的：** 这是使技能自动激活的钩子。

**集成：**
```bash
# 复制两个文件
cp skill-activation-prompt.sh your-project/.claude/hooks/
cp skill-activation-prompt.ts your-project/.claude/hooks/

# 设置为可执行
chmod +x your-project/.claude/hooks/skill-activation-prompt.sh

# 安装依赖
cd your-project/.claude/hooks
npm install
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

**自定义：** ✅ 无需自定义 - 自动读取 skill-rules.json

---

### post-tool-use-tracker（PostToolUse）

**用途：** 跟踪文件更改以跨会话维护上下文

**工作原理：**
1. 监控 Edit/Write/MultiEdit 工具调用
2. 记录哪些文件被修改
3. 为上下文管理创建缓存
4. 自动检测项目结构（前端、后端、包等）

**为什么它是必需的：** 帮助 Claude 了解代码库的哪些部分处于活动状态。

**集成：**
```bash
# 复制文件
cp post-tool-use-tracker.sh your-project/.claude/hooks/

# 设置为可执行
chmod +x your-project/.claude/hooks/post-tool-use-tracker.sh
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

**自定义：** ✅ 无需自定义 - 自动检测结构

---

## 可选钩子（需要自定义）

### tsc-check（Stop）

**用途：** 用户停止时的 TypeScript 编译检查

**⚠️ 警告：** 为多服务 monorepo 结构配置

**集成：**

**首先，确定这是否适合你：**
- ✅ 使用如果：多服务 TypeScript monorepo
- ❌ 跳过如果：单服务项目或不同的构建设置

**如果使用：**
1. 复制 tsc-check.sh
2. **编辑服务检测（约第 28 行）：**
   ```bash
   # 用你的服务替换示例服务：
   case "$repo" in
       api|web|auth|payments|...)  # ← 你的实际服务
   ```
3. 在添加到 settings.json 之前手动测试

**自定义：** ⚠️⚠️⚠️ 大量

---

### trigger-build-resolver（Stop）

**用途：** 编译失败时自动启动 build-error-resolver agent

**依赖于：** tsc-check 钩子正常工作

**自定义：** ✅ 无（但 tsc-check 必须先工作）

---

## 给 Claude Code

**为用户设置钩子时：**

1. **首先阅读 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)**
2. **始终从两个必备钩子开始**
3. **添加 Stop 钩子之前询问** - 如果配置错误可能会阻止
4. **设置后验证：**
   ```bash
   ls -la .claude/hooks/*.sh | grep rwx
   ```

**有问题？** 查看 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)
