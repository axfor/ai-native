# Prompt 目录说明

按培训演示顺序，逐个目录执行。⭐ 为必须演示，⚡ 可根据时间跳过。

每个 Prompt 文件包含：**目标** → **详细需求** → **流程时序图** → **Prompt** → **验收标准** → **验证命令** → **讲师要点**。

```
prompts/
├── 01-init/                    Sprint 0: 项目初始化
│   └── ⭐ 01-project-scaffold     创建项目骨架 + 数据库模型
│       ├─ 数据模型 ER 图（User/OAuthAccount/ApiKey/AuditLog/RecoveryCode）
│       ├─ 初始化流程时序图（讲师 → Claude Code → Docker/DB）
│       └─ 7 项验收标准
│
├── 02-core-auth/               Sprint 1-2: 核心认证
│   ├── ⭐ 01-jwt-token-service    JWT RS256 Token 服务
│   │   ├─ JWT 签发与验证时序图
│   │   ├─ RS256 vs HS256 对比图
│   │   └─ 8 项验收标准
│   ├── ⭐ 02-register-login       用户注册 + 登录
│   │   ├─ 注册/登录时序图（Client → Validate → Auth → DB）
│   │   ├─ 密码校验失败场景矩阵（5 种规则 × 错误示例）
│   │   └─ 9 项验收标准
│   └── ⭐ 03-token-refresh-logout Token 刷新 + 登出 + 黑名单
│       ├─ Token 完整生命周期 5 阶段时序图（登录→使用→刷新→登出→失效）
│       ├─ Redis 降级策略决策图
│       └─ 9 项验收标准
│
├── 03-authorization/           Sprint 3: 权限控制
│   └── ⭐ 01-rbac                 角色守卫 + 资源守卫 + 审计日志
│       ├─ 角色守卫时序图（Auth MW → RBAC MW → Controller）
│       ├─ 资源所有权守卫时序图（含 ADMIN 跳过逻辑）
│       ├─ 权限矩阵（3 角色 × 5 接口）
│       ├─ 审计日志写入流程
│       └─ 12 项验收标准
│
├── 04-oauth/                   Sprint 4: 社交登录
│   └── ⚡ 01-github-oauth         GitHub OAuth2 + 账号绑定
│       ├─ OAuth2 Authorization Code Flow 完整时序图
│       ├─ 账号绑定决策树（查绑定→查邮箱→新建）
│       ├─ 解绑安全检查逻辑
│       └─ 10 项验收标准
│
├── 05-mfa/                     Sprint 5: 多因素认证
│   └── ⚡ 01-totp                 TOTP + 恢复码 + QR Code
│       ├─ MFA 启用流程时序图（用户 → Server → Google Authenticator）
│       ├─ MFA 两步登录时序图（密码验证 → TOTP/恢复码验证）
│       ├─ TOTP 时间窗口容差示意图
│       └─ 11 项验收标准
│
├── 06-apikey/                  Sprint 6: API Key
│   └── ⚡ 01-hmac-auth            API Key + HMAC 签名 + 限流
│       ├─ API Key 创建流程时序图
│       ├─ HMAC 签名请求验证时序图
│       ├─ 防篡改 & 防重放原理示意（3 种攻击场景）
│       ├─ Redis 滑动窗口限流示意图
│       └─ 10 项验收标准
│
├── 07-quality/                 Sprint 7: 质量保障
│   ├── ⭐ 01-test-coverage        测试补全 → 覆盖率 80%+
│   │   ├─ 测试执行架构图（Jest → 测试文件 → 覆盖率报告）
│   │   ├─ 测试隔离策略图（SQLite :memory: + 内存 Map）
│   │   ├─ 详细测试用例矩阵（Auth 12 + Token 11 + RBAC 14 场景）
│   │   └─ 8 项验收标准
│   └── ⭐ 02-security-audit       安全审查 + 自动修复
│       ├─ OWASP Top 10 检查矩阵
│       ├─ 5 阶段审查流程（扫描→配置→依赖→报告→修复）
│       ├─ 常见安全问题 Before/After 对比
│       └─ 8 项验收标准
│
└── 08-deploy/                  Sprint 8: 部署就绪
    └── ⚡ 01-docker-docs          Docker + README + CLAUDE.md
        ├─ 一键启动流程时序图（docker-compose up 全过程）
        ├─ 多阶段构建对比图（800MB → 200MB）
        └─ 9 项验收标准
```

## 最小演示路线（40 分钟）

只执行 ⭐ 标记的 Prompt：01 → 02 → 03 → 04 → 05 → 09 → 10

```
01-project-scaffold → 02-jwt-token → 03-register-login → 04-token-refresh
       ↓                                                        ↓
  项目骨架就绪                                              核心认证完成
                                                               ↓
                    10-security-audit ← 09-test-coverage ← 05-rbac
                           ↓                                    ↓
                      安全加固完成                          权限控制就绪
```

## 完整演示路线（90 分钟）

全部执行：01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 09 → 10 → 11

```
01-scaffold → 02-jwt → 03-register → 04-token → 05-rbac
     │                                              │
 Sprint 0                                       Sprint 3
                                                    │
     11-docker ← 10-audit ← 09-test ← 08-hmac ← 07-totp ← 06-oauth
         │                                                      │
     Sprint 8                                              Sprint 4
```

## 每个 Prompt 文件结构

| 章节 | 说明 |
|------|------|
| 目标 | 本 Sprint 要达成的目标（一句话） |
| 详细需求 | 方法签名、输入输出、技术参数、安全约束（表格化） |
| 流程时序图 | ASCII 时序图，展示组件间交互 |
| Prompt | 直接复制给 AI 执行的指令 |
| 验收标准 | 编号表格：验收项 + 验证方式 + 预期结果 |
| 验证命令 | curl / node 命令，可直接运行 |
| 讲师要点 | 演示时需要强调的关键点 |
