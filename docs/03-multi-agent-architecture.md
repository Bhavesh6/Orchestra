# Document 3 — Multi-Agent Architecture

> Read [Document 8 — Critique](08-critique.md) alongside this. This document describes the design *if* we build it; Document 8 argues about *when* it earns its cost. The two were written together and disagree with each other on purpose.

---

## 3.1 The core thesis, restated precisely

The brief's thesis is "a central orchestrator decomposes a project into parallel specialized workstreams." After the research, I want to sharpen it, because the naive version is the version that fails.

> **Parallelism is not the product. Bounded context with enforced interfaces is the product. Parallelism is a side effect we get once interfaces are enforced.**

Why the reframe matters: the failure literature on multi-agent LLM systems (MAST, arXiv 2503.13657, NeurIPS 2025) categorizes failures into **system design issues, inter-agent misalignment, and task verification** — with secondary analyses attributing roughly 42% / 37% / 21% respectively. **[2°]** for the percentages, **[Src]** for the taxonomy. Only one of those three buckets is about the agents being individually bad at their jobs. Two thirds are about **coordination and verification** — that is, about *interfaces*.

So the architecture below is designed around interfaces first. If we build the interfaces and then discover that running the workstreams sequentially on one strong model is faster and cheaper, **we still win**, because the interfaces are what produced the correctness. That is the fallback that makes this project safe to attempt.

---

## 3.2 Layer map

