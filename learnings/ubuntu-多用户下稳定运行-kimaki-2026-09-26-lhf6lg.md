---
title: "Ubuntu 多用户下稳定运行 Kimaki"
author: gao
date: 2026-09-26
tags: [kimaki, deployment, troubleshooting, best-practice]
---

## 背景

在 Ubuntu 多用户主机上，`gao` 和 `gaoshan` 的 Kimaki 是两套独立运行环境：各自有 `~/.kimaki` 配置、会话和数据库文件，也可能使用不同的环境变量、工作目录、Node/npm/npx 安装及文件权限。`systemctl --user` 管理的服务也归启动它的 Linux 用户所有。只看 Discord 里的机器人名称，无法判断服务由哪个用户启动。

Discord Bot ID 标识 Discord 应用，不标识 Ubuntu 用户。本次两个用户的日志都显示 Bot ID `1477605701202481173`；这说明日志中的 Bot 应用身份相同，但没有比较 token 的字节内容，不能据此断言两份配置中的 token 完全相同。不要让多个 Kimaki 实例同时使用同一个 Bot 身份/token，以免 Gateway 连接和事件处理相互干扰。

## 解决方案

1. **盘点运行归属。** 用 `ps -eo user,pid,comm` 查看 Kimaki/OpenCode 进程的 Linux 用户；分别检查各用户的 `systemctl --user status kimaki.service` 和该用户的 Kimaki 日志。核对配置、启动命令、工作目录、Node/npm 路径和服务环境。OpenCode 子进程不一定代表另一份 Kimaki，先检查父进程或服务 cgroup 再判断归属。
2. **选定唯一运行账号和启动入口。** 例如由 `gao` 的终端启动，或由 `gao` 的 user service 托管。记录唯一的配置目录与启动方式，避免临时终端、systemd 服务、容器等多条入口并存。
3. **按顺序切换。** 先在旧账号下停止服务或进程，确认旧实例退出，再启动新账号下的实例；不要重叠启动。对于旧账号的 user service，可登录该账号执行 `systemctl --user stop kimaki.service`。需要代执行时，先用 `id -u <用户名>` 获取 UID，再运行 `sudo -u <用户名> env XDG_RUNTIME_DIR=/run/user/<UID> systemctl --user stop kimaki.service`。
4. **分别处理运行状态和开机自启。** `stop` 只停止当前实例；`disable` 只关闭下次登录/启动时的自动启动；`disable --now` 同时禁用并停止。是否禁用取决于是否还要保留旧账号的启动入口。删除 `~/.kimaki` 会清除该用户的数据，不属于切换步骤；只有确认不再需要其中配置、会话和数据库并做好备份后再单独处理。
5. **验收完整链路。** 新实例日志应出现 Discord Connected/Ready；随后在 Discord 发一条真实测试消息，并确认日志没有 `sendMessage` 错误且消息成功送达。Gateway Ready 只证明连接已就绪，不等于发送链路正常。本次 gaoshan 实例已停止并确认清理完成；gao 实例虽显示 Connected/Ready，但 11:50:38 的日志仍有一次 `sendMessage` 失败，因此需要真实收发验证后才能判定恢复正常。

## 经验总结

- 先核实每个进程的 Linux 用户、父子关系和服务归属，再决定停止对象；不要仅凭 Discord Bot 名称或单个 OpenCode 进程判断。
- 多用户意味着配置、数据、环境和服务生命周期隔离；切换账号后，要在目标账号自己的环境中启动并检查日志。
- 同一 Bot 身份只保留一个活跃 Kimaki 实例；切换采用“停旧、确认退出、启新、实测收发”的顺序。
- 停止服务、禁用自启、删除用户数据是三种不同操作，分开决定和执行。
- 分享日志前检查并遮盖 token、密钥、会话内容等敏感信息。

## 相关 Skills

- diagnosing-bugs
- teamai-share-learnings
