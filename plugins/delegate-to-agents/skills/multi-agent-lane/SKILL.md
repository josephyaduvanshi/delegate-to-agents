---
name: multi-agent-lane
description: Use when an external agent (Codex/Gemini/Qwen/OpenClaude/OpenCode) produces a patch you must review, test, and accept/reject before keeping — isolated git worktree, untrusted-diff reconciliation, independent test run, and accept/partial/reject handoff. Ported and generalized from the hermes-agent kanban-codex-lane skill.
---

# Multi-Agent Lane (untrusted-patch reconciliation)

A discipline for using an external agent as a **bounded implementation lane**
while *you* (the orchestrating Claude Code session) keep ownership of acceptance,
testing, and the final state. Generalized from the Nous `kanban-codex-lane` skill
— the durable SQLite board is Hermes-specific, but the reconciliation rigor is
universal and is the part worth keeping.

**Core stance:** the agent is an *input lane only*. Its output is **not** a
completion signal, **not** a trusted review, and **not** allowed to define the
final state. You review the diff, run the tests yourself, and decide.

## When to use

All of these true:
- A coding/refactor/test/docs task with **clear acceptance criteria**.
- A **bounded diff** you can review in one pass.
- The repo can be checked out in an **isolated worktree/branch**.
- **You can run the tests yourself** after the agent exits.
- The prompt can state every safety constraint and off-limits file.

Do **not** use when: the task needs judgment not captured in the brief; you lack
repo access / agent auth / time to reconcile; the change touches secrets,
credentials, or production-critical paths; a small direct edit is faster and
safer; or you'd be tempted to accept based only on the agent's self-report.

## Ownership rules

1. **You own acceptance.** Treat the agent's commits/diff as untrusted patches
   until reviewed and verified.
2. **You own testing.** The agent may run tests, but those runs are advisory —
   re-run the canonical suite yourself.
3. **You own safety.** If the agent weakens a safety boundary, risk gate, auth,
   or secrets handling, reject the lane **even if tests pass**.
4. **You own cleanup.** Kill stuck agents; remove temp worktrees.

## 1. Isolate (never run in a dirty shared checkout)

```
REPO=/path/to/repo
TASK=issue-123
BASE=$(git -C "$REPO" rev-parse --abbrev-ref HEAD)
SAFE=$(printf '%s' "$TASK" | tr -cd '[:alnum:]_-')
BRANCH="agent/${SAFE}/$(date -u +%Y%m%d%H%M%S)"
WORKTREE="/tmp/${SAFE}-agent-lane"

git -C "$REPO" fetch --all --prune
git -C "$REPO" worktree add -b "$BRANCH" "$WORKTREE" "$BASE"
git -C "$WORKTREE" status --short --branch
```

## 2. Build the prompt (every lane prompt must include)

- Task id, title, and full acceptance criteria.
- Repo path, worktree path, branch name, and **allowed file scope**.
- Explicit: *"You are an input lane only. Do not finalize, do not push, do not
  touch files outside the listed scope."*
- Required output: concise summary, files changed, commits, tests run, known risks.
- Prohibited: secrets access, external messaging, unrelated refactors, dependency
  upgrades unless required.
- The verification commands the agent may run, and the ones you'll run after.
- Repo-specific safety invariants (replace the Hermes "PMB safety block" with
  yours — e.g. "never enable live order entry", "don't weaken auth/rate limits").

## 3. Run the agent in the worktree (delegated, so output stays out of your context)

Use the `cli-delegate` subagent with the right driver skill, pointed at the
worktree. Prefer a sandboxed/auto-edit mode over full bypass.

```
Agent(
  subagent_type: "cli-delegate",
  prompt: "Use the drive-codex skill. workdir=/tmp/issue-123-agent-lane. Run `codex exec -s workspace-write` with this brief: <full brief incl. scope, safety, acceptance, verification>. Make small commits on the current branch. Return summary, files changed, commits, tests run, risks. Do NOT push."
)
```

For very long lanes, run it in the background instead and poll the worktree:
`run_in_background: true` on the Agent call, or a background `Bash` driving the
CLI with a logfile.

## 4. Monitor without interfering

- Poll the log/pane; don't kill on first silence (multi-step work looks idle).
- **Kill** if: no useful output within the budget; the agent asks for secrets or
  external permissions; it edits outside the worktree; it starts unrelated
  rewrites/dependency churn; or it's near your deadline with no safe partial.

## 5. Reconcile — the checklist (do this before accepting anything)

- [ ] `git -C "$WORKTREE" status --short --branch` shows only expected files.
- [ ] `git -C "$WORKTREE" diff --stat` and the full `git diff` reviewed by you.
- [ ] No secrets, credentials, generated caches, or unrelated data included.
- [ ] Repo safety invariants preserved (auth, rate limits, kill switches, etc.).
- [ ] Commits are small enough to cherry-pick or squash cleanly.
- [ ] **You ran the canonical tests yourself** with the repo's documented command.
- [ ] Agent-run tests are listed separately from your runs.
- [ ] Accepted commits/diffs applied to your owned branch; rejected/partial work
      has a concrete reason (and an artifact path if useful).

## 6. Record the outcome

Summarize to the user (and, if you track tasks, into your notes) with this shape:

```json
{
  "agent_lane": {
    "used": true,
    "agent": "codex | gemini | qwen | openclaude | opencode",
    "mode": "exec | run | interactive | skipped",
    "worktree": "/abs/path",
    "branch": "agent/issue-123/20260524…",
    "command": "codex exec -s workspace-write …",
    "result": "accepted | partial | rejected | timed_out",
    "accepted_commits": ["<sha>"],
    "rejected_reason": "empty when fully accepted; else concrete reason",
    "tests_run": [
      {"command": "make test", "exit_code": 0, "owner": "me"},
      {"command": "agent-reported: npm test", "exit_code": 0, "owner": "agent"}
    ],
    "artifacts": ["/abs/path/to/log-or-patch"]
  }
}
```

## 7. Clean up

```
git -C "$REPO" worktree remove "$WORKTREE"
git -C "$REPO" branch -D "$BRANCH"   # only after accepted commits were copied/cherry-picked, or work was rejected
```

Keep the worktree only if it's a needed review artifact — and say so in the handoff.

## Pitfalls

1. Treating the agent's self-report as verification. Always diff + re-test yourself.
2. Running in the user's dirty main checkout. Always isolate.
3. Letting the agent define "done." You decide and record state.
4. Dropping safety invariants from the prompt — that's a setup failure.
5. Accepting broad unrelated cleanup because tests pass — cherry-pick only the scope.
6. Killing a stuck lane without recording why.
