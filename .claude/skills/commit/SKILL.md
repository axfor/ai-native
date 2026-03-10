---
name: commit
description: 生成 git commit，不包含 Co-Authored-By 签名
disable-model-invocation: true
allowed-tools: Bash(git *)
argument-hint: [commit message (optional)]
---

# Git Commit（无签名）

执行 git commit，**不添加** Co-Authored-By 签名行。

## 步骤

1. 运行 `git status` 查看当前变更
2. 运行 `git diff --staged` 查看已暂存的变更
3. 如果没有暂存的变更，将相关修改文件逐个 `git add`（不要用 `git add -A` 或 `git add .`）
4. 分析变更内容，判断类型：feat / fix / docs / refactor / test / chore
5. 生成简洁的 Conventional Commit 格式提交信息：
   - 主题行：`<type>(<scope>): <描述>`（不超过 70 字符）
   - 空行
   - 正文：解释「为什么」而非「做了什么」（如需要）
6. 如果用户提供了 $ARGUMENTS，使用用户提供的信息作为提交信息基础
7. 执行 `git commit`，**不要**在提交信息中添加任何 Co-Authored-By 行
8. 运行 `git status` 确认提交成功

## 提交信息格式

```
<type>(<scope>): <简短描述>

<详细说明（可选）>
```

## 注意事项

- 不要添加 `Co-Authored-By` 签名
- 不要使用 `--no-verify` 跳过 hooks
- 不要使用 `git add -A` 或 `git add .`，逐个添加文件
- 不要提交 .env、credentials 等敏感文件
- 如果 pre-commit hook 失败，修复问题后创建新的 commit（不要 --amend）
