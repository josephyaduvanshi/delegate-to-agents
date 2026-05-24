---
name: delegate
description: Delegate a coding/research task to an external agent CLI (Codex, Gemini, Qwen, OpenClaude, OpenCode) or a Claude subagent
argument-hint: "[codex|gemini|qwen|openclaude|opencode|claude] [--bg] [--review] <task to delegate>"
allowed-tools: Agent, Skill, Bash, AskUserQuestion
---

Delegate the task below to another AI agent. Use the `delegate-to-agents` skill
for the routing logic and the shared mechanism.

Raw user request:
$ARGUMENTS

How to route:

1. **Load the `delegate-to-agents` skill first.** Follow its Step 1–5.
2. **Agent selection:**
   - If the request starts with an agent name (`codex`, `gemini`, `qwen`,
     `openclaude`, `opencode`, `claude`), use that agent's driver skill.
   - If no agent is named, pick the best installed one for the task and state
     which you chose and why. Preflight before committing (`command -v <cli>`).
3. **Mechanism:**
   - For an **external CLI agent**, delegate the driving to the `cli-delegate`
     subagent via the `Agent` tool (`subagent_type: "cli-delegate"`), passing the
     driver name, the task, and the `workdir`. This keeps CLI output out of the
     main context.
   - For **`claude`**, prefer the native `Agent` tool with a general subagent
     unless a separate `claude` process is specifically needed (then use
     `drive-claude-code`).
   - `cli-delegate` is a **subagent**, not a skill — invoke it through the
     `Agent` tool, never `Skill(cli-delegate)`.
4. **Flags in the request:**
   - `--bg` → run the `Agent` call with `run_in_background: true` (long tasks).
   - `--review` → use the `multi-agent-lane` skill: isolate in a worktree, then
     review the diff and re-run tests yourself before accepting.
   - These are routing flags for Claude Code — strip them from the task text you
     forward to the agent.
5. **If no task was supplied**, ask what to delegate and to which agent.

After the agent finishes, report concrete outcomes to the user: which agent ran,
files changed, tests (mark agent-reported as unverified), and residual risk. If
the chosen CLI is missing or unauthenticated, say so and suggest installing/auth
or a different agent — don't silently fall back without saying which agent ran.
