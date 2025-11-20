---
name: route-tester
description: 使用基于 cookie 的身份验证测试项目中的经过身份验证的路由。在测试 API 端点、验证路由功能或调试身份验证问题时使用此技能。包括使用 test-auth-route.js 和模拟身份验证的模式。
---

# 项目路由测试技能

## 目的
此技能提供使用基于 cookie 的 JWT 身份验证测试项目中经过身份验证的路由的模式。

## 何时使用此技能
- 测试新的 API 端点
- 在更改后验证路由功能
- 调试身份验证问题
- 测试 POST/PUT/DELETE 操作
- 验证请求/响应数据

## 项目身份验证概述

项目使用：
- **Keycloak** 用于 SSO（realm: yourRealm）
- **基于 Cookie 的 JWT** 令牌（不是 Bearer headers）
- **Cookie 名称**：`refresh_token`
- **JWT 签名**：使用 `config.ini` 中的密钥

## 测试方法

### 方法 1：test-auth-route.js（推荐）

`test-auth-route.js` 脚本自动处理所有身份验证复杂性。

**位置**：`/root/git/your project_pre/scripts/test-auth-route.js`

#### 基本 GET 请求

```bash
node scripts/test-auth-route.js http://localhost:3000/blog-api/api/endpoint
```

#### 带 JSON 数据的 POST 请求

```bash
node scripts/test-auth-route.js \
    http://localhost:3000/blog-api/777/submit \
    POST \
    '{"responses":{"4577":"13295"},"submissionID":5,"stepInstanceId":"11"}'
```

#### 脚本的作用

1. 从 Keycloak 获取刷新令牌
   - 用户名：`testuser`
   - 密码：`testpassword`
2. 使用 `config.ini` 中的 JWT 密钥签名令牌
3. 创建 cookie header：`refresh_token=<signed-token>`
4. 进行经过身份验证的请求
5. 显示用于手动重现的确切 curl 命令

#### 脚本输出

脚本输出：
- 请求详细信息
- 响应状态和正文
- 用于手动重现的 curl 命令

**注意**：脚本很冗长 - 在输出中查找实际响应。

### 方法 2：使用令牌的手动 curl

使用 test-auth-route.js 输出中的 curl 命令：

```bash
# 脚本输出类似于：
# 💡 To test manually with curl:
# curl -b "refresh_token=eyJhbGci..." http://localhost:3000/blog-api/api/endpoint

# 复制并修改该 curl 命令：
curl -X POST http://localhost:3000/blog-api/777/submit \
  -H "Content-Type: application/json" \
  -b "refresh_token=<COPY_TOKEN_FROM_SCRIPT_OUTPUT>" \
  -d '{"your": "data"}'
```

### 方法 3：模拟身份验证（仅开发 - 最简单）

对于开发，完全绕过 Keycloak 使用模拟身份验证。

#### 设置

```bash
# 添加到服务 .env 文件（例如 blog-api/.env）
MOCK_AUTH=true
MOCK_USER_ID=test-user
MOCK_USER_ROLES=admin,operations
```

#### 使用

```bash
curl -H "X-Mock-Auth: true" \
     -H "X-Mock-User: test-user" \
     -H "X-Mock-Roles: admin,operations" \
     http://localhost:3002/api/protected
```

#### 模拟身份验证要求

模拟身份验证仅在以下情况下工作：
- `NODE_ENV` 是 `development` 或 `test`
- `mockAuth` 中间件已添加到路由
- 永远不会在生产环境中工作（安全功能）

## 常见测试模式

### 测试表单提交

```bash
node scripts/test-auth-route.js \
    http://localhost:3000/blog-api/777/submit \
    POST \
    '{"responses":{"4577":"13295"},"submissionID":5,"stepInstanceId":"11"}'
```

### 测试工作流启动

```bash
node scripts/test-auth-route.js \
    http://localhost:3002/api/workflow/start \
    POST \
    '{"workflowCode":"DHS_CLOSEOUT","entityType":"Submission","entityID":123}'
```

### 测试工作流步骤完成

```bash
node scripts/test-auth-route.js \
    http://localhost:3002/api/workflow/step/complete \
    POST \
    '{"stepInstanceID":789,"answers":{"decision":"approved","comments":"Looks good"}}'
```

### 测试带查询参数的 GET

```bash
node scripts/test-auth-route.js \
    "http://localhost:3002/api/workflows?status=active&limit=10"
```

### 测试文件上传

```bash
# 首先从 test-auth-route.js 获取令牌，然后：
curl -X POST http://localhost:5000/upload \
  -H "Content-Type: multipart/form-data" \
  -b "refresh_token=<TOKEN>" \
  -F "file=@/path/to/file.pdf" \
  -F "metadata={\"description\":\"Test file\"}"
```

## 硬编码的测试凭据

`test-auth-route.js` 脚本使用这些凭据：

- **用户名**：`testuser`
- **密码**：`testpassword`
- **Keycloak URL**：来自 `config.ini`（通常是 `http://localhost:8081`）
- **Realm**：`yourRealm`
- **Client ID**：来自 `config.ini`

## 服务端口

| 服务 | 端口 | 基础 URL |
|---------|------|----------|
| Users   | 3000 | http://localhost:3000 |
| Projects| 3001 | http://localhost:3001 |
| Form    | 3002 | http://localhost:3002 |
| Email   | 3003 | http://localhost:3003 |
| Uploads | 5000 | http://localhost:5000 |

## 路由前缀

检查每个服务中的 `/src/app.ts` 以获取路由前缀：

