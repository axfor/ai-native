# ⭐ Prompt 03 — 用户注册 + 登录

> **Sprint 1** | 预计 AI 执行时间：2-3 分钟

## 目标

实现用户注册和登录的核心业务逻辑，包含密码强度校验、bcrypt 哈希、参数验证中间件和统一响应格式，构成认证系统的入口。

## 最佳实践

| 实践 | 说明 | 参考 |
|------|------|------|
| bcrypt ≥ 12 轮 | 2024 年推荐 12+ 轮，平衡安全性与性能 | OWASP Password Storage |
| 密码强度策略 | 至少 8 位，混合大小写+数字+特殊字符 | NIST SP 800-63B |
| 模糊错误信息 | 登录失败统一返回"用户名或密码错误"，不透露哪个错 | OWASP Authentication |
| 敏感字段过滤 | API 响应永远不返回 passwordHash、mfaSecret | OWASP API Security Top 10 |
| 注册防重放 | 用户名 + 邮箱唯一约束，数据库级别防并发注册 | Database Constraints |
| 登录速率限制 | 同一 IP/用户 5 次失败后锁定 15 分钟 | OWASP Brute Force Prevention |

## 详细需求

### 1. 认证服务 `src/services/auth.service.js`

| 方法 | 输入 | 逻辑 | 输出 |
|------|------|------|------|
| `register(username, email, password)` | 用户名、邮箱、密码 | 密码强度校验 → bcrypt(12轮) → 创建用户 → 签发 Token | `{user, tokens}` |
| `login(username, password)` | 用户名、密码 | 查用户 → 校验密码 → 检查 isActive → 检查 MFA → 签发 Token | `{user, tokens}` 或 `{requireMfa, mfaToken}` |
| `sanitizeUser(user)` | User 对象 | 过滤敏感字段 | 安全的用户对象 |

**密码强度规则**（全部满足才通过）：

| 规则 | 正则 | 错误提示 |
|------|------|----------|
| 最少 8 位 | `.{8,}` | 密码长度至少 8 位 |
| 包含大写 | `[A-Z]` | 必须包含大写字母 |
| 包含小写 | `[a-z]` | 必须包含小写字母 |
| 包含数字 | `[0-9]` | 必须包含数字 |
| 包含特殊字符 | `[!@#$%^&*]` | 必须包含特殊字符 |

**敏感字段过滤列表**：passwordHash、mfaSecret、recoveryCodes

### 2. 校验中间件 `src/middlewares/validate.js`

通用校验中间件，**纯手写不依赖外部库**（如 Joi、express-validator）。

支持的校验规则：

| 规则 | 示例 | 说明 |
|------|------|------|
| `required` | `required: true` | 必填 |
| `minLength` | `minLength: 3` | 最小长度 |
| `maxLength` | `maxLength: 50` | 最大长度 |
| `pattern` | `pattern: /^[a-zA-Z0-9_]+$/` | 正则匹配 |
| `isEmail` | `isEmail: true` | 邮箱格式 |
| `isIn` | `isIn: ['USER','ADMIN']` | 枚举值 |

### 3. 路由 `src/routes/auth.routes.js`

| 方法 | 路径 | 中间件 | 说明 |
|------|------|--------|------|
| POST | `/auth/register` | validate | 用户注册 |
| POST | `/auth/login` | validate | 用户登录 |

### 4. 统一响应格式

```json
// 成功
{ "success": true, "data": { ... } }

// 失败
{ "success": false, "error": { "message": "...", "code": "VALIDATION_ERROR" } }
```

## 注册登录流程

