# Learnings 索引

本目录沉淀会话经验：排障记录、操作指南与流程教训。learnings 由 `teamai contribute` 写入（官方会生成 `主题-YYYY-MM-DD-随机后缀.md` 命名并维护检索索引），也可手动创建后经 push 提交。

## 命名规范

- 官方自动生成：`<主题>-<YYYY-MM-DD>-<6位随机>.md`（主题用连字符拼接，避免空格与特殊字符）。
- 手动创建：遵循同一格式；主题用中文或英文均可，但同一条 learning 内不要混用命名模式。
- 一个文件只记录一个问题；排障类在标题中直接写现象（如 `wsl-中-codex-代理继承与-computer-离线排障`）。

## 索引

### 排障记录

| 文件 | 主题 |
|---|---|
| `git-push-网络容错.md` | git push 网络失败的处理 |
| `kimaki-与-teamai-本地配置排障-freellmapi-鉴权和-cli-恢复-2026-09-20-npqw24.md` | freellmapi 鉴权与 CLI 恢复 |
| `opentag-codex-运行时缺失代理-2026-09-19-yxs0hw.md` | Codex 运行时缺代理 |
| `opentag-qdr-wsl-claude-provider-failed-零凭证-2026-09-19-7xq3kn.md` | WSL Claude provider 零凭证 |
| `opentag-websocket-silent-hang-与-session-上下文累积-2026-09-20-evyuaj.md` | WebSocket 挂起与会话上下文累积 |
| `opentag-wsl-中-codex-代理继承与-computer-离线排障-2026-09-20-6srorc.md` | WSL Codex 代理继承与离线 |
| `opentag-新-agent-provider-failed-全链排障-2026-09-15-pgjzlk.md` | agent provider failed 全链排障 |
| `opentag-飞书不回复排障.md` | 飞书不回复 |
| `teamai-contribute-网络失败后的重试处理-2026-09-15-s86iph.md` | contribute 网络失败重试 |
| `wsl-daemon故障速查.md` | WSL daemon 故障速查 |

### 操作指南

| 文件 | 主题 |
|---|---|
| `WSL2 Ubuntu 整机精简备份与迁移到新发行版实战.md` | WSL2 整机备份迁移 |
| `本地托管-context-tree-搭建与使用指南-2026-09-19-ylq7o2.md` | context-tree 搭建使用 |
| `teamai-contribute工作流.md` | contribute 工作流 |

### 流程与经验

| 文件 | 主题 |
|---|---|
| `session-notes-2026-09-15-832k48.md` | 会话笔记 |
| `使用-teamai-将-session-经验沉淀为团队知识-2026-09-15-67oo4n.md` | session 经验沉淀为团队知识 |

## 维护

- 新增 learning 优先用 `teamai contribute`（自动进检索索引）。
- 定期用 `teamai recall maintenance` 清理低置信度条目、回写置信度。
- 高置信度 learning 可 `teamai recall promote` 晋升为正式 skills/rules/docs。
