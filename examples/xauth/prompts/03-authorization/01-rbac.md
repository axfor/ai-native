# ⭐ Prompt 05 — RBAC 权限控制

> **Sprint 3** | 预计 AI 执行时间：2-3 分钟

## 目标

实现基于角色的访问控制（RBAC），包含角色守卫、资源所有权守卫和审计日志，确保不同角色（ADMIN/USER/GUEST）拥有不同的操作权限，并记录所有敏感操作。

## 最佳实践

| 实践 | 说明 | 参考 |
|------|------|------|
| 最小权限原则 | 默认拒绝一切，仅授予完成任务所需的最小权限 | NIST RBAC Model |
| 资源所有权守卫 | 用户只能操作自己的资源，管理员可跳过 | OWASP IDOR Prevention |
| 角色与权限分离 | 角色是权限的集合，避免硬编码角色判断 | RBAC NIST SP 800-162 |
| 审计日志不可缺 | 所有敏感操作（删除/提权/角色变更）必须记录 | SOC 2 Compliance |
| 防止垂直越权 | 中间件层拦截，Controller 层不做权限判断 | Defense in Depth |
| Seed 数据幂等 | 种子脚本可重复执行，使用 upsert 避免重复创建 | Database Seeding |

## 详细需求

### 1. RBAC 中间件 `src/middlewares/rbac.js`

| 中间件 | 签名 | 说明 |
|--------|------|------|
| `requireRole(...roles)` | 高阶函数 → middleware | 检查 `req.user.role` 是否在允许列表中，不在则 403 |
| `requireOwnership(extractUserId)` | 高阶函数 → middleware | 检查请求者是否为资源所有者，ADMIN 跳过检查 |

**requireOwnership 参数说明**：
- `extractUserId` 是一个函数 `(req) => userId`，从请求中提取目标资源的 userId
- 例如：`requireOwnership(req => req.params.id)` — 从路由参数提取

### 2. 审计服务 `src/services/audit.service.js`

| 方法 | 签名 | 说明 |
|------|------|------|
| `logAction(action, userId, details, req)` | → AuditLog | 记录操作到数据库 |

**记录字段**：

| 字段 | 来源 | 示例 |
|------|------|------|
| action | 业务传入 | LOGIN / LOGOUT / USER_UPDATE / ROLE_CHANGE / USER_DELETE |
| userId | req.user 或传入 | uuid |
| ip | req.ip | 127.0.0.1 |
| userAgent | req.headers['user-agent'] | curl/7.x |
| details | 业务传入 (JSON) | `{targetUserId: "xxx", oldRole: "USER", newRole: "ADMIN"}` |

### 3. 用户管理路由 `src/routes/user.routes.js`

全部需要 JWT 认证中间件。

| 方法 | 路径 | 权限要求 | 说明 |
|------|------|----------|------|
| GET | `/users` | ADMIN only | 查看所有用户列表 |
| GET | `/users/:id` | ADMIN 或本人 | 查看单个用户详情 |
| PUT | `/users/:id` | ADMIN 或本人 | 更新用户信息 |
| DELETE | `/users/:id` | ADMIN only | 删除（停用）用户 |
| PUT | `/users/:id/role` | ADMIN only | 修改用户角色 |

### 4. Seed 脚本 `scripts/seed.js`

| 用户名 | 密码 | 角色 | 用途 |
|--------|------|------|------|
| admin | Admin@123456 | ADMIN | 管理员测试 |
| user | User@123456 | USER | 普通用户测试 |
| guest | Guest@123456 | GUEST | 访客测试 |

## 权限控制流程

