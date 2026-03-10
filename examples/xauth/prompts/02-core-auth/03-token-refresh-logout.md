# ⭐ Prompt 04 — Token 刷新 + 登出 + 认证中间件

> **Sprint 2** | 预计 AI 执行时间：2-3 分钟

## 目标

实现 Token 管理闭环——刷新 Access Token、登出黑名单、JWT 认证中间件，并引入 Redis 作为黑名单存储（含降级策略），完成「登录 → 使用 → 刷新 → 登出 → 失效」的完整生命周期。

## 最佳实践

| 实践 | 说明 | 参考 |
|------|------|------|
| Refresh Token Rotation | 每次刷新签发新 RT，旧 RT 立即失效 | OWASP JWT Cheat Sheet |
| 黑名单 TTL 对齐 | TTL = Token 剩余有效期，自动清理不浪费内存 | Redis Best Practices |
| Redis 降级策略 | Redis 不可用时回退内存 Map，保证核心功能不中断 | Resilience Pattern |
| Bearer Token 规范 | 严格解析 Authorization: Bearer {token} 格式 | RFC 6750 |
| Token 指纹 | 黑名单用 SHA256(token) 而非原文，减少存储暴露面 | Security by Design |
| 登出即失效 | 登出立即加黑名单，不依赖 Token 自然过期 | OWASP Session Management |

## 详细需求

### 1. Redis 工具 `src/utils/redis.js`

| 项目 | 要求 |
|------|------|
| 客户端 | ioredis，从 config 读取连接信息 |
| 降级策略 | Redis 不可用时回退到内存 Map（log 告警） |
| 导出方法 | `get(key)`、`set(key, value, ttl)`、`del(key)` |
| 连接事件 | 监听 connect / error / reconnecting 事件并打日志 |

### 2. Token 黑名单（扩展 `token.service.js`）

| 方法 | 说明 |
|------|------|
| `blacklistToken(token)` | 将 Token 加入黑名单，TTL = Token 剩余有效秒数 |
| `isBlacklisted(token)` | 检查 Token 是否在黑名单中 |

**Redis Key 设计**：
```
Key:   bl:{sha256(token)}
Value: "1"
TTL:   token.exp - now()  (剩余有效秒数，自动过期)
```

### 3. 认证中间件 `src/middlewares/auth.js`

处理流程：
1. 从 `Authorization: Bearer <token>` 提取 Token
2. 调用 `verifyAccessToken` 验签
3. 查黑名单 `isBlacklisted`
4. 注入 `req.user = { userId, role }`
5. 失败返回 401

### 4. 新增路由

| 方法 | 路径 | 中间件 | 说明 |
|------|------|--------|------|
| POST | `/auth/refresh` | 无 | Refresh Token 换新 Token 对 |
| POST | `/auth/logout` | auth | 当前 Token 加入黑名单 |

### 5. Refresh Token 逻辑

- 验证 Refresh Token 的签名和过期时间
- 检查 Refresh Token 是否在黑名单中
- 签发新的 Access Token + Refresh Token
- 旧 Refresh Token 加入黑名单（Rotation 策略）

## Token 生命周期流程

