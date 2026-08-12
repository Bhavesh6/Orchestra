# Document 9 — Answers (A–O) and a Course Correction

**Written after receiving the full project history.** The handoff document contained one fact that none of the earlier briefs did, and it changes the recommendation materially.

---

## 9.0 The thing in section 1 that changes everything

> "My Claude Pro subscription is going to end. I therefore started looking for an alternative that would let me continue having a powerful coding-agent workflow."

That is the actual problem. Everything from section 2 onward is a solution that grew away from it.

Two questions got fused into one, and they have completely different answers:

| Question | Answer | Time to solve |
| --- | --- | --- |
| **"What do I code with next month?"** | OpenCode or Cline, pointed at NVIDIA Build | **~30 minutes** |
| **"What should I build?"** | A contract + verification system | **months** |

The handoff document answers question 1 with the answer to question 2. That means several months of no coding agent while building the thing that was supposed to replace the coding agent. That's the failure mode I most want to prevent, and it has nothing to do with architecture.

### The immediate fix, verified

This is documented and working today, not speculative:

- **NVIDIA Build** exposes 100+ models on an **OpenAI-compatible endpoint** at `https://integrate.api.nvidia.com/v1`. Free account, no credit card. **[2°]**
- **OpenCode supports it directly** via `@ai-sdk/openai-compatible` — a provider block in `~/.config/opencode/opencode.json` plus a key in `auth.json`. There are step-by-step guides for exactly this combination. **[2°]**
- **Tool calling works** across the major models — confirmed on DeepSeek V4/V3.2, GLM-5/5.2, Qwen3, Kimi K2.6. Agentic coding models available include **Qwen3 Coder 480B** (purpose-built for agentic coding), **GLM-5.2**, **DeepSeek V4 Flash** (1M context), **MiniMax M2.7**. **[2°]**

**Do this first, this week.** It is not a compromise or a placeholder — it is a genuinely capable setup, and it makes everything else optional rather than urgent. Building under deadline pressure to replace a tool you no longer have is how architecture decisions get made badly.

### The constraint that matters for the architecture

Free tier: **1,000 inference credits on signup, up to 5,000 on request, and a rate limit of 40 requests per minute.** **[2°]**

That 40 RPM number deserves a moment. Agentic loops are request-heavy — each tool call is a round trip. A single agent working hard will use a meaningful fraction of that. **Six parallel agents will saturate it immediately**, and rate-limited agents don't fail cleanly; they stall, retry, and burn wall-clock while producing nothing.

So for your actual situation, on your actual provider, **parallelism is throttled by infrastructure before it's throttled by coordination overhead**. That's a hard, concrete argument — independent of all the MAST research — that heavy multi-agent execution is not the right early bet *for you*.

It also flips the value of the model gateway: its most important job isn't picking the smartest model, it's **surviving rate limits and outages by failing over across providers**. See §9.N.

---

## 9.1 The scope collapse: the differentiator does not need a platform

Here is the insight I'd most like you to take from this document.

**The contract system can ship as an MCP server.** Not as part of a runtime you build. As a standalone MCP server plus a CLI.

Consequences:

```
        contract-mcp  +  contracts-cli
                 │
    ┌────────────┼────────────┬──────────┬─────────┐
    │            │            │          │         │
 OpenCode   Claude Code     Cline      Codex    Cursor
```

- It works **inside every agent you already use**, on day one. You don't need to build an agent runtime to get the value.
- It's useful **with a single agent** — which is what the evidence says you'll mostly run anyway.
- It's testable immediately against real ESP32 work.
- If the multi-agent thesis never pans out, **you still have a thing worth having**.
- It's maybe 2–4 weeks of work instead of 12 months.

Compare that to the architecture in section 32 of the handoff: orchestrator, contract layer, agent runtime, MCP layer, skills layer, model gateway, verification, synthesis. That's 8–10 subsystems, and section 35 of your own document says *"do not build ten subsystems before proving the core idea."*

The MCP-server framing is how you honor that instruction. Your own §12 principle — "existing systems become resources/workers/adapters rather than things we recreate" — applies to the agent runtime too. OpenCode *is* the runtime. You supply the part nobody has.

---

