---
title: "OpenTag qdr-wsl-claude provider_failed 零凭证排障"
author: gshappy365
date: 2026-09-19
tags: [opentag, claude-code, feishu, provider-failed, workspace, settings-json, zero-credentials]
---

## 问题

飞书群里 @ `qdr-wsl-claude` agent 后无回复。daemon 日志显示 turn 正常启动（`Turn started`），但所有 turn `outcome=failed, errorReason=provider_failed`，`usage.inputTokens=0 / outputTokens=0`，`outgoingReplies.replies=[]`。从 09-15 到 09-19 持续复现。

## 根因

agent workspace 缺 `.claude/settings.json` → claude-code provider 零凭证 → 不向模型 API 发请求 → 合成 `<synthetic>` 空响应 → opentag 判定 `provider_failed`。

证据链（全部来自本地文件，已交叉验证）：

1. **agent 身份** — `effective-snapshots/…/s-481323…json`：`qdr-wsl-claude` = agentId `5a5e8f59…`，provider `claude-code`，跑在 prod 实例。
2. **飞书消息已送达** — `~/.opentag/logs/client.log`：多次 `Turn started`，binding 正常。
3. **每个 turn 失败、零 token、无回复** — `session-bindings/…/s-cef3f14….json` 的 `recentRecordedInputs[].report`：5 次 turn 全部 `outcome=failed, errorReason=provider_failed, usage.inputTokens=0/outputTokens=0, outgoingReplies.replies=[]`。
4. **claude-code 根本没发 API 请求** — `~/.claude/projects/…/0f58c62a….jsonl`：assistant 行 `"model":"<synthetic>"`，`usage.input_tokens=0/output_tokens=0`，`stop_reason:"stop_sequence"`。`<synthetic>` 是 claude-code 在无凭证时合成的空响应，不是真实模型输出。
5. **workspace 没有配置** — `~/.opentag/data/workspaces/a-15a0e83…/` 是纯空目录，无 `.claude/`，无 `settings.json`。
6. **provider 凭证目录也空** — `~/.opentag/data/runtime/provider-credentials/` 无任何文件。

机制：daemon 用 `--setting-sources project` 启动 claude-code，只读 workspace 下的 project 级配置。workspace 是空目录 → 零凭证 → claude 不发任何 HTTP → `<synthetic>` 空响应 → turn 0 token → `provider_failed`。这是 [[opentag-新-agent-provider-failed-全链排障-2026-09-15-pgjzlk]] 和 [[opentag-飞书不回复排障]] 根因在新 agent 上的又一次重现。

## 诊断快速路径

1. 看 `session-bindings` 中对应 agent 的 `recentRecordedInputs[].report`：若 `outcome=failed, errorReason=provider_failed, usage.inputTokens=0` → 几乎一定是 workspace 缺 settings.json。
2. 看 claude transcript（`~/.claude/projects/…workspace…/<sessionId>.jsonl`）里 assistant 行：`"model":"<synthetic>"` + `input_tokens=0` → 确认零凭证、未发请求。
3. 看 workspace 目录 `~/.opentag/data/workspaces/<workspace-id>/` 是否空、有无 `.claude/settings.json`。

## 修复

在该 agent 的 workspace 下补一个 `.claude/settings.json`，指向本地 FreeLLMAPI：

```bash
mkdir -p ~/.opentag/data/workspaces/<workspace-id>/.claude
cat > ~/.opentag/data/workspaces/<workspace-id>/.claude/settings.json <<'EOF'
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
EOF
```

本次 qdr-wsl-claude 的 workspace-id 是 `a-15a0e83fb90dc85ec01544954f06187f984384fa`。

**无需重启 opentag.service**：claude-code provider 在每次 turn 启动时拉起 claude 进程并读 project 级 settings，新文件下一次 turn 即生效。

## 验证

补文件后在飞书 @ agent 一次，下一个 turn 报告（`~/.opentag/data/runtime/durability/turn-report.json` 最新条）：
- `outcome` 不再是 `failed`
- `usage.inputTokens=998, outputTokens=400`（真实模型调用）
- `outgoingReplies.replies[]` 含飞书 messageId（如 `om_x100b65d4345130a4b3dd6f854223cf9`）
- `traceSummary.lastSequence=15`（失败时是 5）
- agent 实际回复："收到 ✅ 测试正常，我在线。…抱歉让你久等。有什么我可以帮你的吗？"

## 经验总结

- **新建 agent 必须补 settings.json** — `opentag connect` 不会自动为新 workspace 创建 `.claude/settings.json`。这是最高频的坑：飞书消息能下发到 daemon（turn started），但 claude-code 零凭证导致 provider_failed。每次新建 agent 后都要手动放配置。这与 [[opentag-新-agent-provider-failed-全链排障-2026-09-15-pgjzlk]] 的结论一致，是 recurring 根因。
- **`<synthetic>` 是零凭证的指纹** — claude transcript 里 assistant 行 `model: "<synthetic>"` + `input_tokens=0` 是 claude-code 没向 API 发请求的确定性证据，区别于"发请求被拒"（那会有真实 model 名和非零 token 或明确错误）。
- **诊断 provider_failed 的快速路径** — 看 `session-bindings` 的 `recentRecordedInputs[].report`：`provider_failed + inputTokens=0` 几乎一定是缺 settings.json；再读 transcript 确认 `<synthetic>` 即可定论，不用反复试。
- **多 agent 同机时的陷阱** — 同一台机器上 prod 和 dev 两个 opentag 实例各自有独立 workspace 目录，且不同 agent 用不同 workspace。日志里 im-delivery 的 agentId 要和目标 agent 的 agentId 核对一致，否则会误判"消息没到"。

## 相关 Learnings

- [[opentag-新-agent-provider-failed-全链排障-2026-09-15-pgjzlk]] — 同一根因的首篇全链排障，4 层叠加分析
- [[opentag-飞书不回复排障]] — 首次记录 `--setting-sources project` 只读 project 级的机制
- [[opentag-codex-运行时缺失代理-2026-09-19-yxs0hw]] — 同日另一 agent 的 provider_failed（codex 侧，缺失代理环境变量）
- [[wsl-daemon故障速查]] — daemon 故障通用诊断
