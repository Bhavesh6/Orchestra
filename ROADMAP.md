# Orchestra — Roadmap

**How we're actually building this, step by step.**

Each milestone has a **definition of done** that is checkable, not a feeling. We don't move on until it's met. M5 is a genuine decision point where the answer may be "stop here."

---

## Ordering principle

> Build the deterministic parts first, because they work or they don't — no unproven claims about model behavior. Build the probabilistic parts last, gated on measurement.

That's why gates and contracts come before the orchestrator, even though the orchestrator is the exciting part.

```
M0  Setup & decisions          ← 1 week
M1  Contract engine            ← 2-3 weeks     ← the core mechanism
M2  Gate runner                ← 1 week
M3  MCP server                 ← 1 week
M4  Dogfood on real work       ← 4 weeks       ← the experiment
M5  DECISION GATE              ← does it help? evidence, not vibes
M6  Orchestrator               ← only if M5 says yes
M7  Artifacts / agent manager
M8  IDE surface
```

---

## M0 — Setup & decisions · ~1 week

Get unblocked and lock the choices everything else depends on.

**Tasks**
- [ ] Repo created, research docs migrated *(done)*
- [ ] **Get a working agent again**: OpenCode + NVIDIA Build (`https://integrate.api.nvidia.com/v1`, provider block + `auth.json`). Verify tool calling works with Qwen3 Coder 480B or GLM-5.2.
- [ ] **Language decision**: TypeScript vs Python for the Orchestra tooling. Record as ADR-0005.
- [ ] **Weekend spike**: write a bare agent loop from scratch (~300 lines — model call, tool dispatch, loop until done). Throwaway. The point is to read OpenCode's source with informed eyes afterward.
- [ ] **Read OpenCode's source.** Map exactly where L2 plugs in. Decide *extend vs fork* and record as ADR-0006.
- [ ] Pick the three dogfood projects: one web/API, one ML, one ESP32.

**Definition of done**
You're writing code with an agent again, the language is chosen and justified, and you can point at the specific place in OpenCode where Orchestra attaches.

**Note on the language choice.** TypeScript has the stronger MCP and agent ecosystem and matches OpenCode. Python is better for the ML contract plugins (pandera, pydantic) and for the embedded scripting. This is a real trade-off — decide it deliberately and write down why.

---

## M1 — Contract engine · ~2–3 weeks

The core mechanism. Everything else is scaffolding around this.

**Tasks**
- [ ] Contract file format + JSON Schema for the contract-of-contracts
- [ ] **Plugin interface for contract kinds** — adding a domain must be a plugin, not a core change
- [ ] Validators: deterministic, no model calls
  - [ ] `openapi` — schema lint, breaking-change detection
  - [ ] `sql_schema` — FK integrity, migration ordering
  - [ ] `data_schema` — shapes, dtypes, ranges, nullability
  - [ ] `pinmap` — `unique_pins`, `input_only_pins`, `adc2_unusable_with_wifi`, power budget
- [ ] Generators with `authoritative: true` — generated files are committed *and* verified unmodified
- [ ] Drift detection: hand-edit to a generated file is an error, not a merge
- [ ] CLI: `orchestra validate` · `orchestra generate` · `orchestra status`

**Definition of done — the thesis test**

> Define `pinmap.yaml` for a real ESP32 project. Generate `pins.h`. Have an agent write `digitalWrite(35, HIGH)`. **The build fails with a message naming the violated contract.**

Repeat the equivalent for OpenAPI: change a backend field, watch the frontend typecheck fail.

If both work, the central idea is proven — with no orchestrator, no task graph, no UI.

---

## M2 — Gate runner · ~1 week

**Tasks**
- [ ] Declarative `gates.yaml` (`run`, `pass_if`, `timeout`, `artifacts`)
- [ ] `pass_if` expressions: `exit_zero`, threshold comparisons on parsed metrics (`metrics.auc >= 0.82`)
- [ ] Structured result schema with evidence references
- [ ] **Runtime rejection of unverified `complete`** — the rule that makes "done" mean something
- [ ] Gate artifacts captured: logs, diffs, metric JSON
- [ ] `orchestra gate <name>` and `orchestra gate --all`

**Definition of done**
An agent claims a task is complete while a gate fails. Orchestra rejects it and returns the failing gate plus its evidence.

---

## M3 — MCP server · ~1 week

The step that makes L2 useful everywhere without building a runtime.

**Tasks**
- [ ] `orchestra-mcp` against the **2026-07-28** spec — stateless core, MRTR, cacheable lists. **Do not implement `sampling`, `roots`, or `logging`** (deprecated).
- [ ] Tools: `contract.read`, `contract.propose`, `contract.validate`, `generate.run`, `gate.list`, `gate.run`, `status`
- [ ] Isolate transport behind an interface — the spec broke once and will again
- [ ] Verify in OpenCode, Claude Code, and Cline

