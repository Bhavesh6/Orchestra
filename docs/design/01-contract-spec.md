# Contract Specification (draft v0.1)

**Status:** Draft. Will change during M1 as we hit reality.

---

## What a contract is

A contract is a **versioned, machine-readable statement of a decision** that:

1. can be **validated** by deterministic code (no model call), and
2. ideally **generates** an artifact that makes violating the decision a build error.

A contract is *not* documentation, memory, or a note-to-self. If it can't be validated or generated from, it belongs in `docs/` or the project memory — not in `contracts/`.

> **The test:** if an agent violates this contract, does something fail automatically? If no, it isn't a contract.

---

## Layout

```
.orchestra/
├── orchestra.yaml           # project config: kinds enabled, paths
├── contracts/
│   ├── api/openapi.yaml
│   ├── data/features.yaml
│   ├── db/schema.yaml
│   └── hardware/pinmap.yaml
├── generated/               # optional staging; targets usually live in-tree
├── gates.yaml
└── decisions/               # ADRs for contract changes
```

---

## Common envelope

Every contract carries the same header regardless of kind:

```yaml
contract: HW-PINMAP           # stable ID
kind: pinmap                  # selects the plugin
version: 3
status: accepted              # draft | proposed | accepted | superseded
owner: hardware               # role responsible
supersedes: 2

evidence:
  - ref: docs/research/esp32-devkit-v1.md#adc2-wifi
    quote: "ADC2 is unavailable while WiFi is active"

constraints:                  # executable rule names, never prose
  - { id: C-01, rule: unique_pins }
  - { id: C-02, rule: input_only_pins, params: { pins: [34,35,36,39] } }
  - { id: C-03, rule: adc2_unusable_with_wifi }

generates:
  - target: include/pins.h
    template: pins_h
    authoritative: true       # hand-edits are an error, not a merge

spec: {}                      # kind-specific payload below
```

### Field notes

**`constraints[].rule`** names a function the plugin exposes. It must be deterministic, fast, and free. `unique_pins` is code. `"pins should not conflict"` is prose and is rejected at load time.

**`generates[].authoritative: true`** is the load-bearing flag. It means:
- the generated file **is committed to git** (so the build works without Orchestra installed)
- a gate verifies it **matches what the generator would produce**
- a hand-edit becomes a **failing gate**, not a silent divergence

This is the answer to "who maintains this?" — generated artifacts cannot rot without breaking the build.

**`status`** gates authority. Only `accepted` contracts are enforced. `proposed` ones are validated and reported but don't fail the build, which is how an agent suggests a change without breaking everyone.

---

## Kinds

Kinds are plugins. The general ones come first — they're the ones that prove the core is domain-agnostic. If a mechanism only works for one domain, it isn't a platform.

### `openapi` (web/API)

```yaml
kind: openapi
spec:
  source: api/openapi.yaml      # the real spec; contract wraps it
```

Rules: `schema_lint`, `no_breaking_changes_vs_accepted`, `all_endpoints_have_responses`, `no_untyped_any`.
Generates: typed client, server stubs, contract tests.

**This is where web work gets its value:** backend renames a field → generated client changes → frontend typecheck fails. The classic silent break becomes a compile error.

### `data_schema` (ML)

```yaml
kind: data_schema
spec:
  dataset: training_features
  columns:
    - { name: temp_c,   dtype: float32, range: [-40, 85], nullable: false }
    - { name: humidity, dtype: float32, range: [0, 100],  nullable: false }
    - { name: label,    dtype: int8,    values: [0, 1] }
  invariants:
    - no_train_test_overlap_on: [device_id, timestamp]
```

Rules: `dtype_conformance`, `range_conformance`, `null_policy`, `no_leakage`.
Generates: a pandera/pydantic validator invoked at pipeline boundaries.

`no_leakage` is worth calling out — train/test contamination is invisible, catastrophic, and exactly the kind of thing an agent introduces while "improving the pipeline."

### `sql_schema` (database)

Rules: `fk_integrity`, `migration_ordering`, `no_destructive_without_flag`.
Generates: migrations, models, types.

### `metric_threshold` (ML)

```yaml
kind: metric_threshold
spec:
  baseline: { auc: 0.82, f1: 0.71 }
  holdout: data/holdout.parquet
  tolerance: 0.005
```

Doesn't generate code — it generates a **gate**. This is how "I improved the model" stops being an unverifiable claim.

### `pinmap` (embedded)

One domain plugin among several, implemented after the general kinds work. Included because it's the sharpest illustration of the mechanism: the violation is otherwise invisible until hardware misbehaves.

```yaml
kind: pinmap
spec:
  board: esp32-devkit-v1
  assignments:
    - { signal: SENSOR_ADC, pin: GPIO34, mode: analog_in, adc: ADC1 }
    - { signal: DHT22_DATA, pin: GPIO4,  mode: digital_io, pullup: external_10k }
  power:
    supply_ma: 500
    components:
      - { name: DHT22, typical_ma: 1.5, peak_ma: 2.5 }
```

Rules: `unique_pins`, `input_only_pins`, `adc2_unusable_with_wifi`, `strapping_pin_warning`, `i2c_bus_consistency`, `power_budget_within_supply`.
Generates: `pins.h`, optionally a `platformio.ini` fragment.

---

## Plugin interface (sketch)

```ts
interface ContractKind {
  name: string;
  schema: JSONSchema;                          // validates the spec block
  rules: Record<string, (c: Contract, ctx: RepoContext) => Violation[]>;
  generators: Record<string, (c: Contract) => GeneratedFile[]>;
}
```

Adding a domain is a plugin registration. It must never require touching the core. If we find ourselves special-casing a kind in the engine, the interface is wrong.

---

## Violation output

Structured, so agents and humans read the same thing:

```jsonc
{
  "contract": "HW-PINMAP@3",
  "rule": "input_only_pins",
  "severity": "error",
  "location": { "file": "src/main.c", "line": 42 },
  "message": "GPIO35 is input-only on esp32-devkit-v1; digitalWrite() is invalid",
  "evidence": "docs/research/esp32-devkit-v1.md#input-only",
  "fix_hint": "Use a GPIO with output capability, or update HW-PINMAP if the board changed"
}
```

`fix_hint` matters more than it looks. An agent that gets "violation: C-02" retries blindly. One that gets the sentence above fixes it in a single turn.

---

## Open questions for M1

1. **Where do generated files live** — in-tree (committed, build works without Orchestra) or in `.orchestra/generated/`? Leaning in-tree for exactly that reason.
2. **How does a contract get proposed by an agent** and approved by a human without a full workflow engine? Probably: agent writes `status: proposed`, human flips to `accepted`.
3. **Do we need contract composition** (one contract referencing another)? Defer until something demands it.
4. **How much of OpenAPI do we validate ourselves** vs shelling out to existing linters? Prefer shelling out.
