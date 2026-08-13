# ADR-0005 — TypeScript for the Orchestra engine

**Status:** accepted · **Date:** 2026-08-12

## Context

M1–M3 produce roughly 3–5k lines across five pieces:

| Piece | Responsibility |
| --- | --- |
| Contract engine | load YAML, validate against JSON Schema, run the plugin registry |
| Validators | deterministic functions — `unique_pins`, `fk_integrity`, `no_breaking_changes` |
| Generators | templates → `pins.h`, typed API clients, data validators |
| Gate runner | spawn processes, parse output, evaluate `pass_if`, return structured results |
| `orchestra-mcp` | expose all of the above to any MCP-capable agent |

### What this choice does *not* determine

Worth stating explicitly, because it's the assumption most likely to mislead:

**The engine's language is unrelated to the languages of the projects Orchestra manages.** A gate is `run: "pytest -q"` — a subprocess. Orchestra shells out; it never needs to *be* the language it checks. A TypeScript engine validates a Python ML pipeline, a Rust service, or ESP32 firmware equally well.

It also doesn't affect the model providers (HTTP), the contract files (YAML), or interoperability with OpenCode (MCP is language-neutral).

So the ML/web/embedded spread in our work is **not** an argument for either language.

### Trade-off

**TypeScript** — most mature official MCP SDK; matches OpenCode; clean distribution (`npx orchestra`, or a single binary via `bun build --compile`); the heaviest generator target is TypeScript itself. Costs: ML plugins can't use pandera/pydantic in-process.

**Python** — native pandera/pydantic for ML contracts; excellent YAML tooling. Costs: messy CLI distribution (pip, venvs, version skew); a permanent two-language split if we extend OpenCode; clunkier when generating TypeScript artifacts.

## Decision

**TypeScript**, on **Bun**, with domain plugins free to shell out to other languages.

Ranked reasoning:

1. **[ADR-0006](ADR-0006-extend-vs-fork-opencode.md) is still open.** If we extend OpenCode rather than fork it, TypeScript stops being a preference and becomes a requirement. Choosing Python now risks either a rewrite or a permanent bilingual codebase.
2. **M3 is an MCP server** — TypeScript's strongest ground, and the SDK we'd depend on most.
3. **Distribution.** `npx orchestra` versus explaining virtualenvs is a larger ongoing cost than it appears for a CLI other people install.
4. **The ML gap closes cheaply.** The `data_schema` plugin invokes a small Python script as a subprocess — exactly what the plugin interface exists for, and precisely how gates already work.

### Initial stack

| Concern | Choice |
| --- | --- |
| Runtime | Bun (Node-compatible; `bun build --compile` for single-binary distribution) |
| Language | TypeScript, strict mode |
| Validation | Zod + JSON Schema |
| CLI | a thin argument parser — no heavyweight framework |
| Templating | plain template functions first; a library only if they get unwieldy |
| MCP | official TypeScript SDK, 2026-07-28 spec |
| Tests | `bun test` |

Nothing above is load-bearing enough to need its own ADR. Revisit if any becomes a constraint.

## Consequences

**Good**
- One language if ADR-0006 lands on "extend."
- Best-supported MCP path for the piece that makes L2 useful everywhere.
- Trivial install story.
- Generating TypeScript artifacts (the most common target) is native.

**Bad**
- ML contract validation crosses a process boundary. Slower per call, and an extra dependency for users of that plugin. Acceptable — validation runs at gate time, not in a hot loop.
- Python is likely the stronger language for this project's author. **This is the real cost**, and it was weighed: for a solo project, author velocity usually beats architectural tidiness. It's outweighed here only because ADR-0006 could make TypeScript mandatory anyway.

**Reversible?** Partially. The contract format, gate format, and MCP interface are all language-neutral, so a rewrite would preserve the design and the specs. But it would be a rewrite. Revisit only if ADR-0006 lands on "fork" *and* TypeScript proves a genuine drag.
