# 统一认证中心 — 架构设计文档

---

## 一、技术选型

| 层级 | 技术 | 选型理由 |
|------|------|---------|
| **运行时** | Node.js 20 LTS | 异步 I/O、生态丰富、团队熟悉 |
| **框架** | Express 4.x | 轻量灵活、中间件生态成熟 |
| **数据库** | PostgreSQL 16 | 生产级关系型数据库、JSON 支持好 |
| **ORM** | Prisma | 类型安全、迁移管理好、开发体验优 |
| **缓存** | Redis 7 | Token 黑名单、会话缓存、限流 |
| **认证** | Passport.js | 业界标准、策略模式插件化 |
| **JWT** | jose | 标准库、支持 RS256/ES256 |
| **MFA** | otpauth | TOTP 标准实现、兼容主流 Authenticator |
| **测试** | Jest + Supertest | 单元测试 + 集成测试 |
| **容器** | Docker + Compose | 一键启动全部依赖 |

---

## 二、系统架构图

```
┌─────────────────────────────────────────────────────────┐
│                    客户端 (Browser / App / API Client)     │
└───────────┬────────────────────────────────┬─────────────┘
            │                                │
            ▼                                ▼
┌───────────────────┐              ┌──────────────────┐
│  Web 前端 (可选)    │              │  第三方 API 调用   │
│  登录页 / 回调页    │              │  API Key + HMAC   │
└───────────┬───────┘              └────────┬─────────┘
            │                               │
            ▼                               ▼
┌─────────────────────────────────────────────────────────┐
│                    API Gateway Layer                      │
│  ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐  │
│  │ 限流     │ │ 日志     │ │ CORS     │ │ 请求解析    │  │
│  └─────────┘ └──────────┘ └──────────┘ └────────────┘  │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                 Authentication Layer                      │
│                                                          │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────┐  │
│  │ Local Auth   │  │ OAuth2 Auth   │  │ API Key Auth │  │
│  │ (用户名密码)  │  │ (GitHub/Google │  │ (HMAC 签名)  │  │
│  │              │  │  /微信)        │  │              │  │
│  └──────┬───────┘  └──────┬────────┘  └──────┬───────┘  │
│         │                 │                   │          │
│         ▼                 ▼                   ▼          │
│  ┌─────────────────────────────────────────────────┐     │
│  │              Token Service (JWT)                 │     │
│  │  • Access Token (RS256, 15min)                  │     │
│  │  • Refresh Token (7d, 单设备)                    │     │
│  │  • Token 黑名单 (Redis)                         │     │
│  └─────────────────────────────────────────────────┘     │
│                                                          │
│  ┌─────────────────┐  ┌─────────────────────────────┐   │
│  │ MFA Service     │  │ Session / Token Blacklist    │   │
│  │ (TOTP + 恢复码) │  │ (Redis)                      │   │
│  └─────────────────┘  └─────────────────────────────┘   │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                 Authorization Layer                      │
│  ┌─────────────────┐  ┌──────────────────────────────┐  │
│  │ RBAC 角色守卫    │  │ 资源所有权校验               │  │
│  │ (admin/user/     │  │ (只能操作自己的资源)          │  │
│  │  guest)          │  │                              │  │
│  └─────────────────┘  └──────────────────────────────┘  │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                  Business Layer                          │
│  ┌──────────┐ ┌───────────┐ ┌───────────┐ ┌─────────┐  │
│  │ User     │ │ OAuth     │ │ API Key   │ │ Audit   │  │
│  │ Service  │ │ Service   │ │ Service   │ │ Service │  │
│  └──────────┘ └───────────┘ └───────────┘ └─────────┘  │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                   Data Layer                             │
│  ┌──────────────────┐       ┌─────────────────────┐     │
│  │ PostgreSQL       │       │ Redis               │     │
│  │ • users          │       │ • token:blacklist   │     │
│  │ • oauth_accounts │       │ • rate_limit:*      │     │
│  │ • api_keys       │       │ • session:*         │     │
│  │ • audit_logs     │       │                     │     │
│  │ • mfa_secrets    │       │                     │     │
│  └──────────────────┘       └─────────────────────┘     │
└─────────────────────────────────────────────────────────┘
```

---

## 三、目录结构

