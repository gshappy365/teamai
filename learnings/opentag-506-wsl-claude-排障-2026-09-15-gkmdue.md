# WSL 环境下 OpenTag daemon 常见故障速查

## 故障 1: Provider 鉴权失败（0 token / Not logged in）
- 根因: daemon 传 `--setting-sources project`，claude 只读 `<workspace>/.claude/settings.json`
- 修复: workspace 下创建 `./claude/settings.json`，含 `ANTHROPIC_BASE_URL` + `ANTHROPIC_AUTH_TOKEN`

## 故障 2: 消息不触发 turn（daemon 静默假活）
- 根因: WebSocket silent hang，无 connection lost 事件，但消息投递中断
- 特征: 日志 >10min 无新条目，turn-report.json 停滞
- 修复: `systemctl --user restart opentag.service`

## 故障 3: GitHub push TLS 失败
- 根因: WSL 直连 GitHub https 被 GnuTLS 拒绝
- 修复: `git -c http.proxy=http://127.0.0.1:7897 push`

## 常用诊断命令
```
# daemon 状态
systemctl --user status opentag.service

# WebSocket 活跃度
tail -f ~/.opentag/logs/client.log | grep 'turn\|connection'

# 最近 turn 结果
python3 -c "import json;d=json.load(open('.opentag/data/runtime/durability/turn-report.json'));\
  [print(r['updatedAt'],r['payload']['outcome']) for r in d[-3:]]"

# FreeLLMAPI 健康
curl http://127.0.0.1:3001/api/ping
```