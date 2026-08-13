# AGENTS.md — context for any agent working in this repo

Read this before making changes. It is deliberately short; the detail is linked.

---

## What Orchestra is

An **agentic development environment** — same category as Claude Code, Codex, Antigravity, Cursor, OpenCode, Cline. It differs on three things:

1. **Model-agnostic** — any provider, cloud inference only
2. **Enforced completion** — the runtime rejects `status: complete` unless declared gates pass
3. **Decisions generate code** — violating an architectural decision is a build error, not a review comment

**One-line thesis:** *every spec-driven tool makes specs authoritative by convention; Orchestra makes interface decisions authoritative by construction.*

## Current phase

**M0 — setup and decisions.** No product code exists yet. That is intentional, not an oversight.

Do **not** start building the orchestrator, the UI, or a multi-agent system. See [ROADMAP.md](ROADMAP.md) for why the order is what it is, and [NEXT.md](NEXT.md) for the exact next task.

## Architecture in three layers

```
L3  ORCHESTRATOR · ARTIFACTS · IDE   ← ours, later, gated on evidence
L2  CONTRACTS + GATES                ← ours, NOW — this is the product
L1  AGENT RUNTIME                    ← reused: OpenCode (MIT)
```

We do **not** build L1. See [ADR-0001](docs/decisions/ADR-0001-reuse-opencode-as-runtime.md).

## Rules for contributions

1. **Reuse before build.** Any "let's write our own X" needs a written justification in an ADR.
2. **Deterministic before probabilistic.** If a check can be a validator function instead of a model call, it must be.
3. **Useful with one agent.** Nothing core may require orchestration to deliver value.
4. **Domain-agnostic core.** Every domain is a plugin — web, ML, database, embedded. No special cases in the engine.
5. **Evidence over enthusiasm.** Multi-agent execution is a hypothesis tested at M5, not a premise. See [ADR-0003](docs/decisions/ADR-0003-multi-agent-is-a-hypothesis.md).
6. **Correct, don't quietly patch.** If a decision turns out wrong, amend the ADR visibly — see ADR-0002's amendment for the pattern.

## Stack

TypeScript on Bun, strict mode. Zod + JSON Schema. Official MCP TypeScript SDK (**2026-07-28** spec — do **not** implement `sampling`, `roots`, or `logging`; they're deprecated). `bun test`. Domain plugins may shell out to other languages. See [ADR-0005](docs/decisions/ADR-0005-typescript-for-the-engine.md).

## Terminology — keep these distinct

| Term | Means |
| --- | --- |
| **Contract** | A versioned decision with **executable** constraints, usually generating an artifact. Lives in `.orchestra/contracts/`. |
| **Gate** | A command plus a pass condition. Applies to all work in any language. |
| **Generated artifact** | Code produced from a contract, marked `authoritative: true`, committed, and verified unmodified. |
| **Memory** | Non-authoritative project notes. **Not** a contract. Never enforced. |

If something can't be validated or generated from, it is **not** a contract — it belongs in docs or memory.

## Where things are

| Path | Contents |
| --- | --- |
| [ROADMAP.md](ROADMAP.md) | M0–M8, definitions of done |
| [NEXT.md](NEXT.md) | The exact next task |
| `docs/design/` | Contract spec, gate spec |
| `docs/decisions/` | ADRs — read before changing direction |
| `docs/research/` | Evidence base; **08 (critique)** and **10 (prior art)** matter most |
| `docs/setup/` | Verified local setup |

## Two things to know before proposing changes

- **[Document 08](docs/research/08-critique.md)** argues against parts of this project on evidence. It is not pessimism to be ignored; it's the reason the roadmap is ordered as it is.
- **[Document 10](docs/research/10-prior-art-sdd-landscape.md)** shows the spec-driven-development field is crowded — GitHub Spec Kit alone has ~111k stars. Check prior art before building anything; the answer may be to compose rather than compete.
