---
name: drive-openclaude
description: Use when delegating a coding task to the OpenClaude CLI (`openclaude`) from Claude Code. OpenClaude is a Claude-Code-compatible agent CLI whose standout feature is --provider (anthropic/openai/gemini/github/bedrock/vertex/ollama). Covers print mode, permission modes, providers, and tmux. Load delegate-to-agents first.
---

# Drive OpenClaude CLI

Delegate to [OpenClaude](https://github.com/openclaude) — a Claude-Code-compatible
coding agent CLI. Its flag surface mirrors Claude Code, so the `drive-claude-code`
patterns mostly apply; the key difference is **`--provider`**, which lets one CLI
run against many model backends. Read `delegate-to-agents` first.

Flags verified against **OpenClaude 0.14.0** (`openclaude --help`).

## When to use

- You want a Claude-Code-style agent but pointed at a **non-Anthropic provider**
  (OpenAI, Gemini, GitHub Models, Bedrock, Vertex, Ollama).
- A second Claude-compatible engine for review/cross-checking.

## Preflight

```
command -v openclaude && openclaude --version
```

- **Auth:** depends on `--provider`. OpenClaude reads provider API keys from the
  environment (e.g. `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, …),
  or Anthropic OAuth for the default provider. Confirm the relevant key is set
  before a headless run.

## One-shot (PREFERRED): `-p` / `--print`

```
# headless (workspace-trust dialog is skipped under -p)
Bash(command="cd /repo && openclaude -p 'Add structured logging to the request handler'", timeout=300000)

# against a specific provider
Bash(command="cd /repo && openclaude -p 'Review src/auth.ts for security issues' --provider openai --model gpt-5.4", timeout=300000)

# structured JSON output
Bash(command="cd /repo && openclaude -p 'List all exported functions' --output-format json", timeout=300000)

# long task: detach + log
Bash(command="cd /repo && openclaude -p 'Refactor the cache layer' --permission-mode acceptEdits > /tmp/agent-oc.log 2>&1", run_in_background=true)
```

### Key flags

| Flag | Effect |
|------|--------|
| `-p, --print` | Non-interactive; print result and exit (skips workspace-trust dialog) |
| `--provider <p>` | `anthropic`, `openai`, `gemini`, `github`, `bedrock`, `vertex`, `ollama` (reads keys from env) |
| `--model <model>` | Alias (`sonnet`/`opus`) or full name |
| `--output-format <fmt>` | `text`, `json`, `stream-json` (print mode) |
| `--json-schema <schema>` | Structured output validation |
| `--permission-mode <mode>` | `default`, `acceptEdits`, `plan`, `auto`, `dontAsk`, `bypassPermissions` |
| `--dangerously-skip-permissions` | Auto-approve everything (isolated dirs only) |
| `--allowedTools` / `--disallowedTools` / `--tools` | Scope tool access |
| `--max-budget-usd <n>` | Spend cap (print mode) |
| `--fallback-model <m>` | Fall back when default is overloaded (print mode) |
| `--no-session-persistence` | Don't save the session (print mode) |
| `-c, --continue` / `-r, --resume [id]` | Resume conversations |
| `-w, --worktree [name]` / `--tmux` | Isolated git worktree (+ tmux session) |
| `--from-pr [n]` | Resume a session linked to a PR |
| `--bare` | Minimal: skip hooks/LSP/plugins/CLAUDE.md (needs `ANTHROPIC_API_KEY` or `--settings`) |
| `--append-system-prompt <text>` | Extend the system prompt |

> `--max-turns` is not listed in OpenClaude 0.14.0's `--help` (it's an inherited,
> hidden flag — it's still accepted at runtime). Prefer **`--max-budget-usd`** as
> the documented, guaranteed bound for a print-mode run.

## Interactive (tmux)

Default (no `-p`) is interactive; drive via tmux exactly like Claude Code. The
workspace-trust dialog appears on first visit to a directory — answer it by
sending `Enter` (default is "trust"). With `--dangerously-skip-permissions` a
second confirmation defaults to "No" — send `Down` then `Enter`.

```
Bash(command="tmux new-session -d -s oc -x 200 -y 50")
Bash(command="tmux send-keys -t oc 'cd /repo && openclaude' Enter")
Bash(command="sleep 5 && tmux send-keys -t oc Enter")              # trust dialog → default Yes
Bash(command="sleep 2 && tmux send-keys -t oc 'Add a /health route' Enter")
Bash(command="sleep 25 && tmux capture-pane -t oc -p -S -80")
Bash(command="tmux kill-session -t oc")
```

## Rules

1. **`-p` for automation** — clean, skips the trust dialog.
2. **Set `--provider` + the matching API key env var** when not using Anthropic.
3. **Pick `--permission-mode`** deliberately; reserve `--dangerously-skip-permissions`
   for isolated dirs.
4. **Bound print-mode runs with `--max-budget-usd`** (documented); `--max-turns`
   is hidden-but-accepted and may also be added.
5. **One worktree per parallel run**; clean up tmux/logs.
6. **Review the diff and re-run tests yourself.**
