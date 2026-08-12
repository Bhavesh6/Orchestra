# ADR-0003 — Multi-agent execution is a hypothesis, not a premise

**Status:** accepted · **Date:** 2026-08-12

## Context

The project began with parallel specialized agents as the central idea. The evidence does not support treating it as a premise.

**Academic.** "Why Do Multi-Agent LLM Systems Fail?" (arXiv 2503.13657, NeurIPS 2025) built a failure taxonomy from 150 annotated traces (κ=0.88), extended to 1600+ across 7 frameworks: 14 failure modes in three categories — system design, inter-agent misalignment, task verification. Its own summary: performance gains on popular benchmarks are "often minimal."

**Vendor behavior — the stronger signal.** Everyone shipping this feature hedges it:

| System | Hedge |
| --- | --- |
| Claude Code agent teams | Behind `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`, **disabled by default**. Docs state teams use significantly more tokens and that single sessions are more effective for sequential work, same-file edits, or many dependencies. |
| Cursor | Supports 8 parallel agents; guidance is to start with 2–3 because merge complexity scales non-linearly. |
| Codex | `max_threads` default 6; `max_depth` exists specifically to prevent recursive delegation. |

When every organization that shipped a feature also ships guardrails against it, that's data.

**Provider constraint.** NVIDIA Build's free tier is **40 requests/minute**. Agentic loops are request-heavy. Parallel agents saturate that before coordination overhead is even the binding constraint.

## Decision

**Multi-agent execution is a hypothesis to be tested at M5, not a premise to build on.**

- Milestones M1–M4 deliver full value with a **single agent**.
- M5 runs a three-arm benchmark: plain agent / agent + contracts + gates / multi-agent + contracts + gates.
- **If the multi-agent arm doesn't clearly win, we stop at L2.** That is a planned outcome, not a failure.
- If we do build it: default single, fan-out is the exception that must argue for itself, and the orchestrator explains when it declines to parallelize.

**Prediction on record:** arm two (single agent + contracts + gates) captures most of the gain. Written down now so it can't be rationalized later.

## Consequences

**Good**
- We can't waste a year on an unvalidated premise.
- M1–M4 are useful regardless of the outcome.
- Forces the single-agent path to be first-class rather than a fallback.

**Bad**
- Less exciting near-term. No impressive parallel-agent demo for months.
- If multi-agent turns out to be a large win, we get there later than a team that gambled and was right.

**Accepted trade.** The downside of being late to a real win is smaller than the downside of spending a year on a false one.
