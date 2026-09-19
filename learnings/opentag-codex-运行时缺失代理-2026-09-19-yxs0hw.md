---
title: "OpenTag codex 运行时缺失代理导致 provider_failed"
author: gshappy365
date: 2026-09-19
tags: [opentag, codex, proxy, network, provider-failed, environment]
---

## 问题

OpenTag daemon 正常启动、WebSocket 连接正常、turn 正常启动，但所有 turn 全部 `provider_failed`，usage 中 input/output tokens 均为 0。飞书机器人无回复。

## 根因

**daemon 启动 codex app-server 时没有传递代理环境变量**。

诊断路径：

1. 检查 codex 进程网络连接：
```bash
ss -tnp | grep codex
```
发现 codex 进程 (PID 13565) 有两条到 OpenAI IP 的连接卡在 `SYN-SENT`：
```
SYN-SENT  192.168.1.17:45720  →  31.13.69.169:443       # OpenAI IPv4
SYN-SENT  [IPv6 addr]:48634   →  [OpenAI IPv6 addr]:443  # OpenAI IPv6
```

2. 检查 daemon 的环境变量 vs codex 的环境变量：
```bash
cat /proc/362/environ | tr '\0' '\n' | grep -i proxy
# → HTTPS_PROXY=http://127.0.0.1:7897 ✅

cat /proc/13565/environ | tr '\0' '\n' | grep -i proxy
# → (空) ❌
```

3. 检查 daemon 启动 codex 时的环境过滤器：
daemon 用 `shell_environment_policy.inherit="all"` + 白名单 filters 控制 codex 环境：
```
filters={
  PATH, LANG, LC_ALL,
  OPENTAG_HOME, OPENTAG_PROVIDER_ENV_FILE, OPENTAG_SESSION_PROOF_FILE
}
```
**HTTPS_PROXY / HTTP_PROXY / NO_PROXY 不在白名单中**。

4. 对比独立运行的 codex (PID 19337)，它有完整的代理变量并通过代理正常工作。代理 `127.0.0.1:7897` 本身正常。

## 影响范围

- 影响：所有需要通过代理访问 OpenAI API 的环境
- 现象：turn started → provider_failed → 0 token → 无回复
- codex app-server 直连 OpenAI 失败，SYN 包无应答

## 修复方案

### 方案 A：通过 OPENTAG_PROVIDER_ENV_FILE 注入（临时）

在 provider credential env file 中添加代理变量：
```bash
# ~/.opentag/data/runtime/provider-credentials/<session-id>.sh
# 在已有的 Lark 配置后追加：
export HTTPS_PROXY='http://127.0.0.1:7897'
export HTTP_PROXY='http://127.0.0.1:7897'
export NO_PROXY='127.0.0.1,localhost'
export https_proxy='http://127.0.0.1:7897'
export http_proxy='http://127.0.0.1:7897'
export no_proxy='127.0.0.1,localhost'
```

### 方案 B：在 systemd service 中设置代理（推荐）

编辑 `~/.config/systemd/user/opentag.service`，在 `[Service]` 段添加：
```
Environment="HTTPS_PROXY=http://127.0.0.1:7897"
Environment="HTTP_PROXY=http://127.0.0.1:7897"
Environment="NO_PROXY=127.0.0.1,localhost"
Environment="https_proxy=http://127.0.0.1:7897"
Environment="http_proxy=http://127.0.0.1:7897"
Environment="no_proxy=127.0.0.1,localhost"
```
然后 `systemctl --user daemon-reload && systemctl --user restart opentag.service`。

**注意**：方案 B 只能确保 daemon 有代理变量，但 codex 子进程是否继承取决于 daemon 的 shell_environment_policy.filter 是否放行。

### 方案 C：根本修复（需要 daemon 更新）

daemon 需要在 `shell_environment_policy.filters` 白名单中添加 `HTTPS_PROXY`、`HTTP_PROXY`、`NO_PROXY` 及其小写版本。

## 快速诊断命令

```bash
# 检查 codex 是否有代理
cat /proc/$(pgrep -f "codex app-server" | head -1)/environ | tr '\0' '\n' | grep -i proxy

# 检查 codex 的网络连接状态
ss -tnp | grep codex | grep SYN-SENT

# 检查 turn 失败原因
python3 -c "
import json
d=json.load(open('.opentag/data/runtime/durability/turn-report.json'))
r=d[-1]['payload']
print(f\"outcome={r['outcome']}, error={r['errorReason']}, in={r['usage']['inputTokens']}, out={r['usage']['outputTokens']}\")
"
```

## 经验总结

- **provider_failed + 0 token 不止是缺 settings.json**。还有一个原因：codex 运行时缺少代理环境变量，导致网络不通。
- **比对 daemon 和 codex 子进程的环境变量**是快速定位这类问题的关键。
- **`shell_environment_policy.filters` 白名单**会静默丢弃不在列表中的环境变量，即使 daemon 自身有这些变量。
- 代理 `127.0.0.1:7897` 正常时，`curl` 能通过它访问 OpenAI，但 codex 不使用代理 → 就是环境变量未传递的问题。

## 相关 Learnings

- [[opentag-新-agent-provider-failed-全链排障-2026-09-15-pgjzlk]] — settings.json 缺失导致的 provider_failed
- [[opentag-飞书不回复排障]] — WebSocket silent hang 导致的无回复
- [[wsl-daemon故障速查]] — daemon 故障通用诊断