## Answers to A–O

### A. What are we right about?

1. **Contracts that generate artifacts.** The strongest idea in the project, correctly identified. Nobody in the surveyed field does this.
2. **Verification before "done."** Runtime-enforced, not prompt-requested. Cheap, and it improves output at any agent count.
3. **Multi-agent as optimization, not religion.** Correct, and backed by MAST plus the behavior of every vendor shipping the feature.
4. **Security through capability partitioning.** Genuinely strong, and better motivated than the productivity argument.
5. **Verification moved to Phase 0.** Right, because it's the only thing that makes the rest falsifiable.
6. **Don't rebuild MCP or PlatformIO tooling.** Right. `platformio-mcp` already covers build/flash/monitor/diagnose.
7. **Rejecting local-first inference.** Right, and now doubly so — NVIDIA Build gives you frontier-class models for free.
8. **Model-agnosticism is an architectural requirement, not a differentiator.** Correctly demoted.
9. **Cloud model + local tools split.** Right, and it's exactly what OpenCode already implements.

### B. What are we wrong about?

1. **The framing of the problem.** §9.0. This is the big one.
2. **That you need your own agent runtime to deliver the differentiator.** §9.1. You don't.
3. **The four-layer diagram in §32.** It puts the contract layer *beside* the agent runtime and above the model gateway, with verification as a terminal step. In practice contracts sit **underneath** everything as a validation and generation service, and verification is a **loop**, not a final stage. The picture implies a pipeline; the real thing is a control loop with a validator hanging off it.
4. **"Detect and resolve conflicts" as a headline feature (§34).** Most conflicts should be *prevented* — textually by file-scope disjointness, structurally by generated code. Only genuinely semantic disagreements need adjudication, and those are rare. Building a conflict-resolution subsystem before you have conflicts is speculative.
5. **Treating the ESP32 workflow (§25) as validation of the multi-agent thesis.** That workflow — detect, compile, upload, capture, diagnose, fix, retry — is strictly **sequential**, gated on one physical board. It's a fantastic proof of *contracts and verification*. It proves nothing about parallelism, because it can't be parallelized.

### C. What assumptions are unsupported?

| Assumption | Status |
| --- | --- |
| Parallel specialized agents beat one strong agent on real coding work | **Unproven, evidence leans against.** This is what §37's experiment is for. |
| Capability-based model routing improves outcomes | **Unproven.** Needs per-(role × model) verified-outcome data you won't have for months. |
| Model diversity improves quality | **Unsupported.** Explicitly listed in your own §35, then partly reintroduced in §8. Different models have correlated failure modes. |
| Generated artifacts can be authoritative in messy domains | **Unproven outside crisp domains.** Works beautifully for pin maps and OpenAPI. Unclear for architecture decisions and UI behavior. |
| MCP is sufficient for agent collaboration | **False as stated.** MCP is agent→tool. It has no schema for "agent finished a task with findings and evidence." That layer is ours. |
| The orchestrator can reliably decompose tasks | **Weakest assumption in the project.** MAST puts bad specification in the largest failure bucket. |
| Contract maintenance cost is low | **Unknown.** Generated contracts can't drift. Prose ones will. |

### D. Which features already exist and should be reused?

| Need | Use | Don't build |
| --- | --- | --- |
| Agent runtime | **OpenCode** (MIT) or **Cline** | your own loop, initially |
| Cloud inference | **NVIDIA Build** (free, OpenAI-compatible) | anything local |
| Provider abstraction | **Vercel AI SDK** (in OpenCode) or **LiteLLM** | 40 adapters |
| Model catalogue | **models.dev** | your own |
| Embedded toolchain | **platformio-mcp**, **serial-mcp** | wrappers |
| Browser | **Playwright MCP**, **Chrome DevTools MCP** | automation engine |
| Sandboxing | Landlock/seccomp, Docker, Firecracker | your own |
| Isolation | **git worktrees** | custom workspaces |
| Sessions, permissions, LSP, checkpoints | OpenCode/Cline | all of it |
| Full runtime, if you do build one | **OpenHands SDK** (MIT) — spike it first | from scratch |

### E. Which parts must actually be built?

Short list. That's the point.

