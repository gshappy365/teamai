---
title: "TeamAI contribute 网络失败后的重试处理"
author: gshappy365
date: 2026-09-15
tags: [teamai, troubleshooting, workflow, git]
---

## 背景

使用 `teamai contribute` 将 Session 经验写入团队知识库时，第一次推送可能因为网络、代理或 Git 认证失败。需要确认失败后本地仓库到底处于什么状态，避免重试时产生重复知识条目或把无关提交一起推送。

## 解决方案

1. 正式提交前先检查当前分支和未推送提交：

   ```bash
   git -C ~/.teamai/team-repo status --short --branch
   git -C ~/.teamai/team-repo log --oneline origin/main..main
   ```

2. 先使用全局 dry-run 预览目标路径：

   ```bash
   teamai --dry-run contribute --file /tmp/session-summary.md --title "经验标题" --scope user
   ```

3. 网络失败后重新检查 Git 状态。`teamai contribute` 可能已经在本地完成了 commit，只是在 push 阶段失败；再次执行时会重新生成随机文件名，可能产生重复文档。

4. 网络恢复后重试前，确认已有本地 commit 是否属于本次授权范围，并检查工作树中是否有无关文件。成功推送后确认：

   ```bash
   git -C ~/.teamai/team-repo status --short --branch
   ```

## 经验总结

- `--dry-run` 只预览目标文件名和路径，不验证远程网络是否可用。
- 推送失败不一定代表本地没有 commit；必须同时检查 `git log origin/main..main`。
- 普通远程仓库模式下，TeamAI 沿用当前分支推送，已有的 ahead commits 可能在重试时一并上传。
- 贡献文档应保持简短、中文、可复用，并包含 YAML frontmatter；不要把访问令牌、密码或个人敏感信息写入文档。
- Git remote 建议使用 SSH 或 credential helper，避免在 remote URL 中直接保存访问令牌。

## 相关 Skills

- teamai-share-learnings