```
=== RBAC 角色守卫流程 ===

  Client             Auth MW           RBAC MW            Controller
    |                   |                 |                    |
    | GET /users        |                 |                    |
    | Bearer <token>    |                 |                    |
    |------------------>|                 |                    |
    |                   | 1. 验签         |                    |
    |                   | 2. 查黑名单     |                    |
    |                   | 3. req.user =   |                    |
    |                   |   {userId,      |                    |
    |                   |    role:"USER"} |                    |
    |                   |---------------->|                    |
    |                   |                 | requireRole        |
    |                   |                 | ("ADMIN")          |
    |                   |                 |                    |
    |                   |                 | role="USER"        |
    |                   |                 | not in ["ADMIN"]   |
    |                   |                 | --> 403 Forbidden  |
    |  403              |                 |                    |
    |<------------------|<----------------|                    |

=== 资源所有权守卫流程 ===

  Client             Auth MW           RBAC MW            Controller
    |                   |                 |                    |
    | PUT /users/uuid-B |                 |                    |
    | Bearer <token-A>  |                 |                    |
    |------------------>|                 |                    |
    |                   | req.user =      |                    |
    |                   | {userId: A,     |                    |
    |                   |  role: "USER"}  |                    |
    |                   |---------------->|                    |
    |                   |                 | requireOwnership   |
    |                   |                 | (req=>req.params.id)|
    |                   |                 |                    |
    |                   |                 | 请求者: userId=A   |
    |                   |                 | 目标: params.id=B  |
    |                   |                 | 角色: USER!=ADMIN  |
    |                   |                 | A != B --> 403     |
    |  403              |                 |                    |
    |<------------------|<----------------|                    |

=== ADMIN 跳过所有权检查 ===

    | PUT /users/uuid-B |                 |                    |
    | Bearer <admin-tkn>|                 |                    |
    |------------------>|                 |                    |
    |                   | req.user =      |                    |
    |                   | {userId: X,     |                    |
    |                   |  role: "ADMIN"} |                    |
    |                   |---------------->|                    |
    |                   |                 | role=ADMIN         |
    |                   |                 | --> 跳过检查 OK    |
    |                   |                 |------------------->|
    |  200 OK           |                 |                    |
    |<------------------|<----------------|<-------------------|
```

### 权限矩阵

```
              GET /users   GET /users/:id   PUT /users/:id   DELETE /users/:id   PUT /users/:id/role
  ADMIN       [Y]          [Y]              [Y]              [Y]                 [Y]
  USER        [N]          本人 [Y]         本人 [Y]         [N]                 [N]
  GUEST       [N]          [N]              [N]              [N]                 [N]
```

### 审计日志流程

```
  Controller                  Audit Service              DB
    |                              |                      |
    |  DELETE /users/:id 成功      |                      |
    |  logAction("USER_DELETE",    |                      |
    |   adminId,                   |                      |
    |   {targetUserId, reason},    |                      |
    |   req)                       |                      |
    |---------------------------->|                      |
    |                              |  INSERT AuditLog     |
    |                              |  {action, userId,    |
    |                              |   ip, userAgent,     |
    |                              |   details(JSON),     |
    |                              |   createdAt}         |
    |                              |--------------------->|
    |                              |  OK 记录完成         |
    |                              |<---------------------|
    |  (异步，不阻塞响应)          |                      |
    |<-----------------------------|                      |
```

## API 示例

### POST /auth/login

**以管理员身份登录，获取 ADMIN_TOKEN**

Request:
```bash
curl -s -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"Admin@123456"}'
```

Response (200):
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...xyz",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...abc",
    "user": {
      "id": "b5f7d2a1-3e4c-4f8a-9b6d-1a2b3c4d5e6f",
      "username": "admin",
      "role": "ADMIN"
    }
  }
}
```

---

### GET /users

**管理员查看所有用户列表（成功）**

Request:
```bash
curl -s http://localhost:3000/users \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

Response (200):
```json
{
  "success": true,
  "data": [
    {
      "id": "b5f7d2a1-3e4c-4f8a-9b6d-1a2b3c4d5e6f",
      "username": "admin",
      "email": "admin@example.com",
      "role": "ADMIN",
      "createdAt": "2026-03-10T08:00:00.000Z"
    },
    {
      "id": "c6a8e3b2-4f5d-5a9b-0c7e-2b3c4d5e6f7a",
      "username": "user",
      "email": "user@example.com",
      "role": "USER",
      "createdAt": "2026-03-10T08:00:01.000Z"
    },
    {
      "id": "d7b9f4c3-5a6e-6b0c-1d8f-3c4d5e6f7a8b",
      "username": "guest",
      "email": "guest@example.com",
      "role": "GUEST",
      "createdAt": "2026-03-10T08:00:02.000Z"
    }
  ]
}
```

**普通用户查看所有用户列表（被拒绝）**

