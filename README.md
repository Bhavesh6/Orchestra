# Orchestra

**A contract-driven, verification-gated AI development environment.**

> An AI development environment where the agent has to prove it's done, and where architectural decisions are enforced by the compiler rather than remembered by a model.

**Status:** Research complete. Implementation starting at M0. No product code yet.

---

## The problem

AI coding agents fail in two ways that nothing currently catches:

1. **They forget decisions.** An agent assigns `GPIO34` to a sensor, then an hour later writes `digitalWrite(35, HIGH)`. Or the backend renames a field and the frontend still expects the old one. No linter, no test, and no review catches it until something breaks at runtime — or, in embedded work, until hardware is damaged.

2. **They say "done" without proof.** "I improved the model." "The refactor is complete." These are claims, not facts, and today nothing forces the difference.

Orchestra addresses both mechanically rather than by asking a model to be careful.

## The two mechanisms

**Contracts generate authoritative artifacts.** A pin map generates `pins.h`. An OpenAPI spec generates a typed client. A data schema generates validators. Once generated, violating the decision is a **compile error**, not a code-review finding.

```
  pinmap.yaml  ──generates──►  pins.h  ──►  compiler rejects wrong pin
  openapi.yaml ──generates──►  client.ts ──►  typecheck rejects wrong field
  schema.yaml  ──generates──►  validators ──►  pipeline rejects wrong shape
```

**Gates enforce completion.** An agent's result is rejected by the runtime unless every declared gate passes. Not requested in a prompt — enforced in code.

```yaml
gates:
  build: { run: "npm run build",  pass_if: exit_zero }
  types: { run: "tsc --noEmit",   pass_if: exit_zero }
  eval:  { run: "python eval.py", pass_if: "metrics.auc >= 0.82" }
```

Both work with a **single agent**. Neither requires orchestration. That's deliberate.

## Domains

Orchestra is general-purpose. Contract kinds are plugins, not hardcoded:

| Domain | Contract | Generates | Gates |
| --- | --- | --- | --- |
| **Web/API** | OpenAPI | typed client, server stubs, contract tests | `tsc`, lint, vitest, Playwright |
| **ML** | data/feature schema, metric thresholds | validators, inference wrapper | pytest, eval-beats-baseline, leakage check |
| **Database** | schema | migrations, models, types | migration lint, FK integrity |
| **Embedded** | pin map, power budget | `pins.h` | compile, pin audit, flash smoke test |

## Architecture

Three layers. Only two of them are ours.

```
┌──────────────────────────────────────────────────┐
│  L3  ORCHESTRATOR + ARTIFACTS       ← ours, later│
│      task graph, agent manager, review surface   │
├──────────────────────────────────────────────────┤
│  L2  CONTRACTS + GATES              ← ours, now  │
│      validators, generators, gate runner, MCP    │
├──────────────────────────────────────────────────┤
│  L1  AGENT RUNTIME                  ← reused     │
│      OpenCode (MIT): loop, tools, MCP, LSP,      │
│      sessions, permissions, 75+ providers        │
└──────────────────────────────────────────────────┘
                        │
              MODEL GATEWAY (cloud only)
     NVIDIA Build · Anthropic · OpenAI · Google · OpenRouter
```

**L1 is reused, not rebuilt.** Google forked VS Code to build Antigravity. Cursor and Windsurf are VS Code forks. Nobody builds this from zero, and neither will we — the agent loop is a weekend, but a reliable edit tool is months.

**L2 ships as an MCP server**, so it works inside OpenCode, Claude Code, Cline, Codex and Cursor on day one.

**L3 is a hypothesis**, not a premise. See [the critique](docs/research/08-critique.md).

## Documentation

| | |
| --- | --- |
| **[ROADMAP.md](ROADMAP.md)** | **How we're building it. Start here.** |
| [docs/design/](docs/design/) | Contract spec, gate spec, architecture |
| [docs/decisions/](docs/decisions/) | ADRs — what we decided and why |
| [docs/research/](docs/research/) | The competitive study and evidence base |

**Two research documents matter most:**
- [09 — Answers & Course Correction](docs/research/09-answers-and-correction.md) — the current position
- [08 — Critique](docs/research/08-critique.md) — where this idea is wrong

## Principles

1. **Reuse before build.** Every "we'll write our own X" needs a written justification.
2. **Evidence over enthusiasm.** Multi-agent execution is gated on measurement, not assumed.
3. **Deterministic before probabilistic.** If a check can be a validator instead of a model call, it is.
4. **Useful with one agent.** Nothing core may require orchestration to be valuable.
5. **Cloud inference only.** No local models — the hardware isn't there and doesn't need to be.
6. **Prove it, don't claim it.** Applies to the agents, and to us.

## License

MIT
