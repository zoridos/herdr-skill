# herdr skill

An agent skill for [Herdr](https://herdr.dev) — a terminal-native agent multiplexer. Gives coding agents (Claude, Codex, OpenCode, etc.) the ability to control their Herdr environment: split panes, spawn agents, read pane output, and wait for state changes — all via CLI commands over a local Unix socket.

## What it does

When running inside a Herdr-managed pane (`HERDR_ENV=1`), agents can:

- Read output from any other pane or agent
- Wait for specific output patterns without polling with `sleep`
- Wait for another agent to reach `idle` or `done` status
- Spawn additional agent instances into named panes
- Send tasks to other agents without leaving the current pane
- Split panes and run servers, tests, or watchers in the background

## Prerequisites

- [Herdr](https://herdr.dev) installed and running
- `HERDR_ENV=1` set in your agent's environment

## Installation

### Claude Code

```sh
mkdir -p ~/.claude/skills/herdr
curl -sL https://raw.githubusercontent.com/jransom87/herdr-skill/main/SKILL.md \
  -o ~/.claude/skills/herdr/SKILL.md
```

Add to `~/.claude/settings.json`:
```json
{
  "env": {
    "HERDR_ENV": "1"
  }
}
```

### Other agents

Place `SKILL.md` in your agent's skills directory:

| Agent     | Skills directory              |
|-----------|-------------------------------|
| Claude    | `~/.claude/skills/herdr/`     |
| Clawde    | `~/.clawde/skills/herdr/`     |
| Codex     | `~/.codex/skills/herdr/`      |
| OpenCode  | `~/.config/opencode/skills/herdr/` |
| Pi        | `~/.config/pi/skills/herdr/`  |
| Hermes    | `~/.config/hermes/skills/herdr/` |

## Usage examples

**Spawn an agent and give it a task:**
```bash
herdr agent start reviewer --cwd ~/project -- claude
herdr agent send reviewer "review test coverage in src/api/"
```

**Wait for another agent to finish, then read its output:**
```bash
herdr wait agent-status 1-1 --status done --timeout 120000
herdr pane read 1-1 --source recent-unwrapped --lines 100
```

**Run a server and wait until it's ready:**
```bash
NEW_PANE=$(herdr pane split 1-1 --direction right --no-focus | \
  python3 -c 'import sys,json; print(json.load(sys.stdin)["result"]["pane"]["pane_id"])')
herdr pane run "$NEW_PANE" "npm run dev"
herdr wait output "$NEW_PANE" --match "ready on port" --timeout 30000
```

## License

MIT
