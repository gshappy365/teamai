# TeamAI 资源仓库

本仓库承载团队的 AI 协作资源：可复用技能、强制规则、经验知识与工具配置。它由 teamai-cli 分发到各 AI 工具，本身不是应用代码树。

## Language

**Skill**:
一个可被 AI 工具自动发现的指令单元：目录内含 `SKILL.md`，frontmatter 带 `name` 与 `description`，使模型能按需调用。
_Avoid_: command、tool、插件

**Rule**:
团队约定，随会话强制注入 AI 工具（`rules/<name>.md`）。
_Avoid_: 规范、指南文件（特指非强制类文档时用 docs）

**Learning**:
从真实会话沉淀的经验笔记（排障、操作指南、教训），经 `teamai contribute` 提交到 learnings 目录或 teamai-learnings 分支。
_Avoid_: note、日记、随笔

**ADR**:
记录"难逆转、缺上下文会惊讶、有真实权衡"的决策的文档（`docs/adr/NNNN-slug.md`）。
_Avoid_: 决定、决策备忘

**MCP server**:
在 `mcp/mcp.yaml` 声明、由 teamai pull 注入各工具配置的模型上下文协议服务器。
_Avoid_: 插件、连接器

**Source**:
外部公开 skill 订阅源（`teamai.yaml` 的 `sources` 列表），订阅方的 skills 经 pull 自动同步。
_Avoid_: 上游、外部仓库（特指 skill 订阅时用 source）

**Scope**:
安装范围。user scope 将资源安装到用户主目录（本机全局）；project scope 安装到项目目录。
_Avoid_: 模式、级别

**Pull**:
将团队仓库的最新资源拉取并注入本机各 AI 工具。
_Avoid_: 同步、更新（特指 teamai pull 动作用 pull）

**Push**:
将本机对团队资源的贡献推回仓库（经 PR/MR）。
_Avoid_: 提交、上传

**Contribute**:
将 session 经验沉淀为 learning 的动作。
_Avoid_: 分享、记录

**Recall**:
对团队知识库的语义检索（BM25 + 图谱增强），`teamai recall <query>`。
_Avoid_: 搜索、查找

**Doctor**:
全量体检命令，验证团队配置与资源是否真正送达各工具（`teamai doctor`）。
_Avoid_: 检查、诊断（命令名用 doctor）

**Team repo**:
本仓库：团队 AI 资源的唯一远端来源。
_Avoid_: repository、codebase

**Junction**:
历史手动部署时代用于分发 skill 的目录符号链接；官方 CLI 接管后仅保留在历史文档中。
_Avoid_: symlink（Windows 上实际是 junction，避免混用术语）
