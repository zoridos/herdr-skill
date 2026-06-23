---
name: herdr
description: >
  Control Herdr from inside a Herdr-managed pane — split panes, spawn agents,
  read another pane's output, and wait for state changes via CLI commands over
  a local Unix socket. Use this skill whenever HERDR_ENV=1 is set, or whenever
  the task involves running things in parallel panes, waiting for a test/server/
  agent to finish, reading what another agent is doing, or coordinating between
  multiple Claude/Codex/opencode instances in the same Herdr session. If you're
  orchestrating agents and HERDR_ENV=1, default to this skill rather than
  improvising shell patterns.
---

# herdr — agent skill

Before using any herdr commands, check that `HERDR_ENV=1`. If it is not set, say you are not running inside a Herdr-managed pane and stop — do not attempt to inspect or control panes from outside Herdr.

You are running inside Herdr, a terminal-native agent multiplexer. Herdr gives you workspaces, tabs, and panes — each pane is a real terminal running its own shell, agent, server, or log stream — and you can control all of it from the CLI.

This means you can:

- See what other panes and agents are doing
- Create tabs for separate subcontexts inside one workspace
- Split panes and run commands in them without stealing your focus
- Start servers, watch logs, and run tests in sibling panes
- Block until specific output appears, rather than polling with sleep
- Block until another agent finishes
- Spawn more agent instances

The `herdr` binary is in your PATH. Its commands talk to the running Herdr instance over a local Unix socket.

## Concepts

**Workspaces** are project contexts. Each workspace has one or more tabs. A workspace's label follows the first tab's root pane — usually the repo name, otherwise the root pane's current folder.

**Tabs** are subcontexts inside a workspace. Each tab has one or more panes.

**Panes** are terminal splits inside a tab. Each pane runs its own process.

**Agent status** is detected automatically. The values are: `idle`, `working`, `blocked`, `done`, `unknown`. `done` means the agent finished but you haven't looked at that pane yet.

**IDs** — workspace IDs look like `1`, `2`. Tab IDs look like `1:1`, `1:2`. Pane IDs look like `1-1`, `1-2`, `2-1`.

IDs compact when panes/tabs/workspaces close — never treat them as durable. Re-read IDs from `workspace list`, `tab list`, or `pane list` before using them. Don't guess that `1-3` from earlier is still the same pane.

## Discover yourself

See what panes exist and which one is focused (yours):

```bash
herdr pane list
```

List workspaces:

```bash
herdr workspace list
```

## Read another pane

```bash
herdr pane read 1-1 --source recent --lines 50
```

- `--source visible` — current viewport
- `--source recent` — recent scrollback as rendered
- `--source recent-unwrapped` — recent text with soft wraps rejoined (best for pattern matching)

## Split a pane and run a command

Always use `--no-focus` when splitting so your terminal context stays focused:

```bash
NEW_PANE=$(herdr pane split 1-2 --direction right --no-focus | python3 -c 'import sys,json; print(json.load(sys.stdin)["result"]["pane"]["pane_id"])')
herdr pane run "$NEW_PANE" "npm run dev"
```

Split downward instead of right:

```bash
herdr pane split 1-2 --direction down --no-focus
```

## Wait for output

Block until specific text appears in a pane — far better than polling with sleep:

```bash
herdr wait output 1-3 --match "ready on port 3000" --timeout 30000
```

With regex:

```bash
herdr wait output 1-3 --match "server.*ready" --regex --timeout 30000
```

Use `--source recent-unwrapped` to match against the same unwrapped transcript that `wait output` uses internally. Exit code is `1` on timeout.

## Wait for an agent to finish

```bash
herdr wait agent-status 1-1 --status done --timeout 60000
```

Or wait for idle (still running but no active task):

```bash
herdr agent wait <target> --status idle --timeout 60000
```

`done` and `idle` are distinct: `done` means the agent session ended; `idle` means it's waiting for input.

## Send text or keys to a pane

```bash
herdr pane send-text 1-1 "some text"   # no Enter
herdr pane send-keys 1-1 Enter          # press Enter
herdr pane run 1-1 "echo hello"         # send text + Enter in one shot
```

## Tab management

