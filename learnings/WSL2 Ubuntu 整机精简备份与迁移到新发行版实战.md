---
title: "WSL2 Ubuntu 整机精简备份与迁移到新发行版实战"
author: gshappy365
date: 2026-09-20
tags: [wsl, backup, migration, devops, troubleshooting]
---

## 背景

WSL2 里的 Ubuntu(旧,8.8G)需要迁移到新安装的 Ubuntu2(Ubuntu 26.04.1 LTS)。目标:完整保留 kimaki、opentag(含 context-tree 全部配置)、teamai、codex/claude/opencode 的会话聊天内容与相关文件,同时剔除可重建内容(node_modules、nvm/npm/pnpm store、claude versions、opencode/opentag 运行时、各类 caches、~/.opentag-dev 开发渠道),实现"整机精简备份"。

## 解决方案

1. **整机精简备份**:`backup-userdata.sh` 用 tar 打包 /home/gaoshan,配一组 --exclude 剔除全部可重建目录;产物 94M / 5172 项。备份前自动停掉 kimaki/opentag 服务,保证 SQLite 一致。
2. **新系统安装并存**:`wsl --install Ubuntu-26.04 --name Ubuntu2 --location D:\WSL\Ubuntu2 --no-launch` 成功(注意:仅用命名参数 `-n Ubuntu2` 会报 WSL_E_DISTRO_NOT_FOUND)。root 预建用户 gaoshan + 临时密码,`wsl --manage Ubuntu2 --set-default-user gaoshan`。
3. **一键还原**:`restore-userdata.sh` 解压备份 → 恢复被剔除的运行时 tar → nvm + node v24.21.0 → npm 全局 kimaki/pnpm → 重装 opencode → pnpm install → systemd 用户服务 enable。
4. **运行时零差异恢复**:被剔除的 opentag/claude/codex 程序本体,从旧 Ubuntu vhdx 打包(`runtime-restore.tar.gz`,514M),新系统解压即版本一致(claude 2.1.277、opentag 0.0.5 portable node v24.19.0、codex 0.154.0),比重新下载更可靠。
5. **验证**:kimaki 100 个 Discord 命令注册成功、opencode serve 子进程被拉起、opentag daemon 稳定(NRestarts=0);数据完整性核验:codex 18 会话、claude 7 jsonl、opencode.db 52M、context-tree 35 文件、.env.local/setup-env.sh 全部在位;opentag agent 复活后自报环境与旧系统完全一致。

## 经验总结

- **两个 systemd Ubuntu 不能同时运行**:新旧系统用户同为 UID 1000,共享内核 cgroup,后启动者的 `user@1000.service` 报 `Failed to spawn executor: Device or resource busy`(EBUSY)。并存期间必须 `wsl --terminate` 一个再启动另一个。
- **PowerShell 外层引号包 `wsl -e bash -lc '...'` 会拆坏 `$`、`$()`、内层双引号和转义**(实测 `tr -d "\r"` 变成删所有字母 r)。复杂命令一律写成 .sh 放到 /mnt/d 再 `wsl -d X -e bash /mnt/d/xxx.sh` 执行。
- **nohup/disown 经 wsl.exe 启动不驻留**,服务必须用 systemd 用户服务(`systemctl --user` + `export XDG_RUNTIME_DIR=/run/user/$(id -u)`,或 root 用 `-M gaoshan@`)。
- **kimaki 自升级会误调 Windows 侧 npm**,升级不到 WSL 内版本,形成升级-退出循环;须在 WSL 内 `npm i -g`。kimaki 服务还需 bun、opencode 在 PATH。
- **tar 备份运行中 SQLite 会漏文件**,必须先停服务再打包。
- **WSL 发行版"默认"标记只影响敲 `wsl` 进哪个,不释放磁盘空间**;磁盘空间只有 `wsl --unregister` 才释放。Stopped 的发行版不占内存(WSL2 共享 VM,按需启动)。
- **opentag 是 portable 模式,自带独立 node 运行时**(v24.19.0,与 nvm 的 v24.21.0 是两套,互不影响);其数据 home 在 `~/.opentag`(工作区 data/workspaces、context-tree CLI 在 ~/.opentag/context-tree/bin、provider-cli 双层 shim,lark-cli 1.0.92),树本体在 `~/.context-tree/trees/`。
- **WSL 内 Grep 工具读 `\\wsl$\...` 网络路径返回 0 匹配**,必须回 bash 文件方式处理。

## 相关 Skills

- teamai-share-learnings
- opentag(含 context-tree)
- wsl / bash / systemd
