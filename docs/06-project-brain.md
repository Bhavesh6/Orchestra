# Document 6 — Project Brain

---

## 6.1 The problem it solves

A codebase records **what** the code is. It does not record **why**. Every decision, constraint, rejected alternative, and piece of supporting evidence lives in a chat transcript that is discarded at session end.

For a single agent this is annoying. For a multi-agent system it is fatal, because:

- Agents have **no shared conversation** to inherit from — Claude Code documents exactly this: teammates load project context fresh and "the lead's conversation history does not carry over." **[Doc]**
- If the only shared state is source code, agents must **re-derive** intent from it, and they will derive it differently. That is precisely the "inter-agent misalignment" bucket in the MAST failure taxonomy. **[Src]**

So the Brain is not a nice-to-have memory feature. **It is the coordination substrate.** Without it, this architecture is six agents guessing at each other's intentions.

Antigravity independently arrived at something similar — a knowledge base that agents both read from and contribute to, so the system learns across tasks. **[2°]** That convergence is encouraging.

---

## 6.2 Design principles

1. **Files in git, not a database.** Human-readable, diffable, reviewable, forkable, and it survives the platform being uninstalled. A database index can be *derived* from files; files cannot be derived from a database.
2. **Structured where it matters, prose where it doesn't.** Contracts and constraints are YAML with schemas. Research notes are markdown.
3. **Every claim carries evidence.** A statement without a source is an assumption, and it's labeled as one.
4. **Append-only decisions.** Decisions are never edited, only superseded — with a pointer to what superseded them and why.
5. **Exposed via MCP.** The Brain is an MCP server (Doc 2 §2.3), so our agents and external tools read it through one interface.
6. **Writes are typed and validated.** No free-text writes into structured regions. This is also the T8 (blackboard poisoning) control from [Document 5](05-security-architecture.md).

---

## 6.3 Layout

```
.project/
├── brain.yaml                  # index, schema version, project identity
│
├── requirements/
│   ├── functional.yaml         # REQ-001 … with priority, source, status
│   └── constraints.yaml        # budget, deadline, regulatory, physical
│
├── architecture/
│   ├── overview.md
│   ├── components.yaml         # component graph, ownership
│   └── diagrams/
│
├── contracts/                  # ← the coordination interfaces
│   ├── hardware/pinmap.yaml
│   ├── hardware/power.yaml
│   ├── api/openapi.yaml
│   ├── data/schema.sql
│   └── _generated/             # code generated FROM contracts (pins.h, clients)
│
├── decisions/                  # ADR-style, append-only
│   └── ADR-0007-mq9-single-channel.md
│
├── research/
│   ├── sources/                # cached PDFs, datasheets, pages (hashed)
│   └── notes/                  # extracted findings, each citing a source
│
├── agent-results/              # structured results, by task
│   └── T-014/result.json
│
├── artifacts/                  # review surfaces: plans, screenshots, logs, diffs
│   └── T-014/{build.log,serial-capture.log,diff.patch}
│
├── verification/
│   ├── gates.yaml              # gate definitions: how to run, how to judge
│   └── runs/                   # gate results with evidence refs
│
├── conflicts/                  # open + resolved, with adjudication record
│
├── memory/
│   ├── project.md              # durable facts (like CLAUDE.md/AGENTS.md)
│   └── learned.yaml            # observed patterns: "this repo's tests need X"
│
└── history/
    └── audit.jsonl             # immutable action log
```

**Compatibility note:** `memory/project.md` should be kept in sync with `CLAUDE.md` / `AGENTS.md` if present. Those conventions already exist and are already read by other tools; competing with them is pointless. Ours is a superset that also *generates* them.

---

## 6.4 Worked example: the MQ-9 conflict from the brief

The brief (§14) uses a great example. Here is how the Brain makes it tractable.

**1. Hardware agent proposes:**

```yaml
# contracts/hardware/sensors.yaml  (proposed)
- id: SENS-01
  part: MQ-9
  measures: [CO, CH4, LPG]
  interface: analog
  evidence: [research/notes/mq9-overview.md]
  confidence: 0.7
```

**2. Research agent writes a contradicting claim with better evidence:**

```yaml
# research/notes/mq9-selectivity.md
claim: "MQ-9 outputs a single analog value; it cannot independently
        report CO, CH4, and LPG concentrations simultaneously."
confidence: 0.95
evidence:
  - { kind: datasheet, ref: research/sources/mq9-datasheet.pdf#p2,
      quote: "single sensitive layer … output is a composite response" }
```

