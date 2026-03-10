# ⚡ Prompt 08 — API Key + HMAC 签名

> **Sprint 6** | 预计 AI 执行时间：2-3 分钟 | 可选演示

## 目标

实现面向第三方 API 调用者的 API Key 认证机制，通过 HMAC-SHA256 签名保证请求防篡改和防重放，配合 Redis 滑动窗口限流保护 API 安全。

## 最佳实践

| 实践 | 说明 | 参考 |
|------|------|------|
| Secret 只存哈希 | API Secret 仅创建时展示一次，数据库只存 SHA256 哈希 | OWASP API Key Management |
| HMAC-SHA256 签名 | 对 Method+Path+Timestamp+BodyHash 签名，防篡改 | AWS Signature V4 |
| 时间窗口防重放 | 请求时间戳 ±5 分钟内有效，过期拒绝 | Anti-Replay Pattern |
| 滑动窗口限流 | Redis Sorted Set 实现精确滑动窗口，优于固定窗口 | Rate Limiting Best Practices |
| 标准限流响应头 | 返回 X-RateLimit-* 和 Retry-After，客户端可自适应 | IETF Draft RateLimit Headers |
| Key 权限粒度 | 每个 API Key 独立权限列表，最小授权原则 | API Security Pattern |
| 时间安全比较 | 签名比对使用 crypto.timingSafeEqual，防时间侧信道 | CWE-208 |

## 详细需求

### 1. API Key 服务 `src/services/apikey.service.js`

| 方法 | 输入 | 逻辑 | 输出 |
|------|------|------|------|
| `createApiKey(userId, name, permissions)` | 用户ID、名称、权限列表 | 生成 ak_xxx + sk_xxx → 只存哈希 | `{apiKey, apiSecret}` ⚠️ Secret 仅此一次展示 |
| `verifyRequest(apiKey, signature, method, path, timestamp, bodyHash)` | 签名参数 | 查 Key → 重建签名 → 比对 → 检查时间窗口 | `{userId, permissions}` |
| `listApiKeys(userId)` | 用户ID | 列出该用户的所有 API Key | ApiKey[]（不含 Secret） |
| `revokeApiKey(userId, keyId)` | 用户ID + Key ID | 标记 isActive=false | void |

**Key 生成规格**：

| 项目 | 格式 | 说明 |
|------|------|------|
| API Key | `ak_` + 32 位 hex | 公开标识，可传输 |
| API Secret | `sk_` + 64 位 hex | 私密密钥，仅创建时返回一次 |
| 存储 | sha256(apiKey) → keyHash | 数据库只存哈希，不存明文 |
| 前缀 | apiKey 前 8 位 → keyPrefix | 便于用户识别哪个 Key |

### 2. HMAC 认证中间件 `src/middlewares/apikey-auth.js`

**请求头**：

| Header | 说明 | 示例 |
|--------|------|------|
| `X-API-Key` | API Key 标识 | `ak_a1b2c3d4...` |
| `X-Timestamp` | Unix 秒级时间戳 | `1700000000` |
| `X-Signature` | HMAC-SHA256 签名 | `hex string` |

**签名算法**：

```
StringToSign = "{METHOD}\n{PATH}\n{TIMESTAMP}\n{BODY_HASH}"
Signature = HMAC-SHA256(apiSecret, StringToSign)

其中：
  METHOD    = 大写 HTTP 方法（GET/POST/PUT/DELETE）
  PATH      = 请求路径（不含 query string）
  TIMESTAMP = Unix 秒级时间戳
  BODY_HASH = SHA256(requestBody)，无 body 则 SHA256("")
```

**安全检查**：

| 检查项 | 规则 | 失败响应 |
|--------|------|----------|
| 时间窗口 | `|now - timestamp| ≤ 300秒`（±5 分钟） | 401，请求已过期 |
| 签名匹配 | 重建签名与 X-Signature 比对 | 401，签名无效 |
| Key 有效 | isActive = true | 401，API Key 已吊销 |
| 更新 lastUsedAt | 每次成功验证后更新 | — |

### 3. 限流中间件 `src/middlewares/rate-limit.js`

| 项目 | 要求 |
|------|------|
| 算法 | Redis 滑动窗口（Sorted Set） |
| 维度 | 按 API Key / IP 双维度 |
| 默认限制 | 100 请求 / 分钟（可配置） |
| 429 响应头 | `Retry-After`、`X-RateLimit-Limit`、`X-RateLimit-Remaining`、`X-RateLimit-Reset` |

