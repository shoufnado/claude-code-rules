# Global Claude Code Rules

> Applies to every session regardless of project directory.
> Project-specific `CLAUDE.md` files add on top of these.

---

## Model Discipline (cost control)

Claude Code does not auto-pick a model per task, so the cost floor is set by the **default** and
the rest by **self-routing**. Start every task at the cheap tier and escalate only when the task
genuinely needs it — and **announce it** when you do (e.g. "switching to Opus — architecture
decision").

| Tier | When to use |
|---|---|
| **Haiku** (floor) | File reads, searches, log greps, status checks, renames, trivial single-line edits. Delegate broad exploration to the `Explore` subagent instead of burning the main model on 20 file reads. |
| **Sonnet** (default) | Feature work, multi-file edits, refactors, routine debugging, builds, releases, doc/site updates — nearly everything. Global default: `model: sonnet`, `effortLevel: medium`. |
| **Opus** (escalate) | Genuinely hard reasoning only: architecture/design decisions, gnarly multi-system debugging, security/regulatory/compliance analysis, ambiguous problem decomposition. Use `/model opus` or `/model opusplan` (auto-switches to Sonnet for execution after planning). Effort: `high` for deliberative reasoning; `xhigh` for agentic/coding Opus tasks (start here); `max` for genuinely novel research-level problems only. Announce the escalation. |
| **opus[1m]** (skip) | Only when the context genuinely cannot fit in ~200K tokens. The 1M window re-bills a huge history every turn — very expensive for routine work. |

**Workflow subagents:** always pass `model: 'sonnet'` + `effort: 'medium'` on `agent()` opts for
feature-work fan-outs. Reserve Opus/high only for the genuinely hard verify/judge stage. A fan-out
on opus[1m] spawns N Opus[1m] agents and can exhaust the session limit mid-run.

**`opusplan` is the practical default** for sessions that mix design and coding. It runs Opus while in
plan mode (reasoning/architecture) and **auto-switches to Sonnet for execution** — the escalation
decision is made per-phase, so you don't manage it by hand. Set it in `~/.claude/settings.json`
(`"model": "opusplan"`), not via `/model` — the `/model` slash command is **not available in the
VSCode extension** (use the model picker by the input box there, or just set settings.json once).
Note: opusplan is strictly **two-model (Opus + Sonnet)** — it **never** routes to Haiku, and no
built-in main-loop model spans all three tiers. Haiku enters only via subagents (below) or a manual
pick for a trivially cheap session. Use the plain `opusplan` alias, **not** a `[1m]` variant — the
1M context window re-bills the whole history every turn.

**529 / overload is NOT a capability problem.** A weak model never *errors* on a hard task — it just
gives a worse answer. If a request errors and "goes away when you bump to Opus," that's because Opus
and Sonnet have **separate rate-limit/capacity buckets** and you jumped to a less-congested one — not
because the task needed a bigger model. 529 Overloaded is Anthropic-wide capacity, not your quota, and
is auto-retried up to ~10× before you ever see it. **Don't escalate to dodge a 529.** Instead set a
`fallbackModel` (array) in settings.json so Claude Code auto-switches buckets for the congested turn
and returns: `"fallbackModel": ["claude-opus-4-8"]`. There is **no** built-in difficulty-based
auto-downgrade — the model is fixed per session until changed; this tiered discipline IS the routing.

**Caching hygiene** (free savings): switch tiers only at *task boundaries* (a mid-task model switch
invalidates the prompt cache). Keep the top of `CLAUDE.md`/`MEMORY.md` stable — a change near the
top re-bills the entire prefix.

**Per-project overrides:** some repos default to Opus via `.claude/settings.local.json` because they
carry hard reasoning in every session (architecture, regulatory, biomechanics). Sonnet-default projects
can still escalate with `/model opus` for a single hard task.

Pricing reference (per 1M tokens, in/out): Haiku $1/$5 · Sonnet $3/$15 · Opus $5/$25.

---

## Session & Subagent Cost Hygiene

Usage telemetry shows the real spend drivers are *structural*, not per-token. Address them:

1. **Subagent model is enforced, not remembered.** Subagents inherit the main-loop model unless told
   otherwise — so a fan-out from an Opus session spawns N Opus agents. The global cap
   `CLAUDE_CODE_SUBAGENT_MODEL=sonnet` (set in settings.json `env`) forces **every** subagent —
   Task/Agent spawns and Workflow `agent()` spawns — onto Sonnet. ⚠️ This env var **overrides** both
   per-agent `model:` frontmatter and per-call `model:` opts, so it is all-or-nothing: it cannot tier
   "Haiku for grunt work, Sonnet for substantive work." The Sonnet cap is the deliberate 80/20 — it
   kills the expensive Opus-subagent blowups (the dominant usage driver) and Sonnet is already cheap.
   To get true 3-tier routing (Haiku on Explore/grunt agents, Sonnet elsewhere) you must set
   `CLAUDE_CODE_SUBAGENT_MODEL=inherit` **and** pin `model:` in each agent definition — more setup and
   more fragile (an unpinned agent then inherits the main loop, i.e. Opus during plan mode). The
   Workflow tool's `agent()` spawns are the "workflow-subagent" usage line; the same cap governs them.

2. **Context discipline.** Long sessions are expensive even when cached. `/clear` when switching to an
   unrelated task; `/compact` once a single task pushes past ~150k tokens. Don't let one session
   accumulate many unrelated tasks.

3. **Long & parallel sessions are intentional, not ambient.** Background/loop sessions (8+ hours) and
   4+ parallel sessions all draw on **one shared rate limit** — parallelism doesn't buy more quota, it
   just drains it faster and triggers overload. Time-box background work; prefer queueing sequential
   sessions over running 4+ at once unless you genuinely need them simultaneously.

---

## Quality & Verification

Before reporting a task done:

1. **Diff-review first** — re-read every changed line in the diff.
   Ask: does this do exactly what was asked, and nothing else?

2. **Verify by task type** (show the actual output, not a summary):
   - *Code/data pipeline* → run it; paste the real counts/rows/errors.
   - *UI/frontend* → open the browser; describe what you see on the golden path.
   - *API/endpoint* → hit it; show the response.
   - *Library/engine* → call it with a known input; show the return value.

3. **Build output is the evidence** — quote the last few lines of the
   build/typecheck/run. "It should work" is not verification.

4. **Scope discipline** — if review uncovers a secondary bug:
   - Same function you're already in → fix it.
   - Elsewhere → flag it explicitly; don't silently expand scope.

---

## Hard Rules (universal)

1. **NEVER publish to GitHub** unless the user explicitly says "publish", "release to GitHub", or
   "ready for release".
2. **NEVER hardcode secrets** — all keys/tokens in `.env` (gitignored).
3. **NEVER skip pre-commit hooks** (`--no-verify`) unless the user explicitly asks.
4. **NEVER amend published commits** — create new commits instead.
5. **Match scope to request** — a bug fix does not refactor; a feature does not build an abstraction.