```
┌─────────────────────────────────────────────────────────────┐
│  CLIENTS   TUI · web · IDE ext · CI · SDK        (thin)     │
└───────────────────────────┬─────────────────────────────────┘
                            │  HTTP + SSE event stream
┌───────────────────────────▼─────────────────────────────────┐
│  PLATFORM SERVER (headless, the actual product)             │
│                                                             │
│  ┌───────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ ORCHESTRATOR  │  │  SCHEDULER   │  │  VERIFIER        │  │
│  │ decompose     │→ │  DAG, caps,  │→ │  gates, evidence │  │
│  │ plan, integrate│  │  conflicts   │  │  failure→task    │  │
│  └───────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │
│          │                 │                   │            │
│  ┌───────▼─────────────────▼───────────────────▼─────────┐  │
│  │  BLACKBOARD  (Project Brain + contracts + artifacts)  │  │
│  └───────────────────────────┬───────────────────────────┘  │
│                              │                              │
│  ┌───────────────────────────▼───────────────────────────┐  │
│  │  AGENT RUNTIME   loop · context assembly · budget     │  │
│  └───────┬───────────────────────────────────┬───────────┘  │
│          │                                   │              │
│  ┌───────▼────────┐              ┌───────────▼───────────┐  │
│  │  TOOL LAYER    │              │   MODEL GATEWAY       │  │
│  │  builtin + MCP │              │   capability router   │  │
│  └───────┬────────┘              └───────────┬───────────┘  │
│          │                                   │              │
│  ┌───────▼────────┐              ┌───────────▼───────────┐  │
│  │ WORKSPACE MGR  │              │ providers: Anthropic, │  │
│  │ worktrees,     │              │ OpenAI, Google, NVIDIA│  │
│  │ sandboxes      │              │ OpenRouter, local     │  │
│  └────────────────┘              └───────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Everything above the client line is a client.** Following OpenCode's proven split (Doc 1 §1.6): the TUI never executes agent work. This is non-negotiable and it is what brief §31 is really asking for.

---

## 3.3 Agent runtime

One runtime, many configurations. An "agent" is not a class — it's a **configuration record** the runtime executes.

```jsonc
{
  "id": "firmware-01",
  "role": "firmware",
  "objective": "Implement sensor read loop per HW-CONTRACT v3",
  "model_policy": {
    "requires": ["coding", "tool_calling", "long_context"],
    "prefers": ["low_cost"],
    "escalate_to": ["reasoning"]        // on repeated failure
  },
  "tools":  ["fs.*", "platformio.*", "serial.read", "brain.read_*"],
  "skills": ["esp32-sensor-debug", "project-c-style"],
  "context_grant": {
    "brain_paths": ["contracts/hardware", "requirements/sensors"],
    "files": ["src/sensors/**", "include/pins.h"],
    "prior_results": ["hardware-01"]     // explicit, not ambient
  },
  "workspace": { "mode": "worktree", "branch": "agent/firmware-01" },
  "budget": { "tokens": 400000, "wall_clock_s": 900, "tool_calls": 200 },
  "limits": { "consecutive_failures": 3, "depth": 1 }
}
```

Design commitments visible here:

- **Context is granted, never ambient.** An agent gets exactly the Brain paths, files, and prior results the orchestrator names. This is the single biggest lever on both cost and correctness. Claude Code's docs make the same point about subagents; we make it explicit and machine-checkable. **[Doc]** for the principle, **[Inf]** for the mechanism.
- **Budgets are hard.** Tokens, wall clock, and tool calls. An agent that exhausts budget returns `status: "budget_exhausted"` with partial results — it does not silently truncate.
- **`consecutive_failures`** is Cline's mistake-limit idea (Doc 1 §1.7), which is a cheap circuit breaker most designs omit.
- **`depth: 1`** — delegation depth cap, from Codex's `max_depth`. **[2°]** Agents may not spawn agents beyond the configured depth. Claude Code independently arrived at the same constraint ("no nested teams"). Two independent implementations converging on this is strong evidence it's necessary.

---

## 3.4 The task graph

A DAG. Nodes are tasks; edges are dependencies. This is the orchestrator's actual output — not prose.

```jsonc
{
  "id": "T-014",
  "title": "Implement DHT22 read path",
  "role": "firmware",
  "depends_on": ["T-003"],                  // hardware pin contract
  "consumes_contracts": ["HW-PINMAP@v3"],
  "produces_contracts": [],
  "file_scope": ["src/sensors/dht22.*", "include/pins.h"],   // ← ownership
  "verification": ["build.esp32", "test.unit.sensors", "flash.smoke"],
  "acceptance": "Serial emits TEMP=<float> HUM=<float> at 1Hz for 10s",
  "parallel_safe": true,
  "estimated_cost_usd": 0.40
}
```

### `file_scope` is the conflict-prevention mechanism

This is the piece I consider most important and it is missing from every system I researched.

**The scheduler refuses to run two tasks concurrently whose `file_scope` globs intersect.** Not "detect the conflict afterwards and ask a review agent to resolve it" — *never create it*.

The brief (§14) proposes a conflict-resolution flow with a Review Agent adjudicating between Hardware, Research, and Firmware agents. That flow is valuable for **semantic** conflicts (an MQ-9 sensor can't do what the API assumes). It is the **wrong tool for textual** conflicts (two agents edited `pins.h`). Textual conflicts should be made structurally impossible by the scheduler, because resolving them costs model calls, adds latency, and — per MAST — is exactly where multi-agent systems fail.

Cursor's own guidance corroborates the cost: merge-conflict complexity "scales non-linearly with agent count," which is why they recommend 2–3 agents despite supporting 8. **[2°]**

So:

| Conflict type | Mechanism | Cost |
| --- | --- | --- |
| **Textual** (same file/region) | **Prevented** by scheduler `file_scope` disjointness | ~0 |
| **Contract** (agent violates a published interface) | **Detected** by contract validator, deterministic | ~0 (no model call) |
| **Semantic** (claims are mutually inconsistent) | **Adjudicated** by Reviewer agent with evidence | expensive, rare |

Only the third tier needs an LLM. The brief's design spends model calls on all three.

---

## 3.5 Communication: blackboard, not chat

**Agents do not talk to each other.** This is a deliberate departure from Claude Code's agent teams (where teammates message each other directly via mailboxes) and from most agent frameworks.

```
        ┌──────────────────────────────────┐
        │          BLACKBOARD              │
        │  Project Brain · contracts ·     │
        │  artifacts · evidence · results  │
        └──┬─────────┬─────────┬─────────┬─┘
     read/ │         │         │         │
     write │         │         │         │
        ┌──▼──┐   ┌──▼──┐   ┌──▼──┐   ┌──▼──┐
        │ HW  │   │ FW  │   │ Web │   │ Res │
        └─────┘   └─────┘   └─────┘   └─────┘
             (no edges between agents)
