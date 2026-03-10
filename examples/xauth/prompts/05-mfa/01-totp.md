# ⚡ Prompt 07 — MFA 双因素认证

> **Sprint 5** | 预计 AI 执行时间：2-3 分钟 | 可选演示

## 目标

实现兼容 Google Authenticator 的 TOTP 双因素认证，包含 QR Code 生成、恢复码机制和加密存储，为高安全需求账号提供二次验证能力。

## 最佳实践

| 实践 | 说明 | 参考 |
|------|------|------|
| Secret 加密存储 | AES-256-GCM 加密 TOTP Secret，密钥分离管理 | OWASP Cryptographic Storage |
| 时间容差 ±1 窗口 | 允许前后各 30 秒，兼容用户操作延迟和时钟偏移 | RFC 6238 §5.2 |
| 恢复码一次性 | bcrypt 哈希存储，验证后标记已使用，仅展示一次 | NIST SP 800-63B §5.1.3 |
| MFA 不绕过 | 登录流程强制二步验证，mfaToken 短期有效（5分钟） | OWASP MFA Cheat Sheet |
| QR Code 安全 | Secret 通过 otpauth:// URI 编码，支持主流 Authenticator | Google Authenticator Key URI |
| 备选验证方式 | 提供恢复码作为 TOTP 备选，防止手机丢失无法登录 | Account Recovery Pattern |

## 详细需求

### 1. MFA 服务 `src/services/mfa.service.js`

| 方法 | 输入 | 逻辑 | 输出 |
|------|------|------|------|
| `generateSecret(username)` | 用户名 | 生成 TOTP Secret + otpauth URI + QR Code | `{secret, qrCodeUrl}` |
| `verifyToken(secret, token)` | Secret + 6位码 | 验证 TOTP，允许 ±1 时间窗口 | Boolean |
| `enableMfa(userId, secret, token)` | 用户ID + Secret + 验证码 | 先验证码正确 → AES 加密 Secret → 存 DB → 生成恢复码 | `{recoveryCodes}` |
| `disableMfa(userId, token)` | 用户ID + 验证码 | 验证码正确 → 清除 MFA 数据 | void |
| `generateRecoveryCodes(userId)` | 用户ID | 生成 8 个一次性恢复码，bcrypt 存储 | 明文恢复码数组（仅此一次展示） |
| `verifyRecoveryCode(userId, code)` | 用户ID + 恢复码 | 遍历匹配 → 标记已使用 | Boolean |

### 2. TOTP 技术规格

| 参数 | 值 | 说明 |
|------|------|------|
| 算法 | SHA-1 | Google Authenticator 标准 |
| 位数 | 6 位 | 标准 TOTP |
| 周期 | 30 秒 | 标准时间窗口 |
| 容差 | ±1 窗口 | 允许前后各 30 秒 |
| 依赖 | otpauth + qrcode | npm 包 |

### 3. Secret 加密存储

| 项目 | 要求 |
|------|------|
| 算法 | AES-256-GCM |
| 密钥 | 从环境变量 `MFA_ENCRYPTION_KEY` 读取（32 字节 hex） |
| 格式 | `{iv}:{authTag}:{ciphertext}`（Base64 编码） |
| 原因 | TOTP Secret 等同密码，泄露即可生成有效验证码 |

### 4. 恢复码设计

| 项目 | 要求 |
|------|------|
| 数量 | 8 个 |
| 格式 | `XXXX-XXXX`（8 位大写字母数字） |
| 存储 | bcrypt 哈希存储到 RecoveryCode 表 |
| 使用 | 一次性，验证后标记 `isUsed = true` |
| 展示 | 仅在 enableMfa 时返回一次，之后无法再查看 |

### 5. 修改登录流程

MFA 启用时，login 不直接签发 JWT，而是返回临时 mfaToken（5 分钟过期）。

### 6. 新增路由

