---
name: drive-claude-code
description: Use when delegating a task to a separate Claude Code process via the `claude` CLI (print mode or interactive tmux) — e.g. headless runs in an isolated worktree, a different account/settings, or nested orchestration. For ordinary Claude subagents prefer the built-in Agent tool. Load delegate-to-agents first.
---

# Drive Claude Code CLI (nested)

Delegate to a separate [Claude Code](https://code.claude.com/docs/en/cli-reference)
process via the `claude` CLI. Read `delegate-to-agents` first.

Flags verified against **claude 2.1.150**.

## First: do you even need the CLI?

For an ordinary Claude subagent, use the built-in **`Agent` tool** — it gives an
isolated context, returns a summary, supports `run_in_background` and
`isolation: "worktree"`, and needs no subprocess plumbing. That is the default.

Drive the **`claude` CLI** only when you specifically need a *separate OS
process*: a different account/auth or `--settings`, a fully detached long run, a
headless run inside a worktree launched from a `cli-delegate` subagent, or
explicit nested orchestration.

## Preflight

```
command -v claude && claude --version       # need 2.x+
claude auth status --text                    # human-readable login status
```

## One-shot (PREFERRED): `-p` / `--print`

Print mode runs once and exits. It **skips all interactive dialogs** (no
workspace-trust, no permission prompts), so it's the clean automation path.

```
# bounded headless task
Bash(command="cd /repo && claude -p 'Add error handling to all API calls in src/' --allowedTools 'Read,Edit' --max-turns 10", timeout=300000)

# structured JSON result (parse session_id, total_cost_usd, result)
Bash(command="cd /repo && claude -p 'Audit auth.py for security issues' --output-format json --max-turns 5", timeout=300000)

# pipe content in
Bash(command="git -C /repo diff main...feature | claude -p 'Review this diff for bugs and missing tests' --max-turns 1", timeout=180000)

# long WRITE task: detach + log. --permission-mode acceptEdits lets it edit files
# non-interactively; add Bash to --allowedTools (or bypassPermissions) if it must run tests/git.
Bash(command="cd /repo && claude -p 'Refactor the database layer' --permission-mode acceptEdits --allowedTools 'Read,Edit,Write,Bash' --max-turns 20 > /tmp/agent-claude.log 2>&1", run_in_background=true)
```

### Key flags (delegation-relevant)

| Flag | Effect |
|------|--------|
| `-p, --print` | Non-interactive; exits when done |
| `--max-budget-usd <n>` | **Documented** spend cap (print mode) — the primary bound. Very low values are rejected outright (e.g. `0.10` errors "Exceeded USD budget" on some accounts); use a realistic cap like `1` or fall back to `--max-turns` |
| `--max-turns <n>` | Caps agentic loops (print mode). Not shown in `--help` as of 2.1.150 but accepted at runtime — useful, but prefer `--max-budget-usd` as the guaranteed bound |
| `--output-format <fmt>` | `text`, `json`, `stream-json` |
| `--json-schema <schema>` | Structured output |
| `--model <alias>` | `sonnet`, `opus`, `haiku`, or full name |
| `--fallback-model <m>` | Fall back on overload (print mode) |
| `--permission-mode <mode>` | `default`, `acceptEdits`, `plan`, `auto`, `dontAsk`, `bypassPermissions` |
| `--allowedTools` / `--disallowedTools` | Scope tools (e.g. `Read` only for reviews) |
| `--dangerously-skip-permissions` | Auto-approve all (isolated dirs only) |
| `-w, --worktree [name]` / `--tmux` | Isolated git worktree (+ tmux) |
| `-c, --continue` / `-r, --resume <id>` | Resume sessions |
| `--from-pr [n]` | Review/resume a PR-linked session |
| `--bare` | Skip hooks/plugins/MCP/CLAUDE.md (fastest; needs `ANTHROPIC_API_KEY`) |
| `--no-session-persistence` | Don't save the session (print mode) |

## Interactive (tmux)

Use tmux for multi-turn TUI driving (see `delegate-to-agents` Mode 2). Handle
dialogs:

- **Workspace trust** (first visit to a dir): default is "Yes" → send `Enter`.
- **Bypass-permissions warning** (only with `--dangerously-skip-permissions`):
  default is "No" → send `Down` then `Enter`.

```
Bash(command="tmux new-session -d -s cc -x 200 -y 50")
Bash(command="tmux send-keys -t cc 'cd /repo && claude' Enter")
Bash(command="sleep 5 && tmux send-keys -t cc Enter")          # trust → default Yes
Bash(command="sleep 2 && tmux send-keys -t cc 'Refactor auth to JWT, then add tests' Enter")
Bash(command="sleep 30 && tmux capture-pane -t cc -p -S -80")  # a '>' prompt at the bottom = waiting for input
Bash(command="tmux kill-session -t cc")
```

## Parallel Claude processes

```
Bash(command="git -C /repo worktree add -b feat/a /tmp/cc-a main")
Bash(command="git -C /repo worktree add -b feat/b /tmp/cc-b main")
Bash(command="cd /tmp/cc-a && claude -p 'Build feature A' --max-turns 15 > /tmp/cc-a.log 2>&1", run_in_background=true)
Bash(command="cd /tmp/cc-b && claude -p 'Build feature B' --max-turns 15 > /tmp/cc-b.log 2>&1", run_in_background=true)
```

## Rules

1. **Prefer the `Agent` tool** for Claude subagents; use the CLI only for a real
   separate process.
2. **`-p` for automation** — skips dialogs, supports JSON output.
3. **Bound every print-mode run** — `--max-budget-usd` is the documented cap;
   `--max-turns` also works (hidden but accepted) and is worth adding too.
4. **Write tasks need a write flag.** In `-p` mode a plain run can't edit — pass
   `--permission-mode acceptEdits` (file edits) and add `Bash` to `--allowedTools`
   if it must run tests/git. Use `--allowedTools` to restrict to what's needed
   (e.g. `Read` for a review).
5. **One worktree per parallel process**; clean up tmux sessions and logs.
6. **Review and re-run tests yourself** before keeping changes.
