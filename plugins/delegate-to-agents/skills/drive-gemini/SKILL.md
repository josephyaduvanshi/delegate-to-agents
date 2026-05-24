---
name: drive-gemini
description: Use when delegating a coding or research task to the Google Gemini CLI (`gemini`) from Claude Code. Covers non-interactive `-p`, approval modes, output formats, sessions, worktrees, and tmux interactive driving. Load delegate-to-agents first for the shared mechanism.
---

# Drive Gemini CLI

Delegate to the [Gemini CLI](https://github.com/google-gemini/gemini-cli),
Google's provider-agnostic-ish coding agent. Read `delegate-to-agents` first for
the shared Claude Code mechanism.

Flags verified against **gemini 0.43.0** (`gemini --help`).

## When to use

Coding, research, large-context analysis, or when you want Gemini's model
specifically. Strong for big-context reads.

## Preflight

```
command -v gemini && gemini --version
```

- **Auth:** set `GEMINI_API_KEY` or `GOOGLE_API_KEY`, **or** run `gemini` once
  interactively to complete Google OAuth. If neither is present and there's no
  cached OAuth, a headless `-p` run will fail — fall back accordingly.

## One-shot / headless (PREFERRED): `-p`

Gemini defaults to **interactive** mode; a bare positional prompt opens the TUI.
For automation you MUST pass `-p`/`--prompt`.

```
# headless, capture stdout
Bash(command="cd /repo && gemini -p 'Explain what src/server.ts does and list its exports'", timeout=300000)

# headless write task with auto-approved edits
Bash(command="cd /repo && gemini -p 'Add retry logic to the HTTP client and update tests' --approval-mode auto_edit", timeout=600000)

# stdin is appended to the -p prompt
Bash(command="git -C /repo diff | gemini -p 'Review this diff for bugs and security issues'", timeout=300000)

# long task: detach + log
Bash(command="cd /repo && gemini -p 'Refactor the data layer' --approval-mode auto_edit > /tmp/agent-gemini.log 2>&1", run_in_background=true)
```

### Key flags

| Flag | Effect |
|------|--------|
| `-p, --prompt <text>` | Headless (non-interactive) run |
| `-i, --prompt-interactive <text>` | Run the prompt, then stay interactive |
| `-m, --model <name>` | Model override |
| `--approval-mode <mode>` | `default` (prompt), `auto_edit` (auto-approve edits), `yolo` (auto-approve all), `plan` (read-only) |
| `-y, --yolo` | Shorthand for approve-all |
| `-o, --output-format <fmt>` | `text` (default), `json`, `stream-json` |
| `-s, --sandbox` | Run tools in a sandbox |
| `--skip-trust` | Trust the workspace for this session (avoids the trust prompt) |
| `-w, --worktree [name]` | Start in a fresh git worktree |
| `--include-directories <dirs>` | Add directories to the workspace (monorepos) |
| `-e, --extensions <list>` | Restrict to specific extensions |

### Sessions / resume

```
gemini --list-sessions                 # see sessions for this project
gemini -r latest -p 'Continue: now add tests'   # resume most recent
gemini -r 5 -p '...'                    # resume by index
gemini --session-id <uuid> -p '...'     # pin a session id
```

## Interactive (tmux)

```
Bash(command="tmux new-session -d -s gem -x 200 -y 50")
Bash(command="tmux send-keys -t gem 'cd /repo && gemini --skip-trust' Enter")
Bash(command="sleep 6 && tmux capture-pane -t gem -p -S -40")
Bash(command="tmux send-keys -t gem 'Add a CLI subcommand for export' Enter")
Bash(command="sleep 25 && tmux capture-pane -t gem -p -S -80")
Bash(command="tmux kill-session -t gem")
```

`--skip-trust` avoids the workspace-trust prompt. Without it, answer the trust
dialog by capturing the pane and sending the right key.

## Rules

1. **Always `-p` for automation** — without it, Gemini opens the TUI and hangs a
   non-interactive `Bash` call.
2. **Pick the approval mode** — `auto_edit` for building, `plan` for read-only
   review, `default` when you want it to ask.
3. **`--output-format json`** when you need to parse the result.
4. **`--skip-trust`** (or pre-trust) in automation to avoid the trust prompt.
5. **One worktree per parallel run**; clean up tmux sessions and logs.
6. **Verify the diff and tests yourself** before keeping changes.