1. **Contract schema + validators** — pin maps, OpenAPI, DB schema; deterministic checks.
2. **Generators** — `pinmap.yaml → pins.h`, OpenAPI → typed client. The core mechanism.
3. **Verification gate runner** — declarative gates, structured results with evidence refs.
4. **Structured result schema** — with runtime rejection of unverified `complete`.
5. **The MCP server** exposing 1–4 to any agent.
6. *(Later, only if the experiment justifies it)* orchestrator, task graph, scheduler.

Items 1–5 are the product. Item 6 is a hypothesis.

### F. Smallest useful prototype

**`contracts-mcp` + `contracts` CLI.** Concretely:

```
contracts validate            # check contracts + code against them
contracts generate            # emit pins.h, clients, configs
contracts gate <name>         # run a verification gate, structured result
contracts status              # what's out of sync
```

plus an MCP server exposing `contract.read`, `contract.propose`, `contract.validate`, `gate.run`, `gate.list`.

**Acceptance test:** point it at one of your real ESP32 projects. Define `pinmap.yaml`. Generate `pins.h`. Have OpenCode + a NVIDIA model make a change that violates the pin map. **The build must fail, with a message naming the contract it violated.**

If that works, you have proven the central idea, in weeks, with no orchestrator.

### G. Technical stack

| Concern | Choice | Why |
| --- | --- | --- |
| Language | **TypeScript** (Bun or Node) | Best MCP SDK; matches OpenCode if you extend it later |
| Schemas | **JSON Schema + Zod** | Validation at every boundary |
| Contract files | **YAML** | Human-editable, git-diffable, reviewable |
| Generation | **Eta** or plain template functions | Keep it boring |
| MCP | official TypeScript SDK, **2026-07-28** | Don't implement deprecated `sampling`/`roots`/`logging` |
| Gates | shell out to `pio`, `pytest`, `tsc` | Existing tools are the gates |
| Storage | files in git + SQLite index (derived) | Brain must survive your tooling |
| Agent runtime | **OpenCode**, initially not yours | Reuse |
| Models | **NVIDIA Build** primary, OpenRouter fallback | Free tier + failover |

Python is a reasonable alternative if you prefer it for the embedded side — but the MCP and agent ecosystem is stronger in TypeScript, and you'll be writing MCP servers.

### H. Contract format

```yaml
# .project/contracts/hardware/pinmap.yaml
contract: HW-PINMAP
version: 3
status: accepted          # draft | proposed | accepted | superseded
owner: hardware
supersedes: 2
evidence:
  - ref: research/esp32-devkit-v1.md#adc2-wifi
    quote: "ADC2 is unavailable while WiFi is active"

board: esp32-devkit-v1

assignments:
  - signal: MQ9_ADC
    pin: GPIO34
    mode: analog_in
    adc: ADC1
    note: "ADC1 required — ADC2 conflicts with WiFi"
  - signal: DHT22_DATA
    pin: GPIO4
    mode: digital_io
    pullup: external_10k

constraints:
  - id: C-01
    rule: unique_pins
  - id: C-02
    rule: input_only_pins
    params: { pins: [34, 35, 36, 39] }
  - id: C-03
    rule: adc2_unusable_with_wifi

generates:
  - target: include/pins.h
    template: pins_h
    authoritative: true        # hand-edits to this file are an error
```

Three properties that make it work:

1. **`constraints` are executable rule names**, not prose. `unique_pins` is a function. It runs in milliseconds, costs nothing, and needs no model.
2. **`generates` with `authoritative: true`** means the generated file is checked into git *and* verified as unmodified by a gate. Drift becomes a build failure, which is the answer to "who maintains this."
3. **`evidence` and `supersedes`** give you the decision trail without a separate ADR system for simple cases.

Same shape for `api/openapi.yaml` (constraints = schema lint + conformance tests, generates = typed client) and `data/schema.sql` (generates = migrations + models).

### I. Agent-to-agent communication format

**There isn't one. That's the design.**

Agents do not message each other. They read and write a typed blackboard. An agent's entire output is one structured result object (full schema in [Doc 3 §3.7](03-multi-agent.md)):