```

Rationale, and I want to be honest that this is a bet:

1. **MAST's largest single failure category is inter-agent misalignment.** **[Src]** Removing the edges removes the category. You cannot have a misaligned conversation you never had.
2. **Chat is lossy and unauditable.** "Hardware told me GPIO34" in a transcript is not evidence. `contracts/hardware/pinmap.yaml@v3` is.
3. **It's replayable.** Blackboard state at time T fully determines what an agent saw. Transcripts between six agents do not.
4. **It scales without N² coordination.** Adding an agent adds one reader, not N edges.

The cost is real: an agent cannot ask a peer a clarifying question. It must instead write a **`question` into its structured result** and let the orchestrator route it — to the Brain, to a human, or to a new investigation task. That is slower per hop but it produces an auditable decision record, and it keeps the orchestrator as the single point of integration (which the brief's §27 hierarchy also demands).

> **Where I'd revisit this:** Claude Code's docs describe a genuinely compelling use case for direct messaging — adversarial debate, where teammates try to *disprove* each other's theories, and the theory that survives is likelier to be the root cause. **[Doc]** That's a real mechanism that a blackboard doesn't reproduce well. If we want debate, we should implement it as an explicit **`debate` task type** with structured rounds, not as ambient chat. Noted as an open question in Doc 8.

---

## 3.6 Contracts

Contracts are the typed interfaces between workstreams (brief §16). They are **versioned, validated, machine-readable files in the Project Brain**, not prose.

```yaml
# .project/contracts/hardware/pinmap.yaml
contract: HW-PINMAP
version: 3
status: accepted
owner: hardware
decided_by: T-003
evidence: [".project/research/esp32-devkit-v1-pinout.md#adc2-wifi-conflict"]

board: esp32-devkit-v1
assignments:
  - { signal: MQ9_ADC,   pin: GPIO34, mode: analog_in, adc: ADC1, notes: "ADC1 required; ADC2 unusable with WiFi" }
  - { signal: DHT22_DATA,pin: GPIO4,  mode: digital_io, pullup: external_10k }
  - { signal: LCD_SDA,   pin: GPIO21, mode: i2c_sda }
  - { signal: LCD_SCL,   pin: GPIO22, mode: i2c_scl }

constraints:
  - id: C-01, rule: "no pin assigned twice"
  - id: C-02, rule: "GPIO34-39 are input-only"
  - id: C-03, rule: "ADC2 pins unusable while WiFi active"
