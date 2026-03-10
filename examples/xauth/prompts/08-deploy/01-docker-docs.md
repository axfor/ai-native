# ⚡ Prompt 11 — Docker 化 + 文档

> **Sprint 8** | 预计 AI 执行时间：1-2 分钟 | 可选演示

## 目标

完成项目的生产化准备，包含多阶段 Docker 构建、一键启动的 docker-compose、完整的 README 文档和 CLAUDE.md 项目上下文配置，使项目可以"git clone → docker-compose up → 可用"。

## 最佳实践

| 实践 | 说明 | 参考 |
|------|------|------|
| 多阶段构建 | builder + runner 分离，镜像体积减少 75%+ | Docker Multi-stage Builds |
| 非 Root 用户 | 容器内创建专用用户运行应用，降低逃逸风险 | CIS Docker Benchmark |
| .dockerignore | 排除 node_modules/.git/coverage/keys，加速构建 | Docker Build Context |
| 健康检查 | HEALTHCHECK 指令 + /health 端点，编排器自动重启 | Docker HEALTHCHECK |
| 依赖锁定 | npm ci 而非 npm install，确保确定性构建 | npm ci Best Practice |
| Secret 不入镜像 | 密钥通过环境变量/挂载注入，不 COPY 到镜像中 | Docker Secrets Management |
| CLAUDE.md 上下文 | 为 AI 编码工具提供项目约定，团队新人即插即用 | Claude Code Context |

## 详细需求

### 1. Dockerfile — 多阶段构建

| 阶段 | 基础镜像 | 作用 |
|------|----------|------|
| builder | node:20-alpine | 安装依赖、生成 Prisma Client |
| runner | node:20-alpine | 仅复制必要文件运行 |

**要求**：
- 使用 `.dockerignore` 排除 node_modules、.git、coverage、keys/ 等
- 非 root 用户运行（安全最佳实践）
- 暴露 3000 端口
- 健康检查指令

### 2. 更新 docker-compose.yml

在已有 PostgreSQL + Redis 基础上加入 app 服务：

| 服务 | 配置要求 |
|------|----------|
| app | 依赖 db + redis，等待健康检查后启动 |
| app | 自动运行 prisma migrate deploy |
| app | 环境变量从 .env 读取 |
| app | 端口映射 3000:3000 |

### 3. README.md

| 章节 | 内容 |
|------|------|
| 项目简介 | XAuth 是什么、技术栈 |
| 快速开始 | git clone → docker-compose up 三步启动 |
| 本地开发 | npm install → 配置 .env → npm run dev |
| API 列表 | 所有 API 端点，按模块分组 |
| 环境变量 | 所有变量说明 + 默认值 |
| 测试 | npm test / npm run test:coverage |
| 项目结构 | 目录树 + 各模块说明 |

### 4. CLAUDE.md — AI 编码上下文

| 章节 | 内容 |
|------|------|
| 项目概述 | 技术栈、架构风格 |
| 目录结构 | 关键文件路径 |
| 代码规范 | CommonJS、错误处理方式、响应格式 |
| 运行命令 | dev/test/build/migrate |
| 注意事项 | 安全要求、不可使用的模式 |

## 部署流程

```
=== 一键启动流程 (docker-compose up) ===

  用户              Docker Compose                容器
    |                    |                          |
    | docker-compose     |                          |
    | up --build         |                          |
    |------------------>|                          |
    |                    |                          |
    |                    | 1. 启动 PostgreSQL        |
    |                    |------------------------->| db
    |                    |    等待 healthy           |
    |                    |                          |
    |                    | 2. 启动 Redis             |
    |                    |------------------------->| redis
    |                    |    等待 healthy           |
    |                    |                          |
    |                    | 3. 构建 App 镜像          |
    |                    |   +-- Stage 1: builder   |
    |                    |   |   npm ci              |
    |                    |   |   prisma generate     |
    |                    |   +-- Stage 2: runner     |
    |                    |       复制依赖+代码       |
    |                    |       创建非root用户      |
    |                    |                          |
    |                    | 4. 启动 App               |
    |                    |------------------------->| app
    |                    |    prisma migrate         |
    |                    |    deploy                 |
    |                    |    npm start              |
    |                    |                          |
    | All services up    |                          |
    |<-------------------|                          |
    |                    |                          |
    | curl :3000/health  |                          |
    |------------------->|------------------------->|
    | {status: "ok"}     |                          |
    |<-------------------|<-------------------------|
```

### 多阶段构建对比

```
  [BAD] 单阶段构建                  [GOOD] 多阶段构建

  node:20                           Stage 1: builder
  npm install                       npm ci
  (含 devDeps)                      prisma generate
                                    ---
  镜像大小: ~800MB                  Stage 2: runner
                                    npm ci --omit=dev
                                    COPY --from=builder

                                    镜像大小: ~200MB
```

## Prompt

```
完成生产化：
1. Dockerfile — 多阶段构建
2. 更新 docker-compose.yml — 加入 app 服务
3. README.md — 项目简介、快速开始、API 列表、环境变量说明
4. CLAUDE.md — 项目上下文配置（技术栈、目录结构、代码规范、运行命令）
```

## 验收标准

| # | 验收项 | 验证方式 | 预期结果 |
|---|--------|----------|----------|
| 1 | Docker 构建成功 | `docker-compose build` | 无报错 |
| 2 | 一键启动 | `docker-compose up` | 3 个服务全部启动 |
| 3 | 健康检查通过 | `curl http://localhost:3000/health` | `{status: "ok"}` |
| 4 | 多阶段构建 | 检查 Dockerfile | 包含 builder + runner 两个 FROM |
| 5 | 非 root 运行 | 检查 Dockerfile | 包含 USER 指令 |
| 6 | README 完整 | 检查 README.md | 包含快速开始、API 列表、环境变量 |
| 7 | API 文档齐全 | 检查 README | 所有 /auth /users /mfa /oauth /api-keys 路由列出 |
| 8 | CLAUDE.md 可用 | 检查 CLAUDE.md | 包含技术栈、目录结构、运行命令 |
| 9 | .dockerignore | 检查文件 | 排除 node_modules/.git/coverage/keys |

## 验证

```bash
docker-compose up --build
curl http://localhost:3000/health
```

## 讲师要点

- 检查 Dockerfile 是否使用多阶段构建减小镜像体积
- 检查 README 中 API 文档是否完整
- CLAUDE.md 的价值：后续团队成员用 AI 编码时自动获取项目上下文
