# codex-fusion

[中文](./README.md)

A Codex CLI configuration. Expensive model makes decisions, cheap model does the work, a third model reviews independently.

If you're using OpenCode instead of Codex, see [opencode-fusion](https://github.com/faceair/opencode-fusion).

Inspired by Cognition's [Devin Fusion](https://cognition.com/blog/devin-fusion) — the sidekick architecture comes from them.

## Installation

Copy the [github.com/faceair/codex-fusion](https://github.com/faceair/codex-fusion) repo's configuration files into `~/.codex`. You can clone and copy manually, or send this prompt to Codex and let it install for you:

```
Install configuration from https://github.com/faceair/codex-fusion to ~/.codex:
1. Merge config.toml into ~/.codex/config.toml (preserve the user's other config; on key conflict, this repo's value wins)
2. Copy the agents/ directory to ~/.codex/agents/, overwriting same-named files
3. Copy the instructions/ directory to ~/.codex/instructions/, overwriting same-named files
4. List the final effective file paths
```

Restart Codex after installation for the configuration to take effect.

## What it solves

A few common problems when using a single-model agent for engineering work:

- Frontier model time gets spent running tests and reading files — wasting money. Switch everything to a cheap model and decision quality drops. Cognition's data shows the sidekick architecture cuts cost 35% while maintaining frontier performance; delegating test suites to sidekick saves 62%, mechanical removal tasks save 32%.
- One model writes the code, reviews the code, and approves the code. No independent perspective, edge cases get missed.
- "Ask another model" tools lose context cache on every cross-model call, and you pay the full prompt cost again. On a long task, this adds up fast.
- Long tasks stall mid-way waiting for you to type "continue". Codex's native goals auto-continue until the objective is done or a concrete blocker stops progress.
- Context compaction wipes working memory. Codex's goals and thread tools recover the objective and prior history.

## How it works

Codex does not proactively spawn subagents by default — even with multi-agent enabled, the default mode is `ExplicitRequestOnly` (spawning only happens when the user or AGENTS.md explicitly asks). codex-fusion uses `usage_hint_text` config to inject a developer authorization so fusion treats proactive delegation as the default path for non-trivial work without waiting for an explicit request. `usage_hint_text` is only injected into the primary agent, not subagents.

A system-level hard constraint is enforced via `agents.max_depth = 1` + multi-agent v1 mode: the root session (depth 0) can spawn subagents (depth 1), but subagents cannot spawn further — the v2 spawn handler ignores `max_depth` by design, while v1 enforces it and returns `"Agent depth limit reached"`.

Two parallel agents, each with its own tools and cached context. The main agent decides which work to give the sidekick and which to do itself:

![Sidekick architecture: a frontier main agent and a small sidekick agent running in parallel, each with its own cached context](https://cognition.com/_next/static/media/sidekick-diagram.153unbtaywtzg.png)

codex-fusion adds a third read-only agent (reviewer) for independent review on top of this. Three agents, each with its own model and context:

```
┌─────────────────────────────────────────────────┐
│  fusion (expensive model)                       │
│  owns: decisions, final review                  │
│                                                 │
│  delegates via spawn_agent ─────────┐           │
│  consults via spawn_agent ──────┐   │           │
│                                 ▼   ▼           │
│                     ┌──────────────┐ ┌─────────┐│
│                     │ reviewer     │ │ sidekick││
│                     │ (read-only)  │ │ (cheap) ││
│                     │ risk review   │ │ execute ││
│                     │ + adversarial │ │ discover││
│                     │               │ │ verify  ││
│                     └──────────────┘ └─────────┘│
└─────────────────────────────────────────────────┘
```

**fusion** is the main agent you talk to, defaulting to `glm-5.2`. It takes minimal actions — reads only what's necessary, makes the calls that need judgment, and delegates the rest by default. It owns understanding requirements, making decisions, and controlling delivery quality, doing the final review itself rather than letting sidekick do it.

**sidekick** defaults to `gpt-5.5` (reasoning effort `medium`). It handles the mechanical load: reading code, editing files, running tests, diagnosing failures. It returns locatable evidence and observations to fusion, not conclusions.

**reviewer** defaults to `gemini-3.5-flash`, read-only. It reviews high-risk changes before implementation and does independent code review before delivery. For changes touching untrusted input, persistence, or concurrency, it walks each input path from an attacker's perspective.

fusion creates subagents with `spawn_agent`, gets back an `agent_id`, and continues the conversation with `send_input` — no need to start fresh each time. The same sidekick and reviewer carry through an entire workflow, accumulating context.

### When to delegate to sidekick

This is not a single-prompt router that picks one model for the whole task. Fusion decides per-step which agent should do what:

- **Hand off slow verification.** Sidekick runs the test suite while fusion moves on to the next decision.
- **Take back judgment-heavy work.** When sidekick hits a decision point — API shape, error semantics, cross-module boundary — fusion takes it back instead of letting the cheap model guess.
- **Send targeted follow-ups.** Sidekick found something unexpected? Fusion sends a focused question back instead of re-reading the code itself.
- **Don't delegate when judgment is the deliverable.** Hard features that need subtle intent (e.g. cross-team search UI decisions) lose intent when delegated to a cheap model — the result comes out wrong.

### When to consult reviewer

Two scenarios:

**Before implementation**, when the change is high-risk: shared API contracts, cross-subsystem boundaries, lifecycle/concurrency/persistence semantics, security/credentials/privacy, production-critical paths, the same approach failing repeatedly, confidence still low after local verification.

**Before delivery**, for any non-trivial change, reviewer does a code review pass: correctness, completeness, regressions, architectural coherence. For high-risk changes, it also performs adversarial review — probing the diff from an attacker's perspective:

- *What if this input is 50MB instead of 5KB?*
- *What if a timestamp comes from the future?*
- *What if a background worker gets killed mid-task and retries?*
- *What if two users submit the same request simultaneously?*

Each finding traces the full path: entry point → processing → storage → output → side effects.

### Reviewer loop for open-ended work

Some tasks can't be fully planned upfront: performance optimization, ambiguous root-cause investigation, architecture cleanup. These use a reviewer loop:

1. Complete a todo, bring evidence to reviewer
2. Reviewer decides the next step: `continue` (same direction), `pivot` (change direction), `stop` (no meaningful next step), `blocked` (missing evidence or prerequisite)
3. Execute the next step, loop back to reviewer
4. Until reviewer says `stop` and the work is verified, or `blocked` with a concrete blocker

### First principles

When a bug fix, architecture decision, or approach choice is on the table, agents reason from fundamental facts and constraints instead of reaching for the closest pattern in training data.

- *Symptom fix:* "The feed is broken, let me fix the fetcher." The same bug comes back next week.
- *Root cause fix:* "The traffic routing layer has a latent failure mode. The fetcher was just the first victim. Fix the routing." The bug class is eliminated.

## Good fit for

- Long tasks that need to keep running without stalling
- Complex debugging where the obvious fix treats the symptom
- Multi-stage refactors with judgment-heavy decisions
- High-risk changes that need independent review before shipping
- Open-ended work where the next step can't be planned upfront and a reviewer loop helps converge

If you just want a lightweight chat assistant, this is probably overkill.

## License

[MIT](./LICENSE)
