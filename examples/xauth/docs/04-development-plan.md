# 统一认证中心 — 开发计划

> **演示方式**：按 Sprint 顺序逐步执行 Prompt，每个 Sprint 用 AI 从零编码实现。
> **预计总演示时间**：60-90 分钟（跳过等待时间），可根据培训时长选择性演示。

---

## 开发阶段总览

```
Sprint 0  项目初始化              (5 min)    ████
Sprint 1  用户注册登录 + JWT      (15 min)   ████████████
Sprint 2  Token 管理 + 登出       (10 min)   ████████
Sprint 3  RBAC 权限控制           (10 min)   ████████
Sprint 4  OAuth2 社交登录         (10 min)   ████████
Sprint 5  MFA 双因素认证          (10 min)   ████████
Sprint 6  API Key + HMAC         (10 min)   ████████
Sprint 7  测试补全 + 安全审查      (10 min)   ████████
Sprint 8  Docker 化 + 文档        (5 min)    ████
```

---

## Sprint 0：项目初始化（5 分钟）

### Prompt 0.1 — 创建项目骨架

```
基于以下技术栈创建项目骨架：
- Node.js + Express
- Prisma + PostgreSQL
- Redis (ioredis)
- Jest + Supertest

请创建：
1. package.json（包含所有依赖和 scripts）
2. .env.example（所有需要的环境变量，含注释说明）
3. .gitignore
4. docker-compose.yml（PostgreSQL 16 + Redis 7，含健康检查）
5. src/app.js（Express 应用入口，包含基础中间件：JSON解析、CORS、请求日志）
6. src/config/index.js（统一配置管理，从环境变量读取）
7. src/middlewares/error-handler.js（统一错误处理中间件）
8. src/utils/errors.js（自定义错误类：AppError, NotFoundError, ValidationError, UnauthorizedError, ForbiddenError）

项目使用 CommonJS 模块规范。
不需要 TypeScript。
```

### Prompt 0.2 — 数据库模型

```
根据以下 Prisma Schema 创建数据库模型文件 prisma/schema.prisma：

数据模型：
1. User: id(uuid), username(unique), email(unique), passwordHash, role(enum: ADMIN/USER/GUEST), mfaEnabled(bool), mfaSecret, recoveryCodes(string[]), isActive(bool), timestamps
2. OAuthAccount: id(uuid), provider, providerId, accessToken, refreshToken, profile(json), userId(FK→User), createdAt。provider+providerId 联合唯一。
3. ApiKey: id(uuid), name, keyHash(unique), keyPrefix, permissions(string[]), rateLimit(int, default 1000), expiresAt, lastUsedAt, userId(FK→User), createdAt
4. AuditLog: id(uuid), action, ip, userAgent, details(json), userId(FK→User nullable), createdAt

配置 PostgreSQL provider，数据库 URL 从环境变量 DATABASE_URL 读取。

然后运行 prisma migrate dev 初始化数据库。
```

### 讲师要点
- 强调 AI 在 30 秒内生成了完整的项目骨架
- 传统方式：查文档 + 复制粘贴 + 调配置，至少 1-2 小时

---

## Sprint 1：用户注册登录 + JWT（15 分钟）

### Prompt 1.1 — JWT Token 服务

```
创建 src/services/token.service.js，实现 JWT Token 管理：

1. 使用 jsonwebtoken 包
2. 使用 RS256 非对称加密签名
3. 生成 RSA 密钥对的脚本 scripts/generate-keys.js（生成到 keys/ 目录）
4. 实现方法：
   - generateAccessToken(user) → 签发 Access Token，payload 包含 {userId, role}，15 分钟过期
   - generateRefreshToken(user) → 签发 Refresh Token，payload 包含 {userId, type:'refresh'}，7 天过期
   - verifyAccessToken(token) → 验证并返回 payload，失败抛出 UnauthorizedError
   - verifyRefreshToken(token) → 验证 Refresh Token
   - generateTokenPair(user) → 同时返回 {accessToken, refreshToken}
5. 密钥路径从 config 读取

参考项目架构文档 02-architecture.md 中的 JWT 认证流程设计。
```

