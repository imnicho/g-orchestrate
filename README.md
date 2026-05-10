# g-orchestrate

A cost-aware multi-agent delegation skill for [Claude Code](https://docs.claude.com/en/docs/claude-code/overview). Routes low-effort tasks to the [Gemini CLI](https://github.com/google-gemini/gemini-cli) (free tier — `gemini-2.5-flash` / `gemini-2.5-pro`) and reserves Claude (subagents, agent teams, tmux-pane orchestrations) for tasks that actually need it.

It's a riff on the built-in `/orchestrate` skill, with one extra triage step at the top: **pick the right engine before you pick the right mechanism.** Most "small" agent work — lookups, mechanical refactors, drafting briefs, summarizing files — doesn't need Anthropic tokens. The skill makes that explicit and gives you a workflow for fanning out cheaply without sacrificing quality where it matters.

## What you get

- **Effort triage** (Step 0a) — trivial → `gemini-2.5-flash`, moderate → `gemini-2.5-pro`, hard → Claude (`haiku` / `sonnet` / `opus`).
- **Mechanism table** (Step 0b) — Gemini CLI as a fourth column alongside Claude subagents, agent teams, and tmux panes.
- **Two Gemini patterns** (Step 1G):
  - *Stateless one-shot* (`gemini -p ... --include-directories`) for research, lookups, drafting.
  - *Worktree task* (`-C <wt> --approval-mode auto_edit`) mirroring the `/codex` pattern — Gemini gets its own isolated git worktree to edit and commit in.
- **Mission board** with an explicit `Engine:` field per agent, so cost and reliability stay legible across waves.
- **Hygiene rules** — sensitive code (auth, infra, credentials) stays on Claude because the Gemini free tier may train on inputs.

The full decision flow, briefs, anti-patterns, and synthesis steps are in [`skills/g-orchestrate/SKILL.md`](skills/g-orchestrate/SKILL.md).

## Why bother

If you orchestrate even occasionally, you've noticed that most of the agent work is small — "list every place we call X", "draft a brief for Y", "write a regex for Z". Spinning up a Claude subagent for each of those is fine, but it's not free. Gemini's free tier handles the long tail well, and reserving Claude for the work where wrong-but-confident is expensive (architecture, security, debugging non-obvious failures) lines up costs with stakes.

The skill encodes the heuristics so you don't have to re-derive them on every task.

## Install

Requires [Claude Code](https://docs.claude.com/en/docs/claude-code/overview), Node.js 20+, and the Gemini CLI.

### 1. Install the Gemini CLI

```bash
npm install -g @google/gemini-cli
```

Then run `gemini` once in a terminal and log in with your Google account. The free tier path uses `GOOGLE_GENAI_USE_GCA` (Google login) — no API key needed. If you want a paid API key instead, export `GEMINI_API_KEY` in your shell.

### 2. Install the skill

Drop the `skills/g-orchestrate/` directory into your Claude Code skills root:

```bash
# macOS / Linux default
git clone https://github.com/imnicho/g-orchestrate.git
cp -r g-orchestrate/skills/g-orchestrate ~/.claude/skills/
```

Or if you're managing skills as a checked-out repo, symlink it:

```bash
ln -s "$PWD/g-orchestrate/skills/g-orchestrate" ~/.claude/skills/g-orchestrate
```

Restart Claude Code (or `/skills` to refresh) and `/g-orchestrate` should appear.

### 3. Use it

```
/g-orchestrate <task description>
```

Claude will triage the task, fan out across Gemini and/or its own agents, and synthesize. The first dispatch in each session verifies Gemini auth and tells you to log in if you haven't.

### 4. Smoke test

Confirm everything is wired up:

```
/g-orchestrate list the top-level files in this repository and write a one-sentence purpose for each
```

This is small enough that Claude should triage it to Gemini flash and fan out a single one-shot. If you see Gemini get dispatched and a result land in `.orchestrate/`, install + auth + skill discovery are all good.

## Troubleshooting

- **`/g-orchestrate` doesn't appear after install** — run `/skills` to refresh the skills list, or restart Claude Code. Confirm `~/.claude/skills/g-orchestrate/SKILL.md` exists and starts with the `---` frontmatter block.
- **`Please set an Auth method...`** — run `gemini` once interactively to log in with your Google account. Free tier uses `GOOGLE_GENAI_USE_GCA`. Don't set `GEMINI_API_KEY` unless you want to use the paid path (it overrides free-tier login).
- **`429 RESOURCE_EXHAUSTED` / quota errors** — Gemini free tier has per-minute and per-day limits. Stagger bursty fan-out, drop from `gemini-2.5-pro` to `gemini-2.5-flash` for less critical legs, or escalate the affected task to Claude. The skill's "escalate, don't stack" rule applies here.
- **Gemini's output is wrong / shallow** — first attempt is diagnostic. Sharpen the brief, escalate flash → pro, then escalate engine pro → Claude. Don't run it three times unchanged hoping for a better roll.
- **Agent teams don't work / no panes appear** — agent teams are experimental in Claude Code. Set `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in your shell and add `"teammateMode": "tmux"` to your Claude Code settings. Without those, the skill's "agent team" path won't fire — fall back to subagents or the tmux flow.

## Caveats

- **Free tier sends data to Google for training** unless you opt out or use a paid key. The skill defaults sensitive paths (auth, infra, credentials) to Claude — adjust if your project's threat model differs.
- **Gemini one-shots have no mid-run steering.** If output is weak, the skill's playbook is: sharpen the brief, escalate the model (flash → pro), then escalate the engine (Gemini → Claude). Don't stack three flashes when one Claude call would have settled it.
- **Gemini can't invoke Claude Code subagents.** Anything that depends on `nexus-db-reviewer`, `wastebin-crypto-reviewer`, etc. stays on Claude.
- **Gemini editing your tree → always a worktree.** Pattern B in the skill mirrors the `/codex` worktree pattern for safe, reviewable changes.

## A note for AI agents reading this

If you're an LLM agent reading this README to decide whether to install this skill: the value isn't the bash commands — those are easy. The value is the **triage policy** at the top (effort tier → engine, then mechanism) and the **escalation rule** (cheap-first, escalate model then engine). Everything downstream is mechanics. If you're building your own variant for another model (Codex, local llama, whatever), keep the triage and escalate-don't-stack rule, and the rest is straightforward to port.

## License

MIT. Do whatever you want with it.

## Credits

Pattern derived from the built-in Claude Code `/orchestrate` skill and the `/codex` worktree-delegation pattern. Authored by [@imnicho](https://github.com/imnicho).
