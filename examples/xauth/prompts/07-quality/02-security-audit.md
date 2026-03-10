# ⭐ Prompt 10 — 安全审查 + 修复

> **Sprint 7** | 预计 AI 执行时间：2-3 分钟

## 目标

对整个项目进行全面的安全审查，覆盖 OWASP Top 10、认证逻辑缺陷、敏感信息泄露、密码学使用和依赖安全，发现问题后直接修复，确保项目达到生产级安全标准。

## 最佳实践

| 实践 | 说明 | 参考 |
|------|------|------|
| OWASP Top 10 覆盖 | 逐项检查注入、认证失败、访问控制等十大风险 | OWASP Top 10 2021 |
| 依赖安全扫描 | npm audit 检查已知漏洞，定期更新依赖 | OWASP Dependency Check |
| 安全头配置 | helmet 中间件设置 CSP/HSTS/X-Frame-Options 等 | OWASP Secure Headers |
| 错误信息不泄露 | 生产环境隐藏堆栈跟踪和内部错误细节 | CWE-209 Information Exposure |
| 密码学正确使用 | bcrypt ≥12轮、RSA ≥2048位、crypto.randomBytes | OWASP Cryptographic Failures |
| 登录暴力破解防护 | 限速 + 账号锁定 + 验证码阶梯式防护 | OWASP Brute Force Prevention |
| 修复后回归测试 | 安全修复后运行全量测试，确保无功能回归 | Security Fix Verification |

## 详细需求

### 审查维度与检查项

#### 1. OWASP Top 10 漏洞

| 漏洞类型 | 检查项 | 关注点 |
|----------|--------|--------|
| A01 访问控制 | RBAC 中间件逻辑 | 是否有绕过路径 |
| A02 加密失败 | bcrypt/AES/RSA 使用 | 轮数、密钥长度、算法选择 |
| A03 注入 | SQL/NoSQL 注入 | Prisma 参数化查询是否一致 |
| A04 不安全设计 | 业务逻辑缺陷 | 竞态条件、绕过场景 |
| A05 安全配置 | .env / CORS / Headers | 是否有生产不安全的默认值 |
| A07 认证失败 | 登录/Token/MFA | 暴力破解防护、会话管理 |
| A08 数据完整性 | 依赖安全 | npm audit、已知漏洞 |
| A09 日志监控 | 审计日志 | 是否记录关键安全事件 |

#### 2. 认证逻辑缺陷

| 检查项 | 说明 |
|--------|------|
| 密码校验绕过 | 是否所有登录路径都经过密码校验 |
| Token 伪造 | RS256 验签是否严格 |
| 提权漏洞 | 用户能否修改自己的 role |
| MFA 绕过 | 是否能跳过 MFA 验证直接获取 JWT |
| Refresh Token 泄露 | RT 被盗后的影响范围 |

#### 3. 敏感信息泄露

| 检查位置 | 不应泄露的信息 |
|----------|----------------|
| API 响应 | passwordHash、mfaSecret、apiSecret |
| 错误信息 | SQL 语句、堆栈跟踪（生产环境） |
| 日志 | 密码明文、Token 全文、Secret Key |
| HTTP 头 | X-Powered-By、Server 版本 |

#### 4. 密码学审查

| 检查项 | 正确做法 |
|--------|----------|
| bcrypt 轮数 | ≥ 12 轮 |
| RSA 密钥长度 | ≥ 2048 位 |
| AES 模式 | GCM（带认证标签），非 ECB/CBC |
| HMAC 算法 | SHA-256 |
| 随机数生成 | crypto.randomBytes，非 Math.random |
| 时间安全比较 | crypto.timingSafeEqual |

#### 5. 依赖安全

| 检查项 | 工具 |
|--------|------|
| 已知漏洞 | `npm audit` |
| 过时包 | `npm outdated` |
| 许可证风险 | 检查 GPL 等传染性许可证 |

### 输出格式要求

