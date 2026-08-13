# ADR-0004 — Cloud inference only, no local models

**Status:** accepted · **Date:** 2026-08-12

## Context

Local hardware cannot run models at the quality and speed required. The alternative — cloud inference with local tooling — turns out to be strictly better, not a compromise.

**NVIDIA Build** provides 100+ models on an OpenAI-compatible endpoint (`https://integrate.api.nvidia.com/v1`), free, no credit card. Agentic coding models include Qwen3 Coder 480B (purpose-built for agentic coding), GLM-5.2, DeepSeek V4 Flash (1M context), MiniMax M2.7. Tool calling is confirmed working across the major models. OpenCode supports it directly via `@ai-sdk/openai-compatible`.

Free tier limits: **1,000 credits on signup (up to 5,000 on request), 40 requests/minute.**

**Groq** was evaluated as a second free provider. OpenAI-compatible at `https://api.groq.com/openai/v1`, no credit card, custom LPU hardware delivering 300–800 tokens/sec against roughly 50–100 for GPU providers. Open-weight models only — Llama, Qwen, DeepSeek-R1, gpt-oss, Gemma. Function calling supported.

Its free tier advertises **30 RPM / 14,400 requests per day**, but the binding constraint is **tokens per minute: 6,000 TPM default** (12K for Llama 3.3 70B, 8K for gpt-oss-120b).

> ⚠️ One secondary source reports NVIDIA acquired Groq in December 2025 and pitched a "Groq 3 LPU" at GTC 2026 targeting ~1,500 tok/s for agentic inference. Found in a single place — **unconfirmed**. If true, these two providers may converge. Re-check before finalising provider config.

## Decision

**Cloud inference only.** The local machine runs editor, terminal, agent runtime, compiler, browser, git, and toolchains. Model inference is remote.

Local model support (Ollama, vLLM) stays in the adapter list as a **last-resort fallback**, not a supported path.

### Provider tiering

| Tier | Provider | Used for |
| --- | --- | --- |
| **Heavy** | NVIDIA Build — Qwen3 Coder 480B, GLM-5.2, DeepSeek V4 | coding, decomposition, review, adjudication |
| **Fast** | Groq | file triage, classification, summaries, commit messages |
| **Failover** | whichever tier is not currently throttled | continuity |

**Groq is the fast tier and a failover path — never the driver.** Two reasons:

1. **6K TPM cannot carry an agentic coding loop.** A single turn with real repo context is 30–50K tokens, so the free budget allows roughly one such request every 5–8 minutes. It is sized for chat, not for agents. It fits small-context, high-frequency work perfectly.
2. **Open-weight ceiling.** Token *rate* is excellent; model *capability* sits below the heavy tier. For hard agentic coding, capability beats speed — a fast model needing four tool-call iterations instead of two is slower end to end and burns more quota.

Two free providers with different limits is strictly better than one, which is the continuity argument below.

### Rule: read TPM before RPM

Providers advertise request counts; **token throughput is what actually constrains an agent.** Any new provider is evaluated on TPM against a realistic context size first, and RPM second. This is the metric that decided Groq's role here, and it will decide the next one's.

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