Request:
```bash
curl -s http://localhost:3000/users \
  -H "Authorization: Bearer $USER_TOKEN"
```

Response (403):
```json
{
  "success": false,
  "error": {
    "code": "FORBIDDEN",
    "message": "需要 ADMIN 权限"
  }
}
```

---

### GET /users/:id

**管理员查看任意用户详情（成功）**

Request:
```bash
curl -s http://localhost:3000/users/c6a8e3b2-4f5d-5a9b-0c7e-2b3c4d5e6f7a \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

Response (200):
```json
{
  "success": true,
  "data": {
    "id": "c6a8e3b2-4f5d-5a9b-0c7e-2b3c4d5e6f7a",
    "username": "user",
    "email": "user@example.com",
    "role": "USER",
    "createdAt": "2026-03-10T08:00:01.000Z",
    "updatedAt": "2026-03-10T08:00:01.000Z"
  }
}
```

**普通用户查看自己的信息（成功）**

Request:
```bash
curl -s http://localhost:3000/users/c6a8e3b2-4f5d-5a9b-0c7e-2b3c4d5e6f7a \
  -H "Authorization: Bearer $USER_TOKEN"
```

Response (200):
```json
{
  "success": true,
  "data": {
    "id": "c6a8e3b2-4f5d-5a9b-0c7e-2b3c4d5e6f7a",
    "username": "user",
    "email": "user@example.com",
    "role": "USER",
    "createdAt": "2026-03-10T08:00:01.000Z",
    "updatedAt": "2026-03-10T08:00:01.000Z"
  }
}
```

**普通用户查看其他用户信息（被拒绝）**

Request:
```bash
curl -s http://localhost:3000/users/b5f7d2a1-3e4c-4f8a-9b6d-1a2b3c4d5e6f \
  -H "Authorization: Bearer $USER_TOKEN"
```

Response (403):
```json
{
  "success": false,
  "error": {
    "code": "FORBIDDEN",
    "message": "无权访问该资源"
  }
}
```

---

### PUT /users/:id

**管理员更新任意用户信息（成功）**

Request:
```bash
curl -s -X PUT http://localhost:3000/users/c6a8e3b2-4f5d-5a9b-0c7e-2b3c4d5e6f7a \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email":"newuser@example.com"}'
```

Response (200):
```json
{
  "success": true,
  "data": {
    "id": "c6a8e3b2-4f5d-5a9b-0c7e-2b3c4d5e6f7a",
    "username": "user",
    "email": "newuser@example.com",
    "role": "USER",
    "updatedAt": "2026-03-10T09:15:30.000Z"
  }
}
```

**普通用户更新自己的信息（成功）**

Request:
```bash
curl -s -X PUT http://localhost:3000/users/c6a8e3b2-4f5d-5a9b-0c7e-2b3c4d5e6f7a \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email":"mynewemail@example.com"}'
```

Response (200):
```json
{
  "success": true,
  "data": {
    "id": "c6a8e3b2-4f5d-5a9b-0c7e-2b3c4d5e6f7a",
    "username": "user",
    "email": "mynewemail@example.com",
    "role": "USER",
    "updatedAt": "2026-03-10T09:16:00.000Z"
  }
}
```

**普通用户更新其他用户信息（被拒绝）**

Request:
```bash
curl -s -X PUT http://localhost:3000/users/b5f7d2a1-3e4c-4f8a-9b6d-1a2b3c4d5e6f \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email":"hacked@example.com"}'
```

Response (403):
```json
{
  "success": false,
  "error": {
    "code": "FORBIDDEN",
    "message": "无权访问该资源"
  }
}
```

---

### DELETE /users/:id

**管理员删除（停用）用户（成功）**

Request:
```bash
curl -s -X DELETE http://localhost:3000/users/d7b9f4c3-5a6e-6b0c-1d8f-3c4d5e6f7a8b \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

Response (200):
```json
{
  "success": true,
  "data": {
    "message": "用户已停用",
    "userId": "d7b9f4c3-5a6e-6b0c-1d8f-3c4d5e6f7a8b"
  }
}
```

**普通用户尝试删除用户（被拒绝）**