### Prompt 1.2 — 用户注册与登录

```
创建以下文件实现用户注册和登录功能：

1. src/services/auth.service.js：
   - register(username, email, password)
     → 密码强度校验（≥8位，含大小写+数字+特殊字符）
     → bcrypt(12轮) 加密
     → 创建用户
     → 返回用户信息（不含 passwordHash）
   - login(username, password)
     → 查找用户 → 校验密码 → 检查账户是否激活
     → 如果 MFA 已启用，返回 {requireMfa: true, mfaToken}
     → 否则直接签发 JWT Token Pair
   - 使用 Prisma 操作数据库

2. src/services/user.service.js：
   - getUserById(id) → 返回用户信息（排除 passwordHash, mfaSecret）
   - getUserByUsername(username)
   - updateUser(id, data)
   - 定义 sanitizeUser(user) 函数，过滤敏感字段

3. src/routes/auth.routes.js：
   - POST /auth/register → 注册
   - POST /auth/login → 登录
   - 输入校验：username(3-30字符), email(格式校验), password(强度校验)
   - 统一响应格式 { success: true, data: {...} }

4. 在 app.js 中注册路由

5. src/middlewares/validate.js：
   - 通用校验中间件，不依赖外部校验库
   - 支持: required, minLength, maxLength, pattern, isEmail
```

### 讲师要点
- 检查 AI 是否正确实现了密码强度校验
- 检查 bcrypt 是否使用了 12 轮
- 检查 API 响应中是否排除了 passwordHash
- 现场用 curl 测试注册和登录

---

## Sprint 2：Token 管理 + 登出（10 分钟）

### Prompt 2.1 — Token 刷新与黑名单

```
实现 Token 刷新和登出功能：

1. src/utils/redis.js：
   - Redis 客户端封装（使用 ioredis）
   - 连接配置从环境变量读取
   - 优雅的错误处理和重连机制

2. 在 token.service.js 中增加方法：
   - blacklistToken(token) → 将 Token 加入 Redis 黑名单
     → Key: "bl:{tokenHash}"，TTL = Token 剩余有效时间
   - isBlacklisted(token) → 检查 Token 是否在黑名单

3. src/middlewares/auth.js：
   - JWT 校验中间件
   - 从 Authorization: Bearer xxx 提取 Token
   - 验签 → 检查黑名单 → 注入 req.user = {userId, role}
   - 失败返回 401

4. 在 auth.routes.js 中增加路由：
   - POST /auth/refresh → 用 Refresh Token 换取新 Access Token
   - POST /auth/logout → 将当前 Access Token 加入黑名单
   - 都需要验证对应的 Token

5. Redis 降级策略：
   - 如果 Redis 不可用，降级为内存 Map 存储黑名单
   - 记录告警日志
```

### 讲师要点
- 重点展示 AI 实现 Redis 降级策略的方案质量
- 检查 Token 黑名单的 TTL 是否设置正确（不是固定时间，是 Token 剩余有效期）

---

## Sprint 3：RBAC 权限控制（10 分钟）

### Prompt 3.1 — 角色权限中间件

