# Document 2 — MCP and Skills Architecture

---

## 2.1 The headline: our mental model of MCP was out of date

The **2026-07-28 specification** is a substantial redesign. If you learned MCP in 2025, at least four things you know are now wrong. **[Doc]** (from the official spec blog; `modelcontextprotocol.io` itself was egress-blocked)

| Area | Before (2025-11-25 and earlier) | Now (2026-07-28) |
| --- | --- | --- |
| **Core model** | Stateful; `initialize`/`initialized` handshake, `Mcp-Session-Id` header | **Stateless.** Handshake and session header **removed**. Each request self-describes protocol version, client identity, and capabilities in `_meta`. Optional `server/discover` RPC. |
| **Server→client requests** | `sampling/createMessage`, `elicitation/create` over an open bidirectional stream | **Multi Round-Trip Requests (MRTR).** Server returns `resultType: "input_required"`; client retries the original call with responses attached. Server-initiated requests may **only** be issued while the server is actively processing a client request. |
| **Tasks** | In core (2025-11-25) | **Moved to an Extension** (contributed by AWS). Poll-based `tasks/get` / `tasks/update`. |
| **Deprecated** | — | **`roots`, `sampling`, `logging` deprecated.** Legacy HTTP+SSE transport deprecated. **DCR deprecated** in favour of Client ID Metadata Documents (CIMD). Minimum 12-month support window. |

Plus: **header-based routing** (`Mcp-Method`, `Mcp-Name` HTTP headers let gateways route/authorize without parsing JSON bodies), **cacheable list results** (`ttlMs`, `cacheScope` on tool/prompt/resource lists), and **authorization hardening** (RFC 9207 issuer validation against mix-up attacks, `application_type` for desktop/CLI localhost redirects).

### Why this matters enormously to us

**A design that has agents borrow the host's model via MCP `sampling` is a design built on a deprecated primitive.** That was a natural-looking pattern — "MCP server asks the orchestrator for an LLM call" — and it is now the wrong answer.

**The stateless core is a gift.** Our orchestrator wants to run N agents concurrently, each needing tool access. A stateful protocol would have forced per-agent sessions, sticky routing, and shared session storage. A stateless protocol means we can pool MCP connections and fan them out across agents behind a plain load balancer. **[Inf]** — this is my reading of the consequence, not a spec statement.

**Header-based routing is exactly what a multi-agent gateway needs.** We can enforce per-agent tool policy at the HTTP layer by inspecting `Mcp-Method`/`Mcp-Name`, without deserializing every payload. That is our permission enforcement point ([Document 5](05-security.md)).

> ⚠️ **Verify before building:** re-read the actual spec text from an unblocked network. The summary above is from the official spec blog post, which is authoritative but is still a summary. Specifically confirm the exact `_meta` shape and the MRTR retry semantics.

---

## 2.2 What we build vs what we adopt

The brief (§6) says: do not create a proprietary replacement for what MCP already standardizes. Agreed, with one carve-out.

| Concern | Decision | Rationale |
| --- | --- | --- |
| Tool discovery & invocation | **MCP** | Class A. Thousands of servers exist. Zero reason to invent. |
| Resources (files, docs, data) | **MCP** | Class A. |
| Prompts | **MCP** | Class A, though we lean on Skills more. |
| Elicitation (ask the user mid-call) | **MCP MRTR** | Class A. Maps onto our approval system. |
| Long-running work | **MCP Tasks extension** | Class A. Adopt the extension rather than a bespoke async protocol. |
| Authorization | **MCP authz + CIMD** | Class A. Do **not** hand-roll OAuth. |
| **Agent-to-agent coordination** | **OURS** | ← the carve-out, see below |
| **Structured agent results** | **OURS** | MCP has no schema for "an agent finished a task with findings, evidence, and conflicts." |
| **Task graph / scheduling** | **OURS** | Out of MCP's scope entirely. |
| **Project Brain** | **OURS** (exposed *via* MCP) | The store is ours; we expose it as an MCP server so any client can read it. |

### The A2A question