### 4. 路由 `src/routes/apikey.routes.js`（需 JWT 认证）

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api-keys` | 创建 API Key |
| GET | `/api-keys` | 列出我的 API Keys |
| DELETE | `/api-keys/:id` | 吊销 API Key |

### 5. 示例脚本 `scripts/test-apikey.js`

展示完整签名计算过程的 Node.js 脚本，用于演示和测试。

## HMAC 签名认证流程

```
=== API Key 创建流程 ===

  用户 (已登录)            XAuth Server                  DB
    |                         |                          |
    | POST /api-keys          |                          |
    | Bearer <jwt>            |                          |
    | {name: "test-key",      |                          |
    |  permissions: ["read"]} |                          |
    |------------------------>|                          |
    |                         | 1. 生成 ak_xxx (32位hex)|
    |                         | 2. 生成 sk_xxx (64位hex)|
    |                         | 3. sha256(ak_xxx)->hash  |
    |                         | 4. 存储 keyHash           |
    |                         |------------------------->|
    |                         |    INSERT ApiKey         |
    |                         |    (keyHash, keyPrefix,   |
    |                         |     permissions, userId)  |
    |                         |<-------------------------|
    |                         |                          |
    | {apiKey: "ak_xxx",      |                          |
    |  apiSecret: "sk_xxx"}   |                          |
    | ⚠️ Secret 仅此一次展示  |                          |
    |<------------------------|                          |

=== HMAC 签名请求验证流程 ===

  第三方调用者                    XAuth Server                    DB
    |                                |                            |
    | +---------------------+        |                            |
    | | 客户端签名计算：      |        |                            |
    | |                      |        |                            |
    | | timestamp = now()    |        |                            |
    | | bodyHash = SHA256(   |        |                            |
    | |   requestBody)       |        |                            |
    | | stringToSign =       |        |                            |
    | |   "POST\n" +         |        |                            |
    | |   "/api/data\n" +    |        |                            |
    | |   timestamp + "\n" + |        |                            |
    | |   bodyHash           |        |                            |
    | | signature = HMAC-    |        |                            |
    | |   SHA256(sk_xxx,     |        |                            |
    | |   stringToSign)      |        |                            |
    | +---------------------+        |                            |
    |                                |                            |
    | POST /api/data                 |                            |
    | X-API-Key: ak_xxx              |                            |
    | X-Timestamp: 1700000000        |                            |
    | X-Signature: abc123...         |                            |
    | Body: {"key":"value"}          |                            |
    |------------------------------->|                            |
    |                                |                            |
    |                                | 1. 检查 |now-ts| <= 300s   |
    |                                |    --> 防重放 ✅            |
    |                                |                            |
    |                                | 2. sha256(ak_xxx) 查 DB    |
    |                                |--------------------------->|
    |                                |<---------------------------|
    |                                |                            |
    |                                | 3. 用 apiSecret 重建      |
    |                                |    HMAC-SHA256 签名        |
    |                                |                            |
    |                                | 4. 比对签名 --> 防篡改 ✅  |
    |                                |                            |
    |                                | 5. 更新 lastUsedAt         |
    |                                |                            |
    |  200 OK                        |                            |
    |<-------------------------------|                            |

=== 防篡改 & 防重放原理 ===

  攻击场景 1：篡改请求体
  ---------------------
  原始: Body={"amount":100}  -->  签名 S1 (基于 SHA256(Body))
  篡改: Body={"amount":999}  -->  SHA256 变化 --> 重建签名 != S1 --> ❌ 拒绝

  攻击场景 2：重放攻击
  ---------------------
  抓包: timestamp=T1, signature=S1 (5分钟前的合法请求)
  重放: 当前 now()-T1 > 300s --> ❌ 请求已过期

  攻击场景 3：篡改时间戳绕过
  ---------------------
  修改: timestamp=now() (更新时间戳以绕过过期检查)
  但是: stringToSign 变化 --> 重建签名 != 原签名 --> ❌ 拒绝
