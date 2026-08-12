# Document 7 — Implementation Roadmap

---

## 7.1 Why I reordered the brief's phases

The brief proposed: runtime → gateway → tools/MCP → task graph → isolation → browser/docs → verification → embedded → UI → autonomy. That order is mostly right, with **one change I'd argue for strongly**:

> **Verification must move from Phase 7 to Phase 1.**

Three reasons:

1. **It's the gate that makes every later phase measurable.** Without verification you cannot tell whether adding a second agent helped. The entire multi-agent thesis is an empirical claim, and Phase 1 needs to be able to test it.
2. **MAST puts ~21% of multi-agent failures in the task-verification bucket** — and the other two buckets are only *detectable* through verification. **[Src]/[2°]**
3. **It's the cheapest thing here.** "Run the build, run the tests, parse the result" is a day of work. Deferring it to Phase 7 means six phases of unfalsifiable progress.

Second change: **isolation (worktrees) moves earlier**, because it's a prerequisite for parallelism rather than a follow-on, and it costs almost nothing.

Third: **embedded moves earlier** than Phase 8 — not because it's foundational, but because the MCP servers already exist (Doc 2 §2.4) so it's mostly configuration, and it's our sharpest differentiator. It's a cheap way to have something nobody else has.

---

## 7.2 Revised phases

```
P0  Walking skeleton + verification            ← must exist before anything else
P1  Single-agent runtime, sandboxed by default
P2  Model gateway + conformance suite
P3  MCP client + skills
P4  Project Brain + contracts
P5  Task graph + PARALLEL AGENTS   ← the actual bet; gated on P0–P4 evidence
P6  Workspaces, merge queue, conflict handling
P7  Verification depth: review, browser, documents
P8  Embedded / ESP32 vertical
P9  UI
P10 Autonomy, learning, external agent workers
```

---

### P0 — Walking skeleton + verification (2–3 weeks)

The smallest thing that proves the shape.

- Headless server: HTTP + SSE event stream. **No UI.**
- Session and turn model; append-only event log.
- Three tools: `read_file`, `write_file`, `run_command`.
- One provider.
- **Verification gates**: `gates.yaml` defining how to run build/test/lint and how to judge the result; structured results with evidence refs.
- Structured agent result schema (Doc 3 §3.7), with the runtime **rejecting `status: complete` when gates haven't passed**.

**Exit criterion:** the agent fixes a failing test in a real repo, and the system *proves* it via gates rather than the agent asserting it.

---

### P1 — Single-agent runtime, sandboxed (3–4 weeks)

- Agent loop: context assembly, tool dispatch, error handling, retries with backoff.
- **Sandbox on by default** — Landlock + seccomp on Linux, Seatbelt on macOS. Network egress denied by default with per-task allowlists.
- Permission engine: deny → ask → allow, shell-operator aware (Doc 1 §1.2 — copy Claude Code's semantics, they've clearly been through the edge cases).
- Budgets: tokens, wall clock, tool calls. Circuit breaker on consecutive failures.
- Checkpoints (git-backed) + rewind.

**Exit criterion:** competitive with a basic single-agent CLI on real tasks, with a sandbox that survives a deliberate escape attempt.

---

### P2 — Model gateway + conformance (2–3 weeks)

- Wrap an existing library (LiteLLM or Vercel AI SDK — Doc 4 §4.2). Do not write forty adapters.
- Capability catalogue as hot-reloadable data, seeded from models.dev.
- Normalization: tool schemas, stop reasons, streaming, reasoning, structured output.
- **Conformance suite** (Doc 4 §4.5) running in CI.
- Router: hard filters + weighted scoring + fallback chains. **No LLM in the routing path.**
- Cost tracking per task/agent/run.

**Exit criterion:** the same task completes on ≥3 providers, and the conformance suite catches a deliberately mis-declared capability.

---

### P3 — MCP + Skills (3–4 weeks)

- **MCP client, 2026-07-28 spec**: stateless core, MRTR, cacheable lists, header routing, CIMD authz. Do not implement deprecated `sampling`/`roots`/`logging`.
- Connection pool + **per-agent scoped tool views**.
- MCP Tasks extension for long-running work.
- Skills: `SKILL.md` loader with progressive disclosure + our frontmatter extensions.
- **MCP server role**: expose sessions and (later) the Brain.
- Supply-chain controls: pinning, digest, re-approval on tool-description change.