```bash
herdr tab list --workspace 1
herdr tab create --workspace 1 --label "logs"
herdr tab rename 1:2 "logs"
herdr tab focus 1:2
herdr tab close 1:2
```

## Workspace management

```bash
herdr workspace create --cwd /path/to/project --label "api server"
herdr workspace focus 2
herdr workspace rename 1 "api server"
herdr workspace close 2
```

## Close a pane

```bash
herdr pane close 1-3
```

## Recipes

### Spawn a named agent and give it a task

`herdr agent start` is the right command when you want to spawn another coding agent. It creates a new pane, launches the agent, and registers it under a name you can reuse — so you never need to track pane IDs afterward.

```bash
herdr agent start reviewer --cwd ~/project --split right -- claude
```

- `reviewer` is the name you'll use to address this agent from now on
- `--split right` opens the pane to the right; use `--split down` for vertical
- Default is `--no-focus` — your terminal stays focused

Send it a task without switching panes:

```bash
herdr agent send reviewer "review the test coverage in src/api/"
```

Read its output without switching panes:

```bash
herdr agent read reviewer --source recent --lines 50
```

Wait for it to finish:

```bash
herdr agent wait reviewer --status idle --timeout 120000
```

### Spawn a low-level pane and run a command

When you need a pane without agent registration (e.g. a server or test runner):

```bash
NEW_PANE=$(herdr pane split 1-2 --direction right --no-focus | python3 -c 'import sys,json; print(json.load(sys.stdin)["result"]["pane"]["pane_id"])')
herdr pane run "$NEW_PANE" "npm run dev"
```

Always parse the pane ID from the JSON response — never assume the next sequential number.

### Run a server and wait until it's ready

```bash
NEW_PANE=$(herdr pane split 1-2 --direction right --no-focus | python3 -c 'import sys,json; print(json.load(sys.stdin)["result"]["pane"]["pane_id"])')
herdr pane run "$NEW_PANE" "npm run dev"
herdr wait output "$NEW_PANE" --match "ready" --timeout 30000
herdr pane read "$NEW_PANE" --source recent --lines 20
```

### Run tests in a separate pane and inspect the result

```bash
NEW_PANE=$(herdr pane split 1-2 --direction down --no-focus | python3 -c 'import sys,json; print(json.load(sys.stdin)["result"]["pane"]["pane_id"])')
herdr pane run "$NEW_PANE" "cargo test"
herdr wait output "$NEW_PANE" --match "test result" --timeout 60000
herdr pane read "$NEW_PANE" --source recent --lines 30
```

### Coordinate with another agent — wait then read

`done` means the agent session exited. `idle` means the agent is still running but waiting for input. Choose based on what you expect the agent to do when finished:

```bash
# codex exited after finishing — use done
herdr wait agent-status 1-1 --status done --timeout 120000
herdr pane read 1-1 --source recent-unwrapped --lines 100

# claude is staying open, waiting for next prompt — use idle
herdr agent wait reviewer --status idle --timeout 120000
herdr agent read reviewer --source recent-unwrapped --lines 100
```

Use `--source recent-unwrapped` when you want to inspect or pattern-match the transcript — it rejoins soft-wrapped lines so patterns don't break at terminal width.

### Watch a pane robustly

```bash
# inspect what's already there
herdr pane read 1-3 --source recent --lines 40

# block until the next expected output
herdr wait output 1-3 --match "ready" --timeout 30000

# read the same unwrapped transcript the waiter matched against
herdr pane read 1-3 --source recent-unwrapped --lines 40
```

## Notes

- Most commands print JSON on success. `pane read` prints text, not JSON.
- Parse new IDs from create/split responses: `workspace create` returns `result.workspace`, `result.tab`, `result.root_pane`; `tab create` returns `result.tab` and `result.root_pane`; `pane split` returns `result.pane.pane_id`.
- `pane read` reads output that already exists. `wait output` blocks for future output. Use the right one for the situation.
- `--no-focus` on split/tab-create/workspace-create keeps your terminal focused.
- `pane read --format ansi` returns a rendered ANSI snapshot, useful for TUI feedback loops.
- For raw protocol or the full socket API reference: https://herdr.dev/docs/socket-api/
