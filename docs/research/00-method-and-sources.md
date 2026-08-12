# Document 0 — Research Method, Evidence Grading, and Sources

**Date of research:** 2026-08-12
**Status:** Research phase. No product code has been written yet, by design.

---

## 0.1 Why this document exists first

The brief asked for research "using current official documentation and authoritative sources," and explicitly warned against designing around imaginary internals. That warning is the single most important constraint in this whole project, so the grading scheme comes before the findings.

Agentic coding tools move fast. Several things I "knew" going in were wrong by the time I checked:

| I assumed | Reality as of 2026-08 |
| --- | --- |
| MCP is a stateful protocol with `initialize` handshake + `Mcp-Session-Id` | The **2026-07-28** spec removed both. The core is now **stateless**. |
| MCP `sampling` is the way a server borrows the client's model | `sampling`, `roots`, and `logging` are **deprecated** as of 2026-07-28. |
| MCP `tasks` is a core protocol feature | Moved **out of core into an Extension** (contributed by AWS). |
| Cursor agents are mostly cloud VMs | Since Cursor 2.0, local agents run in **git worktrees**; cloud VMs are one of several modes. |
| Codex subagents are ad-hoc | They are **declarative TOML** files under `.codex/agents/` with `max_threads`/`max_depth` caps. |

If we had started coding two months ago against MCP sampling, we would now be building on a deprecated primitive. This is the concrete argument for a research phase.

---

## 0.2 Evidence grades

Every substantive claim in Documents 1–8 carries one of these markers.

| Marker | Meaning |
| --- | --- |
| **[Doc]** | Stated in the vendor's own current documentation, which I fetched and read. |
| **[Src]** | Read from public source code or a public spec/SEP repository. |
| **[2°]** | Secondary source (technical blog, aggregator, press release). Directionally reliable, details may drift. |
| **[Inf]** | My inference from observable behavior or documented constraints. Explicitly *not* vendor-confirmed. |
| **[Unk]** | Not publicly known. We must not design as if we know it. |

### Egress caveat — read this

The research container's network policy **blocked several primary domains**:

`modelcontextprotocol.io`, `antigravity.google`, `cursor.com`, `opencode.ai`, `developers.openai.com`, `deepwiki.com`, `smartscope.blog`.

Where a vendor's own docs were unreachable I fell back to secondary sources and marked the claim **[2°]**. This affects **Antigravity, Cursor, OpenCode, and Codex most heavily**. Claude Code (`code.claude.com`) and the MCP spec blog (`blog.modelcontextprotocol.io`) were reachable, so those sections are the best-evidenced.

**Action item before we commit to the roadmap:** re-verify every [2°] claim about Cursor, Codex, Antigravity, and OpenCode from an unblocked network. I have flagged the ones that would actually change a design decision with ⚠️.

---

## 0.3 The A–E reuse classification (brief §29)

Applied to every capability in Document 1's table.

| Class | Meaning | Our posture |
| --- | --- | --- |
| **A** | Open standard / open protocol | **Implement the standard.** Never fork it. |
| **B** | Open-source implementation under a usable license | **Reuse the library.** Don't rewrite. |
| **C** | Public API we can legitimately integrate with | **Integrate as an optional backend.** Never a dependency of the core. |
| **D** | Architecture/pattern we can independently reproduce | **Build ourselves.** This is where our engineering effort goes. |
| **E** | Proprietary internals, not publicly known | **Do not design around it.** Assume nothing. |

The discipline this enforces: if a capability is class **E**, we are not allowed to write "works like Claude Code's router" in a spec. We only get to write what we can observe or standardize.

---

## 0.4 Explicit non-knowledge (class E)

I want these on the record so nobody later mistakes speculation for spec:

- **Anthropic's system prompts for Claude Code**, its internal context-compaction heuristics, and any model-side routing. The docs describe *user-facing* model selection (`/model`, `availableModels` allowlist, per-subagent `model:`), not an internal quality router. **[Unk]**
- **OpenAI's Codex server-side orchestration** for Codex Cloud, and how GPT-5.5 is served. **[Unk]**
- **Google Antigravity's agent runtime internals** and how its browser subagent model differs from the main model beyond "it is a different, specialized model." **[2°/Unk]**
- **Cursor's cloud agent scheduler and snapshot infrastructure.** **[Unk]**
- **Any provider's training data, tool-choice fine-tuning, or hidden reranking.** **[Unk]**

Where a competitor's behavior is class E but the *capability* is desirable, we reproduce the capability from first principles (class D) and say so.

---

## 0.5 Source list

### Primary — fetched and read directly