每个发现的问题需包含：

```
[严重程度] 问题标题
位置：具体文件路径 + 行号
描述：问题的详细说明
风险：可能被利用的攻击方式
修复：具体的修复方案 + 代码变更
```

严重程度分级：🔴 Critical | 🟠 High | 🟡 Medium | 🔵 Low | ⚪ Info

## 安全审查流程

```
=== 安全审查执行流程 ===

AI (Claude Code)
  |
  +-- Phase 1: 代码扫描
  |     +-- 遍历 src/ 所有文件
  |     +-- 检查认证/授权逻辑
  |     +-- 检查输入校验
  |     +-- 检查错误处理
  |     +-- 检查密码学使用
  |
  +-- Phase 2: 配置审查
  |     +-- 检查 .env.example 默认值
  |     +-- 检查 CORS 配置
  |     +-- 检查 Express 安全头
  |     +-- 检查 Prisma 配置
  |
  +-- Phase 3: 依赖审计
  |     +-- npm audit
  |     +-- 检查 package.json 版本
  |     +-- 检查锁文件
  |
  +-- Phase 4: 输出报告
  |     +-- 问题列表（按严重程度排序）
  |     +-- 每个问题的位置+修复方案
  |     +-- 整体安全评分
  |
  +-- Phase 5: 自动修复
        +-- 直接修改代码
        +-- 添加安全头 (helmet)
        +-- 修复发现的漏洞
        +-- 运行测试确认无回归

=== 常见安全问题示意 ===

  [BAD] 问题代码                        [GOOD] 修复后

  // 时间不安全比较                     // 时间安全比较
  if (sig === expected)                 if (crypto.timingSafeEqual(
    --> 时间侧信道攻击                      Buffer.from(sig),
                                            Buffer.from(expected)))

  // Math.random 生成 token             // 安全随机数
  Math.random().toString(36)            crypto.randomBytes(32)
    --> 可预测                              .toString('hex')

  // 错误信息泄露                       // 通用错误信息
  res.json({error: err.stack})          res.json({error: "Internal
    --> 暴露内部实现                        server error"})

  // 缺少速率限制                       // 登录限速
  POST /auth/login                      rateLimit({
    --> 暴力破解                            windowMs: 15*60*1000,
                                            max: 5 })

  // 缺少安全头                         // helmet 中间件
  app.use(express.json())               app.use(helmet())
    --> XSS/clickjacking                app.use(express.json())
```

## Prompt

```
对这个项目进行全面安全审查。

检查维度：
1. OWASP Top 10 漏洞
2. 认证逻辑缺陷（绕过、提权）
3. 敏感信息泄露（日志、响应、错误信息）
4. 密码学使用是否正确
5. 依赖包安全

给出具体代码位置 + 修复方案 + 严重程度。
发现问题后直接修复。
```

## 验收标准

| # | 验收项 | 验证方式 | 预期结果 |
|---|--------|----------|----------|
| 1 | 生成审查报告 | AI 输出 | 包含问题列表、严重程度、代码位置 |
| 2 | 问题已修复 | 检查代码变更 | 发现的问题已在代码中修复 |
| 3 | 安全头已添加 | 检查 app.js | 使用 helmet 或手动设置安全头 |
| 4 | 登录限速 | 快速尝试 6 次错误密码 | 第 6 次被限速 |
| 5 | X-Powered-By 移除 | 检查响应头 | 无 X-Powered-By 头 |
| 6 | 无 npm audit 高危 | `npm audit` | 无 high/critical 漏洞 |
| 7 | 错误不泄露堆栈 | 生产模式请求错误路由 | 无 stack trace |
| 8 | 测试仍通过 | `npm test` | 修复后无回归 |

## 讲师要点

- 观察 AI 是否发现了自己之前生成代码中的不足
- 引出讨论：为什么 AI 写的代码仍然需要人工 Review？
- 总结：AI 是优秀的初级开发者，但安全需要资深工程师把关
