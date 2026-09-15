---
title: "使用 TeamAI 将 Session 经验沉淀为团队知识"
author: gshappy365
date: 2026-09-15
tags: [teamai, workflow, knowledge, tool-usage]
---

## 背景

本次 Session 需要确认 TeamAI 如何把 AI 编码过程中的经验变成团队可检索的知识。重点是区分 `teamai contribute`、自动经验分享技能和 `teamai session save` 的职责，并确认 WSL 本地安装的实际行为。

## 解决方案

1. 先查看本地安装包中的 README、中文 README、命令帮助和 `teamai-share-learnings` 技能说明。
2. 用中文 Markdown 编写经验文档，并在文件开头加入 YAML frontmatter：`title`、`author`、`date` 和 2 到 5 个 `tags`。
3. 先执行 dry-run 检查生成的目标文件名和 `learnings/` 路径：

   ```bash
   teamai --dry-run contribute --file /tmp/session-summary.md --title "经验标题"
   ```

4. 确认仓库状态、分支和未推送提交后，再执行正式贡献：

   ```bash
   teamai contribute --file /tmp/session-summary.md --title "经验标题" --scope user
   ```

5. 团队成员通过 `teamai pull` 获取内容；若要让 AI 在新任务开始时自动检索团队知识，还需要开启 `teamai recall`。

## 经验总结

- `teamai contribute` 只负责读取文档、写入 `learnings/` 并提交到团队仓库，不负责自动总结当前 Session。
- `teamai-share-learnings` 负责总结 Session、生成临时文档，再调用 `teamai contribute`。
- `teamai session save --push` 记录的是脱敏的 Session 活动摘要，主要供 `teamai digest` 使用，不等同于可复用的 troubleshooting 知识。
- 普通远程仓库模式下，`contribute` 会沿用当前 Git 分支提交并推送；执行前必须检查是否存在无关的未推送提交或未跟踪文件。
- `teamai contribute` 的文档规范要求中文内容和 YAML frontmatter。CLI 主要检查文件是否存在、非空以及仓库是否可写，因此格式质量需要在提交前人工或由 AI 检查。
- 团队仓库的 Git 认证应使用 SSH 或 credential helper，避免把访问令牌直接嵌入 remote URL。

## 相关 Skills

- teamai-share-learnings
