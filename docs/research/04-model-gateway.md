# Document 4 — Model Gateway

---

## 4.1 Goal and honest scope

The brief (§8, §25, §26) wants a provider-independent model layer with capability-based routing. I agree with the goal. I want to be precise about how much of it is a solved problem and how much is genuinely hard, because the two halves have very different risk profiles.

| Half | Difficulty | Who's solved it |
| --- | --- | --- |
| **Provider abstraction** — call 10 providers with one interface | **Solved.** Reuse, don't build. | LiteLLM (100+ providers, OpenAI-format, Rust core), Vercel AI SDK (used by OpenCode), models.dev (capability catalogue) **[2°]** |
| **Capability routing** — pick the *right* model per task | **Unsolved, and harder than it looks.** | Nobody. See §4.6 and Doc 8 §8.5. |

Building the first half from scratch would be months of adapter maintenance for zero differentiation. Building the second half is where the actual research question lives.

---

## 4.2 Architecture

```
                        ┌────────────────────────────┐
   task requirements ──►│      CAPABILITY ROUTER     │
   {requires, prefers,  │  filter → score → select   │
    budget, context_sz} │  + fallback chain          │
                        └─────────────┬──────────────┘
                                      │  ModelRef
                        ┌─────────────▼──────────────┐
                        │      NORMALIZED CLIENT      │
                        │ text · tools · stream ·     │
                        │ structured · reasoning ·    │
                        │ images · files · cache      │
                        └─────────────┬──────────────┘
                                      │
                        ┌─────────────▼──────────────┐
                        │   ADAPTER (reuse LiteLLM /  │
                        │   AI SDK where possible)    │
                        └─┬────┬────┬────┬────┬────┬─┘
                          │    │    │    │    │    │
                    Anthropic OpenAI Google NVIDIA OpenRouter local
                                                    (Ollama/vLLM)
```

**Decision: wrap, don't write.** Use an existing multi-provider library as the transport adapter layer, and put **our** value in the two boxes above it: the **capability model** and the **router**. If we later need a provider the library lacks, we write one adapter — not forty.

Which library depends on our language choice ([Document 7](07-original-roadmap.md) §7.6): **Vercel AI SDK** if TypeScript, **LiteLLM** if Python or as a sidecar gateway. LiteLLM's gateway mode is attractive independent of language because it gives us cost tracking, load balancing, and fallbacks as infrastructure rather than code. **[2°]**

---

## 4.3 The capability model

Routing by model *name* is a dead end — names change monthly and encode nothing useful. Route by **declared capability**, with the catalogue as data, not code.

```jsonc
{
  "id": "provider/model-name",
  "family": "...",
  "capabilities": {
    "tool_calling":       { "supported": true, "parallel": true, "strict_schema": true },
    "structured_output":  { "supported": true, "mode": "json_schema" },
    "vision":             { "supported": true, "formats": ["png","jpeg","webp"] },
    "pdf_native":         { "supported": false },
    "reasoning":          { "supported": true, "controllable": true, "levels": ["low","medium","high"] },
    "context_window":     { "input": 200000, "output": 64000 },
    "prompt_caching":     { "supported": true, "ttl_s": 3600, "min_tokens": 1024 },
    "streaming":          { "supported": true, "tool_calls_streamed": true },
    "computer_use":       { "supported": false }
  },
  "economics": { "usd_per_mtok_in": 0.0, "usd_per_mtok_out": 0.0, "cache_read_discount": 0.1 },
  "performance": { "p50_ttft_ms": 0, "p50_tok_per_s": 0, "observed": true },
  "reliability": { "success_rate_30d": 0.0, "observed": true }
}
```

Three things to note:

- **`observed: true` fields are measured by us, not declared by the vendor.** Latency and reliability are our telemetry. Vendors don't publish them honestly and they vary by region and time of day.
- **The catalogue is a data file, hot-reloadable.** A new model release is a config change, never a deploy. Seed it from **models.dev** (which is exactly this, and is what OpenCode uses **[2°]**) and layer our observations on top.
- **Capabilities are asserted, then verified.** See §4.5 — the vendor's claim about tool calling and the reality of tool calling under load are different objects.

---

## 4.4 Normalization: what actually breaks

The brief says "do NOT assume every provider supports every capability." Correct. From experience, here is where the abstraction leaks, ranked by how much pain each causes:

| Leak | Reality | Our handling |
| --- | --- | --- |
| **Tool-call schema dialects** | JSON Schema subsets differ; some reject `oneOf`/`$ref`/nested arrays; strict modes vary | Author tools in a **restricted common subset**; per-adapter down-conversion; validate at registration, fail loudly at startup not at runtime |
| **Parallel tool calls** | Some emit N calls per turn, some one | Router requires `parallel: true` for fan-out-heavy roles; runtime serializes transparently otherwise |
| **Reasoning tokens** | Different names, different visibility, sometimes billed but hidden | Normalize into `reasoning_summary` + `reasoning_tokens`; **never** put another provider's reasoning trace back into a different provider's context |
| **System prompt handling** | Separate field vs first message vs multiple blocks | Adapter concern; core always emits a structured system block |
| **Prompt caching** | Explicit breakpoints vs automatic vs none; different minimums | Cache-aware context assembly: **stable prefix first** (system, skills, Brain), volatile last. Big real-world cost lever. |
| **Structured output** | Native JSON schema vs "please output JSON" vs constrained decoding | Capability-gated: native if available, else tool-call trick, else parse-and-repair with retry cap |
| **Images/PDF** | Some take PDFs natively; others need pre-extraction | Router requires `pdf_native`, else routes through a **document pre-processing step** (Doc brief §21) |
| **Stop reasons & errors** | Wildly inconsistent | Normalize to a single taxonomy: `length`, `tool_use`, `stop`, `refusal`, `filter`, `error(retryable|fatal)` |
| **Token counting** | Different tokenizers; estimates drift | Per-provider counters; budget with a safety margin; never assume portability of a token count |

