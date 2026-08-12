# Multi-Agent AI Development Platform — Research & Architecture Phase

**Status:** Research complete. No product code written, by design.
**Date:** 2026-08-12

This repository contains the research and architecture specification requested before implementation begins. It deliberately contains **no application code** — per the brief, the first task was to research, analyze, and specify.

---

## The documents

| # | Document | What's in it |
| --- | --- | --- |
| 0 | [Research Method & Sources](docs/00-research-method-and-sources.md) | Evidence grading, the A–E reuse classification, full source list, and what we explicitly *don't* know |
| 1 | [Competitive Architecture Study](docs/01-competitive-architecture-study.md) | Claude Code, Codex, Antigravity, Cursor, OpenCode, Cline + the capability matrix |
| 2 | [MCP & Skills Architecture](docs/02-mcp-and-skills-architecture.md) | The 2026-07-28 spec changes, our three MCP roles, tools vs skills |
| 3 | [Multi-Agent Architecture](docs/03-multi-agent-architecture.md) | Orchestrator, task graph, contracts, blackboard, conflicts, verification |
| 4 | [Model Gateway](docs/04-model-gateway.md) | Provider independence, capability model, conformance suite, router |
| 5 | [Security Architecture](docs/05-security-architecture.md) | Threat model, sandbox/approval axes, trifecta-breaking, secrets, supply chain |
| 6 | [Project Brain](docs/06-project-brain.md) | Structured project state, contracts, decisions, evidence, memory |
| 7 | [Implementation Roadmap](docs/07-implementation-roadmap.md) | Revised phases, risks, tech choices, what we won't build |
| 8 | **[Critique](docs/08-critique.md)** | **Where this idea is wrong. Read this one first.** |

---

## Answers to the ten questions

### 1. What can we confidently reproduce?
Agent loop, tool dispatch, permission engine, context assembly, task graph and scheduler, contract system, verification gates, structured results, workspace management, project state. All class **D** — publicly observable architecture, independently reproducible. The permission semantics in particular are worth copying closely from Claude Code, whose docs expose the edge cases they've already hit (shell-operator awareness, wrapper stripping, deny→ask→allow precedence).

### 2. What should we reuse from open source?
Provider adapters (**LiteLLM** or **Vercel AI SDK**), model catalogue (**models.dev**), MCP SDK, browser automation (**Playwright MCP**, **Chrome DevTools MCP**), embedded tooling (**platformio-mcp**, **serial-mcp**), sandboxing (**Landlock/seccomp**, **gVisor**, **Firecracker**), git worktrees, LSP clients. Also worth a two-week spike: the **OpenHands Agent SDK** (MIT) could collapse much of Phases 0–1.

### 3. What should we integrate through MCP/API?
Everything tool-shaped. Tools, resources, prompts, elicitation (via MRTR), long-running work (Tasks extension), authorization (CIMD). Plus GitHub/GitLab APIs and, at the perimeter only, external coding agents as optional workers.

### 4. What should remain proprietary/provider-specific?
Model weights and inference, provider-specific reasoning controls and caching semantics, provider auth. We consume these through the gateway and never depend on any single one. Anthropic's/OpenAI's/Google's internal orchestration is class **E** — we don't design around it.

### 5. What's missing from current systems?
Six things nobody does (Doc 8 §8.8): contract-driven development where **contracts generate code**; **runtime-enforced** verification-gated completion; evidence-graded claims with automatic demotion; **cross-provider** independent review; trifecta-breaking agent decomposition as security architecture; and **a scheduler that refuses to parallelize** and says why.

### 6. Minimum viable architecture?
Headless server → agent runtime (sandboxed) → tool layer (built-in + MCP) → model gateway → verification gates → structured results. Single agent. No orchestrator, no UI. Everything else is additive.

### 7. Technologies?
TypeScript (Node/Bun) core; Vercel AI SDK or LiteLLM; official MCP TypeScript SDK; HTTP + SSE; files-in-git + SQLite index; Landlock/seccomp → Docker → Firecracker tiering; git worktrees; JSON Schema + Zod. Go is the credible alternative if concurrency and single-binary distribution matter more than AI-library ecosystem.

