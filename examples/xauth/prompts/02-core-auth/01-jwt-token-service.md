# ⭐ Prompt 02 — JWT RS256 Token 服务

> **Sprint 1** | 预计 AI 执行时间：1-2 分钟

## 目标

实现基于 RS256 非对称加密的 JWT Token 服务，支持 Access Token + Refresh Token 双 Token 机制，为整个认证系统提供签发和验证能力。

## 最佳实践

| 实践 | 说明 | 参考 |
|------|------|------|
| 非对称签名 (RS256) | 私钥签发、公钥验证，微服务架构下无需共享私钥 | RFC 7518 §3.3 |
| 短期 Access Token | 15 分钟过期，降低 Token 泄露风险窗口 | OWASP JWT Cheat Sheet |
| Payload 最小化 | 仅存 userId + role，不放邮箱/手机等 PII | RFC 7519 §4.1 |
| 密钥轮换机制 | 支持多密钥 (kid header)，实现零停机密钥更换 | RFC 7517 JWK |
| 私钥安全存储 | 密钥文件 600 权限，生产环境使用 KMS/Vault | NIST SP 800-57 |
| 禁用 alg:none | 验证时明确指定算法，防止算法降级攻击 | CVE-2015-9235 |

## 详细需求

### 1. RSA 密钥管理 `scripts/generate-keys.js`

| 项目 | 要求 |
|------|------|
| 算法 | RSA 2048 位 |
| 格式 | PEM（PKCS#8 私钥 + SPKI 公钥） |
| 输出 | `keys/private.pem` + `keys/public.pem` |
| 安全 | keys/ 目录加入 .gitignore，脚本检测已有密钥则跳过 |

### 2. Token 服务 `src/services/token.service.js`

| 方法 | 签名 | 说明 |
|------|------|------|
| `generateAccessToken(user)` | → String | RS256 签名，payload: `{userId, role}`，15min 过期 |
| `generateRefreshToken(user)` | → String | RS256 签名，payload: `{userId, type:'refresh'}`，7 天过期 |
| `verifyAccessToken(token)` | → Payload | 验证签名+过期时间，失败抛 UnauthorizedError |
| `verifyRefreshToken(token)` | → Payload | 验证 Refresh Token，检查 type 字段 |
| `generateTokenPair(user)` | → Object | 返回 `{accessToken, refreshToken, expiresIn}` |

### 3. 安全约束

- 私钥路径从 `config` 模块读取，不硬编码
- Access Token 过期时间：15 分钟（可配置）
- Refresh Token 过期时间：7 天（可配置）
- 验证失败统一抛出 `UnauthorizedError`，不泄露具体原因给客户端
- Token Payload 不包含敏感信息（不含 email、password 等）

## JWT 签发与验证流程

```
=== JWT RS256 签发流程 ===

  Client                Auth Service            Token Service
    |                       |                        |
    | 1. login(user, pass)  |                        |
    |---------------------->|                        |
    |                       | 2. generateTokenPair   |
    |                       |----------------------->|
    |                       |                        |
    |                       |    +-------------------+
    |                       |    | 3. 读取 private.pem
    |                       |    | 4. RS256 签名
    |                       |    |    Header:
    |                       |    |    {alg:"RS256"}
    |                       |    |    Payload:
    |                       |    |    {userId, role,
    |                       |    |     exp, iat}
    |                       |    | 5. 生成 accessToken
    |                       |    | 6. 生成 refreshToken
    |                       |    +------------------->|
    |                       |                        |
    |                       | {accessToken,          |
    |                       |  refreshToken,         |
    |                       |  expiresIn: 900}       |
    |                       |<-----------------------|
    | 7. 返回 Token 对      |                        |
    |<----------------------|                        |

=== JWT RS256 验证流程 ===

  Client                Middleware               Token Service
    |                       |                        |
    | Authorization:        |                        |
    | Bearer <token>        |                        |
    |---------------------->|                        |
    |                       | verifyAccessToken      |
    |                       |----------------------->|
    |                       |    +-------------------+
    |                       |    | 读取 public.pem
    |                       |    | 验证 RS256 签名
    |                       |    | 检查 exp 过期时间
    |                       |    +------------------->|
    |                       |                        |
    |                       | +- 验证成功 -> payload |
    |                       | +- 验证失败 -> throw   |
    |                       |    UnauthorizedError   |
    |                       |<-----------------------|
    |                       |                        |
    | req.user = payload    |                        |
    | -> next()             |                        |
    |<----------------------|                        |
```

### RS256 vs HS256 对比

