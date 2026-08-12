# ADR-0004 — Cloud inference only, no local models

**Status:** accepted · **Date:** 2026-08-12

## Context

Local hardware cannot run models at the quality and speed required. The alternative — cloud inference with local tooling — turns out to be strictly better, not a compromise.

**NVIDIA Build** provides 100+ models on an OpenAI-compatible endpoint (`https://integrate.api.nvidia.com/v1`), free, no credit card. Agentic coding models include Qwen3 Coder 480B (purpose-built for agentic coding), GLM-5.2, DeepSeek V4 Flash (1M context), MiniMax M2.7. Tool calling is confirmed working across the major models. OpenCode supports it directly via `@ai-sdk/openai-compatible`.

Free tier limits: **1,000 credits on signup (up to 5,000 on request), 40 requests/minute.**

## Decision

**Cloud inference only.** The local machine runs editor, terminal, agent runtime, compiler, browser, git, and toolchains. Model inference is remote.

Local model support (Ollama, vLLM) stays in the adapter list as a **last-resort fallback**, not a supported path.

Consequences for the gateway: its primary job is **continuity, not intelligence**.

```
429 rate_limit     → backoff, then next provider
5xx / timeout      → next provider, same tier
credits_exhausted  → next provider
context_overflow   → larger window, or compact and retry
refusal            → different provider (never retry the same model)
```

Plus: never carry context across a provider switch — rebuild from contracts and project state, since reasoning traces, cache breakpoints and tool formats don't transfer. Recompute budgets too; token counts aren't portable across tokenizers.

## Consequences

**Good**
- Frontier-class models on modest hardware, at zero cost.
- No GPU spend, no local serving stack, no quantization tuning.
- Provider abstraction pays for itself immediately through failover.

**Bad**
- Requires internet. No offline work.
- Rate limits are a real architectural constraint — see ADR-0003; this is a hard cap on parallelism.
- Free tiers can change terms without notice. *Mitigated by multi-provider failover being built in from the start rather than added later.*
- Code leaves the machine. Acceptable for personal projects; would need revisiting for anything under NDA.
