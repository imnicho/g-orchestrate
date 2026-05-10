---
name: g-orchestrate
description: Cost-aware multi-agent delegation. Like /orchestrate, but routes low-effort tasks to Gemini CLI (free tier — `gemini-2.5-flash` / `gemini-2.5-pro`) and reserves Claude (Agent / teams / tmux) for tasks that actually need it. Picks the right model AND the right mechanism per task.
user-invocable: true
allowed-tools: Bash Read Write Glob Grep Agent ToolSearch TeamCreate TaskCreate TaskUpdate TaskList TaskGet SendMessage
argument-hint: [task description]
---

# G-Orchestrate — Cost-Aware Multi-Agent Delegation

You are the **Orchestrator**. Same craft as `/orchestrate` — picking the right mechanism, brief-writing, synthesis — with one extra dimension: **picking the right LLM**. Cheap tasks go to Gemini. Hard tasks stay on Claude.

## Input

The user's task: **$ARGUMENTS**

## Step 0a: Pick the Engine (Effort Triage)

Before picking a mechanism, classify each agent task by effort. The goal is to spend Anthropic tokens only where they buy you something Gemini can't.

| Effort tier | Examples | Engine | Model |
|---|---|---|---|
| **Trivial** | grep / file lookup, "list all routes that call X", "summarize this file in 5 bullets", boilerplate generation, simple regex/refactor on one file | **Gemini CLI** | `gemini-2.5-flash` |
| **Moderate** | structured research across a few files, drafting a brief, writing tests for a known spec, doc generation, straightforward refactors | **Gemini CLI** | `gemini-2.5-pro` |
| **Hard** | cross-file reasoning, architecture decisions, debugging non-obvious failures, security review, anything where wrong-but-confident is expensive, anything that must call into the user's Claude Code agents/tools | **Claude** | `haiku` / `sonnet` / `opus` per `/orchestrate` rules |

**Heuristics for staying honest:**

- If you'd be embarrassed by a confidently-wrong answer, don't use Gemini.
- If the task fits in a single prompt with a single concrete deliverable, Gemini is probably fine.
- If the task needs to use Claude Code's specialized subagents (e.g. `nexus-db-reviewer`, `wastebin-crypto-reviewer`), it stays on Claude — Gemini can't invoke them.
- If the task needs `git`, `npm test`, `cargo build`, etc. as part of the loop, Gemini can do that with `--yolo` in a worktree, but only delegate it there if the task is independent.
- When in doubt, start cheap (flash → pro → Claude). Re-dispatch on Claude if Gemini's output is weak.

## Step 0b: Pick the Mechanism

Same table as `/orchestrate`:

