# Claude Code Rules

> Global `CLAUDE.md` for all projects — tiered model self-routing, quality verification, and hard rules.

This is the global instruction file loaded by [Claude Code](https://claude.ai/code) at the start of every session, regardless of project. Project-specific `CLAUDE.md` files add on top of these.

---

## Model Discipline (cost control)

Claude Code does not auto-pick a model per task, so the cost floor is set by the **default** and the rest by **self-routing**. Start every task at the cheap tier and escalate only when the task genuinely needs it — and **announce it** when you do.

| Tier | When to use |
|---|---|
| **Haiku** (floor) | File reads, searches, log greps, status checks, renames, trivial single-line edits. Delegate broad exploration to the `Explore` subagent instead of burning the main model on 20 file reads. |
| **Sonnet** (default) | Feature work, multi-file edits, refactors, routine debugging, builds, releases, doc/site updates — nearly everything. Global default: `model: sonnet`, `effortLevel: medium`. |
| **Opus** (escalate) | Genuinely hard reasoning only: architecture/design decisions, gnarly multi-system debugging, security/regulatory/compliance analysis, ambiguous problem decomposition. Use `/model opus` or `/model opusplan` (auto-switches to Sonnet for execution after planning). Effort: `high` for deliberative reasoning; `xhigh` for agentic/coding Opus tasks (start here); `max` for genuinely novel research-level problems only. Announce the escalation. |
| **opus[1m]** (skip) | Only when the context genuinely cannot fit in ~200K tokens. The 1M window re-bills a huge history every turn — very expensive for routine work. |

**Workflow subagents:** always pass `model: 'sonnet'` + `effort: 'medium'` on `agent()` opts for feature-work fan-outs. Reserve Opus/high only for the genuinely hard verify/judge stage.

**Caching hygiene:** switch tiers only at task boundaries — a mid-task model switch invalidates the prompt cache. Keep the top of `CLAUDE.md`/`MEMORY.md` stable.

**Per-project overrides:** repos carrying hard reasoning in every session (architecture, regulatory, biomechanics) default to Opus via `.claude/settings.local.json`. Sonnet-default projects can still escalate with `/model opus` for a single hard task.

Pricing reference (per 1M tokens, in/out): Haiku $1/$5 · Sonnet $3/$15 · Opus $5/$25.

---

## Quality & Verification

Before reporting a task done:

1. **Diff-review first** — re-read every changed line in the diff. Ask: does this do exactly what was asked, and nothing else?

2. **Verify by task type** (show the actual output, not a summary):
   - *Code/data pipeline* → run it; paste the real counts/rows/errors.
   - *UI/frontend* → open the browser; describe what you see on the golden path.
   - *API/endpoint* → hit it; show the response.
   - *Library/engine* → call it with a known input; show the return value.

3. **Build output is the evidence** — quote the last few lines of the build/typecheck/run. "It should work" is not verification.

4. **Scope discipline** — if review uncovers a secondary bug:
   - Same function you're already in → fix it.
   - Elsewhere → flag it explicitly; don't silently expand scope.

---

## Hard Rules (universal)

1. **NEVER publish to GitHub** unless the user explicitly says "publish", "release to GitHub", or "ready for release".
2. **NEVER hardcode secrets** — all keys/tokens in `.env` (gitignored).
3. **NEVER skip pre-commit hooks** (`--no-verify`) unless the user explicitly asks.
4. **NEVER amend published commits** — create new commits instead.
5. **Match scope to request** — a bug fix does not refactor; a feature does not build an abstraction.

---

## Usage

Copy `CLAUDE.md` to `~/.claude/CLAUDE.md` (global, applies to all projects) or to any project root (project-scoped). Project files layer on top of the global file.

```bash
cp CLAUDE.md ~/.claude/CLAUDE.md
```
