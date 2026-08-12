# ADR-0001 — Reuse OpenCode as the agent runtime

**Status:** accepted · **Date:** 2026-08-12 · **Supersedes:** none

## Context

The original plan was to build our own agent runtime — loop, tools, context management, permissions, sessions, MCP client, LSP, TUI.

Research into what that actually costs:

| Part | From scratch |
| --- | --- |
| Agent loop | ~1 week |
| **Edit tool that reliably applies changes** | **4–8 weeks** |
| Permission system with real edge cases | 3–6 weeks |
| Context management / compaction | 4+ weeks, ongoing |
| Sessions, checkpoints, resume | 2–3 weeks |
| MCP client, skills, LSP, TUI | ~10 weeks |

Roughly **12–18 months solo** before it's something you'd use daily.

Two findings settled it:

1. **Google forked.** Antigravity is a heavily modified fork of VS Code — with credible code analysis suggesting it's actually a fork of Windsurf (itself a VS Code fork), including references to Cascade in its codebase. Cursor and Windsurf are also VS Code forks. A company with a reported $2.4B did not build an IDE from zero.
2. **The loop is easy; tool reliability is hard.** Published experience building an internal coding agent found that investment in tool design — targeted edit model, read-before-edit policy, layered shell guardrails — produced more reliability gain than any prompt engineering. That's the part that takes months and that nobody demos.

## Decision

**Reuse OpenCode (MIT) as layer 1.** Build Orchestra as layers on top:

- **L1 Agent runtime** — OpenCode. Loop, tools, MCP, LSP, sessions, permissions, 75+ providers.
- **L2 Contracts + gates** — ours, shipped as an MCP server so it works in any agent.
- **L3 Orchestrator + artifacts** — ours, later, gated on evidence.

Extend-vs-fork is deferred to ADR-0006 after reading the source.

## Consequences

**Good**
- Inherit ~80% of the work; 3–5 months to something differentiated instead of 12–18.
- L2 as an MCP server works in Claude Code, Cline, Codex and Cursor too — not just OpenCode.
- Effort concentrates on the part nobody else has.

**Bad**
- We inherit OpenCode's abstractions and bugs.
- Upstream changes can break us.
- Less deep understanding of the runtime than building it would give. *Mitigated by the M0 throwaway spike — build a bare loop first so we can read their source with informed eyes.*

**Reversible?** Yes, and cheaply, because L2 talks MCP rather than OpenCode-specific APIs. If OpenCode becomes a bad host we swap L1 without rewriting L2. That's the main reason for the MCP boundary.
