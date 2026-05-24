---
name: drive-codex
description: Use when delegating a coding, refactoring, or code-review task to the OpenAI Codex CLI (`codex`) from Claude Code. Covers non-interactive `codex exec`, sandbox/approval flags, interactive tmux driving, resume, review, and parallel worktrees. Load delegate-to-agents first for the shared mechanism.
---

# Drive Codex CLI

Delegate coding to [Codex](https://github.com/openai/codex), OpenAI's autonomous
coding-agent CLI. Read `delegate-to-agents` first for the shared Claude Code
mechanism (Bash one-shot, background+log, tmux interactive, `cli-delegate`).

Flags below are verified against **codex-cli 0.133.0**. Run `codex exec --help`
if behavior differs — the CLI moves fast.

## When to use

Building features, refactors, code review, batch issue fixing — when you want a
strong second engine or to parallelize work.

## Preflight (always run first)

```
command -v codex && codex --version
codex doctor          # diagnoses install, config, auth, runtime
```

- **Auth:** `OPENAI_API_KEY`, or a Codex CLI OAuth session at `~/.codex/auth.json`.
  A missing `OPENAI_API_KEY` is **not** proof auth is absent — `codex doctor` is
  the real check. Set up with `codex login`.
- Codex normally requires a **git repo**. Use `--skip-git-repo-check` to run
  outside one (replaces the old `mktemp -d && git init` dance).

## One-shot (PREFERRED): `codex exec`

`codex exec [PROMPT]` runs non-interactively and exits when done. Alias: `codex e`.

```
# bounded, capture stdout (foreground)
Bash(command="cd /path/to/project && codex exec 'Add a --dry-run flag to the importer'", timeout=300000)

# write-enabled, sandboxed to the workspace
Bash(command="cd /path/to/project && codex exec -s workspace-write 'Refactor auth to use JWT'", timeout=600000)

# long task: detach + log, then poll (harness re-invokes you on exit)
Bash(command="cd /path/to/project && codex exec -s workspace-write 'Migrate tests to pytest' > /tmp/agent-codex-mig.log 2>&1", run_in_background=true)
Bash(command="tail -n 80 /tmp/agent-codex-mig.log")
```

### Sandbox & approval (get this right)

**`--full-auto` and `--yolo` do not exist in codex 0.133.0 at any level** — the
old Hermes examples target an earlier Codex. Don't use them; they'll fail.
Control sandbox/approval on `codex exec` with these instead:

| Flag | Effect |
|------|--------|
| `-s read-only` | Model may read but not write/execute outside reads |
| `-s workspace-write` | Auto-writes inside the workspace (the usual "build" mode) |
| `-s danger-full-access` | No sandbox restrictions |
| `--dangerously-bypass-approvals-and-sandbox` | Skip ALL prompts + sandbox. EXTREMELY dangerous — only in an externally isolated/throwaway dir |
| `--skip-git-repo-check` | Allow running outside a git repository |
| `--add-dir <DIR>` | Extra writable directory beyond the workspace |
| `-C, --cd <DIR>` | Set the working root (alternative to `cd …`) |
| `-m, --model <MODEL>` | Override model |
| `--ephemeral` | Don't persist session files |

> `-p` on `codex exec` means `--profile`, NOT print. Don't confuse it with
> Claude's `-p`.

### Capturing output programmatically

```
# JSONL event stream
Bash(command="codex exec --json -s workspace-write 'Fix the failing test' > /tmp/codex-events.jsonl 2>&1", run_in_background=true)

# final message only, to a file
Bash(command="codex exec -o /tmp/codex-final.txt 'Summarize the architecture of src/'", timeout=300000)

# structured output against a schema
Bash(command="codex exec --output-schema /tmp/schema.json 'List all API routes'", timeout=300000)
```

## Code review

```
# review current repo against a base
Bash(command="cd /repo && codex review --base origin/main", timeout=300000)

# review a PR in an isolated clone
Bash(command="REVIEW=$(mktemp -d) && git clone https://github.com/USER/REPO.git $REVIEW && cd $REVIEW && gh pr checkout 42 && codex review --base origin/main", timeout=600000)
```

(`codex exec review` is the exec-subcommand equivalent.)

## Resume

```
Bash(command="cd /repo && codex exec resume --last 'Continue and add connection pooling'", timeout=300000)
```

`codex resume` / `codex fork` (no `exec`) open interactive session pickers.

## Interactive (tmux) — multi-turn

`codex` with no subcommand launches the TUI; drive it via tmux (see
`delegate-to-agents` Mode 2).

```
Bash(command="tmux new-session -d -s codex -x 200 -y 50")
Bash(command="tmux send-keys -t codex 'cd /repo && codex' Enter")
Bash(command="sleep 5 && tmux capture-pane -t codex -p -S -40")   # check it started / answer any trust prompt
Bash(command="tmux send-keys -t codex 'Add rate limiting to the API' Enter")
Bash(command="sleep 25 && tmux capture-pane -t codex -p -S -80")
Bash(command="tmux kill-session -t codex")
```

If Codex asks a y/n question while running, answer with
`tmux send-keys -t codex 'y' Enter`. To interrupt: `tmux send-keys -t codex C-c`.

## Parallel issue fixing (worktrees)

```
Bash(command="git -C /repo worktree add -b fix/78 /tmp/iss-78 main")
Bash(command="git -C /repo worktree add -b fix/99 /tmp/iss-99 main")
Bash(command="cd /tmp/iss-78 && codex exec -s workspace-write 'Fix issue #78: <desc>. Commit when done.' > /tmp/agent-78.log 2>&1", run_in_background=true)
Bash(command="cd /tmp/iss-99 && codex exec -s workspace-write 'Fix issue #99: <desc>. Commit when done.' > /tmp/agent-99.log 2>&1", run_in_background=true)
# poll each log; then push + PR; then `git worktree remove`
```

For "review before keeping it", use the `multi-agent-lane` skill.

## Rules

1. **`codex exec` for automation** — it runs and exits cleanly.
2. **Choose the sandbox deliberately** — `-s workspace-write` for building;
   reserve `--dangerously-bypass-approvals-and-sandbox` for throwaway/isolated dirs.
3. **`--skip-git-repo-check`** for scratch work outside a repo.
4. **Background + logfile** for long tasks; poll with `tail`, don't interrupt.
5. **One worktree per parallel Codex** — never share a checkout.
6. **Verify yourself** — review the diff and re-run tests; don't trust the summary.
7. **Clean up** tmux sessions, worktrees, and logfiles.