```
实现 RBAC 权限控制：

1. src/middlewares/rbac.js：
   - requireRole(...roles) → 高阶函数，返回中间件
     → 检查 req.user.role 是否在允许的角色列表中
     → 不匹配返回 403 Forbidden
   - requireOwnership(resourceUserIdExtractor) → 高阶函数
     → 检查当前用户是否是资源的拥有者（ADMIN 跳过检查）
     → 例如：requireOwnership(req => req.params.userId)

2. src/routes/user.routes.js — 用户管理路由：
   - GET /users → 仅 ADMIN（获取用户列表，分页）
   - GET /users/:id → ADMIN 或本人
   - PUT /users/:id → ADMIN 或本人
   - DELETE /users/:id → 仅 ADMIN
   - PUT /users/:id/role → 仅 ADMIN（修改角色）

3. 所有用户管理路由需要先通过 auth 中间件认证

4. src/services/audit.service.js：
   - logAction(action, userId, details, req)
     → 记录操作到 AuditLog 表
     → 自动提取 ip 和 userAgent
   - 在关键操作（注册、登录、登出、角色变更）中调用

5. src/middlewares/audit-log.js：
   - 自动记录所有认证相关请求的中间件
```

### 讲师要点
- 展示 AI 生成的 requireOwnership 高阶函数的设计
- 测试越权访问：用 USER 角色访问 ADMIN 接口，验证 403

---

## Sprint 4：OAuth2 社交登录（10 分钟）

### Prompt 4.1 — GitHub OAuth2

```
实现 GitHub OAuth2 登录，使用 Passport.js：

1. src/config/passport.js：
   - 配置 passport-github2 策略
   - OAuth2 回调中：
     → 用 providerId 查找已绑定的本地用户
     → 如果找到，直接登录
     → 如果未找到，自动创建新用户（用 GitHub 用户名和邮箱）
     → 创建 OAuthAccount 关联记录

2. src/services/oauth.service.js：
   - findOrCreateByOAuth(provider, profile, tokens)
   - linkOAuthAccount(userId, provider, profile)
   - unlinkOAuthAccount(userId, provider)
   - getLinkedAccounts(userId)

3. src/routes/oauth.routes.js：
   - GET /oauth/github → 重定向到 GitHub 授权页
   - GET /oauth/github/callback → 授权回调，签发 JWT
   - GET /oauth/accounts → 查看已绑定的社交账号（需认证）
   - POST /oauth/link/github → 绑定 GitHub 到现有账号（需认证）
   - DELETE /oauth/unlink/github → 解绑（需认证）

4. 在 .env.example 中添加 GitHub OAuth 配置项

注意：
- state 参数必须校验以防 CSRF
- 回调成功后重定向到前端页面并携带 Token
- 如果用户已登录状态下进行 OAuth 授权，执行"绑定"而非"登录"
```

### Prompt 4.2 — Google OAuth2（可选加速演示）

```
参照 GitHub OAuth2 的实现，增加 Google OAuth2 登录支持：

1. 添加 passport-google-oauth20 策略
2. 实现 PKCE (Proof Key for Code Exchange) 增强安全
3. 路由：GET /oauth/google, GET /oauth/google/callback
4. 复用 oauth.service.js 的 findOrCreateByOAuth 方法

代码结构与 GitHub OAuth 保持一致。
```

### 讲师要点
- 展示 AI 如何复用已有的 OAuth 架构快速添加第二个提供商
- 讨论 state 参数防 CSRF 的重要性

---

## Sprint 5：MFA 双因素认证（10 分钟）

### Prompt 5.1 — TOTP 实现

