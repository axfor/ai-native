---
name: commit-push
description: 提交并推送 —— 依次执行 /commit 和 /push
disable-model-invocation: true
argument-hint: "commit message (optional)"
context:
  - .claude/skills/commit/SKILL.md
  - .claude/skills/push/SKILL.md
---

# Git Commit & Push

一键完成提交和推送，依次执行 `/commit` 和 `/push` 两个流程。

## 步骤

### Phase 1: Commit（参照 /commit 规范）

1. 运行 `git status` 查看当前变更
2. 运行 `git diff --staged` 查看已暂存的变更
3. 如果没有暂存的变更，将相关修改文件逐个 `git add`（不要用 `git add -A` 或 `git add .`）
4. 分析变更内容，判断类型：feat / fix / docs / refactor / test / chore
5. 生成 Conventional Commit 格式提交信息（**必须使用英文**）
6. 如果用户提供了 $ARGUMENTS，使用用户提供的信息作为提交信息基础
7. 执行 `git commit`，**不添加** Co-Authored-By 签名
8. 运行 `git status` 确认提交成功

### Phase 2: Push（参照 /push 规范）

9. 运行 `git branch -vv` 确认远程跟踪信息
10. 如果没有远程跟踪分支，使用 `git push -u origin <branch>`
11. 执行 `git push`
12. 运行 `git log --oneline -3` 确认推送成功

## 注意事项

- 如果 Phase 1 提交失败，**不要**继续 Phase 2
- 遵守 `/commit` 和 `/push` 各自的所有约束规则