**A2A** (Agent2Agent, now Linux Foundation) hit **v1.0 stable** in April 2026: 150+ organizations, 22k+ GitHub stars, SDKs in five languages, GA in Copilot Studio / Azure AI Foundry / Bedrock AgentCore, plus signed Agent Cards. **[2°]**

The clean framing from the ecosystem: *MCP is vertical* (agent → tools), *A2A is horizontal* (agent ↔ agent across organizational boundaries). **[2°]**

**Recommendation: do not adopt A2A for our internal orchestration.** Reasons:

1. Our agents are **inside one trust boundary, one process group, one project**. A2A solves cross-organizational discovery and negotiation — a problem we don't have. Adopting it would import authentication, agent cards, and message envelopes to talk between two of our own goroutines.
2. Critical analyses in 2026 note A2A's adoption lagged relative to MCP precisely because of implementation burden vs use case. **[2°]**
3. Internal coordination in our design is **not conversational** — it is a blackboard plus a task graph (see [Document 3](03-multi-agent.md)). A2A's message-passing model would push us toward chat-between-agents, which the failure literature says is where multi-agent systems break.

**But:** expose an **optional A2A adapter at the perimeter** in a late phase, so external A2A agents can be recruited as workers (brief §23). Class **C** — integrate, never depend.

---

## 2.3 Our MCP posture: three roles

We are all three MCP roles simultaneously. Being explicit about this prevents a lot of confusion.

```
                         ┌──────────────────────────────┐
                         │        OUR PLATFORM          │
                         │                              │
   external MCP  ───────►│  (1) MCP HOST                │
   servers              │      owns connections,        │
   (filesystem,         │      consent, policy          │
    github, playwright, │            │                  │
    platformio, …)      │            ▼                  │
                        │  (2) MCP CLIENT per agent     │
                        │      scoped tool view         │
                        │            │                  │
                        │            ▼                  │
   other tools  ◄───────│  (3) MCP SERVER               │
   & IDEs               │      we expose Project Brain, │
                        │      task graph, artifacts    │
                        └──────────────────────────────┘
```

### Role 1 — Host
One **connection pool** per MCP server, owned by the platform, not by individual agents. Agents never open their own connections. This gives us a single enforcement point, connection reuse across parallel agents, and one place to cache tool lists (now cheap thanks to `ttlMs`/`cacheScope`).

### Role 2 — Client, with **per-agent scoped tool views**
This is the important part and it is where we differ from every competitor.

An agent does not see "all tools." It sees the tools its **role** grants, filtered by the **task's** declared needs. A Firmware Agent gets `platformio.*`, `serial.*`, filesystem within its worktree. It does **not** get `github.push` or `browser.*`.

Two reasons, and the second is the one people forget:
- **Security** — least privilege ([Document 5](05-security.md)).
- **Accuracy** — tool-choice quality degrades as tool count grows. An agent with 200 tools in context picks worse than one with 12. This is the same logic as Claude Code's context-economy argument for subagents. **[Inf]**, though widely observed.

### Role 3 — Server
We expose our own state **as an MCP server**: Project Brain reads, contracts, task graph status, artifacts. Consequences:
- Any MCP-capable client (Claude Code, Codex, Cursor, Cline) can inspect a running project. Interop for free.
- Our own agents read the Brain through the same interface external tools do — one code path, no privileged backdoor.
- External agents recruited as workers (brief §23) get context through a standard interface.

---

## 2.4 The MCP servers we should consume, not build

Class **B/C**. Every one of these is a thing we do *not* write.

| Domain | Server | Notes |
| --- | --- | --- |
| Browser (driving) | **Playwright MCP** | Exposes **accessibility-tree snapshots** — the LLM reads structured text, not pixels. Cross-browser. Best for E2E/verification. **[2°]** |
| Browser (debugging) | **Chrome DevTools MCP** | Performance tracing, network, Web Vitals. Chrome-only. "Playwright tells you *what* happened; DevTools tells you *why*." **[2°]** |
| Embedded | **platformio-mcp** | Board discovery, build, flash, monitor, diagnostics. Agent-oriented tools incl. `agent-safe-pin-audit`, `agent-flash-monitor-verify`, `agent-build-diagnose`. ~1000 boards. **[2°]** |
| Embedded | **serial-mcp** | Non-blocking serial I/O for Arduino/ESP32. **[2°]** |
| Git hosting | GitHub / GitLab MCP | PRs, issues, CI status, reviews. |
| Filesystem | official filesystem MCP | Scoped roots. |
| Docs/search | various | Research agent's tooling. |

