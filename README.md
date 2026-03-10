# AI Native Development Training

> **从"写代码"到"指挥 AI 写代码" —— 面向未来的开发范式转型培训**

---

> **清醒认识：AI 正在取代开发者——Block 裁员 40%，Klarna 裁员 47%，初级岗位持续萎缩。**
> **唯一的出路是转型：从「写代码的人」变成「驾驭 AI 写代码的人」。**
>
> 未来的工程师不是被 AI 辅助的程序员，而是指挥 AI 代理军团的架构师。
> 掌握 Spec-Driven Development + AI Skills + 并行代理工作流，让自己成为不可替代的那个人。

**📖 核心教材：[AI-Native-Development-Training.md](AI-Native-Development-Training.md)**（10 章完整培训内容，含行业案例、工具指南、最佳实践和 90 天落地路线图）

## Overview

本项目是一套部门级 AI Native 开发转型培训教材，包含完整的培训文档、可执行的示例项目和 Prompt 工程模板。目标是帮助开发团队系统性地从传统开发模式转型为 AI Native 开发模式。

## Project Structure

```
.
├── AI-Native-Development-Training.md   # 核心培训教材（10 章）
├── examples/
│   └── xauth/                          # 实战示例：认证授权系统
│       ├── docs/                        # 需求、架构、评审、开发计划
│       └── prompts/                     # 8 阶段 Prompt 链
├── images/                             # SVG 图表资源（11 张）
├── prompts/                            # 项目级 Prompt 模板
└── .claude/skills/                     # Claude Code 自定义命令
    ├── ai-native-training/             # 项目规范（自动加载）
    ├── commit/                         # /commit 无签名提交
    └── push/                           # /push 安全推送
```

## Training Content

| Chapter | Topic |
|---------|-------|
| 1 | 为什么要转型 AI Native |
| 2 | 行业裁员警示：不转型的代价 |
| 3 | 行业趋势与成熟案例 |
| 4 | AI Coding 工具全景图 |
| 5 | 开发模式演进路线 |
| 6 | 核心工具实战指南（Claude Code / GitHub Copilot / Cursor） |
| 7 | 实战示范：五大场景 Before/After |
| 8 | AI Native 工作流最佳实践（含 Spec-Driven Development） |
| 9 | 常见挑战与应对策略 |
| 10 | 团队落地路线图（90 天） |

## Example Project: xAuth

一个完整的认证授权系统，展示 AI Native 开发的全流程：

- **需求分析** → 架构设计 → 架构评审 → 开发计划
- **8 阶段 Prompt 链**：项目脚手架 → JWT 服务 → 注册登录 → Token 刷新 → RBAC → GitHub OAuth → TOTP MFA → HMAC API Key → 测试覆盖 → 安全审计 → Docker 部署

每个 Prompt 都是可直接喂给 AI Coding 工具执行的完整指令。

## Quick Start

1. 阅读 [AI-Native-Development-Training.md](AI-Native-Development-Training.md) 培训教材
2. 浏览 [examples/xauth/prompts/](examples/xauth/prompts/) 了解 Prompt 链设计
3. 使用 Claude Code 打开本项目，体验 `/commit` `/push` 等自定义命令

## License

MIT
