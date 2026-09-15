# OpenTag 506-wsl-claude 飞书不回复排障

## 问题
飞书群里 @ 506-wsl-claude agent 后无回复，提示 `provider_failed`，transcript 显示 `Not logged in · Please run /login`。

## 根因
1. Agent `506-wsl-claude` 用 `provider: claude-code`
2. Daemon 起 claude 时带 `--setting-sources project`，只读 `<cwd>/.claude/settings.json`（project 级），忽略 `~/.claude/settings.json`（user 级）
3. CWD 是 agent workspace `~/.opentag/data/workspaces/a-3ccc97.../`，里面没有该文件 → claude 零凭证
4. 零凭证时 claude 输出 synthetic 消息 `Not logged in`，不发任何 HTTP → 网关零记录、turn 0 token

## 修复
在 workspace 下创建 `.claude/settings.json`（从 `~/.claude/settings.json` 复制 env 块），包含 `ANTHROPIC_BASE_URL` 指向本地 FreeLLMAPI

## 另一个坑：WebSocket silent hang
- daemon 到 `app.opentag.build` 的 WebSocket 跑久了假活（不 trigger connection lost，但收不到新消息）
- 诊断：日志超过 10 分钟无新条目
- 修复：`systemctl --user restart opentag.service`
- 验证：重启后积压消息涌入，turn `e9b5cb1b` completed（288k in / 4k out token）