```
=== 用户注册流程 ===

  Client             Validate MW         Auth Service          DB
    |                    |                    |                  |
    | POST /auth/register                    |                  |
    | {username, email,  |                    |                  |
    |  password}         |                    |                  |
    |------------------->|                    |                  |
    |                    | 1. 校验必填        |                  |
    |                    | 2. 校验用户名格式  |                  |
    |                    | 3. 校验邮箱格式    |                  |
    |                    | 4. 校验密码长度    |                  |
    |                    |------------------->|                  |
    |                    |                    | 5. 密码强度校验  |
    |                    |                    |    (大小写+数字  |
    |                    |                    |     +特殊字符)   |
    |                    |                    | 6. bcrypt 12轮   |
    |                    |                    | 7. 查重(用户名+邮箱)
    |                    |                    |----------------->|
    |                    |                    | 8. 创建用户      |
    |                    |                    |<-----------------|
    |                    |                    | 9. 签发 TokenPair|
    |                    |                    | 10. sanitizeUser |
    |                    |                    |                  |
    | {success: true,    |                    |                  |
    |  data: {user,      |<-------------------|                  |
    |   accessToken,     |                    |                  |
    |   refreshToken}}   |                    |                  |
    |<-------------------|                    |                  |

=== 用户登录流程 ===

  Client             Validate MW         Auth Service          DB
    |                    |                    |                  |
    | POST /auth/login   |                    |                  |
    | {username, password}                    |                  |
    |------------------->|                    |                  |
    |                    | 1. 校验必填        |                  |
    |                    |------------------->|                  |
    |                    |                    | 2. 查用户        |
    |                    |                    |----------------->|
    |                    |                    |<-----------------|
    |                    |                    |                  |
    |                    |                    | 3. bcrypt.compare |
    |                    |                    | 4. 检查 isActive  |
    |                    |                    | 5. 检查 mfaEnabled|
    |                    |                    |                  |
    |                    |             +------+                  |
    |                    |             |MFA OFF: 签发 TokenPair  |
    |                    |             |MFA ON:  返回 mfaToken   |
    |                    |             +----->|                  |
    |                    |                    |                  |
    | {success: true,    |<-------------------|                  |
    |  data: {...}}      |                    |                  |
    |<-------------------|                    |                  |

=== 密码校验失败场景 ===

  注册 "123"            -> 400 { error: "密码长度至少 8 位" }
  注册 "abcdefgh"       -> 400 { error: "必须包含大写字母" }
  注册 "Abcdefgh"       -> 400 { error: "必须包含数字" }
  注册 "Abcdefg1"       -> 400 { error: "必须包含特殊字符" }
  注册 "Demo@123456"    -> 201 注册成功
  重复注册同一用户名    -> 409 { error: "用户名已存在" }
  登录错误密码          -> 401 { error: "用户名或密码错误" }（不透露具体哪个错）
```

## API 示例

### POST /auth/register

**成功注册**

Request:
```bash
curl -s -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "johndoe",
    "email": "john@example.com",
    "password": "Secure@123456"
  }' | jq
```

Response (201):
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "username": "johndoe",
      "email": "john@example.com",
      "role": "USER",
      "isActive": true,
      "mfaEnabled": false,
      "createdAt": "2024-01-15T08:30:00.123Z",
      "updatedAt": "2024-01-15T08:30:00.123Z"
    },
    "accessToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiJmNDdhYzEwYi01OGNjLTQzNzItYTU2Ny0wZTAyYjJjM2Q0NzkiLCJyb2xlIjoiVVNFUiIsImlhdCI6MTcwNTMwMjIwMCwiZXhwIjoxNzA1MzAzMTAwfQ.signature...",
    "refreshToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiJmNDdhYzEwYi01OGNjLTQzNzItYTU2Ny0wZTAyYjJjM2Q0NzkiLCJ0eXBlIjoicmVmcmVzaCIsImlhdCI6MTcwNTMwMjIwMCwiZXhwIjoxNzA1OTA3MDAwfQ.signature...",
    "expiresIn": 900
  }
}
```

**弱密码被拒绝**

Request:
```bash
curl -s -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "weakuser",
    "email": "weak@example.com",
    "password": "123"
  }' | jq
```

Response (400):
```json
{
  "success": false,
  "error": {
    "message": "密码长度至少 8 位",
    "code": "VALIDATION_ERROR"
  }
}
```

**用户名已存在**

Request:
```bash
curl -s -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "johndoe",
    "email": "another@example.com",
    "password": "Secure@123456"
  }' | jq
