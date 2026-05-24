# delegate-to-agents

A Claude Code plugin for handing work to other AI agents. Use it to send a
coding or research task to an external agent CLI (Codex, Gemini, Qwen,
OpenClaude, OpenCode) or to a Claude subagent, run a few of them in parallel, and
review what they hand back before you keep it.

It's a port of the
[`autonomous-ai-agents`](https://github.com/NousResearch/hermes-agent/tree/main/skills/autonomous-ai-agents)
skill from Nous Research's `hermes-agent`. Hermes drives other agents with its
own `terminal()`, `process()`, and `delegate_task()` tools. Claude Code already
has equivalents for all three, so this is mostly translation: the same patterns,
rewritten against Claude Code's native `Bash` (background mode), `Monitor`, tmux,
and the `Agent` tool. Nothing runs in the background as a daemon. It's plain
skill files on top of tools you already have.

## Why bother

Claude Code can already delegate. The `Agent` tool spawns Claude subagents, and
there are separate rescue plugins for Codex, Gemini, Qwen, and OpenClaude. What
was missing is a single place that ties it together: something that knows when to
reach for the `Agent` tool versus an external CLI, gives each agent a driver with
that agent's actual flags, and keeps the parallel and review patterns in one
spot. So instead of remembering six different CLIs and which rescue plugin does
what, you say "delegate this to Codex" and it routes.

## What's inside

- `delegate-to-agents` — the router. Picks the mechanism and points you at the
  right driver. Holds the Hermes-to-Claude-Code mechanism mapping so the rest of
  the skills don't repeat it.
- `drive-codex`, `drive-gemini`, `drive-qwen`, `drive-openclaude`,
  `drive-claude-code`, `drive-opencode` — one driver per agent. Each has that
  CLI's flags, an auth preflight, the one-shot and interactive patterns, and the
  gotchas worth knowing.
- `multi-agent-lane` — for when a driven patch needs reviewing before you trust
  it. Isolate in a git worktree, read the diff, run the tests yourself, then
  accept, partially accept, or reject. Generalized from Hermes' `kanban-codex-lane`.
- `cli-delegate` — a Bash-only subagent that drives a CLI to completion and
  returns a summary, so a long or noisy run never floods your main context.
- `/delegate` — a one-line entry point: `/delegate codex fix the failing auth test`.

## How it maps to Hermes

| Hermes | Claude Code |
|--------|-------------|
| `terminal(command, background=true)` | `Bash(run_in_background=true)` writing to a logfile |
| `terminal(pty=true)` for a TUI | tmux driven through `Bash` |
| `process(poll / log / kill)` | `tail` the logfile, or `tmux capture-pane`; kill the shell or session |
| `process(wait)` | let the background `Bash` exit; the harness re-invokes you |
| `delegate_task(goal)` | `Agent(subagent_type, prompt)` |
| `delegate_task(tasks=[...])` | several `Agent` calls in one message |

The `delegate-to-agents` skill has the full table.

## Install

```
/plugin marketplace add josephyaduvanshi/delegate-to-agents
/plugin install delegate-to-agents@delegate-to-agents
```

Or try it for a single session without installing, by cloning the repo and
pointing Claude Code at the plugin directory:

```
git clone https://github.com/josephyaduvanshi/delegate-to-agents
claude --plugin-dir delegate-to-agents/plugins/delegate-to-agents
```

## Use

Plain language works once the plugin is installed:

> delegate the auth refactor to Codex and keep my context clean

So does the command:

```
/delegate gemini review src/server.ts for security issues
/delegate --review codex migrate the tests to pytest
/delegate --bg qwen add integration tests for the API
```

`--review` runs the task in an isolated worktree and walks you through the
review-and-reconcile checklist before anything is kept. `--bg` runs it in the
background. Ask for several independent tasks at once and they go to separate
subagents, one git worktree each, so they don't step on each other.

## A note on the flags

Every flag in the drivers was checked against the CLI version installed when this
was written: Codex 0.133.0, Gemini 0.43.0, Qwen 0.15.6, OpenClaude 0.14.0, Claude
Code 2.1.150. These tools change quickly, so each driver tells you to re-run
`<cli> --help` if something looks off. A few that already drifted from the
upstream Hermes skill: Codex `exec` dropped `--full-auto`/`--yolo` in favor of
`-s` sandbox modes, Gemini uses `auto_edit` while Qwen uses `auto-edit`, and
Qwen needs `--chat-recording` before a session can be resumed.

## Credits

Adapted from the `autonomous-ai-agents` skill in
[NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) (MIT).
The orchestration patterns and the per-agent driver structure are theirs; the
Claude Code translation, the verified flags, and the extra drivers are this
plugin's. Licensed under MIT.