| 方法 | 路径 | 认证 | 说明 |
|------|------|------|------|
| POST | `/mfa/setup` | JWT | 生成 QR Code（不立即启用） |
| POST | `/mfa/enable` | JWT | 用验证码确认启用 |
| POST | `/mfa/disable` | JWT | 用验证码确认禁用 |
| POST | `/auth/mfa-verify` | 无 | mfaToken + TOTP Code → 签发正式 JWT |
| POST | `/auth/mfa-recovery` | 无 | mfaToken + Recovery Code → 签发 JWT |

## MFA 启用流程

```
=== MFA 启用流程（首次设置） ===

  用户 (已登录)            XAuth Server              Google Authenticator
    |                         |                            |
    | 1. POST /mfa/setup      |                            |
    |------------------------>|                            |
    |                         | 2. generateSecret           |
    |                         |    生成 TOTP Secret          |
    |                         |    生成 otpauth:// URI       |
    |                         |    生成 QR Code Data URL     |
    |                         |                            |
    | 3. 返回 QR Code         |                            |
    | {qrCodeUrl:             |                            |
    |  "data:image/png;       |                            |
    |   base64,...",          |                            |
    |  secret: "..."}         |                            |
    |<------------------------|                            |
    |                         |                            |
    | 4. 用户用手机扫码        |                            |
    |----------------------------------------------------->|
    |                         |              5. App 开始生成|
    |                         |                 每 30s 一个 |
    |                         |                 6 位验证码  |
    |                         |                            |
    | 6. POST /mfa/enable     |                            |
    | {token: "123456"}       |                            |
    |------------------------>|                            |
    |                         | 7. verifyToken              |
    |                         |    (secret, "123456")       |
    |                         |    验证码匹配 ✅            |
    |                         |                            |
    |                         | 8. AES-256-GCM 加密 Secret|
    |                         | 9. 存入 User.mfaSecret    |
    |                         | 10. User.mfaEnabled=true   |
    |                         | 11. generateRecoveryCodes  |
    |                         |     生成 8 个恢复码        |
    |                         |     bcrypt 存入 DB         |
    |                         |                            |
    | 12. {recoveryCodes:     |                            |
    |  ["ABCD-1234",          |                            |
    |   "EFGH-5678", ...]}    |                            |
    | ⚠️ 仅此一次展示         |                            |
    |<------------------------|                            |
```

## MFA 登录流程

```
=== MFA 登录流程（两步验证） ===

  Client                  XAuth Server             Token Service
    |                         |                         |
    | ======== 第一步：用户名密码 ========                |
    |                         |                         |
    | POST /auth/login        |                         |
    | {username, password}    |                         |
    |------------------------>|                         |
    |                         | 1. 验证用户名密码 ✅     |
    |                         | 2. 检查 mfaEnabled       |
    |                         |    --> true              |
    |                         | 3. 签发临时 mfaToken    |
    |                         |    (短期，5min 过期)     |
    |                         |------------------------>|
    |                         |                         |
    | {requireMfa: true,      |                         |
    |  mfaToken: "xxx"}       |                         |
    | (不返回正式 JWT)        |                         |
    |<------------------------|                         |
    |                         |                         |
    | ======== 第二步A：TOTP 验证码 ========              |
    |                         |                         |
    | POST /auth/mfa-verify   |                         |
    | {mfaToken: "xxx",       |                         |
    |  token: "123456"}       |                         |
    |------------------------>|                         |
    |                         | 4. 验证 mfaToken        |
    |                         | 5. 解密 AES --> Secret  |
    |                         | 6. verifyToken ✅       |
    |                         | 7. 签发正式 JWT         |
    |                         |------------------------>|
    |                         |                         |
    | {accessToken,           |                         |
    |  refreshToken}          |                         |
    |<------------------------|                         |
    |                         |                         |
    | ======== 第二步B（备选）：恢复码登录 ========        |
    |                         |                         |
    | POST /auth/             |                         |
    |   mfa-recovery          |                         |
    | {mfaToken: "xxx",       |                         |
    |  recoveryCode:          |                         |
    |  "ABCD-1234"}           |                         |
    |------------------------>|                         |
    |                         | 8. 验证 mfaToken        |
    |                         | 9. bcrypt.compare        |
    |                         |    遍历恢复码 ✅         |
    |                         | 10. 标记 isUsed=true    |
    |                         | 11. 签发正式 JWT        |
    |                         |                         |
    | {accessToken,           |                         |
    |  refreshToken,          |                         |
    |  remainingCodes: 7}     |                         |
    |<------------------------|                         |
```