```
=== Token 完整生命周期（登录 -> 使用 -> 刷新 -> 登出） ===

  Client                Auth MW               Token Service          Redis
    |                      |                       |                    |
    | === 阶段 1：登录获取 Token ===                |                    |
    |                      |                       |                    |
    | POST /auth/login     |                       |                    |
    |--------------------->|---------------------->|                    |
    |                      |  generateTokenPair    |                    |
    | {accessToken: AT1,   |<----------------------|                    |
    |  refreshToken: RT1}  |                       |                    |
    |<---------------------|                       |                    |
    |                      |                       |                    |
    | === 阶段 2：使用 Access Token ===             |                    |
    |                      |                       |                    |
    | GET /users           |                       |                    |
    | Bearer AT1           |                       |                    |
    |--------------------->|                       |                    |
    |                      | 1. 验签 AT1            |                    |
    |                      |---------------------->|                    |
    |                      | 2. 查黑名单            |                    |
    |                      |-----------------------+------------------>|
    |                      |    不在黑名单 OK       |                    |
    |                      |<----------------------+-------------------|
    |                      | 3. req.user={userId}  |                    |
    |  200 OK              |                       |                    |
    |<---------------------|                       |                    |
    |                      |                       |                    |
    | === 阶段 3：AT1 过期，用 RT1 刷新 ===         |                    |
    |                      |                       |                    |
    | POST /auth/refresh   |                       |                    |
    | {refreshToken: RT1}  |                       |                    |
    |--------------------->|                       |                    |
    |                      | 1. verifyRefreshToken |                    |
    |                      |---------------------->|                    |
    |                      | 2. 查 RT1 黑名单      |                    |
    |                      |-----------------------+------------------>|
    |                      | 3. 签发新 AT2 + RT2   |                    |
    |                      | 4. RT1 加入黑名单     |                    |
    |                      |-----------------------+------------------>|
    |                      |  SET bl:{hash(RT1)}   |  TTL=剩余秒数     |
    | {accessToken: AT2,   |<----------------------|                    |
    |  refreshToken: RT2}  |                       |                    |
    |<---------------------|                       |                    |
    |                      |                       |                    |
    | === 阶段 4：登出 ===                          |                    |
    |                      |                       |                    |
    | POST /auth/logout    |                       |                    |
    | Bearer AT2           |                       |                    |
    |--------------------->|                       |                    |
    |                      | 1. 验签 AT2            |                    |
    |                      | 2. blacklistToken     |                    |
    |                      |-----------------------+------------------>|
    |                      |  SET bl:{hash(AT2)}   |  TTL=剩余秒数     |
    | {message: "已登出"}  |                       |                    |
    |<---------------------|                       |                    |
    |                      |                       |                    |
    | === 阶段 5：AT2 已失效 ===                    |                    |
    |                      |                       |                    |
    | GET /users           |                       |                    |
    | Bearer AT2           |                       |                    |
    |--------------------->|                       |                    |
    |                      | 1. 验签 AT2 OK        |                    |
    |                      | 2. 查黑名单 -> 命中!  |                    |
    |                      |-----------------------+------------------>|
    | 401 Unauthorized     |    在黑名单中          |                    |
    |<---------------------|<----------------------+-------------------|
```

### Redis 降级策略

```
Redis 连接状态
  |
  +-- 可用？
        |
        +-- Yes --> Redis 正常
        |           SET/GET
        |           带 TTL
        |
        +-- No  --> 内存 Map 降级模式
                    map.set/get
                    setTimeout 模拟 TTL
                    [!] 重启丢失
                    [!] 日志告警
```

## API 示例

### POST /auth/refresh

**刷新成功**

Request:
```bash
curl -s -X POST http://localhost:3000/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{
    "refreshToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiJmNDdhYzEwYi01OGNjLTQzNzItYTU2Ny0wZTAyYjJjM2Q0NzkiLCJ0eXBlIjoicmVmcmVzaCIsImlhdCI6MTcwNTMwMjIwMCwiZXhwIjoxNzA1OTA3MDAwfQ.signature..."
  }' | jq
```

Response (200):
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiJmNDdhYzEwYi01OGNjLTQzNzItYTU2Ny0wZTAyYjJjM2Q0NzkiLCJyb2xlIjoiVVNFUiIsImlhdCI6MTcwNTMwODAwMCwiZXhwIjoxNzA1MzA4OTAwfQ.signature...",
    "refreshToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiJmNDdhYzEwYi01OGNjLTQzNzItYTU2Ny0wZTAyYjJjM2Q0NzkiLCJ0eXBlIjoicmVmcmVzaCIsImlhdCI6MTcwNTMwODAwMCwiZXhwIjoxNzA1OTEyODAwfQ.signature...",
    "expiresIn": 900
  }
}
```

**Refresh Token 过期或已在黑名单**

Request:
```bash
curl -s -X POST http://localhost:3000/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{
    "refreshToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.expired-or-blacklisted-token..."
  }' | jq
