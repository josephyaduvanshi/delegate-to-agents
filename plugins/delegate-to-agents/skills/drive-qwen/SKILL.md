---
name: drive-qwen
description: Use when delegating a coding or research task to the Qwen Code CLI (`qwen`) from Claude Code. Covers one-shot positional prompts, approval modes, sessions (and the --chat-recording gotcha), output formats, and tmux interactive driving. Load delegate-to-agents first for the shared mechanism.
---

# Drive Qwen Code CLI

Delegate to [Qwen Code](https://github.com/QwenLM/qwen-code), a coding agent CLI
(a Gemini-CLI-style fork). Read `delegate-to-agents` first for the shared
Claude Code mechanism.

Flags verified against **qwen 0.15.6** (`qwen --help`).

## When to use

Coding, refactors, review, research — when you want the Qwen models or a cheap,
fast second engine.

## Preflight

```
command -v qwen && qwen --version
qwen auth        # configure auth: OpenRouter, Coding Plan, API Key, or Qwen-OAuth
```

Auth backends: OpenRouter, Qwen Coding Plan, API key, or Qwen-OAuth — set up via
`qwen auth`. No valid backend → headless runs fail; fall back.

## One-shot (PREFERRED): positional prompt

Unlike Gemini, **Qwen's positional prompt defaults to one-shot** ("use
`-i/--prompt-interactive` for interactive"). So a bare prompt is already headless.

```
# one-shot, capture stdout
Bash(command="cd /repo && qwen 'Summarize the responsibilities of src/scheduler.go'", timeout=300000)

# write task, auto-approve edits
Bash(command="cd /repo && qwen 'Add input validation to the signup handler and tests' --approval-mode auto-edit", timeout=600000)

# stdin appended to the prompt
Bash(command="git -C /repo diff | qwen 'Review this diff; flag bugs and missing tests'", timeout=300000)

# long task: detach + log
Bash(command="cd /repo && qwen 'Refactor the parser' --approval-mode auto-edit > /tmp/agent-qwen.log 2>&1", run_in_background=true)
```

`-p/--prompt` still works but is deprecated in favor of the positional form.

### Key flags

| Flag | Effect |
|------|--------|
| `<prompt>` (positional) | One-shot run (default) |
| `-i, --prompt-interactive <text>` | Run prompt, then stay interactive |
| `-m, --model <name>` | Model override |
| `--approval-mode <mode>` | `plan` (read-only), `default` (prompt), `auto-edit` (auto-approve edits), `yolo` (approve all) |
| `-y, --yolo` | Approve all actions |
| `-o, --output-format <fmt>` | `text`, `json`, `stream-json` |
| `-s, --sandbox` | Run tools in a sandbox |
| `--system-prompt` / `--append-system-prompt` | Replace / extend the system prompt |
| `--include-directories`, `--add-dir <dirs>` | Add workspace directories |
| `--bare` | Minimal mode: skip startup auto-discovery |
| `--max-session-turns <n>` | Cap agent turns |

> Note: `--approval-mode auto-edit` (hyphen). Gemini uses `auto_edit`
> (underscore). They differ — don't copy across drivers.

### Sessions / resume — the `--chat-recording` gotcha

Resume/continue only work if chat history was recorded. **`--continue`/`--resume`
do nothing unless `--chat-recording` was enabled on the original run.**

```
qwen --chat-recording 'Start the migration'   # record so it can be resumed
qwen -c 'Continue the migration'               # -c is a boolean (most-recent); the string is the positional prompt
qwen -r <session-id> 'Continue the migration'  # -r takes the ID; the prompt stays positional
qwen --session-id <session-id> 'Continue …'
```

## Interactive (tmux)

```
Bash(command="tmux new-session -d -s qwen -x 200 -y 50")
Bash(command="tmux send-keys -t qwen 'cd /repo && qwen -i \"Add a health-check endpoint\"' Enter")
Bash(command="sleep 6 && tmux capture-pane -t qwen -p -S -60")
Bash(command="tmux send-keys -t qwen 'now add a test for it' Enter")
Bash(command="sleep 25 && tmux capture-pane -t qwen -p -S -80")
Bash(command="tmux kill-session -t qwen")
```

## Rules

1. **Positional prompt = one-shot** — no extra flag needed for headless.
2. **`--approval-mode auto-edit`** (hyphen) for building; `plan` for read-only review.
3. **Enable `--chat-recording`** if you intend to resume later.
4. **`--output-format json`** to parse results.
5. **One worktree per parallel run**; clean up tmux/logs.
6. **Review the diff and re-run tests yourself.**
