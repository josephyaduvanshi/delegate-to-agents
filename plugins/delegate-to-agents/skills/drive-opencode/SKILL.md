---
name: drive-opencode
description: Use when delegating a coding task to the OpenCode CLI (`opencode`) from Claude Code. Covers one-shot `opencode run`, interactive TUI via tmux, sessions, PR review, and parallel work. NOTE opencode is not installed by default on this machine. Load delegate-to-agents first.
---

# Drive OpenCode CLI

Delegate to [OpenCode](https://opencode.ai), a provider-agnostic open-source
coding agent with a TUI and CLI. Read `delegate-to-agents` first.

> **Not installed by default here.** Preflight will tell you. Install with
> `npm i -g opencode-ai@latest` or `brew install anomalyco/tap/opencode`.

## Preflight

```
command -v opencode || echo "opencode NOT installed — npm i -g opencode-ai@latest"
opencode --version
which -a opencode          # PATH can resolve the wrong binary; pin a path if so
opencode auth list         # should show ≥1 provider
```

If not installed/authed, install it or fall back to a different agent — don't
treat this as a hard failure.

## One-shot (PREFERRED): `opencode run`

`opencode run 'prompt'` is bounded and non-interactive — **no PTY needed**, so a
plain `Bash` call works.

```
# bounded task
Bash(command="cd /repo && opencode run 'Add retry logic to API calls and update tests'", timeout=600000)

# attach context files
Bash(command="cd /repo && opencode run 'Review this config for security issues' -f config.yaml -f .env.example", timeout=300000)

# force a model
Bash(command="cd /repo && opencode run 'Refactor auth module' --model openrouter/anthropic/claude-sonnet-4", timeout=600000)

# machine-readable output
Bash(command="cd /repo && opencode run 'List all API routes' --format json", timeout=300000)

# long task: detach + log
Bash(command="cd /repo && opencode run 'Migrate to the new ORM' > /tmp/agent-opencode.log 2>&1", run_in_background=true)
```

### Common flags

| Flag | Use |
|------|-----|
| `run '<prompt>'` | One-shot execution and exit (no PTY) |
| `-c, --continue` | Continue the last session |
| `-s, --session <id>` | Continue a specific session |
| `--agent <name>` | Choose agent (`build` or `plan`) |
| `--model provider/model` | Force a model |
| `--format json` | Machine-readable output/events |
| `-f, --file <path>` | Attach file(s) |
| `--thinking` | Show model thinking |
| `--variant <level>` | Reasoning effort (`high`, `max`, `minimal`) |
| `--title <name>` | Name the session |

## Interactive (tmux)

The bare `opencode` TUI needs a real PTY — drive via tmux.

```
Bash(command="tmux new-session -d -s oco -x 200 -y 50")
Bash(command="tmux send-keys -t oco 'cd /repo && opencode' Enter")
Bash(command="sleep 6 && tmux capture-pane -t oco -p -S -40")
Bash(command="tmux send-keys -t oco 'Implement OAuth refresh flow and add tests' Enter")
Bash(command="sleep 2 && tmux send-keys -t oco Enter")   # Enter may need pressing twice to submit
Bash(command="sleep 25 && tmux capture-pane -t oco -p -S -80")
Bash(command="tmux send-keys -t oco C-c")                # EXIT with Ctrl+C — never /exit
Bash(command="tmux kill-session -t oco")
```

**Gotchas:**
- `/exit` is NOT valid — it opens an agent selector. Exit with `Ctrl+C` (`C-c`).
- Enter may need pressing twice in the TUI (finalize text, then send).
- PATH mismatch can select the wrong binary/model config — `which -a opencode`.

## Sessions / cost

```
opencode session list
opencode stats
opencode stats --days 7
```

## PR review

```
# built-in PR command
Bash(command="cd /repo && opencode run --agent plan 'Review PR #42 vs main: bugs, security, test gaps'", timeout=300000)

# or in an isolated clone
Bash(command="REVIEW=$(mktemp -d) && git clone https://github.com/USER/REPO.git $REVIEW && cd $REVIEW && gh pr checkout 42 && opencode run 'Review this PR vs main. Report bugs, security risks, test gaps, style.'", timeout=600000)
```

## Parallel work

```
Bash(command="git -C /repo worktree add -b fix/101 /tmp/iss-101 main")
Bash(command="cd /tmp/iss-101 && opencode run 'Fix issue #101 and commit' > /tmp/oco-101.log 2>&1", run_in_background=true)
```

## Rules

1. **Prefer `opencode run`** for automation — simpler, no PTY.
2. **Interactive only when iterating** — and exit with Ctrl+C, never `/exit`.
3. **Scope each session to one repo/worktree**; never share a workdir.
4. **`which -a opencode`** if behavior looks wrong (binary/PATH mismatch).
5. **Clean up** tmux sessions, worktrees, and logs.
6. **Review the diff and re-run tests yourself.**