```

Response (409):
```json
{
  "success": false,
  "error": {
    "message": "用户名已存在",
    "code": "CONFLICT"
  }
}
```

### POST /auth/login

**登录成功**

Request:
```bash
curl -s -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "johndoe",
    "password": "Secure@123456"
  }' | jq
```

Response (200):
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "username": "johndoe",
      "email": "john@example.com",
      "role": "USER",
      "isActive": true,
      "mfaEnabled": false,
      "createdAt": "2024-01-15T08:30:00.123Z",
      "updatedAt": "2024-01-15T08:30:00.123Z"
    },
    "accessToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiJmNDdhYzEwYi01OGNjLTQzNzItYTU2Ny0wZTAyYjJjM2Q0NzkiLCJyb2xlIjoiVVNFUiIsImlhdCI6MTcwNTMwNTgwMCwiZXhwIjoxNzA1MzA2NzAwfQ.signature...",
    "refreshToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiJmNDdhYzEwYi01OGNjLTQzNzItYTU2Ny0wZTAyYjJjM2Q0NzkiLCJ0eXBlIjoicmVmcmVzaCIsImlhdCI6MTcwNTMwNTgwMCwiZXhwIjoxNzA1OTEwNjAwfQ.signature...",
    "expiresIn": 900
  }
}
```

**密码错误**

Request:
```bash
curl -s -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "johndoe",
    "password": "WrongPassword@1"
  }' | jq
```

Response (401):
```json
{
  "success": false,
  "error": {
    "message": "用户名或密码错误",
    "code": "UNAUTHORIZED"
  }
}
```

## Prompt

```
实现用户注册和登录：

1. src/services/auth.service.js：
   - register(username, email, password)：密码强度校验（≥8位，大小写+数字+特殊字符），bcrypt(12轮)，创建用户
   - login(username, password)：查用户 → 校验密码 → 检查 isActive → 检查 MFA → 签发 Token
   - sanitizeUser(user)：过滤 passwordHash, mfaSecret 等敏感字段

2. src/middlewares/validate.js：通用校验中间件，纯手写不依赖外部库
   - 支持 required, minLength, maxLength, pattern, isEmail, isIn

3. src/routes/auth.routes.js：
   - POST /auth/register — 输入校验 + 注册
   - POST /auth/login — 登录

4. 在 app.js 注册路由

5. 统一响应格式：{ success: true/false, data/error }

完成后用 curl 测试注册和登录。
```

## 验收标准

| # | 验收项 | 验证方式 | 预期结果 |
|---|--------|----------|----------|
| 1 | 正常注册 | curl 发注册请求 | 201，返回 user + tokens |
| 2 | 弱密码拒绝 | 密码 "123" 注册 | 400，提示密码不符合要求 |
| 3 | 重复用户名 | 相同用户名注册两次 | 409，提示用户名已存在 |
| 4 | 正常登录 | curl 发登录请求 | 200，返回 accessToken + refreshToken |
| 5 | 错误密码 | 用错误密码登录 | 401，不透露是用户名还是密码错 |
| 6 | 敏感字段过滤 | 检查响应 data.user | 不含 passwordHash、mfaSecret |
| 7 | bcrypt 12 轮 | 检查代码 | bcrypt.hash(password, 12) |
| 8 | 邮箱校验 | 不合法邮箱注册 | 400，提示邮箱格式错误 |
| 9 | 统一响应格式 | 检查所有响应 | 均为 `{success, data/error}` 格式 |

## 验证

```bash
# 注册
curl -s -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"demo","email":"demo@test.com","password":"Demo@123456"}' | jq

# 弱密码被拒绝
curl -s -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"weak","email":"weak@test.com","password":"123"}' | jq

# 登录
curl -s -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"demo","password":"Demo@123456"}' | jq
```

## 讲师要点

- 检查密码强度校验是否完整（大小写+数字+特殊字符）
- 检查 bcrypt 是否使用了 12 轮（不是默认的 10 轮）
- 检查 API 响应中是否排除了 passwordHash
- 测试弱密码应被拒绝