```jsonc
{
  "task_id": "T-014",
  "status": "complete",        // complete|blocked|failed|partial|needs_decision
  "summary": "...",
  "claims":   [ { "statement": "...", "confidence": 0.95, "evidence": [...] } ],
  "contracts_consumed": ["HW-PINMAP@v3"],
  "contracts_proposed": [],
  "changes":  [ { "path": "...", "kind": "modified" } ],
  "verification": [ { "gate": "build.esp32", "result": "pass", "ref": "..." } ],
  "assumptions": [ { "text": "...", "validated": false, "risk": "high" } ],
  "open_questions": [ { "text": "...", "route_to": "human" } ],
  "cost": { "tokens_in": 0, "tokens_out": 0, "usd": 0.0 }
}
```

Two rules the runtime enforces mechanically:
- **`status: complete` is rejected** unless every required gate reports `pass`.
- **A claim without evidence is demoted to an assumption.**

Why no chat: MAST's largest failure category is inter-agent misalignment. Removing the channel removes the category. It also removes the agent-to-agent escalation attack — an agent can't ask another to do what it was denied, because there's no field in the schema where that's expressible.

*(Open question I haven't resolved: Claude Code documents adversarial debate between teammates as genuinely effective for debugging, because sequential investigation anchors on the first hypothesis. If we want that, it should be an explicit `debate` task type with structured rounds — not ambient chat.)*

### J. Single-agent vs multi-agent decision

Deterministic, no model call:

```
multi_agent IF ALL of:
    ├── independent_workstreams ≥ 2      (disjoint file_scope globs)
    ├── estimated_work         ≥ 30 min
    ├── task_spec_confidence   ≥ threshold   (well-defined, not exploratory)
    ├── no_shared_exclusive_resource         (one ESP32 → serialize)
    ├── rate_limit_headroom    sufficient    (40 RPM on NVIDIA free tier!)
    └── user_approved_the_plan
ELSE single_agent
```

**Default is single.** Fan-out is the exception that must argue for itself. And when the orchestrator declines to parallelize, **it says so and shows the estimated saving** — that transparency is a feature, not an apology.

Note `rate_limit_headroom` — that constraint is specific to your provider situation and would be invisible to anyone designing this in the abstract.

### K. Model selection

Three stages, and only the first two ship early:

1. **Hard filter** on capability: tool calling, context size, structured output, vision. Pure code, no LLM in the routing path, ever.
2. **Static per-role profiles**, hand-tuned:
   - decompose/plan → strongest reasoning model
   - code → agentic coding model (Qwen3 Coder 480B, GLM-5.2)
   - review → **different provider than the author** (cheap, real independence)
   - classify/summarize → fast, cheap
3. **Learned routing** — later, from accumulated gate outcomes: *for role=firmware, model X passes gates first-try 71% at $0.31; Y passes 68% at $0.09*. Do not promise this until the data exists.

The metric that matters is **cost per verified task**, not cost per token. A cheap model that needs four tool-call iterations instead of two costs more and burns your 40 RPM budget faster.

### L. How MCP fits

Three roles simultaneously:

- **Host** — one connection pool, one policy enforcement point, tool-list caching (now cheap via `ttlMs`/`cacheScope`).
- **Client** — **per-agent scoped tool views**. An agent sees the 12 tools its role needs, not 200. Security *and* accuracy: tool-choice quality degrades with tool count.
- **Server** — we expose contracts, gates, and project state as an MCP server. This is the whole §9.1 strategy.

Build against the **2026-07-28** spec: stateless core, MRTR instead of server-initiated requests, header routing, CIMD authz. **Do not implement `sampling`, `roots`, or `logging`** — deprecated. Isolate transport behind an interface; the spec broke once and will again.

MCP is **not** sufficient for agent collaboration. It's agent→tool. The result schema in §I is ours because MCP has no equivalent.

### M. Security and isolation

Right-sized for a solo developer, not an enterprise:

**Now (trivial, do it immediately):**
- Sandbox on by default — Landlock+seccomp on Linux. Codex is the only major agent doing this by default; it's the right default.
- Network egress deny-by-default with per-task allowlists.
- API keys in `auth.json`, never in env vars reachable by agent subprocesses, never in the repo.
- `git push` and firmware flashing always ask.
- Protected paths: `.git`, CI config, git hooks, agent config.

**Later (when you actually run multiple agents):**
- Trifecta partitioning — research agent gets network but no secrets; coder gets repo but no network. Each agent holds at most two of {private data, untrusted input, exfiltration}.
- Worktree per writing agent.
- Containers/microVM for untrusted repos.

**Hardware-specific (do this at P8, non-negotiable):**
- Device lease — one agent, one board, serialized.
- Pin audit before flash (`platformio-mcp` already exposes `agent-safe-pin-audit`).
- Known-good firmware rollback.
- No unattended flash→observe→reflash loops beyond N iterations.

Most of [Doc 5](05-security.md) is enterprise-scale and you should ignore the parts that are. The sandbox default, the key handling, and the hardware gates are the parts that matter at your scale.

### N. Surviving provider unavailability

**This is now a top-three requirement, not a nice-to-have** — because your primary provider is a free tier with 40 RPM and a credit budget.

```
Ordered fallback chains per role, triggered by failure class:
  429 rate_limit      → queue with backoff, then next provider in chain
  5xx / timeout       → next provider, same capability tier
  context_overflow    → larger-window model, or compact and retry
  credits_exhausted   → next provider (OpenRouter, or a paid key)
  refusal             → different provider (never retry the same model)
```

Plus:
- **Never carry context across a provider switch.** Rebuild from contracts and the Brain. Reasoning traces, cache breakpoints, and tool formats don't transfer.
- **Recompute budgets on switch** — token counts aren't portable across tokenizers.
- **Degrade rather than fail**: if only a weak model is available, run it with *stronger* gates rather than refusing to work.
- **Local-model escape hatch** stays in the adapter list even though your hardware can't drive it well — it's the last resort when everything else is down.

The gateway's most valuable function in your situation isn't intelligence. It's **continuity**.

### O. Strongest unique differentiator

> **Contracts that generate authoritative artifacts, so that incorrect implementations become mechanically impossible rather than detectable by review — combined with runtime-enforced verification, so no agent can claim completion it hasn't proven.**

Not model-agnosticism (OpenCode, Cline). Not multi-agent orchestration (Claude Code, Codex, Antigravity, Cursor). Not parallel workspaces (Cursor shipped worktrees in 2.0).

The one-liner: **an AI development environment where the agent has to prove it's done, and where architectural decisions are enforced by the compiler rather than remembered by a model.**

---

## 9.2 Revised plan

| Step | What | Time | Depends on |
| --- | --- | --- | --- |
| **0** | **OpenCode + NVIDIA Build. Start coding.** | **30 min** | nothing |
| 1 | `contracts` CLI: schema, validators, `pinmap → pins.h` | 1–2 wk | 0 |
| 2 | Gate runner + structured results | 1 wk | 1 |
| 3 | `contracts-mcp` server | 1 wk | 1, 2 |
| 4 | **Use it on real ESP32 work for a month.** Keep notes. | 4 wk | 3 |
| 5 | Decide from evidence whether orchestration is worth building | — | 4 |
| 6+ | Only then: task graph, parallel agents, the rest | months | 5 |

Step 4 is not a delay. It's the experiment — and it's a better one than §37's benchmark, because it runs on work you were going to do anyway, at zero marginal cost, on the domain you actually care about.

**The §37 benchmark is still right**, but run it at step 5 with three arms — plain single agent / single agent + contracts + gates / multi-agent — because that isolates which of the two ideas is doing the work. My prediction: **arm two captures most of the gain.** If I'm right, you've saved a year. If I'm wrong, you'll have the data to prove it and you'll build the orchestrator on solid ground.

---

## 9.3 The one thing I'd ask you to accept

Your §39 says: *"Do not optimize for making the idea sound impressive. Optimize for building something that actually works."*

Taking that seriously means accepting that the thing that works is smaller than the thing in section 32. It's a contract validator, a code generator, a gate runner, and an MCP server. It has no orchestrator, no task graph, no agent runtime, and no UI.

That is not a reduced version of your idea. It's the part of your idea that nobody else has — separated from the parts that four other companies have already built and that you can use for free.