```

### 滑动窗口限流

```
  Redis Sorted Set: ratelimit:{ak_xxx}
  Score = Unix 时间戳（毫秒），Member = 请求 ID

  窗口: 60秒, 限制: 100次

  时间轴 ------------------------------------------------>
         |<------ 60秒 滑动窗口 ------>|
         |  已有 98 个请求              |
         |                             |
  新请求 -+  ZADD + ZREMRANGEBYSCORE    |
         |  ZCARD = 99 <= 100 --> ✅ 通过 |
         |                             |
  响应头:
    X-RateLimit-Limit: 100
    X-RateLimit-Remaining: 1
    X-RateLimit-Reset: 1700000060

  =========== 超过限制 ===========

  新请求 -+  ZCARD = 101 > 100 --> ❌   |
         |                             |
  响应: 429 Too Many Requests
    Retry-After: 15
    X-RateLimit-Limit: 100
    X-RateLimit-Remaining: 0
```

## API 示例

### POST /api-keys

**创建 API Key（Secret 仅此一次展示）**

Request:
```bash
curl -s -X POST http://localhost:3000/api-keys \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"test-key","permissions":["read","write"]}'
```

Response (201):
```json
{
  "success": true,
  "data": {
    "id": "f1a2b3c4-d5e6-7f8a-9b0c-1d2e3f4a5b6c",
    "name": "test-key",
    "apiKey": "ak_a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6",
    "apiSecret": "sk_1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b",
    "keyPrefix": "ak_a1b2c",
    "permissions": ["read", "write"],
    "createdAt": "2026-03-10T10:00:00.000Z"
  }
}
```

---

### GET /api-keys

**列出当前用户的所有 API Key（不包含 Secret）**

Request:
```bash
curl -s http://localhost:3000/api-keys \
  -H "Authorization: Bearer $TOKEN"
```

Response (200):
```json
{
  "success": true,
  "data": [
    {
      "id": "f1a2b3c4-d5e6-7f8a-9b0c-1d2e3f4a5b6c",
      "name": "test-key",
      "keyPrefix": "ak_a1b2c",
      "permissions": ["read", "write"],
      "isActive": true,
      "lastUsedAt": "2026-03-10T11:30:00.000Z",
      "createdAt": "2026-03-10T10:00:00.000Z"
    }
  ]
}
```

---

### DELETE /api-keys/:id

**吊销 API Key**

Request:
```bash
curl -s -X DELETE http://localhost:3000/api-keys/f1a2b3c4-d5e6-7f8a-9b0c-1d2e3f4a5b6c \
  -H "Authorization: Bearer $TOKEN"
```

Response (200):
```json
{
  "success": true,
  "data": {
    "message": "API Key 已吊销"
  }
}
```

---

### HMAC 签名请求示例

**合法签名请求（成功）**

Request:
```bash
# 签名计算过程：
# TIMESTAMP=$(date +%s)
# BODY='{"key":"value"}'
# BODY_HASH=$(echo -n "$BODY" | shasum -a 256 | cut -d' ' -f1)
# STRING_TO_SIGN="POST\n/api/data\n${TIMESTAMP}\n${BODY_HASH}"
# SIGNATURE=$(echo -ne "$STRING_TO_SIGN" | openssl dgst -sha256 -hmac "$API_SECRET" | cut -d' ' -f2)

curl -s -X POST http://localhost:3000/api/data \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ak_a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6" \
  -H "X-Timestamp: 1741600000" \
  -H "X-Signature: 3a7f2c9e8d1b4f6a5c0e7d2b9f4a1c6e8d3b5f7a0c2e4d6b8f1a3c5e7d9b2f4a" \
  -d '{"key":"value"}'
```

Response (200):
```json
{
  "success": true,
  "data": {
    "message": "请求处理成功",
    "received": {
      "key": "value"
    }
  }
}
```

---

### HMAC 签名验证 — 篡改请求体

**请求体被篡改，签名不匹配（失败）**

Request:
```bash
# 使用原始签名，但修改了请求体内容
curl -s -X POST http://localhost:3000/api/data \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ak_a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6" \
  -H "X-Timestamp: 1741600000" \
  -H "X-Signature: 3a7f2c9e8d1b4f6a5c0e7d2b9f4a1c6e8d3b5f7a0c2e4d6b8f1a3c5e7d9b2f4a" \
  -d '{"key":"tampered_value"}'