```

Response (401):
```json
{
  "success": false,
  "error": {
    "message": "Token 无效或已过期",
    "code": "UNAUTHORIZED"
  }
}
```

### POST /auth/logout

**登出成功**

Request:
```bash
curl -s -X POST http://localhost:3000/auth/logout \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiJmNDdhYzEwYi01OGNjLTQzNzItYTU2Ny0wZTAyYjJjM2Q0NzkiLCJyb2xlIjoiVVNFUiIsImlhdCI6MTcwNTMwODAwMCwiZXhwIjoxNzA1MzA4OTAwfQ.signature..." | jq
```

Response (200):
```json
{
  "success": true,
  "data": {
    "message": "已登出"
  }
}
```

### GET 受保护路由（无 Token）

Request:
```bash
curl -s http://localhost:3000/protected-route | jq
```

Response (401):
```json
{
  "success": false,
  "error": {
    "message": "未提供认证 Token",
    "code": "UNAUTHORIZED"
  }
}
```

### GET 受保护路由（已登出/黑名单 Token）

Request:
```bash
curl -s http://localhost:3000/protected-route \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.blacklisted-token..." | jq
```

Response (401):
```json
{
  "success": false,
  "error": {
    "message": "Token 已失效",
    "code": "UNAUTHORIZED"
  }
}
```

## Prompt

```
实现 Token 管理闭环：

1. src/utils/redis.js — ioredis 封装，含降级策略（Redis 不可用时用内存 Map）
2. token.service.js 增加：blacklistToken(token)、isBlacklisted(token)
   - Redis Key: "bl:{sha256(token)}"，TTL = token 剩余有效秒数
3. src/middlewares/auth.js — JWT 认证中间件
   - 提取 Bearer Token → 验签 → 查黑名单 → 注入 req.user
4. auth.routes.js 增加：
   - POST /auth/refresh — Refresh Token 换新 Access Token
   - POST /auth/logout — 当前 Token 加入黑名单

完成后验证：登录 → 访问受保护接口 → 登出 → 再次访问应返回 401。
```

## 验收标准

| # | 验收项 | 验证方式 | 预期结果 |
|---|--------|----------|----------|
| 1 | Redis 连接正常 | 启动应用查看日志 | 显示 Redis connected |
| 2 | 认证中间件生效 | 无 Token 访问保护接口 | 401 Unauthorized |
| 3 | 正确 Token 通行 | 带 Token 访问 | 200 OK |
| 4 | 登出后失效 | 登出后再用同 Token | 401，Token 在黑名单 |
| 5 | Token 刷新 | POST /auth/refresh | 返回新 Token 对 |
| 6 | 旧 RT 失效 | 用旧 RT 再次刷新 | 401，已在黑名单 |
| 7 | 黑名单 TTL 正确 | 检查代码 | TTL = exp - now()，非固定值 |
| 8 | Redis 降级 | 停止 Redis 后测试 | 应用不崩溃，回退内存 Map |
| 9 | 完整闭环 | 执行验证脚本 | 登录→使用→登出→失效 全通过 |

## 验证

```bash
# 登录拿 Token
TOKEN=$(curl -s -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"demo","password":"Demo@123456"}' | jq -r '.data.accessToken')

# 访问受保护接口（成功）
curl -s http://localhost:3000/health \
  -H "Authorization: Bearer $TOKEN" | jq

# 登出
curl -s -X POST http://localhost:3000/auth/logout \
  -H "Authorization: Bearer $TOKEN" | jq

# 再次访问（应返回 401）
curl -s http://localhost:3000/health \
  -H "Authorization: Bearer $TOKEN" | jq
```

## 讲师要点

- 重点展示 Redis 降级策略的设计
- 检查黑名单 TTL 是否 = Token 剩余有效期（而非固定时间）
- 演示完整的 登录 → 使用 → 登出 → 失效 闭环