```

### Contracts are validated by code, not by a model

`C-01` through `C-03` are executable checks. The contract validator runs on every proposed change and is **deterministic, instant, and free**. If the Firmware agent writes `digitalWrite(34, HIGH)`, a static check fails against `C-02` — no model call, no debate, no token spend.

This generalizes far beyond hardware:

| Domain | Contract | Deterministic validator |
| --- | --- | --- |
| Hardware | pin map, power budget | pin uniqueness, input-only pins, current sum ≤ supply |
| API | OpenAPI schema | schema diff; client/server conformance tests |
| Database | migration + schema | migration linter, FK integrity |
| Frontend↔Backend | typed client | codegen + typecheck |
| Firmware↔Hardware | `pins.h` generated **from** the contract | compile fails on drift |

**The most valuable pattern here: generate code from contracts.** `include/pins.h` should be *generated* from `pinmap.yaml`. Then the Firmware agent physically cannot use a wrong pin — the compiler enforces the hardware agent's decision. That converts a coordination problem into a build error, which is the best trade in this entire document.

---

## 3.7 Structured agent results

Improving on the brief's §13 schema. Changes: everything that makes a claim must carry evidence; confidence is explicit; the "I'm not done" states are first-class.

```jsonc
{
  "task_id": "T-014",
  "agent_id": "firmware-01",
  "status": "complete",
  // complete | blocked | failed | partial | budget_exhausted | needs_decision

  "summary": "DHT22 read path implemented; 1Hz output verified on hardware.",

  "claims": [
    {
      "id": "CL-1",
      "statement": "DHT22 requires ≥2s settle after power-on before first valid read",
      "confidence": 0.95,
      "evidence": [
        { "kind": "datasheet", "ref": ".project/research/dht22.pdf#p3", "quote": "..." },
        { "kind": "observation", "ref": "artifacts/serial-capture-T014.log#L20-38" }
      ]
    }
  ],

  "contracts_consumed": ["HW-PINMAP@v3"],
  "contracts_proposed": [],
  "contract_violations": [],

  "changes": [
    { "path": "src/sensors/dht22.c", "kind": "modified", "lines_added": 84, "lines_removed": 3 }
  ],

  "verification": [
    { "gate": "build.esp32",        "result": "pass", "ref": "artifacts/build-T014.log" },
    { "gate": "test.unit.sensors",  "result": "pass", "detail": "12/12" },
    { "gate": "flash.smoke",        "result": "pass", "ref": "artifacts/serial-capture-T014.log" }
  ],

  "assumptions": [
    { "text": "External 10k pull-up is populated on the board", "validated": false, "risk": "high" }
  ],

  "open_questions": [
    { "text": "Should read failures retry or report stale?", "blocking": false, "route_to": "human" }
  ],

  "cost": { "tokens_in": 180000, "tokens_out": 22000, "usd": 0.41, "wall_clock_s": 412, "model": "..." }
}
```

Rules the runtime enforces on this object:

1. **`status: "complete"` is rejected unless every gate in the task's `verification` list reports `pass`.** An agent cannot declare victory (brief §18). This is checked by the runtime, not requested in a prompt.
2. **A claim without evidence is downgraded to an assumption.** Mechanically. This is the anti-hallucination lever.
3. **Unvalidated `high`-risk assumptions auto-generate investigation tasks.**
4. **Cost is always reported**, so the orchestrator can learn routing (and so we can measure whether any of this is worth it).

---

## 3.8 Workspaces and isolation

| Mode | When | Cost |
| --- | --- | --- |
| **Shared read-only** | research, review, analysis — no writes | free |
| **Git worktree** (default for writers) | any agent that edits files | ~free, seconds to create |
| **Container** | untrusted deps, install steps, `npm install` | seconds |
| **microVM** (Firecracker/Kata) | untrusted repos, full network, hostile input | ~100ms boot, heavier |
| **Physical device lease** | ESP32 flashing — **exclusive, serialized** | n/a |

Default: **worktree per writing agent**, following Cursor. **[2°]** It's cheap, it's git-native, and it gives us a real merge path rather than a bespoke one.

### Integration is a merge queue, not a model call

```
agent/firmware-01 ─┐
agent/web-01     ──┼──► integration branch ──► full verification ──► main
agent/hw-01      ──┘        (sequential merges, ordered by DAG)
```

Merges happen **one at a time in dependency order**, with the verification suite run after each. If merge *N* breaks the build, only *N* is at fault — no bisecting a six-way merge. Slower than merging everything at once; dramatically easier to diagnose. Given that debugging a bad multi-agent merge is the most expensive failure mode in this whole system, that trade is correct.

Note on hardware: **a physical ESP32 is a mutex.** Only one agent may hold the device lease. Any "parallel firmware agents" design collapses at the flash step. Worth stating loudly because it's easy to design a beautiful parallel firmware pipeline that cannot exist.

---

## 3.9 The orchestrator loop

```
1  INTAKE        parse objective; load Brain; ask clarifying Qs (cheap, up front)
2  SURVEY        cheap model maps repo → structure, languages, entry points
3  DECOMPOSE     strong model → task DAG + roles + contracts + file_scope
4  PLAN REVIEW   ◄── HUMAN GATE. Present plan as an artifact. Approve/edit.
5  SCHEDULE      topological order; parallel only where file_scope disjoint
                 AND dependencies met AND budget allows
