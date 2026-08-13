# ADR-0002 — Contracts and gates are the differentiator

**Status:** accepted, **amended 2026-08-12** · **Date:** 2026-08-12

> ### ⚠️ Amendment — the original claim was wrong
>
> This ADR originally stated that nothing in the field does contracts-that-generate-artifacts or enforced verification. That is **false**. [Document 10](../research/10-prior-art-sdd-landscape.md) surveys the Spec-Driven Development landscape, which the original research missed entirely because it looked only at agentic *environments* and never at the methodology tooling layered on top.
>
> **Tessl** generates code from specs with `// GENERATED FROM SPEC - DO NOT EDIT` markers. **agent-gates** gates completion on evidence. **GitHub Spec Kit** has ~111k stars and 30+ agent integrations.
>
> The differentiator survives, but only when stated precisely:
>
> **Every SDD tool's spec is prose an LLM is asked to follow — the authority is convention. Orchestra's constraints are executable functions and generated artifacts the compiler enforces — the authority is the toolchain.**
>
> `unique_pins` and `fk_integrity` are functions that run in milliseconds and cannot be partially obeyed. EARS phrasing is not an executable contract. That gap is named by third-party analysis of the incumbents, not invented by us.
>
> Also revised: Orchestra covers **interfaces only** (~20–30% of work), which deliberately avoids the rigidity criticism that lands on Tessl's 1:1 spec-per-file mapping — an approach reviewers compare to early-2000s Model-Driven Development.
>
> **Strategic consequence:** do not compete with Spec Kit on specs. Compose with it. It answers *what to build*; Orchestra answers *this interface decision is now mechanically enforced*.

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