| Source | Used for |
| --- | --- |
| [Claude Code — Subagents](https://code.claude.com/docs/en/sub-agents) | Subagent frontmatter, context isolation, tool allowlists, per-agent model |
| [Claude Code — Agent teams](https://code.claude.com/docs/en/agent-teams) | Team lead/teammate model, mailboxes, shared task list, file locking, limitations, token-cost warnings |
| [Claude Code — Skills](https://code.claude.com/docs/en/skills) | `SKILL.md` format, frontmatter (`name`, `description`, `allowed-tools`, `context: fork`, `agent:`, `disable-model-invocation`), progressive disclosure |
| [Claude Code — Permissions](https://code.claude.com/docs/en/permissions) | deny→ask→allow precedence, rule syntax, shell-operator awareness, wrapper stripping, sandbox pointer |
| [MCP Blog — The 2026-07-28 Specification](https://blog.modelcontextprotocol.io/posts/2026-07-28/) | Stateless core, MRTR, header routing, cacheable lists, authz hardening, deprecations |
| [MCP SEP 2663 — Tasks extension](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/seps/2663-tasks-extension.md) | Tasks moved to extension |

### Secondary — search-derived, marked [2°]

| Topic | Sources |
| --- | --- |
| MCP 2026-07-28 breaking changes | [Stacktree](https://stacktr.ee/blog/mcp-2026-spec-changes), [mcpservers.org](https://blog.mcpservers.org/posts/mcp-spec-2026-07-28), [WorkOS](https://workos.com/blog/everything-your-team-needs-to-know-about-mcp-in-2026) |
| Codex subagents / TOML / caps | [Codex KB](https://codex.danielvaughan.com/2026/03/26/codex-cli-subagents-toml-parallelism/), [Simon Willison](https://simonwillison.net/2026/Mar/16/codex-subagents/), [Firecrawl](https://www.firecrawl.dev/blog/codex-multi-agent-orchestration) |
| Codex sandbox/approvals | [Vincent Schmalbach](https://www.vincentschmalbach.com/how-codex-cli-flags-actually-work-full-auto-sandbox-and-bypass/), [Digitally Applied](https://www.digitalapplied.com/blog/codex-cli-deep-dive-config-profiles-sandbox-2026) |
| Antigravity | [Google Developers Blog](https://developers.googleblog.com/build-with-google-antigravity-our-new-agentic-development-platform/), [Scalable Path](https://www.scalablepath.com/ai/google-antigravity-review), [Arjan KC](https://www.arjankc.com.np/blog/google-antigravity-agent-manager-explained/) |
| Cursor worktrees / parallel agents | [StartupHub](https://www.startuphub.ai/ai-news/artificial-intelligence/2026/cursor-s-ai-agents-get-worktree-boost), [DEV](https://dev.to/arifszn/git-worktrees-the-power-behind-cursors-parallel-agents-19j1), [Digitally Applied](https://www.digitalapplied.com/blog/cursor-3-agents-window-complete-guide) |
| OpenCode architecture | [Inside OpenCode](https://falexm.medium.com/inside-opencode-understanding-the-architecture-behind-the-ai-runtime-01236d9370ff), [Coding agents internals](https://cefboud.com/posts/coding-agents-internals-opencode-deepdive/) |
| Cline architecture | [Cline repo](https://github.com/cline/cline), [Plan/Act](https://deepwiki.com/cline/cline/3.4-plan-and-act-modes) |
| Sandboxing | [Northflank](https://northflank.com/blog/how-to-sandbox-ai-agents), [coding agent sandbox list](https://gist.github.com/wincent/2752d8d97727577050c043e4ff9e386e) |
| Prompt injection | [Sysdig guide](https://www.sysdig.com/learn-cloud-native/prompt-injection), [Airia](https://airia.com/ai-security-in-2026-prompt-injection-the-lethal-trifecta-and-how-to-defend/), [MDPI review](https://www.mdpi.com/2078-2489/17/1/54) |
| Multi-agent failure evidence | [Why Do Multi-Agent LLM Systems Fail? (arXiv 2503.13657)](https://arxiv.org/abs/2503.13657), [NeurIPS 2025 poster](https://neurips.cc/virtual/2025/poster/121528) |
| OpenHands SDK | [arXiv 2511.03690](https://arxiv.org/abs/2511.03690), [OpenHands blog](https://www.openhands.dev/blog/open-source-ai-coding-agents) |
| A2A status | [Linux Foundation press](https://www.linuxfoundation.org/press/a2a-protocol-surpasses-150-organizations-lands-in-major-cloud-platforms-and-sees-enterprise-production-use-in-first-year) |
| Embedded MCP tooling | [platformio-mcp](https://www.npmjs.com/package/platformio-mcp), [serial-mcp](https://github.com/bmdragos/serial-mcp), [Codex embedded guide](https://codex.danielvaughan.com/2026/05/22/codex-cli-embedded-firmware-development-platformio-zephyr-mcp-agents-md-workflows/) |
| Browser MCP options | [MCP.Directory comparison](https://mcp.directory/blog/chrome-devtools-mcp-vs-playwright-mcp-2026), [Steve Kinney](https://stevekinney.com/writing/driving-vs-debugging-the-browser) |
| Model gateway options | [LiteLLM](https://github.com/BerriAI/litellm), [Vercel AI SDK](https://ai-sdk.dev/docs/introduction) |

---

## 0.6 How to read the rest

| Doc | Read it if you want |
| --- | --- |
| [01 — Competitive Architecture Study](01-competitive-study.md) | What the six systems actually do, and the capability matrix |
| [02 — MCP and Skills Architecture](02-mcp-and-skills.md) | How we consume MCP without forking it, and what a Skill is vs a tool |
| [03 — Multi-Agent Architecture](03-multi-agent.md) | The orchestrator, task graph, contracts, conflicts, verification |
| [04 — Model Gateway](04-model-gateway.md) | Provider independence and capability routing |
| [05 — Security Architecture](05-security.md) | Sandbox, permissions, secrets, injection defense |
| [06 — Project Brain](06-project-brain.md) | Structured project state and memory |
| [07 — Implementation Roadmap](07-original-roadmap.md) | Phases, with a revised dependency order |
| [08 — Critique](08-critique.md) | **Where this idea is wrong.** Read this one before the roadmap. |