6  DISPATCH      per task: pick model (Doc 4), scope tools, grant context,
                 allocate workspace, run
7  COLLECT       structured results → blackboard; validate contracts
8  VERIFY        run gates; failures become new tasks (not retries of the same)
9  INTEGRATE     merge queue in dependency order; verify after each
10 ADJUDICATE    semantic conflicts only → Reviewer agent w/ evidence
11 REPORT        artifacts + decision log + cost breakdown
```

**Step 4 is mandatory and should never become optional.** Antigravity's implementation-plan artifact is the model here. **[2°]** A wrong decomposition is the most expensive failure available to this system: it multiplies across every downstream agent. Per MAST, bad specification is the *largest* failure category. **[Src]** A human reading a one-page DAG for thirty seconds is the cheapest error correction in the entire architecture — orders of magnitude cheaper than any verification we can run afterwards.

---

## 3.10 Agent hierarchy (brief §27)

| Tier | Authority | May write code? | May merge? |
| --- | --- | --- | --- |
| **Orchestrator** | plan, decompose, integrate, final call | no | yes (via queue) |
| **Specialist** | one domain, one task, scoped context | yes, in own worktree | no |
| **Reviewer** | judge, find contradictions, **cannot edit** | **no** | no |
| **Executor** | run approved tools/tests/flash | limited, approved ops | no |
| **Researcher** | read-only + web/docs | no | no |

**Reviewers must not be able to edit.** If a reviewer can fix what it finds, it stops reporting and starts patching, and you lose the independent signal — the entire value of having a second opinion. This should be enforced by the tool scope, not by instructions in a prompt.

---

## 3.11 Parallelism: the three kinds, and honest verdicts

| Kind | Description | Verdict |
| --- | --- | --- |
| **Parameter parallelism** | different dimensions (HW / FW / web) run concurrently | ✅ **The real win.** Disjoint file scopes, genuine speedup. |
| **Tool parallelism** | independent tool calls concurrently | ✅ **Free win.** Do it everywhere. Zero coordination risk. |
| **Model parallelism** | N models answer the same question, then adjudicate | ⚠️ **Rarely worth it.** N× cost for questions where the models mostly agree. |

On model parallelism specifically: the brief (§12) is right that we must not blindly average responses. I'd go further — **restrict it to an explicit, opt-in, gated mode**, as Cursor did with `/best-of-n`. **[2°]** Trigger it only when:
- the decision is **irreversible or high-blast-radius** (schema, protocol, pin assignment), **and**
- a first pass returned **low confidence** or the verification gates disagree, **and**
- an adjudicator with **independent evidence** can decide — not a vote.

Voting between models is not truth-finding; it's a popularity contest between correlated priors. Evidence-based adjudication is the only version worth building.

---

## 3.12 When the orchestrator should refuse to orchestrate

The most important behavior in the system, and the one that will make or break its reputation:

```
if task_count <= 2
   or file_scopes_overlap_heavily
   or the work is inherently sequential
   or the repo is small enough to fit in one context
   or the task is exploratory/ill-defined:
        → run as a SINGLE agent. Say so. Report the estimated saving.
```

This must be a **default**, not a setting. Claude Code's docs say plainly that for "sequential tasks, same-file edits, or work with many dependencies, a single session or subagents are more effective." **[Doc]** Cursor recommends starting with 2–3 agents despite supporting 8. **[2°]** Both vendors are telling us the same thing from experience.

A system that fans out six agents to rename a variable is not impressive. It's broken, and expensively so.
