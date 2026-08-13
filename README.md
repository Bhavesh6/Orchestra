# Orchestra

**An agentic development environment — model-agnostic, verification-first.**

Same category as Claude Code, Codex, Antigravity, Cursor, OpenCode and Cline: an environment where agents read your project, write code, run commands, use tools, and work across a whole codebase. Orchestra differs on three things.

| | Orchestra |
| --- | --- |
| **Model** | Any provider. Nothing is built on one vendor. Cloud inference; the machine runs tools. |
| **Completion** | An agent can't say "done" — the runtime rejects it unless the gates it declared actually pass. |
| **Decisions** | Architectural decisions generate code, so violating one is a build error rather than a review comment. |

**Status:** Research complete. Implementation starting at M0. No product code yet.

---

## What it's for

General software work — web apps, APIs, backends, ML pipelines, and yes, embedded firmware. Nothing in the core is domain-specific.

The two problems it targets are ones every agentic environment has, in every language:

**1. Agents forget decisions.** The backend renames a field and the frontend still expects the old one. A schema changes and three call sites don't. Nothing catches it — not the linter, not the tests, not review — until it breaks at runtime.

**2. Agents say "done" without proof.** *"Refactor complete."* *"I improved the model."* These are claims. Today nothing forces them to be facts.

## The two mechanisms

**Gates enforce completion.** Declarative, and they apply to **everything** — every language, every domain, every task.

```yaml
gates:
  build: { run: "npm run build",  pass_if: exit_zero }
  types: { run: "tsc --noEmit",   pass_if: exit_zero }
  test:  { run: "pytest -q",      pass_if: exit_zero }
  eval:  { run: "python eval.py", pass_if: "metrics.auc >= 0.82" }
```

A result marked `complete` while a gate fails is rejected by the runtime — not asked for in a prompt.

**Contracts generate authoritative artifacts.** At interfaces — where coordination actually breaks — a decision generates the code that enforces it.

```
  openapi.yaml ──generates──►  client.ts    ──►  typecheck rejects wrong field
  schema.sql   ──generates──►  models/types ──►  compile rejects wrong column
  features.yaml──generates──►  validators   ──►  pipeline rejects wrong shape
  pinmap.yaml  ──generates──►  pins.h       ──►  compile rejects wrong pin
```

Gates cover all your work. Contracts cover the interfaces — roughly 20–30%, but it's the part where silent breakage lives.

Both work with **a single agent**. Neither needs orchestration. That's deliberate.

## Domains

Contract kinds are plugins. Adding a domain is a plugin, never a core change.

| Domain | Contract | Generates | Gates |
| --- | --- | --- | --- |
| **Web / API** | OpenAPI | typed client, server stubs, contract tests | tsc, lint, vitest, Playwright |
| **Backend / data** | SQL schema | migrations, models, types | migration lint, FK integrity |
| **ML** | feature schema, metric thresholds | validators, inference wrapper | pytest, eval-beats-baseline, leakage check |
| **Embedded** | pin map, power budget | `pins.h` | compile, pin audit, flash smoke |

## Architecture

Three layers. Only two are ours.

```
┌──────────────────────────────────────────────────┐
│  L3  ORCHESTRATOR · ARTIFACTS · IDE   ← ours,    │
│      the Antigravity-shaped surface     later    │
├──────────────────────────────────────────────────┤
│  L2  CONTRACTS + GATES                ← ours,    │
│      validators, generators, MCP server   now    │
├──────────────────────────────────────────────────┤
│  L1  AGENT RUNTIME                    ← reused   │
│      OpenCode (MIT): loop, tools, MCP, LSP,      │
│      sessions, permissions, 75+ providers        │
└──────────────────────────────────────────────────┘
                        │
              MODEL GATEWAY (cloud)
     NVIDIA Build · Anthropic · OpenAI · Google · OpenRouter
```

**Why L1 is reused.** Google forked VS Code to build Antigravity — code analysis suggests it's actually a fork of Windsurf, itself a VS Code fork. Cursor and Windsurf are forks too. A company with a reported $2.4B did not start from zero, and neither will we. The agent loop is a weekend of work; a *reliable edit tool* is months, and that's the part that decides whether the environment is usable.

**Why L2 ships as an MCP server.** It works inside OpenCode, Claude Code, Cline, Codex and Cursor on day one. It also means L1 is swappable — if OpenCode stops being a good host, we replace it without touching L2.

**Why L3 is last.** An agent manager is impressive because it sits on a mature runtime. Built first, on nothing, it's a dashboard showing agents fail in parallel. See [the critique](docs/research/08-critique.md).

## Interface

Orchestra is an application you use, not a library. But you don't wait until the end to get one — because L1 already has a TUI, and L2 surfaces inside agents you already run.

| Stage | What you actually use | Built by |
| --- | --- | --- |
| **Now (M0)** | OpenCode's TUI, pointed at NVIDIA Build | inherited |
| **M3** | contracts + gates visible *inside* OpenCode, Claude Code, Cline, Codex, Cursor — via MCP | us, ~1 week |
| **M7** | **Agent manager** — the Antigravity-shaped surface: artifacts, plans, diffs, gate results, cost | us |
| **M8** | Orchestra TUI, then a VS Code extension | us |

The reason this ordering works is L1's shape: OpenCode is **client–server**, and its TUI is only a client talking HTTP + SSE. Any interface we build is another client against the same server. That's why the UI can come last without the UI being an afterthought — the API it needs already exists and is already proven by a real client.

**On the app form factor.** Cursor, Windsurf and Antigravity are all VS Code forks — Antigravity's codebase reportedly still carries references to Windsurf's Cascade agent. Forking is the established path, but it's also a permanent maintenance tax: you inherit every upstream release forever. They had teams for that.

Our order is **extension before fork**: a VS Code extension first, and a fork only if the extension API genuinely blocks something we need. See [ADR-0007](docs/decisions/ADR-0007-interface-strategy.md).

## Documentation

| | |
| --- | --- |
| **[NEXT.md](NEXT.md)** | **The exact next task. Start here.** |
| [AGENTS.md](AGENTS.md) | Context for any agent working in this repo |
| [ROADMAP.md](ROADMAP.md) | M0–M8 with checkable definitions of done |
| [docs/setup/](docs/setup/) | Verified local setup — OpenCode + NVIDIA Build |
| [docs/design/](docs/design/) | Contract spec, gate spec |
| [docs/decisions/](docs/decisions/) | ADRs — what we decided and why |
| [docs/research/](docs/research/) | Competitive study of the six systems, and the evidence base |

Two research docs matter most: [09 — Answers & Course Correction](docs/research/09-answers-and-correction.md) (current position) and [08 — Critique](docs/research/08-critique.md) (where this idea is wrong).

## Principles

1. **Reuse before build.** Every "we'll write our own X" needs a written justification.
2. **Evidence over enthusiasm.** Multi-agent execution is gated on measurement, not assumed.
3. **Deterministic before probabilistic.** If a check can be a validator instead of a model call, it is.
4. **Useful with one agent.** Nothing core may require orchestration to be valuable.
5. **Domain-agnostic core.** Every domain is a plugin. No exceptions.
6. **Cloud inference only.** The machine runs tools, not models.
7. **Prove it, don't claim it.** Applies to the agents, and to us.

## License

MIT
