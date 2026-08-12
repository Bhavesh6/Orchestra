# Orchestra — Roadmap

**How we're actually building it, step by step.**

Each milestone has a **definition of done** that is checkable, not a feeling. M5 is a real decision point where the answer may be "stop here."

---

## Ordering principle

> Build the deterministic parts first — they work or they don't, with no unproven claims about model behaviour. Build the probabilistic parts last, gated on measurement.

That's why gates and contracts come before the orchestrator, even though the orchestrator is the exciting part.

```
M0  Setup & decisions          ~1 week
M1  Contract engine            2-3 weeks   ← the core mechanism
M2  Gate runner                1 week      ← applies to all work
M3  MCP server                 1 week      ← works in any agent
M4  Dogfood on real work       4 weeks     ← the experiment
M5  DECISION GATE                          ← evidence, not vibes
M6  Orchestrator
M7  Artifacts / agent manager
M8  IDE surface
```

---

## M0 — Setup & decisions · ~1 week

**Tasks**
- [x] Repo, research docs, ADRs 0001–0004
- [ ] **Working agent again**: OpenCode + NVIDIA Build (`https://integrate.api.nvidia.com/v1`). Verify tool calling on an agentic coding model.
- [ ] **Language decision** → ADR-0005
- [ ] **Weekend spike**: bare agent loop from scratch (~300 lines). Throwaway — the point is to read OpenCode's source with informed eyes afterwards.
- [ ] **Read OpenCode's source.** Map where L2 attaches. Extend vs fork → ADR-0006.
- [ ] Pick 2–3 dogfood projects across different domains

**Definition of done**
You're coding with an agent again, the language is chosen and justified, and you can point at the exact place in OpenCode where Orchestra attaches.

**On the language.** TypeScript matches OpenCode and has the stronger MCP ecosystem. Python is better for ML plugins (pandera, pydantic). A real trade-off — decide deliberately, write down why.

---

## M1 — Contract engine · ~2–3 weeks

The core mechanism.

**Tasks**
- [ ] Contract envelope format + JSON Schema
- [ ] **Plugin interface for contract kinds** — a new domain must never require a core change
- [ ] Validators (deterministic, no model calls), in this order:
  - [ ] `openapi` — schema lint, breaking-change detection vs accepted version
  - [ ] `sql_schema` — FK integrity, migration ordering, destructive-change guard
  - [ ] `data_schema` — dtypes, ranges, nullability, leakage invariants
  - [ ] `pinmap` — the embedded case, once the general path works
- [ ] Generators with `authoritative: true` — generated files committed *and* verified unmodified
- [ ] Drift detection: hand-editing a generated file is a failing gate, not a silent merge
- [ ] CLI: `orchestra validate` · `generate` · `status`

**Definition of done — the thesis test**

> Change a field name in an OpenAPI contract. The generated client regenerates. **The frontend typecheck fails, naming the contract that changed.**

The general case comes first deliberately — if it only works for embedded, it isn't a platform. Once that passes, the same test for `pinmap`: an agent writes `digitalWrite(35, HIGH)`, the build fails naming the violated pin contract.

---

## M2 — Gate runner · ~1 week

The half that applies to **100%** of work, in every language.