```
实现 TOTP 双因素认证（兼容 Google Authenticator）：

1. src/services/mfa.service.js：
   - generateSecret(username) → 生成 TOTP Secret + otpauth:// URI + QR Code Data URL
   - verifyToken(secret, token) → 验证 6 位 TOTP 码，允许 ±1 时间窗口
   - enableMfa(userId, secret, token) → 验证 token 正确后启用 MFA，保存 secret（加密存储）
   - disableMfa(userId, token) → 验证当前 token 后禁用 MFA
   - generateRecoveryCodes(userId) → 生成 8 个一次性恢复码，bcrypt 哈希后存储
   - verifyRecoveryCode(userId, code) → 验证并消耗一个恢复码

2. 修改 auth.service.js 的 login 方法：
   - 密码校验通过后，检查 user.mfaEnabled
   - 如果启用 MFA：返回临时 mfaToken（5分钟有效）+ requireMfa: true
   - 不直接签发正式 JWT

3. src/routes/mfa.routes.js（所有路由需认证）：
   - POST /mfa/setup → 生成 Secret 和 QR Code
   - POST /mfa/enable → 用 TOTP Token 确认启用
   - POST /mfa/disable → 用 TOTP Token 确认禁用
   - GET /mfa/recovery-codes → 查看剩余恢复码数量
   - POST /mfa/regenerate-codes → 重新生成恢复码

4. 在 auth.routes.js 增加：
   - POST /auth/mfa-verify → 用 mfaToken + TOTP Code 完成登录
   - POST /auth/mfa-recovery → 用 mfaToken + Recovery Code 完成登录

使用 otpauth 包生成 TOTP，使用 qrcode 包生成 QR Code。
TOTP Secret 使用 AES-256-GCM 加密后存储到数据库。
```

### 讲师要点
- 现场用手机 Authenticator 扫码演示（如果条件允许）
- 展示恢复码的安全设计：只存哈希、一次性使用

---

## Sprint 6：API Key + HMAC 签名（10 分钟）

### Prompt 6.1 — API Key 管理和 HMAC 验证

```
实现 API Key 认证（面向第三方 API 调用）：

1. src/services/apikey.service.js：
   - createApiKey(userId, name, permissions)
     → 生成随机 API Key（格式：ak_xxxxxxxxxxxxxxxx）
     → 只存储 key 的 SHA-256 哈希 + 前 8 位前缀
     → 生成对应的 API Secret（格式：sk_xxxxxxxxxxxxxxxx）
     → 返回明文 key 和 secret（仅此一次，之后无法再查看）
   - verifyApiKey(keyPrefix, signature, method, path, timestamp, bodyHash)
     → 查找 Key → 校验时间戳（±5分钟）→ 重算签名对比
   - listApiKeys(userId) → 返回用户的 API Key 列表（只显示前缀和名称）
   - revokeApiKey(userId, keyId)
   - updateLastUsed(keyId)

2. src/middlewares/apikey-auth.js：
   - 从请求头提取 X-API-Key, X-Timestamp, X-Signature
   - 校验签名：HMAC-SHA256(secret, "{method}\n{path}\n{timestamp}\n{bodyHash}")
   - 校验通过后注入 req.apiKey = {userId, permissions, keyId}
   - 检查限流（Redis INCR + EXPIRE，按小时窗口）

3. src/routes/apikey.routes.js（需 JWT 认证）：
   - POST /api-keys → 创建 API Key
   - GET /api-keys → 列出我的 API Key
   - DELETE /api-keys/:id → 吊销 API Key

4. src/middlewares/rate-limit.js：
   - 通用限流中间件（基于 Redis）
   - 支持按 IP、按用户、按 API Key 三种维度
   - 返回 429 + Retry-After 头

5. 创建一个示例脚本 scripts/test-apikey.js：
   - 演示如何用 API Key + HMAC 签名调用 API
   - 包含完整的签名计算过程
```

### 讲师要点
- 现场运行 test-apikey.js 演示 HMAC 签名调用
- 修改签名内容验证"防篡改"特性
- 修改时间戳验证"防重放"特性

---

## Sprint 7：测试补全 + 安全审查（10 分钟）

### Prompt 7.1 — 测试补全

