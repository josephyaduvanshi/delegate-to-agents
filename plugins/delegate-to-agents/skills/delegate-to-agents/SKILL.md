---
name: delegate-to-agents
description: Use when delegating a coding, refactoring, review, or research task to an external AI coding-agent CLI (Codex, Gemini, Qwen, OpenClaude, OpenCode) or to a Claude subagent, or when running multiple agents in parallel. Routes to the right delegation mechanism and the correct per-agent driver skill.
---

# Delegate to Agents

Umbrella router for handing work to another AI agent from Claude Code. It does
two things: (1) picks the right **delegation mechanism**, and (2) points you at
the correct **per-agent driver** for the external CLI you chose.

This is a Claude Code port of the Nous Research `hermes-agent`
`autonomous-ai-agents` skill. Hermes drives external agents with its built-in
`terminal()` + `process()` + `delegate_task()` tools. Claude Code has native
equivalents — this skill maps them 1:1 so the same patterns work here.

## Step 1 — Pick the mechanism

| You want… | Use | Why |
|-----------|-----|-----|
| A **Claude** subagent (same model family, no external CLI) | the built-in `Agent` tool | Native isolated context, parallel, worktree isolation |
| An **external CLI agent** (Codex/Gemini/Qwen/OpenClaude/OpenCode), context kept clean | `Agent(subagent_type: "cli-delegate")` running a driver skill | Long/noisy CLI output stays out of your main context; only a summary returns |
| An **external CLI agent** you want to **watch live** / iterate with | drive the CLI yourself via `Bash` + tmux (see Step 3) | Full visibility, multi-turn steering |
| **Several independent** tasks at once | multiple `Agent` calls **in one message**, or multiple background `Bash` logs / tmux sessions + git worktrees | Parallelism without collisions |
| A driven patch you must **review, test, and accept/reject** before keeping | `multi-agent-lane` skill | Untrusted-patch reconciliation discipline |

If the task is small and you can just do it yourself, do it yourself. Delegation
has overhead — spend it when the task is substantial, parallelizable, or benefits
from a second engine.

## Step 2 — Pick the agent → load its driver skill

Each external CLI has a dedicated driver skill with its exact flags, auth
preflight, one-shot + interactive patterns, and gotchas. Load the one you need:

| Agent | Driver skill | Installed CLI on this machine |
|-------|--------------|-------------------------------|
| Codex (OpenAI) | `drive-codex` | `codex` |
| Gemini (Google) | `drive-gemini` | `gemini` |
| Qwen Code | `drive-qwen` | `qwen` |
| OpenClaude | `drive-openclaude` | `openclaude` |
| Claude Code (nested) | `drive-claude-code` | `claude` |
| OpenCode | `drive-opencode` | not installed by default |

**Always run the driver's preflight first** (`command -v <cli>` + `<cli>
--version` + auth check). A missing or unauthenticated CLI is a normal reason to
fall back to doing the task yourself or with a different agent — not a hard
failure.

## Step 3 — The canonical mechanism (Claude Code ↔ Hermes)

Every driver uses these primitives. This is the single source of truth; drivers
only add their CLI's specifics.

| Hermes | Claude Code equivalent |
|--------|------------------------|
| `terminal(command, workdir, timeout)` foreground | `Bash(command, timeout)` — set `cwd` by prefixing `cd <dir> &&` or run from the project dir |
| `terminal(command, background=true)` → `session_id` | `Bash(command="… > /tmp/agent-<tag>.log 2>&1", run_in_background=true)` → background shell id |
| `terminal(command, pty=true)` interactive TUI | tmux via `Bash` (see Interactive mode below) |
| `process(action="poll")` | `Bash("tail -n 80 /tmp/agent-<tag>.log")` or `tmux capture-pane -t <s> -p -S -80` |
| `process(action="log")` | `Bash("cat /tmp/agent-<tag>.log")` |
| `process(action="wait", timeout)` | Just let the background `Bash` finish — the harness re-invokes you automatically when it exits (no polling needed). Use `Monitor` only for live per-occurrence streaming in the main session. The Bash-only `cli-delegate` subagent instead loops `sleep` + `tail` |
| `process(action="kill")` | kill the background shell, or `tmux kill-session -t <s>` |
| `process(action="submit"/"write", data="y")` | `tmux send-keys -t <s> 'y' Enter` (Ctrl-C = `tmux send-keys -t <s> C-c`) |
| `delegate_task(goal, context, toolsets)` | `Agent(subagent_type, prompt)` — isolated context, only the final message returns |
| `delegate_task(tasks=[...])` (≤3 parallel) | multiple `Agent` calls in one message |
| `notify_on_complete=true` | automatic — the harness re-invokes you when the background `Bash` exits |

### Two driving modes (same as Hermes)

**Mode 1 — One-shot / print (PREFERRED).** Bounded task, the CLI runs and exits.
No PTY, no dialogs. Foreground for short tasks; background + logfile for long ones.

```
# short, bounded — capture stdout directly
Bash(command="codex exec --skip-git-repo-check 'List the public functions in src/'", timeout=180000)