### TOTP 时间窗口示意

```
  时间轴（30s 一个窗口）
  ---------------------------------------------------------->
  |  T-1 窗口   |  T 当前窗口  |  T+1 窗口   |
  |  (容差 ✅)  |  (标准 ✅)   |  (容差 ✅)  |  T+2 ❌
  |  验证码 A   |  验证码 B    |  验证码 C   |
  +-------------+--------------+-------------+

  用户输入验证码 B：
    - 在 T 窗口内  --> ✅ 验证通过
    - 在 T-1 窗口  --> ✅ 容差通过（用户看码稍慢）
    - 在 T+1 窗口  --> ✅ 容差通过（时钟略有偏差）
    - 在 T+2 窗口  --> ❌ 验证失败
```

## API 示例

### POST /mfa/setup

**生成 TOTP Secret 和 QR Code（不立即启用）**

Request:
```bash
curl -s -X POST http://localhost:3000/mfa/setup \
  -H "Authorization: Bearer $TOKEN"
```

Response (200):
```json
{
  "success": true,
  "data": {
    "secret": "JBSWY3DPEHPK3PXP",
    "qrCodeUrl": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAOQAAADk..."
  }
}
```

---

### POST /mfa/enable

**用 TOTP 验证码确认启用 MFA，返回一次性恢复码**

Request:
```bash
curl -s -X POST http://localhost:3000/mfa/enable \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"token":"482937"}'
```

Response (200):
```json
{
  "success": true,
  "data": {
    "message": "MFA 已启用",
    "recoveryCodes": [
      "ABCD-1234",
      "EFGH-5678",
      "IJKL-9012",
      "MNOP-3456",
      "QRST-7890",
      "UVWX-2345",
      "YZAB-6789",
      "CDEF-0123"
    ]
  }
}
```

**验证码错误，启用失败**

Request:
```bash
curl -s -X POST http://localhost:3000/mfa/enable \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"token":"000000"}'
```

Response (400):
```json
{
  "success": false,
  "error": {
    "code": "INVALID_TOKEN",
    "message": "验证码错误"
  }
}
```

---

### POST /mfa/disable

**用 TOTP 验证码确认禁用 MFA**

Request:
```bash
curl -s -X POST http://localhost:3000/mfa/disable \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"token":"159263"}'
```

Response (200):
```json
{
  "success": true,
  "data": {
    "message": "MFA 已禁用"
  }
}
```

---

### POST /auth/login（MFA 用户登录第一步）

**已启用 MFA 的用户登录，返回临时 mfaToken**

Request:
```bash
curl -s -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"demo","password":"Demo@123456"}'
```

Response (200):
```json
{
  "success": true,
  "data": {
    "requireMfa": true,
    "mfaToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...mfa"
  }
}
```

---

### POST /auth/mfa-verify

**用 TOTP 验证码完成二步验证，获取正式 JWT**

Request:
```bash
curl -s -X POST http://localhost:3000/auth/mfa-verify \
  -H "Content-Type: application/json" \
  -d '{"mfaToken":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...mfa","token":"739201"}'
```