**Definition of done**
Three different agents, unmodified, can read contracts and run gates through Orchestra.

---

## M4 — Dogfood · ~4 weeks

**This is the experiment.** It runs on work you were doing anyway, at zero marginal cost.

**Tasks**
- [ ] Use Orchestra on all three real projects (web/API, ML, ESP32)
- [ ] Keep a log: every time a contract or gate caught something a review would have missed — and every time it got in the way
- [ ] Track: contract violations caught, gate failures caught, false positives, time spent maintaining contracts
- [ ] Note which contract kinds earn their keep and which are ceremony

**Definition of done**
Four weeks of real usage with written evidence. Specifically: **does the contract maintenance cost less than the errors it prevents?** That question has no armchair answer.

---

## M5 — Decision gate · the honest checkpoint

Not a milestone. A fork in the road.

**Run the three-arm benchmark** on ~20 realistic tasks:

| Arm | Setup |
| --- | --- |
| **A** | Plain single agent |
| **B** | Single agent + contracts + gates |
| **C** | Multi-agent + contracts + gates |

Measure: correctness, compile success, test/eval pass, retries, tokens, wall clock, cost, contradictions, human interventions.

**Decision rules**
- **B ≫ A** → the thesis holds. Contracts and gates are the product.
- **C ≫ B** on wall-clock at acceptable cost → build M6. The orchestrator is justified.
- **C ≈ B or worse** → **stop at L2.** Ship the contract system, skip the orchestrator, and be glad we found out in 3 months rather than 18.

**Prediction on record:** B captures most of the gain. Writing it down now so we can't rationalize later.

⚠️ **Provider constraint to account for:** NVIDIA Build free tier is **40 requests/minute**. Agentic loops are request-heavy. Arm C may be rate-limited before coordination overhead even matters — measure requests/min, not just cost.

---

## M6 — Orchestrator · only if M5 justifies it

- [ ] Task DAG with `file_scope` for conflict *prevention* (not resolution)
- [ ] Scheduler: topological order + file-scope disjointness + concurrency cap + rate-limit headroom
- [ ] **Mandatory human plan-approval gate** — a wrong decomposition is the most expensive failure available
- [ ] Blackboard results, no agent-to-agent chat
- [ ] Structured result schema across agents
- [ ] **Single-agent fallback as the default**, with the orchestrator explaining when it declines to parallelize
- [ ] Worktree per writing agent; sequential merge queue

---

## M7 — Artifacts & agent manager

The Antigravity-shaped part. Only meaningful on a mature L1+L2.

- [ ] Artifacts as the review surface: plans, diffs, gate results, screenshots, cost — not six scrolling transcripts
- [ ] Task graph visualization
- [ ] Commentable plans before execution

## M8 — IDE surface

- [ ] TUI first
- [ ] VS Code extension before considering a fork
- [ ] Fork only if the extension API genuinely blocks us — that's how Cursor, Windsurf and Antigravity all got there, and all of them started later than they needed to

---

## Explicitly not building

| Not building | Because |
| --- | --- |
| Our own agent runtime | OpenCode is MIT and mature |
| Our own tool protocol | MCP |
| Our own OAuth | MCP authz + CIMD |
| Browser automation engine | Playwright MCP, Chrome DevTools MCP |
| Embedded toolchain wrappers | platformio-mcp, serial-mcp |
| Provider adapters | Vercel AI SDK / LiteLLM |
| Local model support | Hardware isn't there; cloud is free and better |
| Agent-to-agent chat | Blackboard — deliberate |
| Intelligent model routing | Needs data we won't have for months; static role profiles first |
| A UI before M8 | The API is the product |

---

## Risks we're actively watching

| Risk | Watch for | Response |
| --- | --- | --- |
| Contract maintenance exceeds benefit | M4 log | Drop contract kinds that don't earn keep |
| Generated artifacts drift | Hand-edits to generated files | `authoritative: true` + drift gate |
| Contracts only work in crisp domains | ML/web vs embedded comparison in M4 | Narrow scope honestly |
| Multi-agent never pays off | M5 benchmark | Stop at L2 — planned outcome, not failure |
| Rate limits throttle everything | Requests/min in M4–M5 | Multi-provider failover |
| Scope explosion | Milestones slipping | The DoD gates. Ruthlessly. |
| Someone else ships this first | — | Ship L2 fast; it's weeks, not years |

---

## Current status

**M0, in progress.** Next concrete action: get OpenCode + NVIDIA Build running, then decide the implementation language.