Request:
```bash
curl -s -X DELETE http://localhost:3000/users/d7b9f4c3-5a6e-6b0c-1d8f-3c4d5e6f7a8b \
  -H "Authorization: Bearer $USER_TOKEN"
```

Response (403):
```json
{
  "success": false,
  "error": {
    "code": "FORBIDDEN",
    "message": "需要 ADMIN 权限"
  }
}
```

---

### PUT /users/:id/role

**管理员修改用户角色（成功）**

Request:
```bash
curl -s -X PUT http://localhost:3000/users/c6a8e3b2-4f5d-5a9b-0c7e-2b3c4d5e6f7a/role \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"role":"ADMIN"}'
```

Response (200):
```json
{
  "success": true,
  "data": {
    "id": "c6a8e3b2-4f5d-5a9b-0c7e-2b3c4d5e6f7a",
    "username": "user",
    "role": "ADMIN",
    "message": "角色已更新"
  }
}
```

**普通用户尝试修改角色（被拒绝）**

Request:
```bash
curl -s -X PUT http://localhost:3000/users/c6a8e3b2-4f5d-5a9b-0c7e-2b3c4d5e6f7a/role \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"role":"ADMIN"}'
```

Response (403):
```json
{
  "success": false,
  "error": {
    "code": "FORBIDDEN",
    "message": "需要 ADMIN 权限"
  }
}
```

## Prompt

```
实现角色权限控制：

1. src/middlewares/rbac.js：
   - requireRole(...roles) — 角色守卫
   - requireOwnership(extractUserId) — 资源所有权守卫（ADMIN 跳过）

2. src/services/audit.service.js：
   - logAction(action, userId, details, req) — 审计日志

3. src/routes/user.routes.js — 用户管理（全部需认证）：
   - GET /users — 仅 ADMIN
   - GET /users/:id — ADMIN 或本人
   - PUT /users/:id — ADMIN 或本人
   - DELETE /users/:id — 仅 ADMIN
   - PUT /users/:id/role — 仅 ADMIN

4. scripts/seed.js — 创建三个测试账号：
   - admin / Admin@123456 / ADMIN
   - user / User@123456 / USER
   - guest / Guest@123456 / GUEST

运行 seed 后验证：admin 可管理所有用户，user 只能看自己。
```

## 验收标准

| # | 验收项 | 验证方式 | 预期结果 |
|---|--------|----------|----------|
| 1 | Seed 成功 | `node scripts/seed.js` | 创建 3 个测试用户无报错 |
| 2 | ADMIN 看全部用户 | admin Token GET /users | 200，返回用户列表 |
| 3 | USER 被拒绝看列表 | user Token GET /users | 403 Forbidden |
| 4 | USER 看自己 | user Token GET /users/:self | 200，返回自己的信息 |
| 5 | USER 看别人 | user Token GET /users/:other | 403 Forbidden |
| 6 | USER 改自己 | user Token PUT /users/:self | 200，修改成功 |
| 7 | USER 改别人 | user Token PUT /users/:other | 403 Forbidden |
| 8 | ADMIN 删用户 | admin Token DELETE /users/:id | 200，用户被停用 |
| 9 | USER 删用户 | user Token DELETE /users/:id | 403 Forbidden |
| 10 | ADMIN 改角色 | admin Token PUT /users/:id/role | 200，角色已更新 |
| 11 | 审计日志 | 操作后查数据库 | AuditLog 表有对应记录 |
| 12 | GUEST 无权限 | guest Token 访问 | 所有管理接口返回 403 |

## 验证

```bash
# 初始化测试账号
node scripts/seed.js

# 用 admin 登录
ADMIN_TOKEN=$(curl -s -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"Admin@123456"}' | jq -r '.data.accessToken')

# admin 查看所有用户（成功）
curl -s http://localhost:3000/users \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq

# 用 user 登录
USER_TOKEN=$(curl -s -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"user","password":"User@123456"}' | jq -r '.data.accessToken')

# user 查看所有用户（应返回 403）
curl -s http://localhost:3000/users \
  -H "Authorization: Bearer $USER_TOKEN" | jq
```

## 讲师要点

- 展示 requireOwnership 高阶函数的设计
- 测试越权访问：USER 角色访问 ADMIN 接口 → 403
- 测试资源守卫：USER 只能查看/修改自己的信息