```
  HS256 (对称)                     RS256 (非对称)  我们选择
  +----------+                     +----------+
  | Secret   |--签名-->Token       | Private  |--签名-->Token
  | Key      |<-验证---+          | Key      |  (仅签发方持有)
  +----------+                     +----------+
  同一个密钥签名+验证              +----------+
  所有服务都需要密钥               | Public   |<-验证---+
  密钥泄露=全部沦陷               | Key      |  (可分发给任意服务)
                                   +----------+
                                   公钥可安全分发，私钥不出签发方
```

## 代码示例

### 生成 Token 对

```js
const tokenService = require('./src/services/token.service');

// Generate token pair
const tokens = tokenService.generateTokenPair({ id: 'a1b2c3d4-5678-90ab-cdef-1234567890ab', role: 'USER' });
console.log(tokens);
// {
//   accessToken: 'eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiJhMWIyYzNkNC01Njc4LTkwYWItY2RlZi0xMjM0NTY3ODkwYWIiLCJyb2xlIjoiVVNFUiIsImlhdCI6MTcwMDAwMDAwMCwiZXhwIjoxNzAwMDAwOTAwfQ.signature...',
//   refreshToken: 'eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiJhMWIyYzNkNC01Njc4LTkwYWItY2RlZi0xMjM0NTY3ODkwYWIiLCJ0eXBlIjoicmVmcmVzaCIsImlhdCI6MTcwMDAwMDAwMCwiZXhwIjoxNzAwNjA0ODAwfQ.signature...',
//   expiresIn: 900
// }
```

### 验证 Access Token

```js
const payload = tokenService.verifyAccessToken(tokens.accessToken);
console.log(payload);
// {
//   userId: 'a1b2c3d4-5678-90ab-cdef-1234567890ab',
//   role: 'USER',
//   iat: 1700000000,
//   exp: 1700000900
// }
```

### 验证 Refresh Token

```js
const refreshPayload = tokenService.verifyRefreshToken(tokens.refreshToken);
console.log(refreshPayload);
// {
//   userId: 'a1b2c3d4-5678-90ab-cdef-1234567890ab',
//   type: 'refresh',
//   iat: 1700000000,
//   exp: 1700604800
// }
```

### 验证失败（过期或无效 Token）

```js
try {
  tokenService.verifyAccessToken('invalid.token.here');
} catch (err) {
  console.log(err.name);    // 'UnauthorizedError'
  console.log(err.status);  // 401
  console.log(err.message); // 'Token 无效或已过期'
}
```

## Prompt

```
创建 src/services/token.service.js：

1. 生成 RSA 密钥对的脚本 scripts/generate-keys.js（输出到 keys/目录，添加到 .gitignore）
2. Token 服务实现：
   - generateAccessToken(user) → RS256，payload: {userId, role}，15min 过期
   - generateRefreshToken(user) → RS256，payload: {userId, type:'refresh'}，7天过期
   - verifyAccessToken(token) → 验证并返回 payload
   - verifyRefreshToken(token) → 验证 Refresh Token
   - generateTokenPair(user) → 返回 {accessToken, refreshToken, expiresIn}
3. 密钥路径从 config 读取
4. 验证失败抛出 UnauthorizedError

先运行 generate-keys.js 生成密钥。
```

## 验收标准

| # | 验收项 | 验证方式 | 预期结果 |
|---|--------|----------|----------|
| 1 | 密钥已生成 | `ls keys/` | 存在 private.pem 和 public.pem |
| 2 | 使用 RS256 | 检查代码 | 签名算法为 RS256，非 HS256 |
| 3 | Access Token 15min | 检查代码 | expiresIn = '15m' 或 900 |
| 4 | Refresh Token 7天 | 检查代码 | expiresIn = '7d' 或 604800 |
| 5 | Payload 最小化 | 检查代码 | 仅含 userId、role，不含敏感信息 |
| 6 | 私钥已忽略 | `cat .gitignore` | 包含 keys/ 或 *.pem |
| 7 | 验证失败处理 | 检查代码 | 抛出 UnauthorizedError |
| 8 | 密钥路径可配置 | 检查代码 | 从 config 读取，不硬编码 |

## 验证

```bash
node scripts/generate-keys.js
ls keys/   # 应看到 private.pem 和 public.pem
```

## 讲师要点

- 检查 AI 是否选择了 RS256（非对称）而非 HS256（对称）
- 强调非对称签名的优势：公钥可分发给其他服务验证，私钥不出服务端
- 检查 .gitignore 中是否排除了 private.pem