Response (200):
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...xyz",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...abc",
    "user": {
      "id": "c6a8e3b2-4f5d-5a9b-0c7e-2b3c4d5e6f7a",
      "username": "demo",
      "role": "USER"
    }
  }
}
```

---

### POST /auth/mfa-recovery

**用一次性恢复码完成二步验证，获取正式 JWT**

Request:
```bash
curl -s -X POST http://localhost:3000/auth/mfa-recovery \
  -H "Content-Type: application/json" \
  -d '{"mfaToken":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...mfa","recoveryCode":"ABCD-1234"}'
```

Response (200):
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...xyz",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...abc",
    "remainingCodes": 7,
    "user": {
      "id": "c6a8e3b2-4f5d-5a9b-0c7e-2b3c4d5e6f7a",
      "username": "demo",
      "role": "USER"
    }
  }
}
```

## Prompt

```
实现 TOTP 双因素认证（兼容 Google Authenticator）：

1. src/services/mfa.service.js：
   - generateSecret(username) → TOTP Secret + QR Code Data URL
   - verifyToken(secret, token) → 验证 6 位码，允许 ±1 时间窗口
   - enableMfa / disableMfa
   - generateRecoveryCodes → 8 个一次性恢复码，bcrypt 存储
   - verifyRecoveryCode → 验证并消耗

2. 修改 auth.service.js 的 login：MFA 启用时返回临时 mfaToken

3. src/routes/mfa.routes.js（需认证）：
   - POST /mfa/setup → 生成 QR Code
   - POST /mfa/enable → 确认启用
   - POST /mfa/disable → 确认禁用

4. auth.routes.js 增加：
   - POST /auth/mfa-verify → mfaToken + TOTP Code → 签发正式 JWT
   - POST /auth/mfa-recovery → mfaToken + Recovery Code → 签发 JWT

TOTP Secret 使用 AES-256-GCM 加密存储，密钥从环境变量读取。
使用 otpauth 和 qrcode 包。
```

## 验收标准

| # | 验收项 | 验证方式 | 预期结果 |
|---|--------|----------|----------|
| 1 | 生成 QR Code | POST /mfa/setup | 返回 qrCodeUrl（data:image/png base64） |
| 2 | QR Code 可扫 | 手机 Authenticator 扫码 | App 开始显示 6 位验证码 |
| 3 | 启用 MFA | POST /mfa/enable + 正确验证码 | 返回 8 个恢复码 |
| 4 | 错误验证码拒绝 | POST /mfa/enable + 错误码 | 400，验证码错误 |
| 5 | 登录返回 mfaToken | MFA 用户登录 | `{requireMfa: true, mfaToken: "..."}` |
| 6 | MFA 二步验证 | POST /auth/mfa-verify | 返回正式 accessToken + refreshToken |
| 7 | 恢复码登录 | POST /auth/mfa-recovery | 返回 JWT + remainingCodes |
| 8 | 恢复码一次性 | 同一恢复码用两次 | 第二次 400 |
| 9 | Secret 加密存储 | 查数据库 mfaSecret 字段 | 密文格式 `{iv}:{tag}:{cipher}` |
| 10 | 禁用 MFA | POST /mfa/disable + 验证码 | mfaEnabled 变 false |
| 11 | .env 含加密密钥 | 检查 .env.example | 包含 MFA_ENCRYPTION_KEY |

## 验证

```bash
# 启用 MFA（获取 QR Code）
curl -s -X POST http://localhost:3000/mfa/setup \
  -H "Authorization: Bearer $TOKEN" | jq

# 将返回的 qrCodeUrl 在浏览器打开，用 Authenticator 扫码
# 然后用 App 显示的 6 位码确认启用
curl -s -X POST http://localhost:3000/mfa/enable \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"token":"123456"}' | jq

# 测试 MFA 登录流程
curl -s -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"demo","password":"Demo@123456"}' | jq
# 应返回 requireMfa: true + mfaToken
```

## 讲师要点

- 如有条件，现场用手机 Authenticator 扫码演示
- 展示恢复码的安全设计：哈希存储、一次性使用
- 讨论 AES-256-GCM 加密 TOTP Secret 的必要性
