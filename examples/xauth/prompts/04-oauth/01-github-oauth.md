# ⚡ Prompt 06 — GitHub OAuth2

> **Sprint 4** | 预计 AI 执行时间：2-3 分钟 | 可选演示

## 目标

实现 GitHub OAuth2 社交登录，支持新用户自动创建和已有用户账号绑定，使用 state 参数防 CSRF 攻击，完成 Authorization Code Flow 的完整实现。

## 最佳实践

| 实践 | 说明 | 参考 |
|------|------|------|
| State 参数防 CSRF | 授权请求携带随机 state，回调时严格验证 | RFC 6749 §10.12 |
| PKCE 增强安全 | 公共客户端必须使用 PKCE，防止授权码拦截攻击 | RFC 7636 |
| 最小 Scope 原则 | 仅请求 user:email，不请求多余权限 | OAuth 2.0 Security BCP |
| Token 安全存储 | OAuth Access Token 不存前端，服务端代管 | OWASP Token Storage |
| 账号绑定策略 | 同邮箱自动绑定，避免重复创建用户 | Account Linking Pattern |
| 解绑安全检查 | 解绑前确保用户至少保留一种登录方式 | UX Security Pattern |

## 前置条件

需要提前在 GitHub 创建 OAuth App：
- Settings → Developer settings → OAuth Apps → New OAuth App
- Homepage URL: `http://localhost:3000`
- Callback URL: `http://localhost:3000/oauth/github/callback`
- 获取 Client ID 和 Client Secret，填入 .env

## 详细需求

### 1. Passport 策略 `src/config/passport.js`

| 项目 | 要求 |
|------|------|
| 策略 | passport-github2 |
| 读取配置 | GITHUB_CLIENT_ID、GITHUB_CLIENT_SECRET |
| Callback URL | `http://localhost:3000/oauth/github/callback` |
| Scope | `['user:email']` |
| 序列化 | 使用 JWT 替代 session，不需要 serializeUser/deserializeUser |

### 2. OAuth 服务 `src/services/oauth.service.js`

| 方法 | 输入 | 逻辑 | 输出 |
|------|------|------|------|
| `findOrCreateByOAuth(provider, profile, tokens)` | provider 名称、用户档案、OAuth tokens | 查已绑定 → 查同邮箱用户 → 创建新用户 | `{user, isNewUser}` |
| `linkOAuthAccount(userId, provider, profile, tokens)` | 用户 ID、provider 信息 | 为已有用户绑定 OAuth 账号 | OAuthAccount |
| `unlinkOAuthAccount(userId, provider)` | 用户 ID、provider 名称 | 解绑，检查至少保留一种登录方式 | void |
| `getLinkedAccounts(userId)` | 用户 ID | 查询该用户所有已绑定的 OAuth 账号 | OAuthAccount[] |

**findOrCreateByOAuth 决策逻辑**：

```
收到 OAuth Profile
  |
  +-- Step 1: 按 (provider, providerId) 查 OAuthAccount
  |     +-- 找到 --> 返回已绑定 User (isNewUser: false)
  |
  +-- Step 2: 按 email 查 User
  |     +-- 找到 --> 自动绑定到该用户 (isNewUser: false)
  |
  +-- Step 3: 创建新用户 + 绑定 OAuth 账号 (isNewUser: true)
        username: github_{providerId}
        email: profile.emails[0]
        passwordHash: null (OAuth 用户无密码)
```

### 3. 路由 `src/routes/oauth.routes.js`

| 方法 | 路径 | 认证 | 说明 |
|------|------|------|------|
| GET | `/oauth/github` | 无 | 重定向到 GitHub 授权页 |
| GET | `/oauth/github/callback` | 无 | OAuth 回调，签发 JWT |
| GET | `/oauth/accounts` | JWT | 查看已绑定的 OAuth 账号 |
| DELETE | `/oauth/unlink/:provider` | JWT | 解绑 OAuth 账号 |

### 4. 安全要求

| 项目 | 要求 |
|------|------|
| CSRF 防护 | 授权请求携带随机 state，回调时验证一致性 |
| state 存储 | Redis/内存，TTL = 10 分钟 |
| 解绑检查 | 无密码且仅一个 OAuth 时拒绝解绑 |

## OAuth2 Authorization Code 流程

