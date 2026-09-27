---
title: "定位 TeamAI 内置技能并核对经验贡献分支"
author: gshappy365
date: 2026-09-27
tags: [teamai, skill-discovery, workflow, troubleshooting]
---

## 背景

在 First Tree 会话中调用 `teamai-share-learnings` 时，当前代理的技能列表没有显示它。此前因此误判技能不存在，改为手动执行经验贡献。复核已发表的 learning 和 2026-09-15 的 TeamAI 使用文档后，需要确认技能位置、去重方式及当前版本的发布分支。

## 解决方案

1. 检查 TeamAI CLI 安装包的 `skills/teamai-share-learnings/SKILL.md` 并完整阅读。它存在于本机全局 npm 包中，即使 First Tree 的技能列表未加载该技能，`teamai skill show teamai-share-learnings` 也可能报告找不到。技能发现结果不能代替对安装包的检查。
2. 先阅读已有 learning，并在团队仓库的 `learnings/` 中检索同主题内容。已有《First Tree 与 TeamAI 的分层协同及 Context Tree Seed 修复》涵盖平台边界和 Seed 修复，本篇只记录技能发现与贡献验证。
3. 按技能模板写中文 Markdown，包含 `title`、`author`、`date`、2–5 个 `tags`，以及背景、解决方案、经验总结和相关 Skills。
4. 检查团队仓库和 learning 工作树的 Git 状态，保护无关改动。执行 `teamai --dry-run contribute --file <path> --title <title>` 预览，再执行 `teamai contribute --file <path> --title <title> --scope user`，最后用 `teamai recall <关键词>` 验证可检索。

## 经验总结

- 当前 TeamAI CLI 0.25.0 的技能说明指定将 learning 发布到独立的 `teamai-learnings` 分支；本机会话中也能看到对应的干净工作树。2026-09-15 文档中“沿用当前 Git 分支提交并推送”的说法已不适用于这套安装，操作前应以当前 CLI、技能说明和实际工作树为准。
- 技能未出现在宿主代理列表，可能是加载范围不同；先检查安装包，再判断技能是否缺失。`teamai skill show` 在本次环境中同样无法发现这个内置技能。
- 贡献前先查重，保留每篇 learning 的独立问题；发布后核对目标文件、分支、提交和 `recall` 结果。

## 相关 Skills

- teamai-share-learnings