**Exit criterion:** run against ≥5 third-party MCP servers, including Playwright, with per-agent scoping demonstrably enforced.

---

### P4 — Project Brain + contracts (3–4 weeks)

- `.project/` layout, schemas, validators.
- Contract types + **deterministic validators** (pin uniqueness, OpenAPI conformance, schema checks).
- **Code generation from contracts** (`pins.h`, typed API clients) — the highest-leverage feature in the design.
- Decision log / ADRs, evidence chains.
- Bootstrap flow for existing repos, with mandatory human review of the draft.
- Brain exposed via MCP.

**Exit criterion:** change a contract → the correct downstream builds break → the correct tasks are generated. **This is the moment the architecture starts paying for itself, and it happens before any parallelism exists.**

---

### P5 — Task graph + parallel agents (4–6 weeks) ← **the bet**

- Orchestrator: decompose → DAG with `file_scope`, roles, contracts, gates.
- **Mandatory human plan-approval gate**, presented as an artifact.
- Scheduler: topological + **file-scope disjointness** + concurrency caps + budget.
- Agent dispatch with granted context, scoped tools, allocated workspace.
- Blackboard results; no agent-to-agent messaging.
- **The single-agent fallback path, as a default** (Doc 3 §3.12).

**Exit criterion — and this one is a decision point, not a checkbox:**

> On a benchmark of ≥20 realistic multi-dimension tasks, the parallel path must beat the single-agent path on **wall-clock time at acceptable quality**, or on **quality at acceptable cost**. If it doesn't, we ship the orchestration *planning* (which P4 already proved valuable) and **keep execution single-agent**.

We should be genuinely willing to take that outcome. See [Document 8](08-critique.md).

---

### P6 — Workspaces and integration (2–3 weeks)

- Git worktree per writing agent.
- Container/microVM escalation for untrusted work.
- **Merge queue**: sequential, dependency-ordered, verify after each merge.
- Conflict handling: textual prevented, contract violations detected deterministically, semantic adjudicated by a Reviewer with evidence.
- Device leases for hardware.

---

### P7 — Verification depth (3–4 weeks)

- Code review agent with the **different-provider-than-author** constraint.
- Browser verification via Playwright MCP (accessibility-tree based) + Chrome DevTools MCP for perf/network.
- Document pipeline: PDF/image → structured extraction → analysis, with capability routing.
- Failures become tasks with diagnostic context, not blind retries.

---

### P8 — Embedded / ESP32 (2–3 weeks — mostly integration)

Cheap because the MCP servers exist (Doc 2 §2.4).

- Integrate `platformio-mcp` + `serial-mcp`.
- Hardware contracts → generated `pins.h`.
- Safety gates: pin audit, power budget, device lease, flash-always-asks, known-good rollback.
- Skills for the ESP32 debug loop.

**Exit criterion:** the brief's §19 workflow — detect board → compile → upload → capture serial → diagnose → fix → re-verify — runs end to end with the safety gates engaged.

---

### P9 — UI (4+ weeks)

Last, per brief §31 — and by now the server API is proven by real clients.