```
=== GitHub OAuth2 Authorization Code Flow ===

  Browser              XAuth Server             GitHub OAuth          Redis
    |                       |                       |                    |
    | 1. GET /oauth/github  |                       |                    |
    |---------------------->|                       |                    |
    |                       | 2. 生成随机 state      |                    |
    |                       |------------------------+------------------>|
    |                       |    SET state:{val}     |    TTL=10min      |
    |                       |                       |                    |
    |  3. 302 Redirect      |                       |                    |
    |  Location:            |                       |                    |
    |  github.com/login/    |                       |                    |
    |  oauth/authorize?     |                       |                    |
    |  client_id=xxx&       |                       |                    |
    |  redirect_uri=xxx&    |                       |                    |
    |  state=abc123&        |                       |                    |
    |  scope=user:email     |                       |                    |
    |<----------------------|                       |                    |
    |                       |                       |                    |
    | === 用户在 GitHub 页面授权 ===                 |                    |
    |                       |                       |                    |
    | 4. Redirect back      |                       |                    |
    | /oauth/github/        |                       |                    |
    | callback?code=xyz&    |                       |                    |
    | state=abc123          |                       |                    |
    |---------------------->|                       |                    |
    |                       | 5. 验证 state          |                    |
    |                       |------------------------+------------------>|
    |                       |    GET state:abc123    |  匹配 OK          |
    |                       |<-----------------------+-------------------|
    |                       |                       |                    |
    |                       | 6. 用 code 换 token    |                    |
    |                       |---------------------->|                    |
    |                       | POST /access_token    |                    |
    |                       | {code, client_id,     |                    |
    |                       |  client_secret}       |                    |
    |                       |                       |                    |
    |                       | 7. access_token       |                    |
    |                       |<----------------------|                    |
    |                       |                       |                    |
    |                       | 8. 获取用户信息        |                    |
    |                       |---------------------->|                    |
    |                       | GET /user             |                    |
    |                       | Bearer {token}        |                    |
    |                       |                       |                    |
    |                       | 9. {id, login,        |                    |
    |                       |     email, avatar}    |                    |
    |                       |<----------------------|                    |
    |                       |                       |                    |
    |                       | 10. findOrCreate      |                    |
    |                       |     ByOAuth           |                    |
    |                       |                       |                    |
    |                       | 11. 签发 JWT          |                    |
    |                       |                       |                    |
    | 12. 返回 JWT          |                       |                    |
    | {accessToken,         |                       |                    |
    |  refreshToken,        |                       |                    |
    |  user, isNewUser}     |                       |                    |
    |<----------------------|                       |                    |
```

### 解绑安全检查

```
解绑请求 (userId, provider=github)
  |
  +-- 检查 1: 用户是否设置了密码？
  |     +-- 有密码 --> 允许解绑（还能密码登录）
  |
  +-- 检查 2: 用户绑定了几个 OAuth？
  |     +-- >= 2 个 --> 允许解绑（还有其他 OAuth）
  |
  +-- 无密码 且 仅 1 个 OAuth
        --> 拒绝解绑 400（解绑后无法登录）
```

## API 示例

### GET /oauth/github

**发起 GitHub OAuth2 授权，重定向到 GitHub 登录页**

Request:
```bash
curl -s -I http://localhost:3000/oauth/github
```

Response (302):
```
HTTP/1.1 302 Found
Location: https://github.com/login/oauth/authorize?client_id=Iv1.a1b2c3d4e5f67890&redirect_uri=http%3A%2F%2Flocalhost%3A3000%2Foauth%2Fgithub%2Fcallback&scope=user%3Aemail&state=f8a2e9c1b4d7563012ef
```

---

### GET /oauth/github/callback?code=xxx&state=xxx

**GitHub 授权回调，用授权码换取 JWT**

Request:
```bash
curl -s "http://localhost:3000/oauth/github/callback?code=7a8b9c0d1e2f3a4b&state=f8a2e9c1b4d7563012ef"
```

Response (200) — 新用户首次登录：
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...xyz",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...abc",
    "isNewUser": true,
    "user": {
      "id": "a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
      "username": "github_12345678",
      "email": "developer@github.com",
      "role": "USER"
    }
  }
}
```

Response (200) — 已有用户再次登录：
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...xyz",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...abc",
    "isNewUser": false,
    "user": {
      "id": "a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
      "username": "github_12345678",
      "email": "developer@github.com",
      "role": "USER"
    }
  }
}
```

---

### GET /oauth/accounts

**查看当前用户已绑定的 OAuth 账号列表**

