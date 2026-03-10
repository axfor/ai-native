---
name: ai-native-training
description: AI Native 开发转型培训教材项目的开发规范和上下文。编辑培训文档、Prompt 文件、示例项目时自动加载。适用于修改 AI-Native-Development-Training.md、examples/xauth/ 目录、images/ 目录中的内容。
user-invocable: false
---

# AI Native 开发转型培训教材 — 项目规范

## 项目概述

部门级 AI Native 转型培训教材项目，核心产出：
- 主教材：`AI-Native-Development-Training.md`（10 章节，含行业数据、工具指南、实战场景、SDD 最佳实践）
- 实战演示项目：`examples/xauth/`（统一认证中心，11 个可执行 Prompt，覆盖 JWT/OAuth2/MFA/HMAC/RBAC）
- SVG 配图：`images/`（11 张信息图）

## 项目结构

```
AINative/
  AI-Native-Development-Training.md    # 主教材（唯一入口）
  images/                              # SVG 配图（主教材引用）
    01-ai-coding-key-stats.svg
    02-ai-layoffs-global-data.svg
    ...
  examples/
    xauth/                             # 实战演示项目
      docs/                            # 需求、架构、计划文档
        01-requirements.md
        02-architecture.md
        03-architecture-review.md
        04-development-plan.md
      prompts/                         # 11 个可执行 Prompt
        README.md                      # Prompt 目录说明
        01-init/
        02-core-auth/
        03-authorization/
        04-oauth/
        05-mfa/
        06-apikey/
        07-quality/
        08-deploy/
  prompts/                             # 项目级 Prompt（obj.md, ops.md）
  .claude/skills/                      # Claude Code Skills
```

## 文档编写规范

### 主教材 AI-Native-Development-Training.md

- 章节结构：目录 → 10 个编号章节 → 附录
- 图片引用格式：`![描述](images/xx-name.svg)`
- 图片文件独立存放在 `images/` 目录，不内嵌
- 数据必须标注来源（公司名 + 年份 + 链接）
- 行业案例必须基于真实已发布的案例，不得虚构
- 保持中文撰写，技术术语保留英文原文（如 Spec-Driven Development）
- 结尾引用风格：直面现实，不粉饰（参考裁员数据）

### Prompt 文件（examples/xauth/prompts/）

每个 Prompt 文件遵循统一结构：

```
## 目标
## 最佳实践（业界标准引用表格：实践 | 说明 | 参考）
## 前置条件（如有）
## 详细需求（表格形式）
## 流程时序图（ASCII，见下方图表规范）
## API 示例（curl + JSON 请求响应）
## Prompt（可直接粘贴给 AI 执行的文本块）
## 验收标准（编号表格：# | 验收项 | 验证方式 | 预期结果）
## 验证（bash 脚本）
## 讲师要点
```

- 核心 Prompt 标记 ⭐，可选 Prompt 标记 ⚡
- Sprint 编号与 `04-development-plan.md` 对齐

### ASCII 图表规范

- 使用纯 ASCII 字符：`|`, `-`, `+`, `>`, `<`, `^`, `v`
- 禁止使用 Unicode Box-Drawing 字符（┌┐└┘├┤─│►◄）
- 原因：中文字符占 2 列宽，Unicode 框线与中文混排会错位
- 图表中的中文文字后用空格补齐对齐

### 最佳实践章节规范

- 每个功能 Prompt 必须包含「最佳实践」表格
- 表格列：实践 | 说明 | 参考
- 参考须标注具体标准（OWASP, NIST, RFC, 12-Factor 等）
- 通常 6-7 行

### API 示例规范

- 使用 curl 命令作为请求示例
- 包含成功和失败两种响应
- JSON 响应统一格式：`{ "success": true/false, "data": {} }` 或 `{ "success": false, "error": { "code": "...", "message": "..." } }`

## 技术栈（xauth 演示项目）

- 运行时：Node.js + Express
- 数据库：Prisma + PostgreSQL
- 缓存：Redis（含降级策略）
- 认证：JWT RS256（非对称签名）、Passport.js
- 安全：bcrypt (12 rounds)、AES-256-GCM、HMAC-SHA256
- 测试：Jest + Supertest
- 部署：Docker + docker-compose

## 培训结构

- 8 个 Sprint（Sprint 0-8）
- 11 个可执行 Prompt
- 最小演示路线：40 分钟（5 个核心 Prompt）
- 完整演示路线：90 分钟（11 个 Prompt）
- 演示预期效果：60-90 分钟完成约 8-12 人天工作量

## 行为约束

Always（总是做）：
  - 数据标注来源和链接
  - ASCII 图表用纯 ASCII 字符
  - 保持 Prompt 文件结构一致性
  - 修改主教材后检查目录编号是否对齐
  - 新增行业案例需验证真实性

Ask（先确认）：
  - 调整章节编号或顺序
  - 删除已有内容
  - 修改 xauth 技术栈选型

Never（绝不做）：
  - 虚构行业数据或案例
  - 在 ASCII 图中使用 Unicode Box-Drawing 字符
  - 在 Prompt 文件中写实现代码（这是 AI 现场生成的）
  - 修改 SVG 图片文件（需要单独工具处理）