```
为项目编写完整的测试，目标覆盖率 > 80%：

1. src/tests/auth.test.js — 认证测试：
   - 注册：成功、重复用户名、重复邮箱、弱密码、缺少字段
   - 登录：成功、密码错误、用户不存在、账户被禁用
   - Token 刷新：成功、过期 Token、无效 Token
   - 登出：成功、登出后 Token 失效

2. src/tests/rbac.test.js — 权限测试：
   - ADMIN 访问管理接口：成功
   - USER 访问管理接口：403
   - USER 访问自己的资源：成功
   - USER 访问别人的资源：403
   - 未认证访问受保护接口：401

3. src/tests/token.test.js — Token 服务测试：
   - Access Token 签发和验证
   - Refresh Token 签发和验证
   - 过期 Token 验证失败
   - 篡改 Token 验证失败
   - 黑名单 Token 验证失败

4. src/tests/mfa.test.js — MFA 测试：
   - 启用 MFA 流程
   - MFA 登录流程
   - 错误的 TOTP Code
   - 恢复码使用
   - 恢复码一次性校验

5. src/tests/apikey.test.js — API Key 测试：
   - 创建和列出 API Key
   - HMAC 签名验证成功
   - 签名错误被拒绝
   - 时间戳过期被拒绝
   - 限流触发

使用内存 SQLite 替代 PostgreSQL 进行测试，模拟 Redis 使用 Map。
每个测试文件独立可运行，不依赖其他测试的执行顺序。
```

### Prompt 7.2 — 安全审查

```
作为安全工程师，对这个认证中心项目进行全面的安全审查：

1. 代码级安全检查：
   - SQL 注入 / NoSQL 注入
   - XSS
   - CSRF
   - 路径遍历
   - 敏感信息泄露（日志、错误响应、API 响应）
   - 不安全的加密算法使用
   - 硬编码密钥

2. 认证/授权逻辑检查：
   - 认证绕过的可能性
   - 权限提升漏洞
   - Token 安全性
   - OAuth2 安全性

3. 基础设施安全：
   - 依赖包是否有已知漏洞
   - 环境变量是否安全管理
   - Docker 配置是否安全

请给出具体的代码位置和修复建议，按严重程度排列。
```

### 讲师要点
- 运行 `npm run test:coverage` 展示覆盖率报告
- 观察 AI 安全审查是否发现了自己之前生成代码中的问题
- 强调"AI 也需要 Review，尤其是安全相关的代码"

---

## Sprint 8：Docker 化 + 文档（5 分钟）

### Prompt 8.1 — 生产就绪

```
完成项目的生产化准备：

1. Dockerfile（多阶段构建，生产镜像尽可能小）
2. docker-compose.yml 更新（加入应用服务、健康检查、依赖顺序）
3. scripts/seed.js — 种子数据脚本（创建 admin/user/guest 三个测试账号）
4. 完善 README.md：
   - 项目简介
   - 快速开始（Docker Compose 一键启动）
   - API 接口文档（表格形式列出所有接口）
   - 认证方式说明
   - 环境变量说明

5. 创建 CLAUDE.md（项目上下文配置文件，方便后续 AI 编码使用）
```

---

## 培训效果度量

### 现场统计（建议讲师记录）

| 指标 | 数据 |
|------|------|
| Sprint 0-8 总用时 | ___ 分钟 |
| AI 生成的总文件数 | ___ 个 |
| AI 生成的总代码行数 | ___ 行 |
| 测试覆盖率 | __% |
| 发现并修复的安全问题 | ___ 个 |

### 对比估算

> **如果用传统方式开发这个项目，预计需要多少人天？**
>
> 参考：一个中级开发者，同等功能 + 测试 + 文档 → **约 8-12 人天**
>
> AI Native 方式演示用时 → **约 60-90 分钟**

---

## 注意事项

1. **网络要求**：OAuth2 演示需要能访问 GitHub API，提前确认网络
2. **备用方案**：如果 OAuth2 无法现场演示，可跳过 Sprint 4，不影响其他模块
3. **Docker 环境**：提前安装 Docker Desktop，拉取 PostgreSQL 和 Redis 镜像
4. **密钥准备**：演示前生成 RSA 密钥对，避免现场等待
5. **演示节奏**：每个 Sprint 完成后 pause 2 分钟，回答学员问题