|  | **Subagents** (`Agent` tool) | **Agent Teams** (`TeamCreate` + `SendMessage`) | **tmux Panes** | **Gemini CLI** (this skill's addition) |
|---|---|---|---|---|
| What it is | In-session helpers; result returns as tool result | Separate Claude Code sessions, shared tasks, direct messaging | Independent `claude` processes in panes | One-shot or background `gemini -p` calls, optionally in a git worktree |
| Inter-agent comms | None | Yes | None (manual) | None |
| Mid-run steering | None | `SendMessage` | `send-keys` | None (one-shot); kill+respawn for steering |
| Token cost | Anthropic | Anthropic (high) | Anthropic (high) | **Free** (Google login, free tier) |
| Best for | Cheap Claude fan-out | Debate / collaboration | Process-level control | Cheap fan-out, drafting, lookups, mechanical work |

**Decision flow:**

1. **Trivial / moderate effort?** → **Gemini CLI** (Step 1G). Default for the long tail of small tasks.
2. **Hard effort, single fan-out wave?** → Claude `Agent` tool. Parallelize independent calls.
3. **Hard effort, agents need to talk to each other?** → Claude **agent team** (`TeamCreate` etc., load schemas via `ToolSearch query="select:TeamCreate,SendMessage,TaskCreate,TaskUpdate,TaskList,TaskGet"`).
4. **Hard effort + process control / dev servers / interactive TUIs?** → **tmux flow** (Steps 2–6 of `/orchestrate`).

You can mix Gemini and Claude in one orchestration — e.g. dispatch four Gemini scouts in parallel, synthesize with one Claude `Agent`, kick off a Claude team for the hard architectural call. Don't mix Claude teams with tmux orchestrations in the same session (the team owns the pane layout).

## Step 0c: Verify Gemini Is Ready (one-time)

Before the first Gemini dispatch in a session, verify auth:

```bash
gemini -p "ok" -m gemini-2.5-flash 2>&1 | head -3
```

If you see `Please set an Auth method...`, the user needs to authenticate once. Tell them:

> Run `gemini` once in a terminal to log in with your Google account (free tier — uses `GOOGLE_GENAI_USE_GCA`). Then re-run `/g-orchestrate <task>`.

Do not attempt to set `GEMINI_API_KEY` automatically — the free tier path is Google login, and a paid API key bypasses the free quota. If the user wants the API key path instead, they can export `GEMINI_API_KEY` in their shell.

## Step 1: Plan & Brief

Same as `/orchestrate` Step 1. Decompose, pick a workflow pattern (parallel fan-out / sequential pipeline / review loop / fan-out-fan-in / iterative refinement), write briefs to `<project>/.orchestrate/<role>-brief.md`. Briefs are mechanism-agnostic.

Brief template stays the same: Mission, Context, Success criteria, Constraints, Output channel, Undercover rule.

**One addition for Gemini briefs:** Gemini does not know about your conversation, your earlier agents, or the mission board unless you put it in the brief or the prompt. Be explicit. Gemini's working directory is wherever you run it from (or `-C <dir>` / `--include-directories`), so spell out which files matter.

## Step 1G: Dispatching to Gemini

Two patterns — **stateless one-shot** (most common) and **worktree task** (when Gemini needs to edit and commit).

### Pattern A — Stateless one-shot (research, lookups, drafting)

Best for: "summarize this", "list X", "draft a brief for Y", "write a regex that does Z", "explain this error", "generate boilerplate".

```bash
# Foreground — small, fast tasks (results back in seconds)
gemini -p "$(cat <<'EOF'
You are <ROLE>. Mission: <one-line mission>.

Context:
- Working directory: <path>
- Relevant files: <list>
- Prior decisions: <bullets>

Task: <concrete deliverable>

Output: write your final answer to <project>/.orchestrate/<role>-result.md.
Do not reference AI, Claude, Gemini, or LLMs in any output.
EOF
)" -m gemini-2.5-flash --include-directories <path> 2>&1 | tee <project>/.orchestrate/<role>-stdout.log
```

For **moderate** tasks, swap `-m gemini-2.5-flash` for `-m gemini-2.5-pro`.

For **read-only research** where you want Gemini to be able to read files but not edit them, add `--approval-mode plan`:

```bash
gemini -p "<prompt>" -m gemini-2.5-pro --approval-mode plan --include-directories <path>
```

Run multiple Gemini calls in parallel by issuing them as separate Bash calls in one message with `run_in_background: true`, then collecting results.

### Pattern B — Worktree task (Gemini edits files / runs commands)

Best for: mechanical refactors, adding tests for an existing spec, doc generation that touches several files, anything where Gemini needs to run `npm test` / `cargo check` in a loop.

```bash
# Create an isolated worktree (same pattern as /codex)
SLUG="<short-task-slug>"
BRANCH="gemini/$SLUG"
WT=".claude/worktrees/$BRANCH"
git -C <repo> worktree add "$WT" -b "$BRANCH" HEAD

# Launch Gemini headless with auto-edit in the worktree
gemini -C "$WT" -p "$(cat <<'EOF'
<full brief — mission, context, files, success criteria, output file>
When finished, write a summary to .gemini-result.md in this directory.
Do not reference AI, Claude, Gemini, or LLMs in any code, comments, or commits.
EOF
)" -m gemini-2.5-pro --approval-mode auto_edit --include-directories "$WT"
```

Key flags:
- `-C <dir>` — working directory
- `--approval-mode auto_edit` — auto-approves edits but still prompts on shell commands. Use `--yolo` only if you accept fully unattended shell execution inside the worktree (it's sandboxed by the filesystem, not by policy).
- `-p` — non-interactive
- `-m gemini-2.5-pro` — pick the right tier

Run with `run_in_background: true` so you can keep working. When the background task completes:

1. `cat "$WT/.gemini-result.md"` — read what it did
2. `git -C "$WT" log --oneline` and `git -C "$WT" diff <base>..HEAD` — review changes
3. Decide: merge / cherry-pick / discard, then `git worktree remove "$WT"` and `git branch -D "$BRANCH"`

This mirrors `/codex` exactly — same safety story (sandboxed to a worktree, never touching your live tree, results captured to a file before cleanup).

### Steering Gemini

Gemini one-shots have no mid-run steering. If output is wrong:

- **Sharper brief, re-run.** First attempt is diagnostic. Edit `<role>-brief.md` and reissue.
- **Escalate the model.** flash → pro. If pro is still weak, escalate the *engine*: re-dispatch on Claude (`Agent` / sonnet / opus).
- **Decompose smaller.** Gemini struggles when the prompt asks for too many decisions at once. Break into smaller calls.

### Capturing results

Always have Gemini write to a result file in `<project>/.orchestrate/`. Don't pipe long output back through stdout into your context — read the file, summarize, discard.

## Steps 2–7: Same as /orchestrate

For Claude tmux orchestrations, follow `/orchestrate` Steps 2–6 verbatim (tmux setup, spawn, mission board, observe/steer, completion). For agent teams, follow `/orchestrate` Step 0 path 2.

**Step 7 — Synthesize.** Same job. With Gemini in the mix, weight findings by engine: Gemini-flash output on a moderate task should be sanity-checked before you build on it. If two Gemini agents disagree, dispatch a Claude tiebreaker rather than a third Gemini.

## Mission Board Addition

In `<project>/.orchestrate/mission.md`, the **Agents** section gains an **engine** field:

```
- Role: scout-1
  Engine: gemini-2.5-flash
  Brief: .orchestrate/scout-1-brief.md
  Result: .orchestrate/scout-1-result.md
  Status: done
- Role: architect
  Engine: claude-opus
  Brief: .orchestrate/architect-brief.md
  Status: running (Agent tool, in-session)
```

This makes cost and reliability legible at a glance.

## Cost & Quota Hygiene

- Gemini free tier has daily quota limits. Burst-firing 30 flash calls in a minute may rate-limit. Stagger if needed.
- Free tier sends data to Google for model training unless you opt out / use a paid key. If the project handles secrets or sensitive code, do **not** route through Gemini — keep it on Claude. Default to Claude for: anything in `infra/`, anything touching credentials, anything in security-reviewer territory.
- Don't pipe full file contents into Gemini prompts when `--include-directories <path>` will let it read them itself — saves tokens and keeps the prompt small.

## Anti-patterns

- **Gemini for security or auth review.** Free tier, possibly trained on, and the cost of a wrong-but-confident answer is high. Always Claude.
- **Gemini for tasks that need the user's Claude Code subagents.** Gemini can't call `Agent` / specialized reviewers. If the brief depends on those, stay on Claude.
- **Gemini for mid-flight debate.** No inter-agent comms, no steering. Use Claude teams.
- **Gemini editing your live tree.** Always a worktree for write tasks. Never `-C <project-root>` with `--yolo`.
- **Forgetting to verify auth.** Step 0c, every fresh session.
- **Dumping Gemini's full stdout into your context.** Result files exist for a reason.

## Persistence

After the initial task, stay in g-orchestrate mode for follow-ups. Re-triage each new task — many follow-ups are smaller and even more obviously Gemini-tier.

## Undercover — No AI Attribution

Same as `/orchestrate`. Every brief — Claude or Gemini — restates: do not reference AI, Claude, Gemini, or LLM in code, comments, commits, briefs, PRs, or docs. Write as a human developer would.

## Key Principles

- **Triage by effort first, then by mechanism.** Engine choice is a real lever; use it.
- **Cheap-by-default for the long tail.** Most "small" agent work doesn't need Claude.
- **Escalate, don't stack.** flash → pro → Claude. Don't run three flashes when one Claude would have settled it.
- **Sensitive code stays on Claude.** Free Gemini is not free in privacy terms.
- **Briefs and synthesis are still your craft.** The model just renders them.
- **Real colors, no emoji. Stay undercover.**