**Finding relevant to brief §19:** the embedded story is *already solved* by existing MCP servers, better than I expected. `platformio-mcp` v2.2.1 (May 2026) exposes the entire build→flash→monitor→diagnose loop, with agent-specific verification tools. **We should write zero embedded tooling in early phases.** We install these servers and write **Skills** that teach agents the ESP32 debugging workflow. That converts brief §19 from a build project into a configuration project — a large saving.

---

## 2.5 MCP vs Skills — the distinction, made operational

The brief's framing is right and worth sharpening:

| | **MCP Tool** | **Skill** |
| --- | --- | --- |
| Answers | *What can the agent do?* | *How should the agent do this well?* |
| Form | Executable endpoint + JSON schema | Markdown procedure + optional bundled scripts |
| Loaded | Schema always in context (when scoped in) | **Body loads only on use** (progressive disclosure) |
| Author | Tool vendor | Domain expert / us / the user |
| Failure mode | Tool errors | Bad advice |
| Versioning | Server version | Git-versioned in the project |

The cleanest way to say it: **a tool is a verb; a skill is a playbook that sequences verbs, with judgment attached.**

Concrete pairing for our ESP32 case:

```
MCP tools available:            Skill "esp32-sensor-debug":
  platformio.list_boards          1. detect board + port; if ambiguous, ASK
  platformio.build                2. audit pin map against Project Brain
  platformio.upload                  contract before building
  serial.open / read / write      3. build; on failure route to
  brain.read_contract                build-diagnose, don't guess
  brain.propose_change            4. upload, open serial, capture ≥10s
                                  5. compare against expected signature
                                  6. DHT22 specifics: check pull-up,
                                     2s settle, timing-sensitive reads
                                  7. record findings as evidence
```

The tools are generic. The skill encodes the hard-won knowledge that DHT22 needs a pull-up and a settle delay — knowledge no tool schema can express and no model reliably volunteers.

### Skill format decision

**Adopt the `SKILL.md` convention** (name/description frontmatter + markdown body + bundled resources). Reasons: it is becoming a de-facto cross-vendor standard (Claude Code, Codex, Cline all use skills; Claude Code merged custom commands into them), it's plain markdown so it's git-diffable and human-reviewable, and adopting it means **the existing skill corpus is portable into our system** on day one.

Extensions we add to frontmatter:

```yaml
---
name: esp32-sensor-debug
description: Diagnose a non-responding sensor on an ESP32 target
# --- our additions ---
requires_tools: [platformio.*, serial.*, brain.read_*]
requires_capabilities: [tool_calling, long_context]
agent_roles: [firmware, hardware]
produces_artifacts: [diagnosis_report, serial_capture]
verification: firmware.flash_and_assert
risk: medium          # gates which approval tier applies
---
```

`requires_tools` lets the router **scope the agent's tool view from the skill**, closing the loop between §2.3 and skills. `verification` names the check that must pass before the agent may claim completion (brief §18).

> One caution, and it applies to us as much as anyone: **a skill is untrusted-ish input.** A skill installed from a marketplace is a text file that instructs an agent. That is an injection vector. Skills need the same provenance treatment as MCP servers — see [Document 5](05-security.md) §5.6. Codex already gates this with `/hooks` trust management for hooks; we should do the same for skills. **[2°]**

---

## 2.6 What we are explicitly NOT building

Recording these so they don't creep back in:

- ❌ A proprietary tool protocol. MCP.
- ❌ Our own OAuth for tool servers. MCP authz + CIMD.
- ❌ A bespoke async-task protocol. MCP Tasks extension.
- ❌ An MCP server registry. Consume existing ones.
- ❌ Browser automation from scratch. Playwright MCP + Chrome DevTools MCP.
- ❌ Embedded toolchain wrappers. platformio-mcp + serial-mcp.
- ❌ A2A as internal transport. Perimeter adapter only, late.