```
xauth/
├── docker-compose.yml
├── .env.example
├── package.json
├── prisma/
│   └── schema.prisma              # 数据库模型定义
├── src/
│   ├── app.js                     # Express 应用入口
│   ├── config/
│   │   ├── index.js               # 统一配置管理
│   │   └── passport.js            # Passport 策略配置
│   ├── middlewares/
│   │   ├── auth.js                # JWT 校验中间件
│   │   ├── rbac.js                # 角色权限守卫
│   │   ├── rate-limit.js          # 限流中间件
│   │   ├── validate.js            # 请求校验中间件
│   │   ├── audit-log.js           # 审计日志中间件
│   │   └── error-handler.js       # 统一错误处理
│   ├── routes/
│   │   ├── auth.routes.js         # 认证路由 (注册/登录/登出/刷新)
│   │   ├── oauth.routes.js        # OAuth2 路由 (GitHub/Google/微信)
│   │   ├── mfa.routes.js          # MFA 路由 (启用/验证/恢复)
│   │   ├── apikey.routes.js       # API Key 路由
│   │   └── user.routes.js         # 用户管理路由
│   ├── services/
│   │   ├── auth.service.js        # 认证业务逻辑
│   │   ├── token.service.js       # JWT Token 管理
│   │   ├── oauth.service.js       # OAuth2 业务逻辑
│   │   ├── mfa.service.js         # MFA 业务逻辑
│   │   ├── apikey.service.js      # API Key 管理
│   │   ├── user.service.js        # 用户管理
│   │   └── audit.service.js       # 审计日志
│   ├── utils/
│   │   ├── errors.js              # 自定义错误类
│   │   ├── crypto.js              # 加密工具函数
│   │   └── redis.js               # Redis 客户端
│   └── tests/
│       ├── auth.test.js
│       ├── oauth.test.js
│       ├── token.test.js
│       ├── mfa.test.js
│       ├── apikey.test.js
│       └── rbac.test.js
├── keys/
│   ├── private.pem                # JWT RS256 私钥 (git ignored)
│   └── public.pem                 # JWT RS256 公钥
└── docs/
    └── api.md                     # API 文档
```

---

## 四、数据模型

```prisma
model User {
  id            String    @id @default(uuid())
  username      String    @unique
  email         String    @unique
  passwordHash  String
  role          Role      @default(USER)
  mfaEnabled    Boolean   @default(false)
  mfaSecret     String?
  recoveryCodes String[]
  isActive      Boolean   @default(true)
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  oauthAccounts OAuthAccount[]
  apiKeys       ApiKey[]
  auditLogs     AuditLog[]
}

model OAuthAccount {
  id           String   @id @default(uuid())
  provider     String   // github, google, wechat
  providerId   String   // 第三方平台的用户 ID
  accessToken  String?
  refreshToken String?
  profile      Json?    // 第三方平台的用户信息
  userId       String
  user         User     @relation(fields: [userId], references: [id])
  createdAt    DateTime @default(now())

  @@unique([provider, providerId])
}

model ApiKey {
  id          String   @id @default(uuid())
  name        String
  keyHash     String   @unique  // API Key 的哈希值（不存明文）
  keyPrefix   String             // 前 8 位用于识别
  permissions String[]
  rateLimit   Int      @default(1000) // 每小时请求上限
  expiresAt   DateTime?
  lastUsedAt  DateTime?
  userId      String
  user        User     @relation(fields: [userId], references: [id])
  createdAt   DateTime @default(now())
}

model AuditLog {
  id        String   @id @default(uuid())
  action    String   // login, logout, login_failed, mfa_enabled, ...
  ip        String?
  userAgent String?
  details   Json?
  userId    String?
  user      User?    @relation(fields: [userId], references: [id])
  createdAt DateTime @default(now())
}

enum Role {
  ADMIN
  USER
  GUEST
}
```

---

## 五、认证流程设计

### 5.1 JWT 认证流程

