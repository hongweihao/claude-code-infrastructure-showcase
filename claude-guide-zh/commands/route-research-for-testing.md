---
description: 映射已编辑的路由并启动测试
argument-hint: "[/extra/path …]"
allowed-tools: Bash(cat:*), Bash(awk:*), Bash(grep:*), Bash(sort:*), Bash(xargs:*), Bash(sed:*)
model: sonnet
---

## 上下文

本次会话更改的路由文件（自动生成）：

!cat "$CLAUDE_PROJECT_DIR/.claude/tsc-cache"/\*/edited-files.log \
 | awk -F: '{print $2}' \
 | grep '/routes/' \
 | sort -u

用户指定的额外路由：`$ARGUMENTS`

## 你的任务

**精确**按照编号步骤执行：

1. 将自动列表与 `$ARGUMENTS` 合并，去重，并解析 `src/app.ts`
   中定义的任何前缀。
2. 对于每个最终路由，输出一个 JSON 记录，包含路径、方法、预期的
   请求/响应结构，以及有效和无效的载荷示例。
3. **现在调用 `Task` 工具**使用：

```json
{
    "tool": "Task",
    "parameters": {
        "description": "route smoke tests",
        "prompt": "在上面的 JSON 上运行 auth-route-tester 子 agent。"
    }
}
```
