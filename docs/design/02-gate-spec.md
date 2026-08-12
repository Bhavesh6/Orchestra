# Gate Specification (draft v0.1)

**Status:** Draft. Will change during M2.

---

## The rule that makes this worth building

> **An agent result with `status: "complete"` is rejected by the runtime unless every required gate reports `pass`.**

Enforced in code, not requested in a prompt. This is the difference between an agent that says it's done and an agent that is done.

Gates are the half of Orchestra that applies to **100% of work** — ML, web, embedded, anything. Contracts only apply at interfaces; gates apply everywhere.

---

## `gates.yaml`

```yaml
gates:
  build:
    run: "npm run build"
    pass_if: exit_zero
    timeout_s: 300
    artifacts: [build.log]

  types:
    run: "tsc --noEmit"
    pass_if: exit_zero

  test:
    run: "pytest -q --json-report"
    pass_if: exit_zero
    artifacts: [.report.json]

  eval:
    run: "python eval.py --out metrics.json"
    parse: { file: metrics.json, format: json }
    pass_if: "metrics.auc >= 0.82 and metrics.f1 >= 0.71"
    artifacts: [metrics.json]

  contracts:
    run: "orchestra validate"
    pass_if: exit_zero

  drift:
    run: "orchestra generate --check"
    pass_if: exit_zero
    description: "generated files match their contracts"

  flash_smoke:
    run: "orchestra serial capture --seconds 10"
    parse: { file: serial.log, format: lines }
    pass_if: "match('TEMP=[0-9.]+') and match('HUM=[0-9.]+')"
    requires_lease: esp32-devkit-v1      # exclusive hardware
    approval: always                      # never auto-run

profiles:
  quick:  [types, contracts]
  full:   [build, types, test, contracts, drift]
  ml:     [test, eval, contracts]
  hw:     [build, contracts, drift, flash_smoke]
```

---

## `pass_if` grammar

Deliberately small. A gate DSL that grows into a programming language is a failure.

| Form | Meaning |
| --- | --- |
| `exit_zero` | process exited 0 |
| `exit_code == N` | specific code |
| `metrics.X >= N` | parsed metric comparison |
| `match('regex')` | regex found in parsed output |
| `not match('regex')` | regex absent — good for "no warnings" |
| `a and b`, `a or b` | conjunction |

Anything more complex belongs in a script that the gate invokes and that exits 0 or 1. **The gate runner is a harness, not an interpreter.**

---

## Result schema

```jsonc
{
  "gate": "eval",
  "result": "pass",              // pass | fail | error | skipped | blocked
  "duration_s": 41.2,
  "exit_code": 0,
  "expression": "metrics.auc >= 0.82 and metrics.f1 >= 0.71",
  "observed": { "auc": 0.843, "f1": 0.72 },
  "artifacts": [".orchestra/runs/T-014/metrics.json"],
  "output_tail": "…last 50 lines…"
}
```

`observed` is what makes a failure actionable. `"eval failed"` sends an agent guessing; `"auc 0.79 < 0.82"` sends it to the right place.

Distinguish `fail` (gate ran, criterion not met) from `error` (gate couldn't run — missing dep, timeout). They demand different responses: `fail` → fix the code, `error` → fix the environment. Conflating them is a common way agents waste turns.

---

## Failures become tasks, not retries

A failed gate produces a **new task with diagnostic context**, rather than "try again":

```jsonc
{
  "from_gate": "types",
  "task": "Fix 3 type errors introduced in src/api/client.ts",
  "context": { "errors": ["…"], "likely_cause": "contract API-V2 regenerated client" },
  "attempt": 1
}
```

With a **retry cap**. After N consecutive failures on the same gate, halt and surface to the human — Cline's mistake-limit pattern. An agent looping on a gate it cannot pass is the most expensive failure mode available, and it's trivially preventable.

---

## Hardware gates

Two extra properties, both non-negotiable:

- **`requires_lease`** — exclusive mutex on a physical device. A board is a mutex; any design that ignores this breaks the moment two agents run.
- **`approval: always`** — flashing is never auto-approved regardless of permission mode.

Plus a pin audit *before* flash (`platformio-mcp` already exposes `agent-safe-pin-audit`), and known-good firmware rollback.

---

## Gate strength vs model strength

A principle worth encoding early:

> **Weaker or cheaper models require stronger gates.**

A cheap model routed to a subtle task produces work that passes shallow checks and fails deep ones — the most expensive failure, because you pay for it twice plus debugging. So the router and the gate profile are linked: if we route down a tier, we escalate the profile.

---

## Open questions for M2

1. **Gate caching** — skip a gate whose inputs are unchanged? Big speedup, real correctness risk. Probably content-hash the file scope, opt-in.
2. **Parallel gate execution** — safe when gates don't share resources. Needs a resource declaration.
3. **Flaky gate handling** — retry-once-then-report, or fail immediately? Leaning fail-immediately with an explicit `flaky: true` opt-in, because auto-retry hides real nondeterminism.
4. **Where do gate definitions live** for a repo that already has CI? Ideally we *read* existing CI config rather than duplicating it.
