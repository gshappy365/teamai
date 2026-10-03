# 迁移到官方 teamai-cli（user scope）

此前本机以手动方式部署团队资源：仓库克隆到 `~/.agents/teamai`，用 Windows junction 将 53 个 skill 链接进 5 个 AI 工具的扫描目录（claude/codex/opencode/workbuddy/豆包工作），并手工编辑 4 个工具的 MCP 配置文件。我们决定全面迁移到官方 teamai-cli（npm 包 `teamai-cli`），以 user scope 安装，仓库由 CLI 克隆到 `~/.teamai/team-repo`，skills/rules/docs/env/MCP 全部自动分发。

替代方案：保留手动 junction 体系仅修补仓库内容——被放弃，因为它没有诊断能力（无从验证资源真正送达）、每次资源变更需手工同步 5 个目录，且 `teamai.yaml` 本就是官方 CLI 格式，维护两套分发机制成本更高。project scope——被放弃，因为需求是"本机全局的公共路径自动发现"，user scope 正是官方定义的对应机制。

后果：手动部署已退役（内容备份于 `~/.agents/teamai-manual-backup`，配置备份于 `~/.agents/_teamai-migrate-20261004-062439`）；weknora 知识库暂不安装（已从 `mcp/mcp.yaml` 移除，待需要时再接入）；豆包工作不在官方工具列表，通过 `.agents/skills` 下的 junction 指向官方 clone 继续消费同一份资源。
