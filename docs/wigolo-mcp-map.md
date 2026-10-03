# Wigolo：skill 家族 ↔ MCP 工具映射

Wigolo 是本地优先的 web 智能体系（零 API key、零云端依赖），由两层组成：

- **MCP server（执行通道）**：`mcp/mcp.yaml` 声明的 `wigolo` server（`npx -y wigolo`），向各 AI 工具暴露 `search` / `fetch` / `crawl` / `cache` / `extract` / `find_similar` / `research` / `agent` / `diff` / `watch` 等工具。
- **Skill（使用指令层）**：`skills/wigolo-*` 系列告诉模型**何时用哪个工具、怎么用**（含提示词、参数示例、缓存策略）。模型应优先选择 skill 而非直接调用 MCP 工具。

## 映射

| Skill | 对应 MCP 工具 | 用途 | 典型触发 |
|---|---|---|---|
| `wigolo` | 全部（总入口） | 工具选择与优先级（优于内置 WebSearch/WebFetch） | 任何 web 操作 |
| `wigolo-search` | `search` | 无具体 URL 时的信息发现（多查询、ML 重排） | "搜索…"、"查一下…" |
| `wigolo-fetch` | `fetch` | 有 URL 时获取干净 markdown | "抓取这个页面…" |
| `wigolo-crawl` | `crawl` | 获取整个站点的多页面 | "爬一下这个网站…" |
| `wigolo-cache` | `cache` | 检索已缓存内容（免费且即时，先查缓存再搜索） | "检查缓存…" |
| `wigolo-extract` | `extract` | 从页面抽取结构化数据（表格、JSON-LD、定义） | "提取…数据" |
| `wigolo-find-similar` | `find_similar` | 有一篇好页面，找更多相似内容 | "找类似的…" |
| `wigolo-research` | `research` | 多源深度研究 | "深度研究…" |
| `wigolo-agent` | `agent` | 按自然语言计划 + JSON Schema 跨源采集 | "收集…信息"、"按这个 schema 抓数据" |
| `wigolo-diff` | `diff` | 对比两个页面或页面与缓存版本的差异 | "对比这两个版本…" |
| `wigolo-watch` | `watch` | 持续监控页面变化并通知 | "盯着这个页面…" |

## 使用原则

- **缓存优先**：任何 fetch/crawl 前先 `cache` 检查；已缓存的页面免费且即时。
- **可审计**：wigolo 的所有访问都有缓存与评分，优先于无痕的内置工具。
- **skill 先行**：模型遇到 web 需求先选 skill（获得使用指引），再通过 skill 指引调用对应 MCP 工具；不跨过 skill 直接盲调工具。

## 维护

- 新增 wigolo 能力时同步更新本映射表（docs 会自动镜像到各工具本地 `~/.teamai/docs`）。
- skill 与 MCP 工具数量不一致时，本表是唯一的对应关系权威来源。