The last row deserves emphasis: **budgets computed against one provider's tokenizer are wrong for another.** If we route a task from one provider to another mid-retry, the budget must be recomputed, not carried over.

---

## 4.5 Capability verification (the part nobody does)

**Vendor capability claims are marketing until you test them.**

We ship a **conformance suite** that runs against every model in the catalogue and writes results back into it:

```
CONF-01  simple tool call with flat schema           → pass/fail
CONF-02  nested object + array-of-objects schema     → pass/fail
CONF-03  parallel tool calls in one turn             → pass/fail
CONF-04  tool call with 20 tools in context          → pass/fail + accuracy
CONF-05  structured output conforming to JSON Schema → pass/fail
CONF-06  long-context needle retrieval at 80% window → accuracy
CONF-07  multi-turn tool loop, 10 iterations         → completion rate
CONF-08  refusal behavior on benign shell commands   → false-positive rate
CONF-09  image input → structured extraction         → accuracy
CONF-10  100k-token prompt cache hit                 → hit rate + cost delta
```

Value: when a provider silently regresses — and they do — we find out from CI, not from an agent mysteriously failing at 2am. `CONF-04` in particular is the one that predicts real agent performance and that nobody publishes.

This suite is also the **only honest way** to build the router's scoring function. Without measured per-model conformance data, "capability routing" is just a config file full of guesses.

---

## 4.6 The router

```
Task requirements
  requires:   [tool_calling, long_context]      # hard filter
  prefers:    [low_cost]                        # soft score
  context:    ~150k tokens
  risk:       medium
  budget:     $0.50
        │
        ▼
1. FILTER    drop models failing any hard requirement or context size
             drop models whose conformance results fail a required gate
        │
        ▼
2. SCORE     w_q·quality + w_c·cost + w_l·latency + w_r·reliability
             (weights come from the task's role profile)
        │
        ▼
3. SELECT    best candidate; record the decision + reason
        │
        ▼
4. FALLBACK  ordered chain on failure:
             rate_limit → next provider, same tier
             context_overflow → larger-window model, or compact and retry
             repeated tool-schema failure → model with strict_schema
             refusal → different provider (do not retry same model)
             N consecutive failures → escalate tier, notify orchestrator
```

### `quality` is the hard part

Cost, latency, reliability, and context size are all **measurable numbers**. Quality is not. Options, in increasing order of honesty:

1. **Static tiers in config** (`strong` / `balanced` / `fast`) — a human's opinion, updated when models ship. Crude but transparent.
2. **Task-family win rates from our own verification gates** — the real prize. Every task already runs verification gates and reports cost. That is a labeled outcome. Over time we can learn: *for role=firmware tasks, model X passes gates first-try 71% of the time at $0.31; model Y passes 68% at $0.09* → route to Y.
3. **Public benchmarks** — weakly correlated with our workload. Use as a prior at best.

**Ship (1), instrument for (2) from day one.** The data is free — we're generating it anyway. But do not claim (2) works until we have enough samples per (role × model) cell to beat noise, which will take longer than anyone expects. See Doc 8 §8.5.

---

## 4.7 Routing profiles (initial, hand-tuned)

| Role | requires | prefers | Rationale |
| --- | --- | --- | --- |
| Orchestrator / decompose | `reasoning`, `long_context`, `structured_output` | quality | Worst place to economize — errors multiply downstream |
| Repo survey / classify | `fast_response` | cost | High volume, low stakes, easily verified |
| Coding (specialist) | `tool_calling`, `coding`, `long_context` | balanced | The bulk of spend |
| Code review | `reasoning` | **different provider than the author** | Independence is the entire point |
| Document/PDF | `vision` or `pdf_native`, `long_context` | quality | Extraction errors poison everything downstream |
| Browser/UI | `vision`, `tool_calling` | balanced | Screenshot reasoning |
| Conflict adjudication | `reasoning`, `long_context` | quality | Rare, high stakes |
| Commit messages, renames | `fast_response` | cost | Trivially verifiable |

**The code-review rule is a genuine architectural insight and it's cheap.** A model reviewing its own output shares its own blind spots. Requiring a *different provider* for review is a one-line routing constraint that buys real independence. None of the systems I researched does this by default.

---

## 4.8 What could go wrong

Honest list:

- **Router overhead exceeds its savings.** If choosing a model costs a model call, we've lost. **Mitigation: the router is pure code — filters and arithmetic over a catalogue. No LLM in the routing path. Ever.**
- **Capability drift.** Providers change behavior silently. Mitigation: conformance suite in CI, nightly.
- **Cross-provider context incompatibility.** Reasoning traces, cache breakpoints, and tool-call formats don't transfer. Mitigation: on model switch mid-task, **rebuild context from the Brain** rather than replaying a foreign transcript.
- **The cheap model is cheap for a reason.** Routing a subtle task to a fast model produces work that passes shallow gates and fails deep ones — the most expensive kind of failure, because you pay for it twice plus debugging. Mitigation: **gate strength must scale with model weakness**, and escalate on gate failure rather than retrying the same tier.
- **Lock-in through the back door.** If we depend on one provider's prompt-caching semantics or reasoning controls for acceptable economics, we are not provider-independent no matter what the adapter layer says. Mitigation: the conformance suite must include a **"can this model do the job at all"** gate per role, and we should periodically run the full pipeline on a second provider end to end.