**3. Conflict detected — deterministically, no model call:**

The contract validator sees `SENS-01.measures` has three entries while a claim asserts a single composite output, and the requirement `REQ-014` demands independent readings. It opens:

```yaml
# conflicts/CF-003.yaml
type: semantic
severity: high
blocks: [T-014, T-021]
positions:
  - { source: hardware/T-003, claim: "MQ-9 satisfies REQ-014", confidence: 0.7 }
  - { source: research/T-009, claim: "MQ-9 cannot",            confidence: 0.95 }
  - { source: firmware/T-014, claim: "API assumes 3 channels", confidence: 1.0 }
```

**4. Reviewer adjudicates on evidence, not on vote count.** Datasheet quote beats a general overview page. Resolution recorded as an ADR:

```markdown
# ADR-0007 — MQ-9 cannot satisfy REQ-014 as written

Status: accepted   Supersedes: none   Conflict: CF-003

## Decision
MQ-9 provides a single composite gas response. REQ-014 (independent CO/CH4/LPG
concentrations) cannot be met with MQ-9 alone.

## Evidence
- research/sources/mq9-datasheet.pdf#p2 — "single sensitive layer"

## Options
A. Reframe REQ-014 as "combustible gas presence + relative level"  ← chosen
B. Add per-gas sensors (cost +, board area +)
C. Multi-sensor array + calibration model (scope +++)

## Consequences
- contracts/api/openapi.yaml: `gas_raw` + `gas_level`, not three channels
- T-014, T-021 respecified; T-009 evidence retained
```

**5. Consequences propagate mechanically.** The API contract changes → the generated client changes → the frontend's typecheck fails → a task is auto-created. Nobody had to *remember* that the web agent depended on the hardware decision. The contract graph knew.

That propagation is the whole point of the Brain, and it's the part that a chat transcript can never do.

---

## 6.5 Context distribution

The Brain is also the **context budget** mechanism. Agents get a **grant**, not the whole thing.

```jsonc
"context_grant": {
  "brain_paths": ["contracts/hardware/**", "requirements/functional.yaml#REQ-014"],
  "files": ["src/sensors/**"],
  "prior_results": ["T-003"],
  "budget_tokens": 120000
}
```

The runtime assembles context in **cache-friendly order** (Doc 4 §4.4): stable prefix first (system prompt → skills → durable Brain facts), volatile last (task, recent results). With prompt caching this is a large, real cost reduction across a run where many agents share the same project preamble.

If a grant exceeds budget, the runtime **summarizes the lowest-priority section and records that it did so** — it never silently truncates. An agent that made a decision on truncated context and didn't know is a debugging nightmare.

---

## 6.6 Learned memory, and its risk

`memory/learned.yaml` accumulates observations: "integration tests need Docker running," "this repo's linter is invoked via `make lint` not `npm run lint`."

This is high value and **genuinely dangerous**. A wrong learned fact is worse than no fact, because it is trusted and it persists. Controls:

- Every learned fact carries **evidence, a confidence score, and an observation count**.
- Facts **decay**: unconfirmed after N runs → demoted to `unverified`.
- Facts that lead to a **failed verification gate are automatically demoted** and flagged.
- Learned facts are **never** treated as constraints. They are hints. Contracts are constraints.
- Learned memory is **project-scoped by default**. Cross-project learning is opt-in — it's how a wrong assumption spreads.

---

## 6.7 Bootstrapping an existing repo

For an existing codebase, the Brain starts empty, and an empty Brain provides no value — the classic cold-start problem.

Bootstrap flow, run once with a cheap model:

```
1. Survey structure, languages, build system, entry points, test commands
2. Extract existing conventions from CLAUDE.md / AGENTS.md / README / CONTRIBUTING
3. Infer contracts from what already exists:
     OpenAPI specs → api contract
     DB migrations → data contract
     platformio.ini + pin #defines → hardware contract (flagged low confidence)
4. Mine git history for recurring decisions and constraints
5. Present as a DRAFT for human review — nothing enters the Brain unreviewed
```

Step 5 is essential. An auto-generated Brain full of confidently wrong inferences is worse than nothing, and it will be trusted by every agent thereafter.
