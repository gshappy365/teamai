---
title: "First Tree 与 TeamAI 的分层协同及 Context Tree Seed 修复"
author: gshappy365
date: 2026-09-27
tags: [first-tree, context-tree, teamai, seed, troubleshooting]
---

## 背景

本次工作需要修复一个已绑定但未完成 Seed 的 Context Tree，并明确 First Tree、项目代码仓库和 TeamAI 的边界。旧树可以被绑定，但 `first-tree tree verify` 失败：根 `NODE.md` 和 `README.md` 没有 YAML frontmatter，`members/` 目录也不存在。

## 关键判断

First Tree 与 TeamAI 负责不同的内容：

- First Tree 管理 Agent 身份、模型和推理设置、聊天路由、任务协同以及 Context Tree 绑定。
- Context Tree 记录持久化的决策、约束、ownership 和跨领域关系，不复制代码、测试、构建步骤或 session 日志。
- TeamAI 分发可复用的 skills、rules、docs、env 和 MCP。`teamai push` 或 `teamai contribute` 不会直接写入 Context Tree。
- TeamAI 内容进入 Context Tree 时，必须先成为具体的 source artifact，例如已确认的决策文档、ADR、会议决定或长期约束，然后再按 First Tree 的 source-backed write 流程写入明确节点。

## 修复方法

1. 先运行 `first-tree tree verify --tree-path <tree>`，把失败输出作为反馈环，不先猜原因。
2. 确认 Team 绑定、远端仓库和分支正确，再读取声明的 source 的固定提交。
3. 因为树处于“已绑定、未 Seed”状态，先请求人类 owner 确认顶层领域；不要用 `unassigned` 等占位领域绕过成员校验。
4. 根据 source 中的稳定信号，只开启 `system`，并建立三个二级关注轴：
   - `catalogue`：技能包、技能条目、共享详情和目录/搜索边界；
   - `content`：专题知识库、任务场景、上游固定快照和内容边界；
   - `delivery`：静态站发布、生成内容和交付验证约束。
5. 补齐根 `NODE.md`、`SCOPE.md`、`members/`、`raw-context/`、领域节点和 Seed source ledger，再运行原始校验环。

## 为什么不默认建立 product 或 team-practice

顶层领域表示团队长期使用的关注轴，不是仓库目录的镜像。`product` 需要产品定位、目标用户、范围、路线或成功标准等持久证据；`team-practice` 需要跨 First Tree、TeamAI 和代码仓库的已确认工作协议。只有网站名称、一次性想法、聊天记录或 session learning 时，不应创建新的顶层领域。

## 结果与边界

Seed 骨架必须通过 `first-tree tree verify` 后才可交付。成员的 `domains` 应引用已批准的顶层领域，Seed ledger 应记录实际读取的 source 身份和固定提交。远程 push 或 PR 是独立的发布动作；本地校验通过不等于已经改变远程 `main`。

## 可复用结论

把 First Tree 当作运行时控制面，把 Context Tree 当作长期决策面，把项目仓库当作实现面，把 TeamAI 当作能力分发面。不要让 TeamAI 的全部 skills 或 session learnings 自动镜像到 Context Tree；只提升未来仍需遵守的稳定决策和约束。
