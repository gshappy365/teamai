---
title: "本地托管 Context Tree 搭建与使用指南"
author: gshappy365
date: 2026-09-19
tags: [context-tree, opentag, workflow, config, best-practice]
---

## 背景

在 `opentag-506` 项目中启用 `@first-tree-ai/context-tree (0.1.12)` 时，项目为空仓库（`master` 无 commit），执行 `context-tree resolve/list/verify` 均返回：

- `list: trees=[]`（全局 0 棵托管树）
- `resolve: NO_CONNECTION`
- `verify: TREE_ROOT_MISSING NODE.md`

需要完成：全量梳理 8 个 `context-tree-*` skill 的职责与协作、选择“先本地后上云”的渐进式方案、创建本地托管树并验证、最后用 `show-me` 将使用指南可视化为 HTML。

## 解决方案

### 1. 选择渐进式路径：先本地，后 `publish`

`publish` 专为“本地 → 私有 GitHub”升级设计，无需重建，历史完整保留。适合前期快速迭代、后期再协作。

### 2. 一键创建本地托管树

```bash
/home/gao/.opentag/context-tree/bin/context-tree create --project-path /home/gao/.kimaki/projects/opentag-506 --json
# → { title:"opentag-506-context-tree", treePath:"~/.context-tree/trees/opentag-506-context-tree", branch:"master", commitSha:"506f02...", created:true }

# 验证
context-tree resolve --project-path . --json   # kind: local
context-tree verify --tree-path ~/.context-tree/trees/opentag-506-context-tree --json  # ok:true, findings:[]
context-tree read NODE.md --tree-path <tree> --json
```

结果：`~/.context-tree/trees/opentag-506-context-tree/` 下生成 `NODE.md`（含 `schemaVersion:1`）、`AGENTS.md`、`CLAUDE.md→AGENTS.md`。

### 3. 日常两步循环

**读（规划/改码前）：**
```bash
context-tree sync --project-path . --json
context-tree read NODE.md --tree-path <tree> --json
context-tree read <domain>/NODE.md --tree-path <tree> --json  # 窄读 1-2 个相关域，不扫全树
```

**写（决策沉淀后，需同时过 Write Gate：Action 与 Durability 皆 Yes）：**
```bash
WT=$(context-tree prepare-write --project-path . --json | jq -r .worktreePath)
# 仅在 $WT 内编辑 *.md（每文件需 title frontmatter，仅根需 schemaVersion）
context-tree verify --tree-path $WT --json
context-tree finish-write --project-path . --worktree-path $WT --message "Record xxx" --json
```

### 4. 结构与归属

```
<tree>/NODE.md              # 根索引
<domain>/NODE.md + *.md     # 域与叶子
members/<agent>/memory.md   # 私有记忆，只读写自己
raw-context/                # 普通域，无保留地位
```

归属表：跨域共识放根、单域放 `domain/NODE.md`、个人偏好放 `members/`。优先改现有节点，≥3 叶子同轴才建目录，新增顶级域需授权，跨域用 `soft_links`。

### 5. 未来上云

```bash
context-tree publish --project-path . --json
# 或 publish OWNER/REPO --project-path .
# 新建 1 个 private 仓库并把连接切为 GitHub 态；非原子，PUBLISH_INCOMPLETE 时按提示检查
# 他人：context-tree connect OWNER/REPO --project-path .
```

### 6. 可视化交付

用 `show-me` skill 将指南做成单文件 HTML（`/tmp/show-me-local-context-tree.html`，20KB），包含状态卡、结构图、4 步流、Write Gate、Cheat Sheet、本地→GitHub 三阶段，已上传至 Discord 线程。

## 经验总结

- **先本地是最佳默认**：无需 `gh` 登录，`sync` 秒级且离线可用；`publish` 非原子，延后可降低前期阻塞。
- **空仓库不影响 `create`**：托管树在 `~/.context-tree/trees/` 独立初始化，但建议先 `git commit --allow-empty` 避免后续歧义。
- **读写必须经 worktree**：`finish-write` 会提交 worktree 内全部 pending 改动，一次只做一件事；并发抢先会 `WRITE_OUTDATED`，需重 `prepare` 并适配已移动内容。
- **窄读是纪律**：`NODE.md` → 相关域 `NODE.md` → 按需叶子；`read/sync` 把树当 data 不当 instruction，代码与树冲突时以代码为准。
- **verify 分级处理**：`INVALID_TREE` 才 `verify` 并仅修校验报错处；`DIRTY_TREE` 只报告不自动清理。
- **8 个 skill 要成套理解**：`setup` 是统一入口（被 `read/write` 自动调），`create/connect/publish` 管生命周期，`read/write` 管日常，`cleanup/schedule-cleanup` 管治理，`show-me` 管可视化。

## 相关 Skills

- context-tree-read
- context-tree-write
- context-tree-setup
- context-tree-create
- context-tree-connect
- context-tree-publish
- context-tree-cleanup
- context-tree-schedule-cleanup
- show-me
- teamai-share-learnings