### 8. First prototype?
**A sandboxed single-agent runtime with real verification gates and structured results, over an HTTP API, with no UI.** Unimpressive to demo, correct to build — it's the baseline against which every multi-agent claim must be measured.

### 9. What NOT to implement initially?
UI, our own tool protocol, our own OAuth, browser engine, embedded wrappers, provider adapters from scratch, agent-to-agent chat, default model-parallel consensus, learned routing, A2A internally, nested agent hierarchies, cloud agent VMs, cross-project memory. Full table in Doc 7 §7.3.

### 10. Biggest technical risks?
Parallelism not paying for itself (R1), cost multiplication (R2), bad decomposition poisoning downstream work (R3), prompt injection via research/browser agents (R4), weak verification gates (R8), debuggability collapse (R9), and — the largest — **scope explosion** (R10): the brief describes 8–10 separate products.

---

## My own read on this

You asked for my thinking, not just a spec. Four things I'd say directly.

**1. The premise is oversold, and I'd reframe the project around what actually holds up.**

I went in expecting the research to support "orchestrate many agents in parallel." It mostly didn't. The academic work (MAST, NeurIPS 2025) found performance gains on benchmarks "often minimal," with 14 failure modes dominated by coordination and verification rather than agent capability. More tellingly: *every* vendor shipping this feature ships it hedged. Anthropic gates agent teams behind an experimental env var, disabled by default, with docs that tell you when not to use it. Cursor supports 8 parallel agents and advises 2–3. Codex caps threads at 6 and added `max_depth` specifically to stop recursive delegation.

When everyone who has shipped a feature also ships guardrails against it, that's data.

The parts of your brief that survive contact with the evidence are the *unglamorous* ones: contracts, verification, structured evidence, provider independence. Those are valuable **on day one with a single agent**. I'd build the project around them and treat parallelism as a capability that appears where it earns its cost.

**2. The single best idea in your brief is §16, and I don't think you've noticed how big it is.**

Contracts between agents. But push it one step further than the brief does: **generate code from the contract**. If `pins.h` is generated from `pinmap.yaml`, the firmware agent physically cannot use a wrong pin — the compiler enforces the hardware agent's decision. You've converted a coordination problem into a build error.

That's the trade that makes this whole architecture work, and it generalizes: OpenAPI → typed clients, schema → migrations, pin map → headers. Every contract that generates code is a category of agent disagreement that can no longer happen, and that cannot silently drift. It's also the answer to "who maintains the Brain" — generated artifacts can't rot without breaking the build.

**3. Verification is the actual moat, and it's cheap.**

"No agent may say Done" is trivial to implement — the runtime rejects `status: complete` unless the gates passed — and it improves output quality regardless of how many agents you run. It's also the only way to *measure* whether any of the fancy parts help. Without gates, every claim in this repo about parallelism or routing is unfalsifiable. That's why I moved verification from your Phase 7 to Phase 0.

**4. There's a security argument for multi-agent that's better than the productivity one.**

This surprised me. The lethal-trifecta framing says an agent with private data + untrusted input + an exfiltration vector is unconditionally vulnerable to injection. 2026 data: attack success rates above 60% on popular agents, and — the part that should change our routing — **more capable models often did worse**, because better instruction-following means better compliance with injected instructions.

Decomposing into agents that each hold at most *two* legs of the trifecta is a real, structural defense (Doc 5 §5.1). The research agent reads the hostile web page but has no repo secrets. The coder agent has the repo but no network and no untrusted input. That's a genuine security property that falls out of the architecture, and I haven't seen anyone build for it deliberately.

If I were pitching this project, I'd lead with **verifiable, contract-driven development with security-motivated agent separation** — and mention parallelism third.

---

### One caveat on the research itself

The container's network policy blocked several vendor domains: `modelcontextprotocol.io`, `antigravity.google`, `cursor.com`, `opencode.ai`, `developers.openai.com`, `deepwiki.com`. Claims about **Antigravity, Cursor, OpenCode, and Codex** therefore lean on secondary sources and are marked **[2°]** throughout. The Claude Code and MCP-spec sections are primary-sourced and solid. Before committing to the roadmap, re-verify the [2°] claims flagged ⚠️ from an unblocked network — they're the ones that would change a design decision.
