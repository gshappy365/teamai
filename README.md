# teamai

Shared TeamAI resources for the team: agent skills, team knowledge, operating notes, and configuration. This repository contains guidance and shared resources; it is not an application source tree or runtime.

## Repository contents

- `docs/` — agent-facing domain, issue-tracker, and triage guidance.
- `env/` — currently a placeholder directory (`.gitkeep` only).
- `learnings/` — accumulated troubleshooting notes, how-to guides, and operational lessons.
- `mcp/` — MCP server configuration.
- `members/` — team member metadata.
- `rules/` — currently a placeholder directory (`.gitkeep` only).
- `sessions/` — dated team and member session notes.
- `skills/` — skill instructions and their supporting files.
- `stats/` — member skill-usage and activity statistics.

At the repository root, `AGENTS.md` provides agent guidance and `teamai.yaml` identifies the team and shared-resource configuration.

## Local AI tools (user scope)

`teamai-cli` (v0.30+) merges `toolPaths` in `teamai.yaml` per tool over its built-in defaults rather than replacing the full table, so excluding built-in agents is configured per machine in `~/.teamai/config.yaml` (`disabledAgents`).

- **Recommended active agents:** `claude`, `codex`, `pi`, `dsh`, `agy`
- **Excluded agents:** `zcode`, `hermes`, `opencode`, `cursor`

On a new machine, initialize only the active agents:

```bash
teamai init https://github.com/gshappy365/teamai.git --scope user --agent claude,codex,pi,dsh,agy
teamai pull
teamai doctor
```

On an existing machine where excluded agents were already detected, stop delivery and record them in `disabledAgents`:

```bash
for a in zcode hermes opencode cursor; do
  teamai uninstall --agent "$a"
done
teamai doctor
```
