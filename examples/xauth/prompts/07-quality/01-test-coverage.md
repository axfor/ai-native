# ⭐ Prompt 09 — 测试补全

> **Sprint 7** | 预计 AI 执行时间：3-5 分钟

## 目标

为所有核心模块编写完整的单元测试和集成测试，覆盖正常流程和各种边界/异常场景，确保测试覆盖率达到 80% 以上，使用隔离的测试环境（内存数据库 + Mock Redis）。

## 最佳实践

| 实践 | 说明 | 参考 |
|------|------|------|
| 测试金字塔 | 大量单元测试 + 适量集成测试 + 少量 E2E 测试 | Martin Fowler Test Pyramid |
| 测试环境隔离 | 内存数据库 + Mock Redis，不依赖外部服务 | Test Isolation Pattern |
| 覆盖率 > 80% | 行业标准门槛，重点覆盖业务逻辑和边界条件 | Google Engineering Practices |
| 边界条件优先 | 弱密码、过期 Token、越权访问等异常路径必须覆盖 | Boundary Value Analysis |
| 测试独立性 | 每个测试文件可单独运行，不依赖执行顺序 | Test Independence Principle |
| CI 集成 | 测试在 CI 流水线中自动运行，失败阻断合并 | Continuous Integration |

## 详细需求

### 1. 测试基础设施

| 项目 | 要求 |
|------|------|
| 框架 | Jest + Supertest |
| 数据库 | SQLite 内存数据库替代 PostgreSQL（Prisma test provider 或 mock） |
| Redis | Mock 为内存 Map |
| 隔离性 | 每个测试文件独立可运行，互不影响 |
| 清理 | 每个测试前/后清理数据 |

### 2. Auth 模块测试 `tests/auth.test.js`

| 测试场景 | 预期结果 |
|----------|----------|
| 正常注册 | 201，返回 user + tokens |
| 重复用户名注册 | 409，用户名已存在 |
| 重复邮箱注册 | 409，邮箱已存在 |
| 弱密码注册（短于8位） | 400，密码不符合要求 |
| 弱密码注册（无大写） | 400，必须包含大写字母 |
| 弱密码注册（无特殊字符） | 400，必须包含特殊字符 |
| 缺少必填字段 | 400，校验失败 |
| 邮箱格式错误 | 400，邮箱格式错误 |
| 正常登录 | 200，返回 tokens |
| 用户名错误 | 401，用户名或密码错误 |
| 密码错误 | 401，用户名或密码错误 |
| 停用用户登录 | 401/403，账号已停用 |

### 3. Token 模块测试 `tests/token.test.js`

| 测试场景 | 预期结果 |
|----------|----------|
| 签发 Access Token | 包含 userId、role |
| 签发 Refresh Token | 包含 userId、type:'refresh' |
| 验证有效 Token | 返回正确 payload |
| 验证过期 Token | 抛 UnauthorizedError |
| 验证篡改 Token | 抛 UnauthorizedError |
| 验证格式错误 Token | 抛 UnauthorizedError |
| Token 加入黑名单 | isBlacklisted 返回 true |
| 未黑名单 Token | isBlacklisted 返回 false |
| Refresh Token 换新 | 返回新 Token 对 |
| 旧 RT 换新后失效 | 旧 RT 在黑名单 |
| 登出后 Token 失效 | 再次使用返回 401 |

### 4. RBAC 模块测试 `tests/rbac.test.js`

| 测试场景 | 预期结果 |
|----------|----------|
| ADMIN 访问 GET /users | 200，返回用户列表 |
| USER 访问 GET /users | 403 Forbidden |
| GUEST 访问 GET /users | 403 Forbidden |
| ADMIN 查看任意用户 | 200 |
| USER 查看自己 | 200 |
| USER 查看他人 | 403 |
| USER 修改自己 | 200 |
| USER 修改他人 | 403 |
| ADMIN 修改任意用户 | 200 |
| ADMIN 删除用户 | 200 |
| USER 删除用户 | 403 |
| ADMIN 修改角色 | 200 |
| USER 修改角色 | 403 |
| 无 Token 访问保护接口 | 401 |

### 5. 覆盖率要求

| 维度 | 目标 |
|------|------|
| Statements | > 80% |
| Branches | > 75% |
| Functions | > 80% |
| Lines | > 80% |

