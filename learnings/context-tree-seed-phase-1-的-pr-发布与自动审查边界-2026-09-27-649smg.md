---
title: "Context Tree Seed Phase 1 的 PR 发布与自动审查边界"
author: gshappy365
date: 2026-09-27
tags: [context-tree, seed, workflow, github, review]
---

## 背景

Context Tree Seed Phase 1 的本地骨架通过 `first-tree tree verify` 后，还需要让远程团队树真正接收变更。本轮先将修复提交推送到任务分支、创建 PR，并让 First Tree 在原聊天中跟踪该 PR。尝试开启自动 Context Reviewer 时，界面曾提示 `GitHub pull-request write access is required`；随后观察到 review-config 从 `enabled=false` 变为 `true`。

## 可复用流程

1. 将 **本地校验、远端发布、主分支生效** 分成三个状态：verify 通过只证明候选树有效；推送任务分支并创建 PR 只让变更可审查；只有 PR 合并后，绑定的远程 `main` 才包含新结构。
2. 在发布前复核工作树、提交范围和 `first-tree tree verify`；推送仅包含本任务变更的分支，创建描述开放/暂缓领域及校验结果的 PR。创建 PR 后用 `first-tree github follow <PR URL>` 将后续 CI、审查和合并事件接回原聊天。follow 是事件跟踪，不是审查或合并授权。
3. 自动审查开启后，仍需等待正式审查结果。App 负责发表正式审查；受信任的 Reviewer Agent 才按其流程使用本机 `gh`，对核对过的精确 head SHA 执行 squash merge。不能把“review-config 已启用”误报为“PR 已通过审查/已合并”。
4. 若 PR 改动根 `SCOPE.md`，需要管理员针对 **精确 PR head** 给出可追踪确认；旧 head 的确认不能自动覆盖后续改动。Phase 2 的决策叶节点应等 Phase 1 结构审查并合并后再开始。

## 权限报错的判断边界

`GitHub pull-request write access is required` 是观察到的提示，之后 review-config 变为 `enabled=true` 也是观察到的状态；本轮没有直接核验 GitHub App 安装权限及其变更历史，因此不能断言“缺少 App 权限”就是已证实根因，也不能仅凭配置开启推断 PR 已完成审查。排障时分别核对 App 仓库覆盖/PR 写权限、First Tree 的 review-config，以及 PR 当前的审查与合并状态。

## 相关 Skills

- `first-tree-seed`：Phase 1 结构、验证和 PR 交付。
- `context-tree-review`：自动审查与精确 head 的合并边界。
- `teamai-share-learnings`：将本次可复用经验发布到团队知识库。