Request:
```bash
curl -s http://localhost:3000/oauth/accounts \
  -H "Authorization: Bearer $TOKEN"
```

Response (200):
```json
{
  "success": true,
  "data": [
    {
      "id": "e5f6a7b8-9c0d-1e2f-3a4b-5c6d7e8f9a0b",
      "provider": "github",
      "providerId": "12345678",
      "displayName": "octocat",
      "email": "developer@github.com",
      "linkedAt": "2026-03-10T10:30:00.000Z"
    }
  ]
}
```

---

### DELETE /oauth/unlink/github

**解绑 GitHub OAuth 账号（成功 — 用户有密码或其他 OAuth）**

Request:
```bash
curl -s -X DELETE http://localhost:3000/oauth/unlink/github \
  -H "Authorization: Bearer $TOKEN"
```

Response (200):
```json
{
  "success": true,
  "data": {
    "message": "已解绑 github 账号"
  }
}
```

**解绑失败 — 无密码且仅剩唯一 OAuth 登录方式**

Request:
```bash
curl -s -X DELETE http://localhost:3000/oauth/unlink/github \
  -H "Authorization: Bearer $TOKEN"
```

Response (400):
```json
{
  "success": false,
  "error": {
    "code": "UNLINK_DENIED",
    "message": "无法解绑：这是您唯一的登录方式，请先设置密码"
  }
}
```

## Prompt

```
实现 GitHub OAuth2 登录：

1. src/config/passport.js — passport-github2 策略配置
2. src/services/oauth.service.js：
   - findOrCreateByOAuth(provider, profile, tokens)
   - linkOAuthAccount / unlinkOAuthAccount / getLinkedAccounts
3. src/routes/oauth.routes.js：
   - GET /oauth/github → 重定向授权
   - GET /oauth/github/callback → 回调处理，签发 JWT
   - GET /oauth/accounts → 查看已绑定账号（需认证）
   - DELETE /oauth/unlink/:provider → 解绑（需认证）
4. OAuth state 参数防 CSRF
5. 更新 .env.example 加入 GITHUB_CLIENT_ID / GITHUB_CLIENT_SECRET

注意：回调地址为 http://localhost:3000/oauth/github/callback
```

## 验收标准

| # | 验收项 | 验证方式 | 预期结果 |
|---|--------|----------|----------|
| 1 | 授权跳转 | 浏览器打开 /oauth/github | 跳转到 GitHub 授权页，URL 含 state 参数 |
| 2 | 回调签发 JWT | GitHub 授权后回调 | 返回 accessToken + refreshToken |
| 3 | 新用户自动创建 | 首次 OAuth 登录 | 创建 User + OAuthAccount |
| 4 | 同邮箱自动绑定 | 已有相同邮箱用户 | 绑定到已有用户，不创建新用户 |
| 5 | 重复登录幂等 | 同一 GitHub 用户再次登录 | 返回同一用户，不重复创建 |
| 6 | 查看绑定账号 | GET /oauth/accounts | 返回已绑定 provider 列表 |
| 7 | 解绑成功 | DELETE /oauth/unlink/github（有密码用户） | 解绑成功 |
| 8 | 解绑拒绝 | 无密码用户解绑唯一 OAuth | 400，拒绝解绑 |
| 9 | state 防 CSRF | 篡改 state 参数回调 | 403，state 校验失败 |
| 10 | .env.example 更新 | 检查文件 | 包含 GITHUB_CLIENT_ID / GITHUB_CLIENT_SECRET |

## 验证

```bash
# 浏览器打开 GitHub 授权
open http://localhost:3000/oauth/github

# 查看已绑定账号
curl -s http://localhost:3000/oauth/accounts \
  -H "Authorization: Bearer $TOKEN" | jq
```

## 讲师要点

- 如果网络无法访问 GitHub，可跳过此步，不影响其他模块
- 展示 AI 如何实现 state 参数防 CSRF
- 讨论"账号绑定 vs 新建用户"的逻辑区分

## 补充 Prompt（可选）— Google OAuth2

```
参照 GitHub OAuth2 的实现，增加 Google OAuth2 登录支持：
1. 添加 passport-google-oauth20 策略
2. 实现 PKCE (Proof Key for Code Exchange) 增强安全
3. 路由：GET /oauth/google, GET /oauth/google/callback
4. 复用 oauth.service.js 的 findOrCreateByOAuth 方法
代码结构与 GitHub OAuth 保持一致。
```
