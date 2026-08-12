# ADR-0007 — Interface strategy: inherit, then extend, fork last

**Status:** accepted · **Date:** 2026-08-12

## Context

Orchestra is meant to be an application you use — the same category of thing as Claude Code (terminal) or Antigravity (IDE). The question is not *whether* it has an interface, but **when** it gets built and **what shape** it takes.

The roadmap puts the UI at M8, which reads like the interface is an afterthought. It isn't — but the reasoning needs recording, because "UI last" is easy to misread as "UI never."

**What the research shows:**

- **OpenCode is client–server.** The TUI is only a presentation client; it never executes agent work itself. Everything flows over HTTP + event streams to a server that owns sessions, context, tools and providers. Clients include TUI, desktop, web, IDE and SDK.
- **Cursor, Windsurf and Antigravity are all VS Code forks.** Antigravity is a heavily modified fork of VS Code, with code analysis suggesting it forked Windsurf (itself a fork) — references to Cascade, Windsurf's agent system, appear in its codebase.
- **Antigravity's real innovation isn't the editor.** It's the **Agent Manager** as the primary surface, with the editor as a peer, and **artifacts** — implementation plans, diffs, screenshots, browser recordings, test reports — as the review surface. The insight: a human cannot supervise N agents by reading N transcripts.

## Decision

**Four stages, in this order.**

**1. Inherit (M0).** OpenCode's TUI is the interface from day one. Nothing to build. This is the main reason "UI last" costs nothing — there is no period without a usable interface.

**2. Surface through MCP (M3).** Contracts and gates appear inside whatever agent the user already runs — OpenCode, Claude Code, Cline, Codex, Cursor. Roughly a week of work for an interface in five products.

**3. Agent manager (M7).** The Antigravity-shaped surface, and the first thing that is *ours*. Artifact-centric: plans, diffs, gate results, cost — not scrolling transcripts. A web client against the L1 server.

**4. Editor integration (M8).** Orchestra TUI, then a **VS Code extension**. **Fork only if the extension API demonstrably blocks a required capability.**

## Why extension before fork

Forking VS Code is the established path and it works — three products prove it. It's also a permanent maintenance tax: every upstream release must be merged forever, and every extension-API change is yours to absorb. Cursor, Windsurf and Google all had teams for that.

More importantly: **a fork is only justified by capabilities the extension API can't reach.** We don't yet know that we need any. Deciding to fork before hitting that wall is buying a large recurring cost against a hypothetical requirement.

If we do hit the wall, forking later is not meaningfully harder than forking now — the agent logic lives in L1/L2 behind an API, not in the editor.

## Consequences

**Good**
- A working interface exists at every stage, including stage zero.
- Interface work targets a server API already proven by a real client, so we're not designing an API and a UI simultaneously.
- No maintenance tax until something forces it.
- If OpenCode is later replaced as L1, the clients survive — they talk to an API, not to OpenCode internals.

**Bad**
- Until M7, the interface is someone else's and looks like it. No Orchestra identity in the product for months.
- A VS Code extension is more constrained than a fork — sandboxed APIs, less control over layout and chrome.
- If we eventually fork, some extension work is thrown away.

**Accepted.** The alternative — building an agent manager first — produces a dashboard displaying agents that fail in parallel, because there is no mature runtime under it. Antigravity's manager is impressive precisely because it sits on one.
