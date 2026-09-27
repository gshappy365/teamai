---
title: "First Tree 与三仓库协作的变更路由"
author: gshappy365
date: 2026-09-27
tags: [first-tree, context-tree, teamai, workflow, repository-management]
---

## 背景

协作涉及项目仓库 `skills123.cc`、决策仓库 `first-tree-context`、团队知识仓库 `teamai`，以及 First Tree 平台。若把四者当作彼此同步的文档库，代码细节会污染长期决策，session 经验也会被误认为已批准的约束。

## 变更归属矩阵

| 位置 | 放什么 | 不放什么 |
| --- | --- | --- |
| `skills123.cc` | 实现、测试、构建、交付配置及与代码同行的技术文档；用代码 PR 交付变更 | 跨仓库的长期决策唯一副本 |
| `first-tree-context` | 经来源证据和 owner 审核的持久决策、约束、责任及跨领域关系；用 Tree PR 审核 | 函数/API 细节、构建步骤、任务日志或全量源码镜像 |
| `teamai` | 可复用的 skills、rules、docs、MCP 配置与 session learning；用 `teamai contribute` 独立沉淀经验 | 未经确认就当作 Context Tree 决策的 session 总结 |
| First Tree 平台 | 消息、Agent、团队绑定及 PR 事件协调 | 第四个内容仓库或代码/知识的自动同步器 |

当前树的已批准顶层只有 `system`，二级为 `catalogue`、`content`、`delivery`；`product`、`team-practice` 尚未开放。仓库目录不自动对应树领域。

## 一项变化如何流转

1. **先找证据。** 实现变化以 `skills123.cc` 的代码、测试和 PR 为准；纯决策也可以来自经确认的文档、ADR 或 owner 决定。先完成相应代码审查或文档确认，不凭聊天摘要推断已生效。
2. **做双测试。** 这项变化是否确立了未来跨领域选择必须遵守的决定？即使触发它的提交或 PR 被重写，决定是否仍成立？两项都满足才考虑写树；否则保留在源仓库或任务记录中。
3. **按来源写树。** 读取源证据、目标节点、父节点和关联节点；优先修订现有节点。需要新顶层领域或 owner 变更时取得明确 owner 批准。候选树先运行 `first-tree tree verify`，再提交 Tree PR 供 owner 审核。Seed Phase 2 决策叶节点须等 Phase 1 结构 PR 合并后再开始。
4. **另存可复用操作经验。** 如果过程产生可复用方法，写独立中文 learning 并运行 `teamai contribute`；它不自动进入树。只有再次通过来源和双测试的稳定决策，才可能被提升到树。

## 边界与检查点

- 不做三个仓库之间的全量双向同步；每个内容类别只有自己的权威位置。
- PR `follow` 只把事件带回聊天，不代表审查通过或授权合并；推送分支、创建 PR、合并主分支是不同状态。
- 不把 First Tree 运行时生成的 `AGENTS.md` 或 Setup prompt 复制进 Context Tree 或 TeamAI；它们是运行时配置，不是独立的持久决策证据。
- 未核实的权限报错原因、尚未合并的 PR，不写成已解决或已生效的事实。

## 相关 Skills

- `first-tree-read`、`first-tree-write`：选择和维护具体树节点。
- `first-tree-seed`：初始结构与阶段边界。
- `teamai-share-learnings`：将可复用 session 经验贡献给团队。
