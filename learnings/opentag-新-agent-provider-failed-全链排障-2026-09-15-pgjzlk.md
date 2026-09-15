---
title: "OpenTag 新 Agent provider_failed 全链排障"
author: gshappy365
date: 2026-09-15
tags: [opentag, claude-code, feishu, provider-failed, workspace]
---

## 问题

在 app.opentag.build setup 页面为飞书机器人绑定新 Agent（agentId 3c95775c）后，飞书群里 @ agent 无回复。daemon 日志显示 turn 正常启动，但所有 turn `outcome=failed, errorReason=provider_failed`，usage 全为 0 token。setup 页面"检查本地 Claude 环境"长时间无响应。

## 根因链（4 层叠加）

1. **WebSocket silent hang** — daemon 到 app.opentag.build 的 WebSocket 跑久了假活：状态显示 `registered`，但收不到新消息。诊断特征：日志超过 10 分钟无新条目，但 connection 不报 lost。这是 [[opentag-飞书不回复排障]] 已记录的同一个坑。

2. **systemd service definition drift** — `opentag doctor` 报 `Daemon service: drifted or unverifiable`。drift 导致 daemon 行为与预期不一致，setup 页面下发的探测请求无法路由到本地。

3. **飞书机器人绑定冲突** — 错误提示"此飞书机器人已连接到另一个 Agent"。根因是旧 computer identity（e1defc8e）上有残留的飞书机器人绑定，新 agent 无法复用同一个机器人。

4. **新 workspace 零凭证** — `opentag connect` 成功后，新 agent 的 workspace（`a-f4c5eac1...`）是空目录，没有 `.claude/settings.json`。daemon 用 `--setting-sources project` 启动 claude-code，只读 workspace 下的 project 级配置 → 零凭证 → 不发任何 HTTP → `provider_failed`，0 token。这是 [[opentag-飞书不回复排障]] 已记录的根因在新 agent 上的重现。

## 修复步骤

### 1. 修复 WebSocket silent hang

```bash
systemctl --user restart opentag.service
```

验证：日志出现 `Runtime connection registered` + `Computer runtime is ready`。

### 2. 修复 service drift

```bash
opentag daemon install
```

可能报 `systemd restart failed: timed out after 15000ms`（因 slack provider probe 卡住），但 service 实际已重启。验证：`opentag daemon status --json` 中 `drifted: false`。

### 3. 修复飞书机器人绑定冲突

在 setup 页面获取一次性 connect code，执行：

```bash
opentag connect --server https://app.opentag.build -- otcc_<code>
```

这会创建新 computer identity（d5d58196），消除旧 identity 的绑定冲突。connect 同时会自动配置 lark-cli（飞书）和尝试配置 slack-cli。

验证：输出 `Connected Computer ... and bound Agent ...`，`[im-cli:lark] ready`。

### 4. 修复新 workspace 零凭证

找到新 agent 的 workspace 目录（通过 session-bindings 或 effective-snapshots 中的 workspaceId），从已知可用的 workspace 复制 `.claude/settings.json`：

```bash
mkdir -p ~/.opentag/data/workspaces/<new-workspace-id>/.claude
cp ~/.opentag/data/workspaces/<known-good-workspace-id>/.claude/settings.json \
   ~/.opentag/data/workspaces/<new-workspace-id>/.claude/settings.json
```

settings.json 内容（指向本地 FreeLLMAPI）：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://localhost:3001",
    "ANTHROPIC_AUTH_TOKEN": "freellmapi-<token>",
    "ANTHROPIC_MODEL": "auto",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "auto",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "auto",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "auto",
    "CLAUDE_CODE_AUTO_COMPACT_WINDOW": "1048576",
    "CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY": "1"
  }
}
```

验证：下一个 turn `outcome=completed`，`inputTokens > 0`，`outputTokens > 0`。

## 经验总结

- **新建 agent 必须补 settings.json** — `opentag connect` 不会自动为新 workspace 创建 `.claude/settings.json`。这是最高频的坑：飞书消息能下发到 daemon（turn started），但 claude-code provider 零凭证导致 provider_failed。每次新建 agent 后都要手动放配置。
- **WebSocket silent hang 是 recurring issue** — daemon 跑久了必然出现。诊断方法：看日志最后一条时间戳与当前时间差，超过 10 分钟且无 connection lost 报警即为假活。修复：重启 daemon。
- **service drift 用 `opentag daemon install` 修** — 不是手动改 unit 文件。install 可能因 slack probe 超时但实际已修复 drift。
- **飞书机器人绑定冲突用 `opentag connect` 解决** — 新 connect code 会创建新 computer identity，绕过旧绑定。不需要在服务端手动解绑。
- **诊断 provider_failed 的快速路径** — 检查 session-bindings 中对应 agent 的 `recentRecordedInputs[].report`：如果 `outcome=failed, errorReason=provider_failed, usage.inputTokens=0`，几乎一定是 workspace 缺 settings.json。

## 相关 Learnings

- [[opentag-飞书不回复排障]] — 首次记录的根因和修复
- [[wsl-daemon故障速查]] — daemon 故障通用诊断
