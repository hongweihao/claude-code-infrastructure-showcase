# 钩子配置指南

本指南解释了如何为你的项目配置和自定义钩子系统。

## 快速开始配置

### 1. 在 .claude/settings.json 中注册钩子

在项目根目录创建或更新 `.claude/settings.json`：

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
    ],
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
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-prettier-formatter.sh"
          },
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-build-check-enhanced.sh"
          },
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/error-handling-reminder.sh"
          }
        ]
      }
    ]
  }
}
```

### 2. 安装依赖

```bash
cd .claude/hooks
npm install
```

### 3. 设置执行权限

```bash
chmod +x .claude/hooks/*.sh
```

## 自定义选项

### 项目结构检测

默认情况下，钩子检测以下目录模式：

**前端：** `frontend/`、`client/`、`web/`、`app/`、`ui/`
**后端：** `backend/`、`server/`、`api/`、`src/`、`services/`
**数据库：** `database/`、`prisma/`、`migrations/`
**Monorepo：** `packages/*`、`examples/*`

#### 添加自定义目录模式

编辑 `.claude/hooks/post-tool-use-tracker.sh`，函数 `detect_repo()`：

```bash
case "$repo" in
    # 在此添加你的自定义目录
    my-custom-service)
        echo "$repo"
        ;;
    admin-panel)
        echo "$repo"
        ;;
    # ... 现有模式
esac
```

### 构建命令检测

钩子基于以下内容自动检测构建命令：
1. 具有 "build" 脚本的 `package.json`
2. 包管理器（pnpm > npm > yarn）
3. 特殊情况（Prisma schemas）

#### 自定义构建命令

编辑 `.claude/hooks/post-tool-use-tracker.sh`，函数 `get_build_command()`：

```bash
# 添加自定义构建逻辑
if [[ "$repo" == "my-service" ]]; then
    echo "cd $repo_path && make build"
    return
fi
```

### TypeScript 配置

钩子自动检测：
- 标准 TypeScript 项目的 `tsconfig.json`
- Vite/React 项目的 `tsconfig.app.json`

#### 自定义 TypeScript 配置

编辑 `.claude/hooks/post-tool-use-tracker.sh`，函数 `get_tsc_command()`：

```bash
if [[ "$repo" == "my-service" ]]; then
    echo "cd $repo_path && npx tsc --project tsconfig.build.json --noEmit"
    return
fi
```

### Prettier 配置

prettier 钩子按以下顺序搜索配置：
1. 当前文件目录（向上遍历）
2. 项目根目录
3. 回退到 Prettier 默认值

#### 自定义 Prettier 配置搜索

编辑 `.claude/hooks/stop-prettier-formatter.sh`，函数 `get_prettier_config()`：

```bash
# 添加自定义配置位置
if [[ -f "$project_root/config/.prettierrc" ]]; then
    echo "$project_root/config/.prettierrc"
    return
fi
```

### 错误处理提醒

在 `.claude/hooks/error-handling-reminder.ts` 中配置文件类别检测：

```typescript
function getFileCategory(filePath: string): 'backend' | 'frontend' | 'database' | 'other' {
    // 添加自定义模式
    if (filePath.includes('/my-custom-dir/')) return 'backend';
    // ... 现有模式
}
```

### 错误阈值配置

更改何时推荐 auto-error-resolver agent。

编辑 `.claude/hooks/stop-build-check-enhanced.sh`：

```bash
# 默认是 5 个错误 - 根据你的偏好更改
if [[ $total_errors -ge 10 ]]; then  # 现在需要 10+ 个错误
    # 推荐 agent
fi
```

## 环境变量

### 全局环境变量

在你的 shell 配置文件中设置（`.bashrc`、`.zshrc` 等）：

```bash
# 禁用错误处理提醒
export SKIP_ERROR_REMINDER=1

# 自定义项目目录（如果不使用默认值）
export CLAUDE_PROJECT_DIR=/path/to/your/project
```

### 每次会话环境变量

在启动 Claude Code 之前设置：

```bash
SKIP_ERROR_REMINDER=1 claude-code
```

## 钩子执行顺序

Stop 钩子按 `settings.json` 中指定的顺序运行：

```json
"Stop": [
  {
    "hooks": [
      { "command": "...formatter.sh" },    // 首先运行
      { "command": "...build-check.sh" },  // 第二运行
      { "command": "...reminder.sh" }      // 第三运行
    ]
  }
]
```

**为什么这个顺序很重要：**
1. 首先格式化文件（清洁代码）
2. 然后检查错误
3. 最后显示提醒

## 选择性钩子启用

你不需要所有钩子。选择适合你项目的：

### 最小设置（仅技能激活）

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

### 仅构建检查（无格式化）

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
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-build-check-enhanced.sh"
          }
        ]
      }
    ]
  }
}
```

### 仅格式化（无构建检查）

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
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-prettier-formatter.sh"
          }
        ]
      }
    ]
  }
}
```

## 缓存管理

### 缓存位置

```
$CLAUDE_PROJECT_DIR/.claude/tsc-cache/[session_id]/
```

### 手动缓存清理

```bash
# 删除所有缓存数据
rm -rf $CLAUDE_PROJECT_DIR/.claude/tsc-cache/*

# 删除特定会话
rm -rf $CLAUDE_PROJECT_DIR/.claude/tsc-cache/[session-id]
```

### 自动清理

build-check 钩子在成功构建时自动清理会话缓存。

## 配置故障排除

### 钩子未执行

1. **检查注册：** 验证钩子在 `.claude/settings.json` 中
2. **检查权限：** 运行 `chmod +x .claude/hooks/*.sh`
3. **检查路径：** 确保 `$CLAUDE_PROJECT_DIR` 设置正确
4. **检查 TypeScript：** 运行 `cd .claude/hooks && npx tsc` 检查错误

### 误报检测

**问题：** 钩子为不应该触发的文件触发

**解决方案：** 在相关钩子中添加跳过条件：

```bash
# 在 post-tool-use-tracker.sh 中
if [[ "$file_path" =~ /generated/ ]]; then
    exit 0  # 跳过生成的文件
fi
```

### 性能问题

**问题：** 钩子很慢

**解决方案：**
1. 仅将 TypeScript 检查限制为更改的文件
2. 使用更快的包管理器（pnpm > npm）
3. 添加更多跳过条件
4. 对大文件禁用 Prettier

```bash
# 在 stop-prettier-formatter.sh 中跳过大文件
file_size=$(wc -c < "$file" 2>/dev/null || echo 0)
if [[ $file_size -gt 100000 ]]; then  # 跳过 > 100KB 的文件
    continue
fi
```

### 调试钩子

向任何钩子添加调试输出：

```bash
# 在钩子脚本顶部
set -x  # 启用调试模式

# 或添加特定调试行
echo "DEBUG: file_path=$file_path" >&2
echo "DEBUG: repo=$repo" >&2
```

在 Claude Code 的日志中查看钩子执行。

## 高级配置

### 自定义钩子事件处理程序

你可以为其他事件创建自己的钩子：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/my-custom-bash-guard.sh"
          }
        ]
      }
    ]
  }
}
```

### Monorepo 配置

对于具有多个包的 monorepo：

```bash
# 在 post-tool-use-tracker.sh 中，detect_repo()
case "$repo" in
    packages)
        # 获取包名称
        local package=$(echo "$relative_path" | cut -d'/' -f2)
        if [[ -n "$package" ]]; then
            echo "packages/$package"
        else
            echo "$repo"
        fi
        ;;
esac
```

### Docker/容器项目

如果你的构建命令需要在容器中运行：

```bash
# 在 post-tool-use-tracker.sh 中，get_build_command()
if [[ "$repo" == "api" ]]; then
    echo "docker-compose exec api npm run build"
    return
fi
```

## 最佳实践

1. **从最小开始** - 一次启用一个钩子
2. **彻底测试** - 进行更改并验证钩子工作
3. **记录自定义** - 添加注释解释自定义逻辑
4. **版本控制** - 将 `.claude/` 目录提交到 git
5. **团队一致性** - 在团队中共享配置

## 另请参阅

- [README.md](./README.md) - 钩子概览
- [../../docs/HOOKS_SYSTEM.md](../../docs/HOOKS_SYSTEM.md) - 完整钩子参考
- [../../docs/SKILLS_SYSTEM.md](../../docs/SKILLS_SYSTEM.md) - 技能集成
