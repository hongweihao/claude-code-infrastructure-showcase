# 配置管理 - UnifiedConfig 模式

后端微服务中管理配置的完整指南。

## 目录

- [UnifiedConfig 概览](#unifiedconfig-概览)
- [永不直接使用 process.env](#永不直接使用-processenv)
- [配置结构](#配置结构)
- [特定环境配置](#特定环境配置)
- [密钥管理](#密钥管理)
- [迁移指南](#迁移指南)

---

## UnifiedConfig 概览

### 为什么选择 UnifiedConfig？

**process.env 的问题：**
- ❌ 无类型安全
- ❌ 无验证
- ❌ 难以测试
- ❌ 分散在代码各处
- ❌ 无默认值
- ❌ 拼写错误导致运行时错误

**unifiedConfig 的好处：**
- ✅ 类型安全的配置
- ✅ 单一事实来源
- ✅ 启动时验证
- ✅ 易于使用模拟测试
- ✅ 清晰的结构
- ✅ 回退到环境变量

---

## 永不直接使用 process.env

### 规则

```typescript
// ❌ 永远不要这样做
const timeout = parseInt(process.env.TIMEOUT_MS || '5000');
const dbHost = process.env.DB_HOST || 'localhost';

// ✅ 总是这样做
import { config } from './config/unifiedConfig';
const timeout = config.timeouts.default;
const dbHost = config.database.host;
```

### 为什么这很重要

**问题示例：**
```typescript
// 环境变量名称拼写错误
const host = process.env.DB_HSOT; // undefined！无错误！

// 类型安全
const port = process.env.PORT; // string！需要 parseInt
const timeout = parseInt(process.env.TIMEOUT); // 如果未设置则为 NaN！
```

**使用 unifiedConfig：**
```typescript
const port = config.server.port; // number，保证
const timeout = config.timeouts.default; // number，带回退
```

---

## 配置结构

### UnifiedConfig 接口

```typescript
export interface UnifiedConfig {
    database: {
        host: string;
        port: number;
        username: string;
        password: string;
        database: string;
    };
    server: {
        port: number;
        sessionSecret: string;
    };
    tokens: {
        jwt: string;
        inactivity: string;
        internal: string;
    };
    keycloak: {
        realm: string;
        client: string;
        baseUrl: string;
        secret: string;
    };
    aws: {
        region: string;
        emailQueueUrl: string;
        accessKeyId: string;
        secretAccessKey: string;
    };
    sentry: {
        dsn: string;
        environment: string;
        tracesSampleRate: number;
    };
    // ... 更多部分
}
```

### 实现模式

**文件：** `/blog-api/src/config/unifiedConfig.ts`

```typescript
import * as fs from 'fs';
import * as path from 'path';
import * as ini from 'ini';

const configPath = path.join(__dirname, '../../config.ini');
const iniConfig = ini.parse(fs.readFileSync(configPath, 'utf-8'));

export const config: UnifiedConfig = {
    database: {
        host: iniConfig.database?.host || process.env.DB_HOST || 'localhost',
        port: parseInt(iniConfig.database?.port || process.env.DB_PORT || '3306'),
        username: iniConfig.database?.username || process.env.DB_USER || 'root',
        password: iniConfig.database?.password || process.env.DB_PASSWORD || '',
        database: iniConfig.database?.database || process.env.DB_NAME || 'blog_dev',
    },
    server: {
        port: parseInt(iniConfig.server?.port || process.env.PORT || '3002'),
        sessionSecret: iniConfig.server?.sessionSecret || process.env.SESSION_SECRET || 'dev-secret',
    },
    // ... 更多配置
};

// 验证关键配置
if (!config.tokens.jwt) {
    throw new Error('JWT secret not configured!');
}
```

**关键点：**
- 首先从 config.ini 读取
- 回退到 process.env
- 开发的默认值
- 启动时验证
- 类型安全访问

---

## 特定环境配置

### config.ini 结构

```ini
[database]
host = localhost
port = 3306
username = root
password = password1
database = blog_dev

[server]
port = 3002
sessionSecret = your-secret-here

[tokens]
jwt = your-jwt-secret
inactivity = 30m
internal = internal-api-token

[keycloak]
realm = myapp
client = myapp-client
baseUrl = http://localhost:8080
secret = keycloak-client-secret

[sentry]
dsn = https://your-sentry-dsn
environment = development
tracesSampleRate = 0.1
```

### 环境覆盖

```bash
# .env 文件（可选覆盖）
DB_HOST=production-db.example.com
DB_PASSWORD=secure-password
PORT=80
```

**优先级：**
1. config.ini（最高优先级）
2. process.env 变量
3. 硬编码默认值（最低优先级）

---

## 密钥管理

### 不要提交密钥

```gitignore
# .gitignore
config.ini
.env
sentry.ini
*.pem
*.key
```

### 在生产环境中使用环境变量

```typescript
// 开发：config.ini
// 生产：环境变量

export const config: UnifiedConfig = {
    database: {
        password: process.env.DB_PASSWORD || iniConfig.database?.password || '',
    },
    tokens: {
        jwt: process.env.JWT_SECRET || iniConfig.tokens?.jwt || '',
    },
};
```

---

## 迁移指南

### 查找所有 process.env 用法

```bash
grep -r "process.env" blog-api/src/ --include="*.ts" | wc -l
```

### 迁移示例

**之前：**
```typescript
// 分散在代码各处
const timeout = parseInt(process.env.OPENID_HTTP_TIMEOUT_MS || '15000');
const keycloakUrl = process.env.KEYCLOAK_BASE_URL;
const jwtSecret = process.env.JWT_SECRET;
```

**之后：**
```typescript
import { config } from './config/unifiedConfig';

const timeout = config.keycloak.timeout;
const keycloakUrl = config.keycloak.baseUrl;
const jwtSecret = config.tokens.jwt;
```

**好处：**
- 类型安全
- 集中化
- 易于测试
- 启动时验证

---

**相关文件：**
- [SKILL.md](SKILL.md)
- [testing-guide.md](testing-guide.md)
