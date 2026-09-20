---
title: "OpenTag WebSocket silent hang 与 session 上下文累积导致反复不回复"
author: gshappy365
date: 2026-09-20
tags: [opentag, websocket, silent-hang, session, context, feishu, codex]
---

## 问题现象

`qdr-wsl-codex` agent 出现反复不回复：刚重启 daemon 后能回复 1-2 条消息，随后再次静默。飞书发消息无响应，daemon 日志无新条目，无 connection lost 事件。

## 根因

两层问题叠加：

### 1. WebSocket silent hang（高频复发）

- daemon 到 opentag server 的 WebSocket 连接状态显示 `registered` 且 TCP ESTABLISHED，但消息无法投递
- 日志特征：超过 5-10 分钟无任何 `turn` / `connection` / `recovery` 活动
- 无 `connection lost` 或 `retry` 事件 → daemon 认为连接正常，不会自动重连
- 修复：`systemctl --user restart opentag-dev.service`

### 2. Session 上下文累积（本次加剧因素）

- session `e7e5fd83` 已运行超过 5 天（9月14日 ~ 9月20日）
- 累计 input tokens 超过 10 万（最后一轮 `in=107490`）
- 大上下文 → codex 处理变慢 / 可能触达上下文窗口上限 → 后续 turn 挂住
- 修复：删除旧 session 绑定，让 daemon 创建新 session

## 完整修复步骤

```bash
# 1. 停止 daemon
systemctl --user stop opentag-dev.service

# 2. 备份并删除旧 session 状态
mkdir -p /tmp/opentag-dev-session-backup
cp ~/.opentag-dev/data/runtime/session-bindings/*/s-*.json /tmp/opentag-dev-session-backup/
cp ~/.opentag-dev/data/runtime/durability/turn-report.json /tmp/opentag-dev-session-backup/

rm ~/.opentag-dev/data/runtime/session-bindings/*/s-*.json
rm ~/.opentag-dev/data/runtime/durability/turn-report.json
rm -f ~/.opentag-dev/data/runtime/session-cli-proofs/s-*.json
rm -rf ~/.opentag-dev/data/runtime/provider-credentials/*/

# 3. 启动 daemon
systemctl --user start opentag-dev.service
```

## 诊断命令速查

```bash
# 检查 daemon 是否静默 hang（最后日志时间 vs 当前时间）
stat ~/.opentag-dev/logs/client.log | grep Modify

# 查看最近 turn 结果和 token 消耗
python3 -c "
import json; from datetime import datetime, timezone, timedelta
d=json.load(open('.opentag-dev/data/runtime/durability/turn-report.json'))
for item in d[-5:]:
    p=item['payload']; u=p.get('usage',{})
    dt=datetime.fromtimestamp(item['updatedAt']/1000,tz=timezone.utc)+timedelta(hours=8)
    print(f\"{dt:%H:%M} turn={p['turnId'][:12]}... outcome={p['outcome']} in={u.get('inputTokens',0)}\")
"

# 检查 codex 进程是否有代理
cat /proc/$(pgrep -f "codex app-server" | tail -1)/environ | tr '\0' '\n' | grep -i proxy

# 检查 codex 连接状态（SYN-SENT = 网络不通）
ss -tnp | grep codex | grep -v LISTEN

# 重启 daemon
systemctl --user restart opentag-dev.service
```

## 经验总结

- **WebSocket silent hang 在 .opentag 和 .opentag-dev 都会发生**，是 opentag daemon 的已知 recurring issue
- **重启 daemon 是快速临时修复**，但要彻底解决需要**开启新 session** 清除积压上下文
- **观察 token 消耗趋势**可以预判 session 生命周期：超过 8 万 tokens 建议考虑新 session
- **不要同时排查两个 home**：`.opentag` 和 `.opentag-dev` 是不同的 opentag 实例，先确认 agent 用哪个
- **代理问题 + silent hang 可能同时存在**：先用 `ss -tnp | grep codex | grep SYN-SENT` 排除网络问题，再排查 hang

## 相关 Learnings

- [[opentag-飞书不回复排障]] — 首次记录的 WebSocket silent hang
- [[wsl-daemon故障速查]] — daemon 故障通用诊断
- [[opentag-新-agent-provider-failed-全链排障-2026-09-15-pgjzlk]] — provider_failed 全链排障
- [[opentag-codex-运行时缺失代理-2026-09-19-yxs0hw]] — codex 代理缺失