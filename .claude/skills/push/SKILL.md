---
name: push
description: 推送当前分支到远程仓库，推送前自动检查状态
disable-model-invocation: true
argument-hint: "remote and branch (optional, default: origin current-branch)"
---

# Git Push

将当前分支推送到远程仓库，推送前进行安全检查。

## 步骤

1. 运行 `git status` 确认工作区干净（无未提交的变更）
2. 运行 `git log --oneline -5` 查看最近的提交
3. 运行 `git branch -vv` 确认当前分支和远程跟踪信息
4. 如果有未提交的变更，提示用户先提交或暂存（不要自动提交）
5. 如果当前分支没有远程跟踪分支，使用 `git push -u origin <branch>` 设置上游
6. 如果用户提供了 $ARGUMENTS，使用用户指定的 remote 和 branch
7. 执行 `git push`
8. 运行 `git log --oneline -3` 确认推送成功

## 注意事项

- **禁止** `git push --force` 或 `git push --force-with-lease`，除非用户明确要求
- **禁止** 向 main/master 分支 force push
- 如果推送被拒绝（远程有更新），提示用户先 `git pull --rebase`
- 不要自动执行 pull 或 rebase，交由用户决定
