# ⭐ Prompt 01 — 项目初始化

> **Sprint 0** | 预计 AI 执行时间：2-3 分钟

## 目标

从零搭建 XAuth 项目骨架，包含完整的工程化配置、基础设施（PostgreSQL + Redis）、数据库模型和统一的错误处理体系，为后续所有认证模块提供基座。

## 最佳实践

| 实践 | 说明 | 参考 |
|------|------|------|
| 环境变量分离 | .env 不入库，.env.example 提供模板，敏感值留空 | 12-Factor App Config |
| 基础设施即代码 | docker-compose 定义所有依赖，一键可复现 | 12-Factor App Dev/Prod Parity |
| 健康检查端点 | /health 返回服务状态，供负载均衡器和容器编排探活 | Kubernetes Liveness Probe |
| 统一错误处理 | 全局错误中间件捕获异常，开发/生产区分错误详情 | Express.js Error Handling |
| 数据库迁移管理 | 使用 Prisma Migrate 版本化 schema 变更，禁止手动改表 | Prisma Best Practices |
| 最小权限原则 | 数据库用户仅授予应用所需权限，不用 superuser | OWASP Secure Configuration |

## 详细需求

### 1. 工程配置

| 配置项 | 要求 |
|--------|------|
| package.json | 包含所有依赖 + 6 个 scripts（start/dev/test/test:coverage/db:migrate/db:seed） |
| .env.example | 所有环境变量，含注释说明用途和默认值 |
| .gitignore | node_modules、.env、keys/、coverage/ |
| 模块系统 | CommonJS（require/module.exports），不用 TypeScript |

### 2. 基础设施

| 服务 | 版本 | 配置要求 |
|------|------|----------|
| PostgreSQL | 16 | 含健康检查、volume 持久化、默认数据库 xauth |
| Redis | 7 | 含健康检查、volume 持久化 |

### 3. 应用入口 `src/app.js`

- Express 实例，JSON body 解析（限制 10kb）
- CORS 跨域支持
- 请求日志（morgan 或手写）
- `GET /health` 健康检查路由，返回 `{ status: 'ok', timestamp }`
- 统一错误处理中间件挂载

### 4. 配置模块 `src/config/index.js`

- 从 `.env` 读取所有配置
- 分组导出：server、database、redis、jwt、oauth、mfa

### 5. 错误处理体系

**自定义错误类** `src/utils/errors.js`：

| 错误类 | HTTP 状态码 | 用途 |
|--------|------------|------|
| AppError | 自定义 | 基础错误类 |
| NotFoundError | 404 | 资源不存在 |
| ValidationError | 400 | 参数校验失败 |
| UnauthorizedError | 401 | 未认证 / Token 无效 |
| ForbiddenError | 403 | 权限不足 |

**错误中间件** `src/middlewares/error-handler.js`：
- 开发环境：返回完整错误栈
- 生产环境：隐藏内部错误细节
- 统一格式：`{ success: false, error: { message, code } }`

### 6. 数据模型 `prisma/schema.prisma`

```
+-----------------------------------------------------------+
|                          User                             |
+-----------------------------------------------------------+
| id            String   @id @default(uuid)                 |
| username      String   @unique                            |
| email         String   @unique                            |
| passwordHash  String                                      |
| role          Role     @default(USER)                     |
| isActive      Boolean  @default(true)                     |
| mfaEnabled    Boolean  @default(false)                    |
| mfaSecret     String?                                     |
| createdAt     DateTime @default(now())                    |
| updatedAt     DateTime @updatedAt                         |
+-----------------------------------------------------------+
| oauthAccounts  OAuthAccount[]                             |
| apiKeys        ApiKey[]                                   |
| auditLogs      AuditLog[]                                 |
| recoveryCodes  RecoveryCode[]                             |
+-----------------------------------------------------------+

+-----------------------------------------------------------+
|                      OAuthAccount                         |
+-----------------------------------------------------------+
| id            String   @id @default(uuid)                 |
| provider      String   (github / google)                  |
| providerId    String                                      |
| accessToken   String?                                     |
| userId        String   -> User                            |
| @@unique([provider, providerId])                          |
+-----------------------------------------------------------+

+-----------------------------------------------------------+
|                         ApiKey                            |
+-----------------------------------------------------------+
| id            String   @id @default(uuid)                 |
| name          String                                      |
| keyHash       String   @unique  (只存哈希)              |
| keyPrefix     String   (ak_xxx 前 8 位，便于识别)     |
| permissions   String[] (read / write / admin)             |
| isActive      Boolean  @default(true)                     |
| lastUsedAt    DateTime?                                   |
| userId        String   -> User                            |
+-----------------------------------------------------------+

+-----------------------------------------------------------+
|                        AuditLog                           |
+-----------------------------------------------------------+
| id          String   @id @default(uuid)                   |
| action      String   (LOGIN / LOGOUT / REGISTER / ...)   |
| userId      String?  -> User                              |
| ip          String?                                       |
| userAgent   String?                                       |
| details     Json?                                         |
| createdAt   DateTime @default(now())                      |
+-----------------------------------------------------------+

+-----------------------------------------------------------+
|                      RecoveryCode                         |
+-----------------------------------------------------------+
| id          String   @id @default(uuid)                   |
| codeHash    String   (bcrypt 存储)                      |
| isUsed      Boolean  @default(false)                      |
| userId      String   -> User                              |
+-----------------------------------------------------------+

enum Role { USER  ADMIN  GUEST }
```