**Tasks**
- [ ] Declarative `gates.yaml` (`run`, `pass_if`, `timeout`, `artifacts`, profiles)
- [ ] `pass_if`: `exit_zero`, metric thresholds, regex match. Deliberately small — a gate DSL that grows into a language has failed.
- [ ] Structured results with evidence refs and `observed` values
- [ ] **Runtime rejection of unverified `complete`**
- [ ] Distinguish `fail` (criterion unmet) from `error` (couldn't run) — they need different responses
- [ ] Failures become tasks with diagnostic context, with a retry cap

**Definition of done**
An agent claims complete while a gate fails. Orchestra rejects it and returns the failing gate plus evidence.

---

## M3 — MCP server · ~1 week

What makes L2 useful everywhere without building a runtime.

**Tasks**
- [ ] `orchestra-mcp` on the **2026-07-28** spec — stateless core, MRTR, cacheable lists. **No `sampling`/`roots`/`logging`** (deprecated).
- [ ] Tools: `contract.read` · `contract.propose` · `contract.validate` · `generate.run` · `gate.list` · `gate.run` · `status`
- [ ] Transport behind an interface — the spec broke once and will again
- [ ] Verify in OpenCode, Claude Code, and Cline

**Definition of done**
Three different agents, unmodified, read contracts and run gates through Orchestra.

---

## M4 — Dogfood · ~4 weeks

**The experiment.** Runs on work you were doing anyway, across **at least two different domains** — that's what tells us whether the core is genuinely domain-agnostic or quietly specialised.

**Tasks**
- [ ] Use Orchestra on 2–3 real projects in different domains
- [ ] Log every catch a review would have missed — and every time it got in the way
- [ ] Track: violations caught, gate failures caught, false positives, contract maintenance time
- [ ] Note which contract kinds earn their keep and which are ceremony

**Definition of done**
Four weeks of real usage with written evidence, in more than one domain. The question with no armchair answer: **does contract maintenance cost less than the errors it prevents?**

---

## M5 — Decision gate

Not a milestone. A fork in the road.

**Three-arm benchmark** on ~20 realistic tasks:

| Arm | Setup |
| --- | --- |
| **A** | Plain single agent |
| **B** | Single agent + contracts + gates |
| **C** | Multi-agent + contracts + gates |

Measure: correctness, build success, test/eval pass, retries, tokens, wall clock, cost, contradictions, human interventions.

**Decision rules**
- **B ≫ A** → thesis holds; contracts and gates are the product
- **C ≫ B** on wall clock at acceptable cost → build M6
- **C ≈ B or worse** → **stop at L2.** A planned outcome, not a failure.

**Prediction on record:** B captures most of the gain. Written now so it can't be rationalised later.

⚠️ NVIDIA Build free tier is **40 requests/minute**. Arm C may be rate-limited before coordination overhead even binds — measure requests/min, not just cost.

---

## M6 — Orchestrator · only if M5 justifies it

- [ ] Task DAG with `file_scope` for conflict *prevention*, not resolution
- [ ] Scheduler: topological order + scope disjointness + concurrency cap + rate-limit headroom
- [ ] **Mandatory human plan-approval gate** — a wrong decomposition is the most expensive failure available
- [ ] Blackboard results; no agent-to-agent chat
- [ ] **Single-agent default**, with the orchestrator explaining when it declines to parallelise
- [ ] Worktree per writing agent; sequential merge queue

## M7 — Artifacts & agent manager

The Antigravity-shaped surface. Meaningful only on a mature L1+L2.

- [ ] Artifacts as the review surface: plans, diffs, gate results, screenshots, cost — not N scrolling transcripts
- [ ] Task graph visualisation
- [ ] Commentable plans before execution

## M8 — IDE surface

- [ ] TUI first
- [ ] VS Code extension before considering a fork
- [ ] Fork only if the extension API genuinely blocks us — that's how Cursor, Windsurf and Antigravity all got there, and all of them later than they needed to

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
| Local model support | Cloud is free and better; hardware isn't there |
| Agent-to-agent chat | Blackboard — deliberate |
| Intelligent model routing | Needs data we won't have for months |
| A UI before M8 | The API is the product |

---

## Risks we're watching

| Risk | Watch for | Response |
| --- | --- | --- |
| Contract maintenance exceeds benefit | M4 log | Drop kinds that don't earn keep |
| Generated artifacts drift | Hand-edits to generated files | `authoritative: true` + drift gate |
| Core quietly becomes domain-specific | M4 across ≥2 domains | Plugin interface, enforced |
| Multi-agent never pays off | M5 benchmark | Stop at L2 — planned |
| Rate limits throttle everything | requests/min in M4–M5 | Multi-provider failover |
| Scope explosion | Milestones slipping | The DoD gates, ruthlessly |
| Someone ships this first | — | L2 is weeks, not years |

---

## Current status

**M0, in progress.** Next: OpenCode + NVIDIA Build running, then the language decision.