## 测试执行流程

```
=== 测试执行架构 ===

  npm test
    |
    +-- Jest 启动
    |     +-- 加载 jest.config.js
    |     +-- 设置 NODE_ENV=test
    |     +-- 加载 test setup
    |
    +-- 测试环境初始化
    |     +-- SQLite 内存数据库 (:memory:)
    |     |     +-- Prisma migrate / mock
    |     +-- Redis --> 内存 Map mock
    |     |     +-- get/set/del 方法模拟
    |     +-- RSA 密钥 --> 测试用密钥对
    |
    +-- 测试文件执行（并行）
    |     +-- tests/auth.test.js
    |     |     +-- beforeAll: 创建测试用户
    |     |     +-- describe("注册")
    |     |     |     +-- it("正常注册") ✅
    |     |     |     +-- it("重复用户名") ✅
    |     |     |     +-- it("弱密码") ✅
    |     |     |     +-- ...
    |     |     +-- describe("登录")
    |     |     |     +-- it("正常登录") ✅
    |     |     |     +-- it("错误密码") ✅
    |     |     |     +-- ...
    |     |     +-- afterAll: 清理数据
    |     |
    |     +-- tests/token.test.js
    |     |     +-- describe("签发")
    |     |     +-- describe("验证")
    |     |     +-- describe("黑名单")
    |     |     +-- describe("刷新")
    |     |
    |     +-- tests/rbac.test.js
    |           +-- describe("角色守卫")
    |           +-- describe("所有权守卫")
    |           +-- describe("越权测试")
    |
    +-- 输出
          +-- Test Results: X passed, Y failed
          +-- Coverage Report:
                Statements: 85% ✅
                Branches:   78% ✅
                Functions:  82% ✅
                Lines:      84% ✅
```

### 测试隔离策略

```
  生产环境
  +----------------------------------------------+
  |  +------------+  +------------+              |
  |  | PostgreSQL |  |   Redis    |              |
  |  +------------+  +------------+              |
  +----------------------------------------------+
            | 测试时替换
            v
  测试环境
  +----------------------------------------------+
  |  +------------+  +------------+              |
  |  | SQLite     |  | 内存 Map   |              |
  |  | :memory:   |  |  mock      |              |
  |  +------------+  +------------+              |
  |  每个测试文件独立实例                         |
  |  测试完成自动销毁                             |
  +----------------------------------------------+
```

## Prompt

```
当前测试覆盖率不足，请补全所有测试：

为以下模块编写完整的测试（单元测试 + 集成测试）：
1. auth — 注册/登录/刷新/登出（含各种失败场景）
2. rbac — 角色守卫/资源守卫/越权测试
3. token — 签发/验证/过期/篡改/黑名单

测试配置：
- 使用 SQLite 内存数据库替代 PostgreSQL（配置 Prisma test provider 或 mock）
- Mock Redis 为内存 Map
- 每个文件独立可运行

目标覆盖率 > 80%。完成后运行 npm run test:coverage 验证。
```

## 验收标准

| # | 验收项 | 验证方式 | 预期结果 |
|---|--------|----------|----------|
| 1 | 所有测试通过 | `npm test` | 0 failed |
| 2 | 覆盖率达标 | `npm run test:coverage` | Statements > 80% |
| 3 | Auth 正常流程 | 检查测试用例 | 注册/登录/刷新/登出均覆盖 |
| 4 | Auth 异常流程 | 检查测试用例 | 弱密码/重复用户/错误密码均覆盖 |
| 5 | Token 边界 | 检查测试用例 | 过期/篡改/黑名单均覆盖 |
| 6 | RBAC 越权 | 检查测试用例 | 3 种角色 × 5 个接口均覆盖 |
| 7 | 测试隔离 | 单独运行任一测试文件 | 独立通过，不依赖其他测试 |
| 8 | 无外部依赖 | 不启动 PG/Redis 运行测试 | 全部通过 |

## 验证

```bash
npm test
npm run test:coverage
```

## 讲师要点

- 运行 `npm run test:coverage` 展示覆盖率报告
- 注意 AI 是否覆盖了边界场景（弱密码、过期 Token、越权访问等）
- 强调"AI 写测试比人快 10 倍，且不会偷懒跳过边界条件"