## 初始化流程

```
  Instructor            Claude Code               Docker / DB
      |                      |                          |
      | 1. provide docs      |                          |
      |--------------------->|                          |
      |                      | 2. generate all files    |
      |                      |------+                   |
      |                      |<-----+                   |
      |                      | 3. docker-compose up     |
      |                      |------------------------->|
      |                      |      PG + Redis ready    |
      |                      |<-------------------------|
      |                      | 4. prisma migrate dev    |
      |                      |------------------------->|
      |                      |      tables created      |
      |                      |<-------------------------|
      |                      | 5. npm run dev           |
      |                      |------+                   |
      | 6. return success    |<-----+                   |
      |<---------------------|                          |
      |                      |                          |
```

## 前置操作

```bash
mkdir xauth && cd xauth

# 将需求和架构文档复制到项目根目录
cp ../docs/01-requirements.md .
cp ../docs/02-architecture.md .

# 启动 Claude Code
claude
```

## API 示例

### GET /health

Request:
```bash
curl -s http://localhost:3000/health | jq
```

Response (200):
```json
{
  "status": "ok",
  "timestamp": "2024-01-15T08:30:00.123Z"
}
```

### GET /not-exist (404 错误处理)

Request:
```bash
curl -s http://localhost:3000/not-exist | jq
```

Response (404):
```json
{
  "success": false,
  "error": {
    "message": "Resource not found",
    "code": "NOT_FOUND"
  }
}
```

## Prompt

```
请阅读 01-requirements.md（需求文档）和 02-architecture.md（架构文档），
理解项目的完整需求和架构设计。

然后初始化项目：
1. package.json — 所有依赖 + scripts（start/dev/test/test:coverage/db:migrate/db:seed）
2. .env.example — 所有环境变量，含注释
3. .gitignore
4. docker-compose.yml — PostgreSQL 16 + Redis 7，含健康检查和 volume 持久化
5. src/app.js — Express 入口，含 JSON 解析、CORS、请求日志、健康检查路由
6. src/config/index.js — 统一配置，从 .env 读取
7. src/utils/errors.js — 自定义错误类（AppError/NotFound/Validation/Unauthorized/Forbidden）
8. src/middlewares/error-handler.js — 统一错误处理中间件，区分开发/生产环境的错误详情
9. prisma/schema.prisma — 按架构文档中的数据模型定义

使用 CommonJS 模块。不用 TypeScript。

创建完成后，复制 .env.example 为 .env，启动 docker-compose，
运行 prisma migrate dev 初始化数据库。
```

## 验收标准

| # | 验收项 | 验证方式 | 预期结果 |
|---|--------|----------|----------|
| 1 | Docker 服务正常 | `docker-compose ps` | PG + Redis 状态 healthy |
| 2 | 数据库表已创建 | `npx prisma studio` | 可浏览 User/OAuthAccount/ApiKey/AuditLog/RecoveryCode 表 |
| 3 | 应用可启动 | `npm run dev` | 无报错，监听 3000 端口 |
| 4 | 健康检查可用 | `curl http://localhost:3000/health` | `{ status: 'ok', timestamp: '...' }` |
| 5 | 错误处理正常 | `curl http://localhost:3000/not-exist` | 404 JSON 响应，格式正确 |
| 6 | 环境变量完整 | 检查 .env.example | 包含 DB/Redis/JWT/OAuth/MFA 所有变量 |
| 7 | 密钥目录被忽略 | 检查 .gitignore | 包含 keys/、.env、node_modules/ |

## 讲师要点

- 强调 AI 在 30 秒内生成了完整的项目骨架
- 传统方式：查文档 + 复制粘贴 + 调配置，至少 1-2 小时
- 注意观察 AI 是否正确理解了架构文档中的数据模型
