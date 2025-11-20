# skill-rules.json - 完整参考

`.claude/skills/skill-rules.json` 的完整架构和配置参考。

## 目录

- [文件位置](#文件位置)
- [完整 TypeScript 架构](#完整-typescript-架构)
- [字段指南](#字段指南)
- [示例：防护栏技能](#示例防护栏技能)
- [示例：领域技能](#示例领域技能)
- [验证](#验证)

---

## 文件位置

**路径：** `.claude/skills/skill-rules.json`

此 JSON 文件定义了自动激活系统的所有技能及其触发条件。

---

## 完整 TypeScript 架构

```typescript
interface SkillRules {
    version: string;
    skills: Record<string, SkillRule>;
}

interface SkillRule {
    type: 'guardrail' | 'domain';
    enforcement: 'block' | 'suggest' | 'warn';
    priority: 'critical' | 'high' | 'medium' | 'low';

    promptTriggers?: {
        keywords?: string[];
        intentPatterns?: string[];  // 正则表达式字符串
    };

    fileTriggers?: {
        pathPatterns: string[];     // Glob 模式
        pathExclusions?: string[];  // Glob 模式
        contentPatterns?: string[]; // 正则表达式字符串
        createOnly?: boolean;       // 仅在文件创建时触发
    };

    blockMessage?: string;  // 用于防护栏，{file_path} 占位符

    skipConditions?: {
        sessionSkillUsed?: boolean;      // 如果会话中已使用则跳过
        fileMarkers?: string[];          // 例如，["@skip-validation"]
        envOverride?: string;            // 例如，"SKIP_DB_VERIFICATION"
    };
}
```

---

## 字段指南

### 顶层

| 字段 | 类型 | 必需 | 描述 |
|------|------|------|------|
| `version` | string | 是 | 架构版本（当前为 "1.0"）|
| `skills` | object | 是 | 技能名称 → SkillRule 的映射 |

### SkillRule 字段

| 字段 | 类型 | 必需 | 描述 |
|------|------|------|------|
| `type` | string | 是 | "guardrail"（强制）或 "domain"（建议）|
| `enforcement` | string | 是 | "block"（PreToolUse）、"suggest"（UserPromptSubmit）或 "warn" |
| `priority` | string | 是 | "critical"、"high"、"medium" 或 "low" |
| `promptTriggers` | object | 可选 | UserPromptSubmit 钩子的触发器 |
| `fileTriggers` | object | 可选 | PreToolUse 钩子的触发器 |
| `blockMessage` | string | 可选* | 如果 enforcement="block" 则必需。使用 `{file_path}` 占位符 |
| `skipConditions` | object | 可选 | 逃生口和会话跟踪 |

*防护栏必需

### promptTriggers 字段

| 字段 | 类型 | 必需 | 描述 |
|------|------|------|------|
| `keywords` | string[] | 可选 | 精确子字符串匹配（不区分大小写）|
| `intentPatterns` | string[] | 可选 | 意图检测的正则表达式模式 |

### fileTriggers 字段

| 字段 | 类型 | 必需 | 描述 |
|------|------|------|------|
| `pathPatterns` | string[] | 是* | 文件路径的 Glob 模式 |
| `pathExclusions` | string[] | 可选 | 要排除的 Glob 模式（例如，测试文件）|
| `contentPatterns` | string[] | 可选 | 匹配文件内容的正则表达式模式 |
| `createOnly` | boolean | 可选 | 仅在创建新文件时触发 |

*如果存在 fileTriggers 则必需

### skipConditions 字段

| 字段 | 类型 | 必需 | 描述 |
|------|------|------|------|
| `sessionSkillUsed` | boolean | 可选 | 如果此会话中已使用技能则跳过 |
| `fileMarkers` | string[] | 可选 | 如果文件包含注释标记则跳过 |
| `envOverride` | string | 可选 | 禁用技能的环境变量名称 |

---

## 示例：防护栏技能

具有所有功能的阻止防护栏技能的完整示例：

```json
{
  "database-verification": {
    "type": "guardrail",
    "enforcement": "block",
    "priority": "critical",

    "promptTriggers": {
      "keywords": [
        "prisma",
        "database",
        "table",
        "column",
        "schema",
        "query",
        "migration"
      ],
      "intentPatterns": [
        "(add|create|implement).*?(user|login|auth|tracking|feature)",
        "(modify|update|change).*?(table|column|schema|field)",
        "database.*?(change|update|modify|migration)"
      ]
    },

    "fileTriggers": {
      "pathPatterns": [
        "**/schema.prisma",
        "**/migrations/**/*.sql",
        "database/src/**/*.ts",
        "form/src/**/*.ts",
        "email/src/**/*.ts",
        "users/src/**/*.ts",
        "projects/src/**/*.ts",
        "utilities/src/**/*.ts"
      ],
      "pathExclusions": [
        "**/*.test.ts",
        "**/*.spec.ts"
      ],
      "contentPatterns": [
        "import.*[Pp]risma",
        "PrismaService",
        "prisma\\.",
        "\\.findMany\\(",
        "\\.findUnique\\(",
        "\\.findFirst\\(",
        "\\.create\\(",
        "\\.createMany\\(",
        "\\.update\\(",
        "\\.updateMany\\(",
        "\\.upsert\\(",
        "\\.delete\\(",
        "\\.deleteMany\\("
      ]
    },

    "blockMessage": "⚠️ BLOCKED - Database Operation Detected\n\n📋 REQUIRED ACTION:\n1. Use Skill tool: 'database-verification'\n2. Verify ALL table and column names against schema\n3. Check database structure with DESCRIBE commands\n4. Then retry this edit\n\nReason: Prevent column name errors in Prisma queries\nFile: {file_path}\n\n💡 TIP: Add '// @skip-validation' comment to skip future checks",

    "skipConditions": {
      "sessionSkillUsed": true,
      "fileMarkers": [
        "@skip-validation"
      ],
      "envOverride": "SKIP_DB_VERIFICATION"
    }
  }
}
```

### 防护栏的关键点

1. **type**：必须是 "guardrail"
2. **enforcement**：必须是 "block"
3. **priority**：通常是 "critical" 或 "high"
4. **blockMessage**：必需，清晰可操作的步骤
5. **skipConditions**：会话跟踪防止重复提醒
6. **fileTriggers**：通常同时具有路径和内容模式
7. **contentPatterns**：捕获技术的实际使用

---

## 示例：领域技能

基于建议的领域技能的完整示例：

```json
{
  "project-catalog-developer": {
    "type": "domain",
    "enforcement": "suggest",
    "priority": "high",

    "promptTriggers": {
      "keywords": [
        "layout",
        "layout system",
        "grid",
        "grid layout",
        "toolbar",
        "column",
        "cell editor",
        "cell renderer",
        "submission",
        "submissions",
        "blog dashboard",
        "datagrid",
        "data grid",
        "CustomToolbar",
        "GridLayoutDialog",
        "useGridLayout",
        "auto-save",
        "column order",
        "column width",
        "filter",
        "sort"
      ],
      "intentPatterns": [
        "(how does|how do|explain|what is|describe).*?(layout|grid|toolbar|column|submission|catalog)",
        "(add|create|modify|change).*?(toolbar|column|cell|editor|renderer)",
        "blog dashboard.*?"
      ]
    },

    "fileTriggers": {
      "pathPatterns": [
        "frontend/src/features/submissions/**/*.tsx",
        "frontend/src/features/submissions/**/*.ts"
      ],
      "pathExclusions": [
        "**/*.test.tsx",
        "**/*.test.ts"
      ]
    }
  }
}
```

### 领域技能的关键点

1. **type**：必须是 "domain"
2. **enforcement**：通常是 "suggest"
3. **priority**："high" 或 "medium"
4. **blockMessage**：不需要（不阻止）
5. **skipConditions**：可选（不太关键）
6. **promptTriggers**：通常有大量关键词
7. **fileTriggers**：可能只有路径模式（内容不太重要）

---

## 验证

### 检查 JSON 语法

```bash
cat .claude/skills/skill-rules.json | jq .
```

如果有效，jq 将美化打印 JSON。如果无效，它将显示错误。

### 常见 JSON 错误

**尾随逗号：**
```json
{
  "keywords": ["one", "two",]  // ❌ 尾随逗号
}
```

**缺少引号：**
```json
{
  type: "guardrail"  // ❌ 键缺少引号
}
```

**单引号（无效的 JSON）：**
```json
{
  'type': 'guardrail'  // ❌ 必须使用双引号
}
```

### 验证清单

- [ ] JSON 语法有效（使用 `jq`）
- [ ] 所有技能名称与 SKILL.md 文件名匹配
- [ ] 防护栏有 `blockMessage`
- [ ] 阻止消息使用 `{file_path}` 占位符
- [ ] 意图模式是有效的正则表达式（在 regex101.com 上测试）
- [ ] 文件路径模式使用正确的 glob 语法
- [ ] 内容模式转义特殊字符
- [ ] 优先级与执行级别匹配
- [ ] 无重复的技能名称

---

**相关文件：**
- [SKILL.md](SKILL.md) - 主技能指南
- [TRIGGER_TYPES.md](TRIGGER_TYPES.md) - 完整触发器文档
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - 调试配置问题