```
Client                    Server                     Redis
  │                         │                          │
  │── POST /auth/login ────▶│                          │
  │   {username, password}  │                          │
  │                         │── 校验密码 ──────────────▶│
  │                         │── 检查 MFA 是否启用       │
  │                         │                          │
  │◀── 200 ────────────────│                          │
  │   { accessToken (15m)   │                          │
  │     refreshToken (7d) } │                          │
  │                         │                          │
  │── GET /api/xxx ────────▶│                          │
  │   Authorization: Bearer │                          │
  │                         │── 验签 + 过期检查         │
  │                         │── 检查黑名单 ───────────▶│
  │                         │◀── 不在黑名单 ──────────│
  │◀── 200 data ───────────│                          │
  │                         │                          │
  │── POST /auth/refresh ──▶│                          │
  │   { refreshToken }      │── 验证 Refresh Token     │
  │                         │── 签发新 Access Token     │
  │◀── 200 { accessToken } │                          │
  │                         │                          │
  │── POST /auth/logout ───▶│                          │
  │   Authorization: Bearer │── 将 Token 加入黑名单 ──▶│
  │◀── 200 ────────────────│                          │
```

### 5.2 OAuth2 Authorization Code Flow

```
Client                  Server                   GitHub/Google
  │                       │                          │
  │── GET /oauth/github ─▶│                          │
  │                       │── 生成 state 参数         │
  │                       │── 302 重定向 ───────────▶│
  │◀── 302 ──────────────│  authorize?              │
  │                       │  client_id=xxx&          │
  │                       │  redirect_uri=xxx&       │
  │                       │  state=xxx               │
  │                       │                          │
  │ (用户在 GitHub 授权)    │                          │
  │                       │                          │
  │◀── 302 callback ─────│◀── ?code=xxx&state=xxx ─│
  │                       │                          │
  │── GET /oauth/github/ ▶│                          │
  │   callback?code=xxx   │── POST token endpoint ──▶│
  │                       │   {code, client_secret}  │
  │                       │◀── {access_token} ──────│
  │                       │                          │
  │                       │── GET /user endpoint ───▶│
  │                       │◀── {id, email, name} ───│
  │                       │                          │
  │                       │── 查找/创建本地用户        │
  │                       │── 签发 JWT               │
  │◀── 200 {jwt tokens} ─│                          │
```

### 5.3 TOTP MFA 流程

```
Client                      Server
  │                            │
  │── POST /mfa/enable ───────▶│
  │   Authorization: Bearer    │── 生成 TOTP Secret
  │                            │── 生成恢复码 (8个)
  │◀── 200 ───────────────────│
  │   { secret, qrCodeUrl,     │
  │     recoveryCodes }        │
  │                            │
  │ (用户扫描二维码到 Authenticator)
  │                            │
  │── POST /mfa/verify ───────▶│
  │   { token: "123456" }      │── 验证 TOTP Token
  │                            │── 启用 MFA
  │◀── 200 { enabled: true } ─│
  │                            │
  │ (下次登录)                   │
  │── POST /auth/login ───────▶│
  │   { username, password }   │── 密码校验通过
  │                            │── 检测 MFA 已启用
  │◀── 200 ───────────────────│
  │   { requireMfa: true,      │
  │     mfaToken: "temp..." }  │
  │                            │
  │── POST /auth/mfa-login ───▶│
  │   { mfaToken, code }       │── 验证 TOTP Code
  │                            │── 签发正式 JWT
  │◀── 200 { accessToken,     │
  │          refreshToken }    │
```

### 5.4 API Key + HMAC 签名

```
签名算法：
  signature = HMAC-SHA256(
    key = api_secret,
    message = "{method}\n{path}\n{timestamp}\n{body_hash}"
  )

请求头：
  X-API-Key: ak_xxxxxxxx
  X-Timestamp: 1709123456
  X-Signature: base64(signature)

服务端校验：
  1. 提取 API Key → 查找对应 Secret
  2. 检查 Timestamp 是否在 ±5 分钟内（防重放）
  3. 用相同算法计算签名并对比
  4. 检查限流是否超出
```

---

## 六、安全设计

| 威胁 | 防护措施 |
|------|---------|
| 暴力破解 | 登录限流（5次/分钟）、账户锁定（连续 10 次失败） |
| Token 泄露 | 短过期时间（15min）、Token 黑名单、HTTPS Only |
| CSRF | SameSite Cookie、CSRF Token |
| XSS | HttpOnly Cookie、Content-Security-Policy |
| SQL 注入 | Prisma 参数化查询（ORM 层防护） |
| 密码泄露 | bcrypt(12 轮)、不返回 passwordHash |
| 重放攻击 | HMAC 签名的 Timestamp 校验（±5 分钟窗口） |
| Token 伪造 | RS256 非对称签名（私钥不出服务端） |