```typescript
// 来自 blog-api/src/app.ts 的示例
app.use('/blog-api/api', formRoutes);          // 前缀：/blog-api/api
app.use('/api/workflow', workflowRoutes);  // 前缀：/api/workflow
```

**完整路由** = 基础 URL + 前缀 + 路由路径

示例：
- 基础：`http://localhost:3002`
- 前缀：`/form`
- 路由：`/777/submit`
- **完整 URL**：`http://localhost:3000/blog-api/777/submit`

## 测试检查清单

测试路由之前：

- [ ] 识别服务（form、email、users 等）
- [ ] 找到正确的端口
- [ ] 检查 `app.ts` 中的路由前缀
- [ ] 构建完整 URL
- [ ] 准备请求正文（如果 POST/PUT）
- [ ] 确定身份验证方法
- [ ] 运行测试
- [ ] 验证响应状态和数据
- [ ] 检查数据库更改（如果适用）

## 验证数据库更改

测试修改数据的路由后：

```bash
# 连接到 MySQL
docker exec -i local-mysql mysql -u root -ppassword1 blog_dev

# 检查特定表
mysql> SELECT * FROM WorkflowInstance WHERE id = 123;
mysql> SELECT * FROM WorkflowStepInstance WHERE instanceId = 123;
mysql> SELECT * FROM WorkflowNotification WHERE recipientUserId = 'user-123';
```

## 调试失败的测试

### 401 Unauthorized

**可能的原因**：
1. 令牌过期（使用 test-auth-route.js 重新生成）
2. 错误的 cookie 格式
3. JWT 密钥不匹配
4. Keycloak 未运行

**解决方案**：
```bash
# 检查 Keycloak 是否运行
docker ps | grep keycloak

# 重新生成令牌
node scripts/test-auth-route.js http://localhost:3002/api/health

# 验证 config.ini 是否有正确的 jwtSecret
```

### 403 Forbidden

**可能的原因**：
1. 用户缺少所需角色
2. 资源权限不正确
3. 路由需要特定权限

**解决方案**：
```bash
# 使用具有 admin 角色的模拟身份验证
curl -H "X-Mock-Auth: true" \
     -H "X-Mock-User: test-admin" \
     -H "X-Mock-Roles: admin" \
     http://localhost:3002/api/protected
```

### 404 Not Found

**可能的原因**：
1. URL 不正确
2. 缺少路由前缀
3. 路由未注册

**解决方案**：
1. 检查 `app.ts` 中的路由前缀
2. 验证路由注册
3. 检查服务是否运行（`pm2 list`）

### 500 Internal Server Error

**可能的原因**：
1. 数据库连接问题
2. 缺少必需字段
3. 验证错误
4. 应用程序错误

**解决方案**：
1. 检查服务日志（`pm2 logs <service>`）
2. 检查 Sentry 以获取错误详细信息
3. 验证请求正文是否与预期 schema 匹配
4. 检查数据库连接

## 使用 auth-route-tester Agent

在进行更改后进行全面的路由测试：

1. **识别受影响的路由**
2. **收集路由信息**：
   - 完整路由路径（带前缀）
   - 预期的 POST 数据
   - 要验证的表
3. **调用 auth-route-tester agent**

Agent 将：
- 使用适当的身份验证测试路由
- 验证数据库更改
- 检查响应格式
- 报告任何问题

## 示例测试场景

### 创建新路由后

```bash
# 1. 使用有效数据测试
node scripts/test-auth-route.js \
    http://localhost:3002/api/my-new-route \
    POST \
    '{"field1":"value1","field2":"value2"}'

# 2. 验证数据库
docker exec -i local-mysql mysql -u root -ppassword1 blog_dev \
    -e "SELECT * FROM MyTable ORDER BY createdAt DESC LIMIT 1;"

# 3. 使用无效数据测试
node scripts/test-auth-route.js \
    http://localhost:3002/api/my-new-route \
    POST \
    '{"field1":"invalid"}'

# 4. 不使用身份验证测试
curl http://localhost:3002/api/my-new-route
# 应返回 401
```

### 修改路由后

```bash
# 1. 测试现有功能仍然有效
node scripts/test-auth-route.js \
    http://localhost:3002/api/existing-route \
    POST \
    '{"existing":"data"}'

# 2. 测试新功能
node scripts/test-auth-route.js \
    http://localhost:3002/api/existing-route \
    POST \
    '{"new":"field","existing":"data"}'

# 3. 验证向后兼容性
# 使用旧请求格式测试（如果适用）
```

## 配置文件

### config.ini（每个服务）

```ini
[keycloak]
url = http://localhost:8081
realm = yourRealm
clientId = app-client

[jwt]
jwtSecret = your-jwt-secret-here
```

### .env（每个服务）

```bash
NODE_ENV=development
MOCK_AUTH=true           # 可选：启用模拟身份验证
MOCK_USER_ID=test-user   # 可选：默认模拟用户
MOCK_USER_ROLES=admin    # 可选：默认模拟角色
```

## 关键文件

- `/root/git/your project_pre/scripts/test-auth-route.js` - 主测试脚本
- `/blog-api/src/app.ts` - Form 服务路由
- `/notifications/src/app.ts` - Email 服务路由
- `/auth/src/app.ts` - Users 服务路由
- `/config.ini` - 服务配置
- `/.env` - 环境变量

## 相关技能

- 使用 **database-verification** 验证数据库更改
- 使用 **error-tracking** 检查捕获的错误
- 使用 **workflow-builder** 进行工作流路由测试
- 使用 **notification-sender** 验证发送的通知