```

Response (401):
```json
{
  "success": false,
  "error": {
    "code": "INVALID_SIGNATURE",
    "message": "签名无效"
  }
}
```

---

### HMAC 签名验证 — 时间戳过期

**时间戳超出 ±5 分钟窗口（失败）**

Request:
```bash
# 使用 10 分钟前的时间戳
curl -s -X POST http://localhost:3000/api/data \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ak_a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6" \
  -H "X-Timestamp: 1741599400" \
  -H "X-Signature: 9b2f4a3c5e7d1a8f6b0c2e4d3a7f9c1e5b8d0f2a4c6e8d1b3f5a7c9e2d4b6f8a" \
  -d '{"key":"value"}'
```

Response (401):
```json
{
  "success": false,
  "error": {
    "code": "REQUEST_EXPIRED",
    "message": "请求已过期"
  }
}
```

---

### 限流 — 超过速率限制

**请求频率超过 100 次/分钟（失败）**

Request:
```bash
# 第 101 次请求
curl -s -X POST http://localhost:3000/api/data \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ak_a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6" \
  -H "X-Timestamp: 1741600005" \
  -H "X-Signature: 5e7d9b2f4a3c1a8f6b0c2e4d3a7f9c1e5b8d0f2a4c6e8d1b3f5a7c9e2d4b6f8a" \
  -d '{"key":"value"}'
```

Response (429):
```
HTTP/1.1 429 Too Many Requests
Retry-After: 15
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1741600060
```

```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "请求过于频繁，请稍后再试"
  }
}
```

## Prompt

```
实现 API Key 认证（面向第三方 API 调用者）：

1. src/services/apikey.service.js：
   - createApiKey(userId, name, permissions) → 生成 ak_xxx + sk_xxx，只存哈希
   - verifyRequest(apiKey, signature, method, path, timestamp, bodyHash)
   - listApiKeys / revokeApiKey

2. src/middlewares/apikey-auth.js：
   - 从 X-API-Key, X-Timestamp, X-Signature 提取
   - HMAC-SHA256(secret, "{method}\n{path}\n{timestamp}\n{bodyHash}")
   - 时间窗口 ±5 分钟

3. src/middlewares/rate-limit.js：
   - Redis 滑动窗口限流
   - 按 API Key / IP 维度
   - 返回 429 + Retry-After + X-RateLimit-* 头

4. src/routes/apikey.routes.js（需 JWT 认证）：
   - POST /api-keys → 创建
   - GET /api-keys → 列出
   - DELETE /api-keys/:id → 吊销

5. scripts/test-apikey.js — 示例调用脚本（展示签名计算过程）
```

## 验收标准

| # | 验收项 | 验证方式 | 预期结果 |
|---|--------|----------|----------|
| 1 | 创建 API Key | POST /api-keys | 返回 apiKey + apiSecret |
| 2 | Secret 一次性 | 再次 GET /api-keys | 列表中无 apiSecret 字段 |
| 3 | 只存哈希 | 查数据库 ApiKey 表 | keyHash 是 SHA256 值，无明文 |
| 4 | 合法签名通过 | 运行 test-apikey.js | 200 OK |
| 5 | 篡改 Body 被拒 | 修改请求体不改签名 | 401，签名无效 |
| 6 | 过期时间戳被拒 | timestamp 设为 10 分钟前 | 401，请求已过期 |
| 7 | 吊销 Key 失效 | DELETE 后再用该 Key | 401，Key 已吊销 |
| 8 | 限流生效 | 快速发 101 次请求 | 第 101 次返回 429 |
| 9 | 限流响应头 | 检查 429 响应 | 包含 Retry-After + X-RateLimit-* |
| 10 | lastUsedAt 更新 | 使用 Key 后查数据库 | lastUsedAt 有值 |

## 验证

```bash
# 创建 API Key
curl -s -X POST http://localhost:3000/api-keys \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"test-key","permissions":["read"]}' | jq
# ⚠️ 保存返回的 apiKey 和 apiSecret，之后无法再查看

# 用 HMAC 签名调用 API
node scripts/test-apikey.js

# 修改签名验证防篡改
# 修改时间戳验证防重放
```

## 讲师要点

- 现场运行 `test-apikey.js` 演示完整签名流程
- 修改请求内容后重发 → 签名不匹配 → 被拒绝（防篡改）
- 修改时间戳为 10 分钟前 → 被拒绝（防重放）
- 讨论 API Key 只存哈希 vs 明文存储的安全差异
