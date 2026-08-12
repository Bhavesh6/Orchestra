# ADR-0002 — Contracts and gates are the differentiator

**Status:** accepted · **Date:** 2026-08-12

## Context

The project originally positioned **model-agnosticism** and **multi-agent orchestration** as the differentiators. Competitive research found neither is:

- **Model-agnostic:** OpenCode supports 75+ providers, Cline 33+. Solved, free, not differentiating.
- **Multi-agent orchestration:** Claude Code (agent teams), Codex (subagents), Antigravity (agent manager), Cursor (parallel agents) all ship it.
- **Parallel isolated workspaces:** Cursor shipped git-worktree agents in 2.0.

Meanwhile, nothing in the surveyed field does:

1. Contracts that **generate authoritative artifacts**, making violations compile errors
2. **Runtime-enforced** verification-gated completion
3. Evidence-graded claims with automatic demotion of unsupported assertions

## Decision

**The differentiator is: contracts that generate authoritative artifacts, plus runtime-enforced verification gates.**

Model independence remains an architectural *requirement* — it's how we survive rate limits and outages — but it is not the product claim.

Two properties are mandatory for anything in the core:

1. **Useful with a single agent.** If a feature only pays off with orchestration, it isn't core.
2. **Deterministic where possible.** If a check can be a validator instead of a model call, it must be.

## Consequences

**Good**
- Scope collapses from a platform to a tool. Weeks, not years.
- Value on day one, with one agent, in every domain.
- Works inside existing agents via MCP — no adoption cliff.
- No AI risk in the core: a YAML parser, a template, and a compiler either work or don't.

**Bad**
- Less impressive to pitch than "orchestrates ten agents across five providers."
- Contracts only cover interfaces — roughly 20–30% of coding work. Gates cover the rest, but the contract half is narrower than the original vision implied.
- Requires discipline to keep prose out of `contracts/`.

**Risk:** contract maintenance may cost more than the errors it prevents. That has no armchair answer — M4 measures it. `authoritative: true` generation is the main mitigation, since generated artifacts can't drift silently.
