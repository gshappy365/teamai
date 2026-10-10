---
title: "First Tree 客户端 WSL2 环境 WebSocket 离线排障与代理持久化"
author: gshappy365
date: 2026-10-10
tags: [first-tree, wsl2, proxy, websocket, systemd, troubleshooting]
---

## 背景

在 WSL2 环境中运行 First Tree 客户端时，CLI 执行 `first-tree login` 或 `doctor` 提示无法连接到云端服务（`cloud.first-tree.ai`），Web 端控制台持续处于 `Waiting for your computer to connect…` 挂起状态。

## 问题定位与根本原因

通过查看 `/root/.first-tree/logs/client.log` 日志，定位到以下关键问题：

1. **WSL2 DNS 域名解析失败 (`getaddrinfo EAI_AGAIN`)**：
   - `/etc/wsl.conf` 中配置了 `generateResolvConf = false`，但系统缺少有效的 `/etc/resolv.conf` 文件，导致底层 libc 域名解析直接报错，无法解析 `cloud.first-tree.ai`。
2. **Node.js 24 原生 Fetch / WebSocket 默认未开启环境变量代理**：
   - First Tree 捆绑的 Node 24 内部实现的 undici 和 WebSocket 需通过 `--use-env-proxy` 参数才能正确读取 `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY`。
   - `first-tree.service` systemd 后台服务仅配置了基础变量，未注入 `NODE_OPTIONS=--use-env-proxy` 和 `daemon.env` 中的代理配置。
3. **便携式 Shim 完整性强校验**：
   - First Tree portable 模式会在服务安装与刷新时校验 `~/.local/bin/first-tree` 的文件内容字节，若直接在 shim 内修改脚本会触发 `Portable CLI shim does not identify the same portable root` 错误，因此环境参数需通过外部环境变量方式注入。

## 解决方案与实施步骤

### 1. 恢复与持久化 WSL2 DNS 解析
在 `/root/.first-tree/bin/sync-proxy.sh` 脚本中增加动态生成 `/etc/resolv.conf` 逻辑，确保每次 WSL 重启或网关 IP 变动时自动指向宿主机网关并添加公共备份 DNS：
```bash
cat <<EOF > /etc/resolv.conf
nameserver ${host_ip}
nameserver 223.5.5.5
nameserver 8.8.8.8
options edns0 timeout:2 attempts:3
EOF
```

### 2. 完善 systemd 后台服务代理配置
更新 `/etc/systemd/system/first-tree.service.d/proxy.conf`，使守护进程加载代理变量并启用 Node 代理选项：
```ini
[Service]
ExecStartPre=/root/.first-tree/bin/sync-proxy.sh
EnvironmentFile=/root/.first-tree/daemon.env
Environment=NODE_OPTIONS=--use-env-proxy
```

### 3. 全局 Shell 环境代理透传
在 `/etc/profile.d/first_tree.sh` 和 `~/.bashrc` 中声明：
```bash
export NODE_OPTIONS="--use-env-proxy"
```
确保用户无论是交互式终端、脚本还是自动化工具调用 `first-tree` CLI 均能自动继承代理。

## 验证结果

执行 `systemctl restart first-tree` 后检查日志：
- WebSocket 握手成功：`ws: registered`
- 客户端成功注册：`client registered: client_d1d3e951`
- 两个本地 Agent（`mth-wsl-agy-ft` 与 `mth-wsl-codex-ft`）成功绑定（`agent bound`）并上线
- 平台 Web 端控制台恢复正常连接状态。
