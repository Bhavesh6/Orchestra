# NEXT — the exact task to pick up

**Updated:** 2026-08-13 · **Phase:** M0 → M1

If you're an agent working in this repo, read [AGENTS.md](AGENTS.md) first, then this.

---

## Status

| | |
| --- | --- |
| ✅ Done | Research (docs 01–10), architecture, ADRs 0001–0005 + 0007, roadmap |
| ✅ Done | Language decided: TypeScript on Bun ([ADR-0005](docs/decisions/ADR-0005-typescript-for-the-engine.md)) |
| ✅ Done | Model verified: `z-ai/glm-5.2` on NVIDIA Build, free ([setup guide](docs/setup/opencode-nvidia.md)) |
| ⬜ Doing | OpenCode running locally, tool calling verified |
| ⬜ Next | **M1 — contract engine, first slice** |

**No product code exists yet.** That's deliberate.

---

## The next task: M1 first slice

Build the smallest thing that proves the thesis end to end. **One contract kind, one validator, one generator, one gate.** Against a fixture, not a real project — a fixture can't hide a bug behind existing tooling, and it removes the "which project" blocker.

### Target structure

```
packages/orchestra/
  package.json
  tsconfig.json
  src/
    contract/
      types.ts          Contract envelope types
      load.ts           YAML → validated Contract (Zod + JSON Schema)
      registry.ts       plugin registry for contract kinds
    kinds/
      openapi.ts        first kind: validator + generator
    generate/
      run.ts            generate files, mark authoritative, detect drift
    gate/
      run.ts            spawn process, parse output, evaluate pass_if
      expr.ts           the tiny pass_if grammar
    result.ts           structured result schema
    cli.ts              validate · generate · gate · status
  test/
examples/demo-api/
  .orchestra/
    contracts/api/openapi.yaml
    gates.yaml
  src/                  a small TS consumer of the generated client
```

### Build order

1. **`contract/types.ts` + `load.ts`** — parse the envelope from [the contract spec](docs/design/01-contract-spec.md). Reject prose constraints at load time: `constraints[].rule` must name a registered function.
2. **`kinds/openapi.ts`** — one validator (`no_breaking_changes_vs_accepted`) and one generator (OpenAPI → typed TS client).
3. **`generate/run.ts`** — write generated files, and implement `--check` for drift: a hand-edited generated file is an **error**, not a merge.
4. **`gate/run.ts` + `expr.ts`** — spawn, capture, evaluate. Support `exit_zero` first; add metric comparison after. Keep the grammar tiny — see [the gate spec](docs/design/02-gate-spec.md).
5. **`result.ts`** — and enforce the rule that makes this worth building: **reject `status: "complete"` when any required gate did not pass.**
6. **`cli.ts`** — thin wrapper. No framework.

### Definition of done — the thesis test

In `examples/demo-api/`:

```bash
orchestra generate          # openapi.yaml → src/generated/client.ts
# now change a field name in openapi.yaml
orchestra generate
orchestra gate types        # tsc --noEmit
```

> **The typecheck must FAIL, and the failure must name the contract that changed.**

That is the entire thesis, proven with no orchestrator, no MCP server, and no UI.

Second check, same slice: hand-edit `src/generated/client.ts`, run `orchestra generate --check`, and it must fail on drift.

### Explicitly not in this slice

- No MCP server (that's M3)
- No second contract kind — `openapi` must work end to end first
- No orchestrator, no agents, no UI
- No `pinmap` yet; the general case has to work before the domain case

---

## After M1

**M2** gate runner depth → **M3** MCP server → **M4** four weeks of real use in ≥2 domains → **M5** decision gate on whether multi-agent is worth building at all.

---

## Open items

| Item | Owner |
| --- | --- |
| Verify tool calling in OpenCode (the 3-point test in the setup guide) | you |
| Delete stale branch on the old `try` repo | you |
| **ADR-0006** — extend vs fork OpenCode. **Not a blocker for M1** — the engine is standalone | either |
| Evaluate GitHub Spec Kit hands-on; adopt rather than rebuild if it fits | either |
| Check whether Tessl's Framework has reached GA | either |
| Pick 2–3 dogfood projects across different domains for M4 | you |

## Watch out for

- **Prior art is dense.** GitHub Spec Kit has ~111k stars; Tessl already generates code from specs. Read [Document 10](docs/research/10-prior-art-sdd-landscape.md) before building anything, and check whether the right move is to compose rather than compete.
- **Scope creep toward the orchestrator.** It's the exciting part and it's gated on M5 evidence for good reason — see [Document 08](docs/research/08-critique.md).
- **40 requests/minute** on the NVIDIA free tier. Agentic loops are request-heavy.
