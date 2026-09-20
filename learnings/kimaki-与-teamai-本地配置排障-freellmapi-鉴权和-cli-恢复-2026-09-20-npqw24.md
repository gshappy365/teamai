---
title: "Kimaki 与 TeamAI 本地配置排障：FreeLLMAPI 鉴权和 CLI 恢复"
author: gshappy365
date: 2026-09-20
tags: [kimaki, teamai, freellmapi, systemd, troubleshooting]
---

## 问题

Kimaki 已将频道模型切换为 `freellmapi/deepseek-v4-pro`，但 OpenCode session 持续返回 `Invalid API key (401)`。同时，本机 TeamAI hook 调用的 `teamai` CLI 报 `MODULE_NOT_FOUND`。

## 定位方法

1. 检查 Kimaki 的 systemd 服务环境，而不是只检查交互式 shell。
2. 用本地 FreeLLMAPI 的 `/v1/models` 做最小鉴权验证：无凭证请求返回 401；带 shell 中已有凭证请求返回 200，且 `deepseek-v4-pro` 可用。
3. 检查 TeamAI 启动器指向的全局 Node 模块路径是否存在。

## 根因

- OpenCode 配置使用 `{env:FREELLMAPI_API_KEY}`，但 Kimaki 的 systemd 服务没有继承交互式 shell 环境变量。
- `~/.teamai/bin/teamai` 指向 `teamai-cli/dist/index.js`，但对应的全局 `teamai-cli` 包已被删除。

## 修复

- 使用 systemd user service drop-in 的 `EnvironmentFile` 注入 FreeLLMAPI 密钥；密钥文件权限设置为 `600`，不写入 OpenCode 配置或日志。
- 执行 `systemctl --user daemon-reload` 后重启 Kimaki；进程环境和带凭证的 `/v1/models` 请求均恢复正常。
- 安装 `teamai-cli@0.24.0` 到当前 Node 全局目录，使 `teamai --version` 恢复为 `0.24.0`，OpenCode/Kimaki PATH 可以解析该命令。

## 后续注意

- TeamAI doctor 仍提示 GitHub CLI 未认证；远程 push/pull 需要先完成 `gh auth login` 或修复 SSH credential helper。
- TeamAI 状态显示本地有未推送的 skills，贡献知识前应检查分支和无关改动。
- 不要在 learnings、日志或聊天中记录实际 API Key。