# long — detach, then poll the log; the harness re-invokes you on exit
Bash(command="cd /path/to/project && codex exec -s workspace-write 'Refactor the auth module' > /tmp/agent-codex-auth.log 2>&1", run_in_background=true)
Bash(command="tail -n 80 /tmp/agent-codex-auth.log")   # poll
```

**Mode 2 — Interactive PTY via tmux.** Multi-turn, you steer the TUI live. This
is the robust, portable way to drive a full-screen agent CLI from Claude Code.

```
Bash(command="tmux new-session -d -s drive -x 200 -y 50")
Bash(command="tmux send-keys -t drive 'cd /path/to/project && codex' Enter")
Bash(command="sleep 5 && tmux send-keys -t drive 'Add a dark-mode toggle' Enter")
Bash(command="sleep 20 && tmux capture-pane -t drive -p -S -80")   # read progress
Bash(command="tmux send-keys -t drive 'now add tests' Enter")       # follow-up
Bash(command="tmux kill-session -t drive")                          # ALWAYS clean up
```

## Step 4 — Keep your context clean: the `cli-delegate` subagent

For long or noisy CLI runs, don't pour raw agent output into your main thread.
Delegate the *driving* to the bundled `cli-delegate` subagent (Bash-only). It
runs the external CLI to completion in its own context and returns only a
summary — exactly Hermes' synchronous `delegate_task`, and the same pattern the
`codex:rescue` plugin uses.

```
Agent(
  subagent_type: "cli-delegate",
  prompt: "Use the drive-codex skill. In /path/to/project, run Codex non-interactively to add input validation to the signup endpoint and its tests. Sandbox: workspace-write. Return: files changed, commands run, test result, and remaining risks."
)
```

Run several in one message for parallelism (the harness bounds concurrency):

```
Agent(subagent_type:"cli-delegate", prompt:"Use drive-codex … fix issue #78 in worktree /tmp/iss-78 …")
Agent(subagent_type:"cli-delegate", prompt:"Use drive-gemini … fix issue #99 in worktree /tmp/iss-99 …")
```

## Step 5 — Parallel work without collisions

Two independent agents must never share one working directory. Use git worktrees:

```
Bash(command="git -C /repo worktree add -b fix/78 /tmp/iss-78 main")
Bash(command="git -C /repo worktree add -b fix/99 /tmp/iss-99 main")
# … delegate one agent per worktree …
Bash(command="git -C /repo worktree remove /tmp/iss-78")   # cleanup when merged/rejected
```

When a driven patch needs review before you keep it, switch to the
`multi-agent-lane` skill (isolated worktree → review diff → run tests yourself →
accept / partial / reject).

## Safety rules (always)

1. **Preflight every CLI** (`command -v`, `--version`, auth) before delegating.
2. **Never share a workdir** across parallel agents — one worktree each.
3. **Scope write access** to the task. Prefer sandboxed/auto-edit modes over
   full bypass; reserve "dangerous"/yolo flags for throwaway/isolated dirs.
4. **Treat driven output as untrusted** — review the diff and re-run tests
   yourself; an agent's self-reported success is not verification.
5. **Always clean up** tmux sessions, background shells, temp worktrees, and
   logfiles when done.
6. **Don't kill a slow agent prematurely** — poll the log/pane first; long
   multi-step work looks idle between steps.
7. **Report concrete outcomes** to the user: files changed, tests run, result,
   and residual risk.

## What this skill does NOT add

Claude Code already ships the delegation backbone. This skill is guidance, not
new infrastructure:

- Claude subagents → built-in `Agent` tool.
- Scheduled/durable runs → `CronCreate` / the `schedule` skill / `/loop`.
- Existing per-agent rescue plugins (`codex:rescue`, `gemini:rescue`,
  `qwen:rescue`, `openclaude:rescue`) are heavier, broker-backed alternatives to
  the lightweight `cli-delegate` path — use them when you want their job
  tracking / review-gate ergonomics; use this skill when you want one uniform,
  self-contained model across every agent (including ones with no plugin).