- TUI first, web second.
- **Artifact-centric review surface** (Antigravity's best idea): plans, diffs, screenshots, gate results, cost — not six scrolling transcripts.
- Task graph visualization; conflict and decision views.

---

### P10 — Autonomy and ecosystem

- Learned routing from accumulated gate outcomes (Doc 4 §4.6 option 2), only once there's enough data.
- External agent workers via public interfaces (brief §23) — optional, at the perimeter.
- Optional A2A adapter.
- Long-running / scheduled autonomous workflows.

---

## 7.3 What we deliberately do NOT build initially

| Not building | Why |
| --- | --- |
| **UI** | §31. The API is the product. |
| **Our own tool protocol** | MCP exists. |
| **Our own OAuth** | MCP authz + CIMD. |
| **Browser automation engine** | Playwright MCP. |
| **Embedded toolchain wrappers** | platformio-mcp, serial-mcp. |
| **Provider adapters from scratch** | LiteLLM / AI SDK. |
| **Agent-to-agent chat** | Blackboard. Deliberate architectural choice. |
| **Model-parallel consensus by default** | Opt-in only; N× cost for correlated opinions. |
| **Learned routing** | Needs data we won't have for months. |
| **Fine-tuning / hosting models** | Not our problem. |
| **A2A internally** | Perimeter only, late. |
| **Nested agent hierarchies** | Depth cap 1–2. Both Claude Code and Codex independently limit this. |
| **Cloud-hosted agent VMs** | Local-first. Cloud is a deployment target, not v1. |
| **Cross-project learned memory** | How wrong assumptions spread. |

---

## 7.4 Biggest technical risks

| # | Risk | Severity | Mitigation |
| --- | --- | --- | --- |
| R1 | **Parallelism doesn't pay** — coordination overhead ≥ speedup | **High** | P5 is an explicit decision point with a measured exit criterion; single-agent fallback is a first-class path, not a failure |
| R2 | **Cost multiplication** — N agents, N contexts | **High** | Context grants, prompt caching, cheap models for cheap roles, hard budgets, cost shown before fan-out |
| R3 | **Decomposition quality** — a bad DAG poisons everything downstream | **High** | Mandatory human plan gate; strongest model on decomposition; MAST says specification is the largest failure bucket |
| R4 | **Prompt injection through research/browser/doc agents** | **High** | Trifecta broken per agent; structured extraction at boundaries; egress deny-by-default |
| R5 | **Provider API churn** breaks adapters | Medium | Conformance suite in CI; reuse maintained libraries |
| R6 | **MCP spec churn** — 2026-07-28 was a big break, more may come | Medium | Version-negotiate; isolate transport behind an interface; track the spec repo |
| R7 | **Integration/merge hell** at agent count > 3 | Medium | `file_scope` disjointness; sequential merge queue; low default concurrency |
| R8 | **Verification gates too weak** — agents pass shallow checks, fail real use | **High** | Gate strength scales with model weakness; browser/hardware verification; escalate on failure |
| R9 | **Debuggability collapse** — nobody can tell why a 6-agent run went wrong | **High** | Full audit log; replayable blackboard state; artifacts as review surface |
| R10 | **Scope explosion** — this brief is ~10 products | **Very High** | The phase gates. Ruthlessly. See Doc 8 §8.9. |
| R11 | **Hardware damage** | Medium | Pin audit, power budget, device lease, flash-asks, rollback |
| R12 | **Blocked-source drift** — [2°] claims about competitors wrong | Low–Med | Re-verify Cursor/Codex/Antigravity/OpenCode claims from an unblocked network before P5 |

---

## 7.5 What the first prototype should implement

Concretely, P0+P1 in one sentence: **a sandboxed single-agent runtime with real verification gates and a structured result format, exposed over an HTTP API with no UI.**

That is deliberately unimpressive to demo and it is the correct first thing to build, because:
- It's the substrate every later phase needs.
- It makes the multi-agent hypothesis **testable** — you cannot measure the value of parallelism without a single-agent baseline instrumented identically.
- If we build the orchestrator first, we'll never know whether it helped.

---

## 7.6 Technology recommendations

| Concern | Recommendation | Why |
| --- | --- | --- |
| **Core language** | **TypeScript** (Node/Bun) | Best MCP SDK support; Vercel AI SDK; OpenCode/Cline prove it; largest tool ecosystem. **Go** is the serious alternative — better concurrency and single-binary distribution — but weaker AI-library ecosystem. |
| Model layer | Vercel AI SDK (TS) or LiteLLM (sidecar) | Don't write adapters |
| Model catalogue | models.dev + our observations | Already exists |
| MCP | Official TypeScript SDK, 2026-07-28 | Class A |
| Transport | HTTP + SSE | What OpenCode uses; simple, proxy-friendly |
| State | Files in git + SQLite index | Brain must be human-readable; SQLite is a derived cache |
| Sandbox | Landlock+seccomp / Seatbelt → Docker → Firecracker | Tiered by trust |
| Worktrees | git CLI | Boring and correct |
| Browser | Playwright MCP + Chrome DevTools MCP | Class B/C |
| Embedded | platformio-mcp + serial-mcp | Class B |
| Schemas | JSON Schema + Zod | Validation at every boundary |
| Events | append-only JSONL + SSE | Replayable |

**On reusing OpenHands' SDK instead of writing our own runtime:** worth a serious spike. It's MIT, it has native sandboxed execution, model-agnostic routing, lifecycle control, and integrated security analysis — explicitly designed as a composable foundation. **[2°]** If it fits, it could collapse P0–P1 substantially. The risk is inheriting its abstractions where they conflict with our blackboard/contract model. **Recommendation: timebox a two-week spike before committing to build P1 from scratch.**
