---
title: "OpenTag WSL 中 Codex 代理继承与 Computer 离线排障"
author: gshappy365
date: 2026-09-20
tags: [opentag, wsl, codex, proxy, websocket]
---

## 问题

OpenTag 后台显示“Computer 已关闭”，飞书消息无法正常触发或回复；同时 Codex turn 长时间显示进行中，没有已发送的回复。

## 根因

这是两个网络层级容易混淆的问题：

1. OpenTag daemon 自己有代理环境，但 OpenTag 0.0.5 启动 Codex app-server 时，`codexAgentRuntimeEnvironment()` 没有保留 `HTTP_PROXY`、`HTTPS_PROXY`、`NO_PROXY` 及小写变量。
2. Codex app-server 因此绕过本地代理，直接连接模型服务，连接长期停在 `SYN-SENT`；独立启动的 Codex 则通过 `127.0.0.1:7897` 正常连接。
3. daemon 重启后，后台可能短暂显示旧的“Computer 已关闭”状态；应以本地 daemon 的注册日志和 WebSocket 状态为准，而不是立即重新 `connect`。

## 诊断方法

```bash
# 1. daemon 与服务定义
opentag daemon status --json
opentag doctor
systemctl --user show opentag.service -p ActiveState -p SubState -p MainPID

# 2. 代理端口和 OpenTag Server
timeout 3 bash -c '</dev/tcp/127.0.0.1/7897'
curl -sS -L --max-time 12 -o /dev/null \
  -w 'http_code=%{http_code} remote_ip=%{remote_ip} time=%{time_total}s\\n' \
  https://app.opentag.build

# 3. 比较 daemon 与 Codex 子进程的代理环境
tr '\\0' '\\n' < /proc/<daemon-pid>/environ | grep -i proxy
tr '\\0' '\\n' < /proc/<codex-app-server-pid>/environ | grep -i proxy

# 4. 查看实际网络连接
ss -tnp | grep -E 'codex|SYN-SENT|127.0.0.1:7897'
```

关键判据：daemon 环境有代理、Codex app-server 环境没有代理，同时 Codex socket 对模型地址为 `SYN-SENT`，而独立 Codex 连接本地代理并为 `ESTAB`。

## 修复

1. 先修复服务定义：

   ```bash
   opentag daemon install
   ```

   验证 `drifted:false`，并确认日志出现 `Runtime connection registered` 和 `Computer runtime is ready`。

2. 如果仍使用 OpenTag 0.0.5，需要让 Codex runtime 白名单保留代理变量。当前没有更高的 prod channel target；本机对安装 bundle 做了可回滚的补丁，加入大写和小写的 HTTP(S) proxy 变量。

3. 重启 daemon 会取消正在进行的 turn，因此应先确认该 turn 已经没有进展，再执行重启。重启后重新提交消息，不要依赖旧 turn 自动恢复。

## 避免复发

- 每次重启或 WSL 恢复后，先运行 `opentag doctor`，确保 daemon、Server health、Context Tree 全部通过。
- 不要只检查 daemon 的代理；必须比较 daemon 和 Codex app-server 的 `/proc/<pid>/environ`。
- 看到“Computer 已关闭”时，先检查最近是否有 `Runtime connection registered`、WebSocket `ESTAB` 和 `drifted:false`；这些都正常时先强制刷新后台页面，不要重复 `opentag connect`，以免生成新的 Computer identity。
- 若日志超过 10 分钟无新连接或 turn 事件，再考虑 `systemctl --user restart opentag.service`。
- OpenTag 升级可能覆盖本地 bundle 补丁；升级后要重新执行上述 Codex 子进程代理继承检查。

## 相关经验

- `opentag-codex-运行时缺失代理-2026-09-19-yxs0hw`
- `wsl-daemon故障速查`
- `opentag-新-agent-provider-failed-全链排障-2026-09-15-pgjzlk`
