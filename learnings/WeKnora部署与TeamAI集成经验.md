---
title: "WeKnora 部署与 TeamAI 集成经验"
author: gshappy365
date: 2026-09-15
tags: [weknora, teamai, docker, mcp, wsl, rag, freellmapi]
---

## 背景

在本地 WSL2（Ubuntu 26.04）部署腾讯 WeKnora（RAG 知识库）并接入 TeamAI 工作流：目标是 Docker 可用 → WeKnora 服务跑起来 → 导入团队知识 → CLI 可用 → 通过 MCP 暴露给 Claude / Codex / OpenCode。全程在 Windows PowerShell 通过 `wsl -d Ubuntu -e ...` 驱动。大模型使用 freellmapi（本地 OpenAI 兼容网关），不使用 Ollama。

## 解决方案

### 1. Docker：弃用 Docker Desktop WSL 集成，改用原生 Engine

- Docker Desktop 的 WSL 集成反复失败（`Wsl/Service/0x8007274c` 连接超时、proxy binary `install: No such file or directory`），判定为竞态不可修 → 弃用，配置恢复默认并备份 `settings-store.json.bak-weknora`。
- 原生安装（Ubuntu 源，download.docker.com 经代理 TLS 不通）：`docker.io` + `containerd` + `docker-compose-v2`；`systemctl enable --now containerd docker`。
- dockerd 走 Hub 直连超时 → drop-in `/etc/systemd/system/docker.service.d/http-proxy.conf`（HTTP_PROXY/HTTPS_PROXY=http://127.0.0.1:7897，NO_PROXY 私网段）→ daemon-reload + restart 后 pull 全通。
- 坑：`~/.docker/config.json` 里 Docker Desktop 残留 `currentContext=desktop-linux` 会导致 WSL 内报 "protocol not available"，备份后改为 `default`。

### 2. WeKnora 部署（docker compose）

- 克隆 `/home/gao/services/WeKnora`；`.env` 关键项：`FRONTEND_PORT=8088`、`APP_PORT=18080`、`STORAGE_TYPE=local`、`RETRIEVE_DRIVER=postgres`（无需额外向量库）。
- 镜像：`weknora-app`、`weknora-ui`、`weknora-docreader`、`paradedb:pg17`、`redis:7.0-alpine`；compose pull 大层可能 EOF 中断，重试即可。
- `docker compose up -d` 后 5 容器全 Up，app/postgres/docreader healthy。
- 坑：WSL 内 nohup/& 后台进程会在 wsl.exe 会话退出后被回收，需用 PowerShell `Start-Process` 持有 wsl.exe 跑后台任务，日志定向到 Windows 文件。

### 3. CLI 构建与认证

- 需要 Go 1.26+；`cd cli && go build -o weknora .`，install 到 `/usr/local/bin/weknora`。
- stateless 认证：`WEKNORA_API_KEY` + `WEKNORA_HOST`；`weknora profile add weknora-local --host http://127.0.0.1:18080 --use` + `echo 'sk-...' | weknora auth login --with-token`。
- keychain 不可用时凭证落 `$XDG_CONFIG_HOME/weknora/secrets/` 0600 文件，`weknora doctor` 的 warn 属正常。

### 4. 大模型接入 freellmapi

- 容器内访问 Windows 侧 freellmapi（127.0.0.1:3001）不可达 → 装 socat 建 systemd 服务 `freellm-forward.service`：`172.18.0.1:3001 → 127.0.0.1:3001`。
- 建模型 API 时 SSRF 拦截直连 IP → `.env` 追加 `SSRF_WHITELIST_EXTRA=172.18.0.1,172.18.0.0/16,127.0.0.1,localhost` 后 `docker compose up -d --force-recreate app`。
- chat 模型（如 `deepseek-v4-pro`，base_url `http://172.18.0.1:3001/v1`，provider generic）可用；embedding 模型（如 bge-m3）可注册绑定，但 freellmapi 侧若无可用 embedding key 则无法真正向量化。

### 5. MCP 接入 TeamAI

- team-repo 写 `mcp/mcp.yaml`（schema：`servers[].{name, description?, transport, command?, args?, env?, timeout?, requires?}`，stdio 必须带 command）：
  ```yaml
  servers:
    - name: weknora
      transport: stdio
      command: weknora
      args: [mcp, serve]
      env:
        WEKNORA_HOST: "http://127.0.0.1:18080"
        WEKNORA_API_KEY: "${WEKNORA_API_KEY}"
      timeout: 60
      requires: [weknora]
  ```
- `teamai mcp inject`：`${WEKNORA_API_KEY}` 在注入时被解析为字面值（不支持透传），因此执行注入的 shell 必须先 export 该变量，否则 dry-run 报 "unresolved variable(s)" 全部跳过。
- 注入成功后 claude / codex / opencode 三客户端配置均写入；重启客户端会话生效。
- 验证：`opencode mcp list` 应显示 `✓ weknora connected`；也可 `printf '{"jsonrpc":"2.0","id":1,"method":"initialize",...}' | weknora mcp serve` 手动握手。
- `weknora skills install` 可把内置 Agent Skills 写入 `~/.claude/skills`。

## 经验总结

- **WeKnora 文档导入强制 embedding**：检索使用向量/关键词混合索引（`NeedsEmbedding = VectorEnabled || KeywordEnabled`），无可用 embedding 模型时上传的文档 parse 会失败（`All providers for embedding family 'X' failed (no usable keys)`），KB 已有文档后服务端拒绝更换 embedding 模型。
- **freellmapi embedding 需要独立上游 key**：chat 可用不代表 embedding 可用；若所有 embedding 家族报 "no usable keys"，需在 freellmapi「嵌入模型」页为 provider（Google/Cloudflare/OpenRouter 等）添加 key。
- **MCP 声明引用环境变量会被展开写死**：`${VAR}` 注入时解析为字面值写入各客户端配置，key 会以明文落在 ~/.claude.json / ~/.codex/config.toml / ~/.config/opencode/opencode.json，注意权限与泄露风险。
- **Windows 侧不需要重复配 agent**：全部使用 WSL 内的 Claude / Codex / OpenCode 即可，Windows 侧 opencode 不必加 wsl 桥接。
- **所有过程脚本、日志、SQLite 备份留存在 Windows 目录**便于复查（本项目在 `C:\Users\李平\DoubaoWork\chats\2026-09-15\new-chat-1\`）。

## 相关 Skills

- weknora（CLI / MCP，`/usr/local/bin/weknora`）
- teamai（CLI，`/home/gao/.npm-global/bin/teamai`）
