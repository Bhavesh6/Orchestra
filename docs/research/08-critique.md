# Document 8 — Critique: Where This Idea Is Wrong

The brief asked me to challenge it rather than agree with it. This document is that. It is written to be uncomfortable, because a design review that flatters the design is worthless.

**Summary judgment up front:** the architecture is sound, but **the central premise is oversold**. The value in this project is roughly 80% in *structured project state, contracts, and verification* and roughly 20% in *parallel agents* — and the brief allocates its excitement in exactly the opposite proportion. If we build it in the brief's priority order, we will spend most of our effort on the least valuable part.

---

## 8.1 The evidence against multi-agent parallelism is stronger than the evidence for it

This is not my opinion; it's what the sources say — including the vendors selling the feature.

**From the research literature.** "Why Do Multi-Agent LLM Systems Fail?" (arXiv 2503.13657, NeurIPS 2025) built the first Multi-Agent System Failure Taxonomy from 150 annotated traces (κ=0.88), extended to 1600+ traces across 7 frameworks. Findings: **14 failure modes in 3 categories** — system design, inter-agent misalignment, task verification. And the headline: **"Despite enthusiasm for Multi-Agent LLM Systems, their performance gains on popular benchmarks are often minimal."** **[Src]**

Secondary analyses report production failure rates between **41% and 86.7%**, and error amplification of **up to 17× in uncoordinated systems vs ~4.4× with centralized validation**. **[2°]** — treat the exact figures with suspicion, but note that even the *good* number is amplification, not reduction.

**From Anthropic's own documentation.** Claude Code's agent-teams page — documentation for a shipping multi-agent feature — states that teams "add coordination overhead and use significantly more tokens," and that "for sequential tasks, same-file edits, or work with many dependencies, a single session or subagents are more effective." The feature is **disabled by default** behind an experimental flag, with a documented limitations list including task-status lag blocking dependents, teammates stopping early on errors, and no session resumption. **[Doc]**

**From Cursor.** Supports 8 parallel agents; the practical guidance is to **start with 2–3** because merge-conflict complexity "scales non-linearly with agent count." **[2°]**

**From OpenAI.** Codex caps concurrent subagent threads at a default of **6** and ships `max_depth` specifically to **prevent recursive delegation**. **[2°]**

Read that list again. Every organization that has shipped this feature ships it **capped, gated, defaulted-off, or with explicit warnings**. That is not what a winning strategy looks like — it is what a feature people keep hurting themselves with looks like.

**Implication for us:** we should not treat "many parallel agents" as the goal. We should treat it as a **capability we deploy narrowly, where the evidence supports it**, and design so the system works well when it declines to parallelize.

---

## 8.2 Where parallelism actively makes coding worse

Specific, not general:

**Coupled code.** Most real changes touch multiple files that must change together. Split them across agents and you get a system that must reconcile decisions that should never have been separated. The `file_scope` mechanism (Doc 3 §3.4) prevents the *textual* collision, but at the cost of **serializing the work anyway** — so you've paid the coordination overhead for no parallelism.

**Exploration and debugging.** Debugging is inherently sequential: form hypothesis → test → update. Five agents testing five hypotheses in parallel sounds efficient until you notice that hypothesis 2 is only worth testing *after* hypothesis 1 fails. Parallel exploration duplicates the setup cost five times and discards four results. (Counterpoint: Claude Code documents adversarial-debate multi-agent debugging as a *strength*, on the grounds that sequential investigation suffers from anchoring — once one theory is explored, subsequent work is biased toward it. **[Doc]** I find this genuinely persuasive and it's the strongest pro-parallel argument I encountered. It is also an argument for **parallel hypothesis generation**, not parallel implementation.)

**Small codebases.** If the project fits in one context window, splitting it destroys the one thing the model is best at: seeing everything at once. Decomposition here is pure loss.

**Ill-defined tasks.** Decomposition requires knowing the structure of the problem. For vague objectives, the orchestrator's DAG is a guess, and per MAST, bad specification is the largest failure category. **[Src]** Six agents executing a confident wrong plan is much worse than one agent noticing the plan is wrong.

**Anywhere a shared mutable resource exists.** One ESP32. One serial port. One test database. One dev server port. Parallelism collapses to a queue plus overhead.

---

## 8.3 The cost argument is worse than it looks

Naive framing: N agents ≈ N× cost. Reality is worse.

| Multiplier | Cause |
| --- | --- |
| **Context duplication** | Each agent reloads project context, skills, contracts. Shared preamble paid N times (prompt caching helps, and is the main reason this isn't catastrophic). |
| **Orchestration overhead** | Decomposition, integration, adjudication are themselves expensive model calls — and they need the *strongest* model. |
| **Re-derivation** | Agents rediscover things a single agent already knew, because context was deliberately withheld. |
| **Rework from conflicts** | Every semantic conflict costs an adjudication plus redoing work. |
| **Verification × N** | Gates run per agent, then again on integration. |
| **The tail** | Wall-clock is set by the *slowest* agent, not the average — so you pay N× cost for maybe 2–3× speedup. |

**Honest expected profile:** roughly **3–5× cost for 2–3× wall-clock speedup** on genuinely parallel work, and **worse than single-agent on both axes** for everything else.

That can still be a good trade — developer wall-clock time is expensive and tokens are not. But it must be stated plainly, shown to the user *before* fan-out, and it means the default must be conservative.

---

## 8.4 Shared state cuts both ways

Document 6 argues the Brain is the coordination substrate. Fair. Now the counterargument, which I take seriously:

- **A wrong fact in shared state is worse than no shared state.** A single agent that misremembers affects one session. A wrong entry in `contracts/` is trusted by every agent, forever, and gets *cited as evidence* by downstream work — laundering a guess into an established fact through repetition.
- **Contracts ossify.** Once `HW-PINMAP@v3` exists, agents optimize within it rather than questioning it. If the pin map is wrong, the system will build increasingly elaborate correct-looking work on a broken foundation. A single agent holding everything in context is more likely to notice the contradiction.
- **The Brain is an injection target (T8).** Anything that writes to it — including a research agent that read a hostile web page — can poison every downstream agent. Typed writes and validators help; they do not eliminate it.
- **Maintenance burden.** A Brain that drifts from reality is actively harmful. Who maintains it when a human edits code directly without going through an agent? That's not a rhetorical question and I don't have a complete answer. Partial answer: contracts that *generate* code (Doc 3 §3.6) can't drift silently, because drift becomes a build failure. **That is a strong argument for making generated-from-contract the dominant pattern and prose-in-the-Brain the exception.**

---

## 8.5 Model routing will disappoint at first

Document 4 designs a capability router. Here's why it will underdeliver initially:

- **The scoring function is the whole thing, and we have no data.** Filtering by capability is easy and mostly free. Choosing *which* qualifying model is best for *this* task requires quality estimates we won't have until we've run thousands of verified tasks per (role × model) cell. Until then, "capability routing" is a config file of guesses with extra machinery.
- **Model quality is task-specific in ways that don't generalize.** A model that's excellent at Python refactoring may be mediocre at embedded C. Our routing table will be coarse for a long time.
- **Cheap models are a false economy in agentic loops.** A weak model that takes 4 tool-call iterations instead of 2 costs more than the strong model, *and* burns wall-clock, *and* is likelier to produce work that passes shallow gates and fails deep ones. Cost-per-token is the wrong metric; **cost-per-verified-task** is right, and we can only measure it after building the gates.
- **Provider convergence.** Frontier models are converging in capability. The gap that routing exploits may be narrowing faster than we can build the router.
- **The security inversion.** Per §5.1: more capable models were often *more* susceptible to indirect prompt injection, because better instruction-following means better compliance with injected instructions. **[2°]** So "route the smart model to the task that reads untrusted web content" is plausibly backwards. Almost nobody accounts for this.

**Honest recommendation:** ship provider *abstraction* (high value, low risk, immediate — swap providers, avoid lock-in, use local models). Ship *routing* as static per-role profiles. Do not promise intelligent routing until the gate data proves it. And measure it against the naive baseline of "use one good model for everything," which is a stronger baseline than anyone wants to admit.

---

## 8.6 When a single strong agent is simply better

Direct answer to the brief's request:

| Situation | Why single wins |
| --- | --- |
| Codebase fits in context | Full visibility beats decomposition, always |
| Tightly coupled changes | Coordination cost exceeds any parallel gain |
| Debugging a specific failure | Sequential hypothesis refinement; context accumulates value |
| Vague or exploratory objective | Decomposition requires structure you don't have yet |
| Refactoring | Cross-cutting by definition; the worst possible fit for file-scope partitioning |
| Anything under ~30 min of work | Orchestration overhead dominates |
| One shared physical/stateful resource | Parallelism becomes a queue |
| High-stakes single decision | You want one coherent reasoner, not a committee |

**That's a lot of software development.** Plausibly the majority. The multi-agent path is genuinely better for: greenfield multi-dimensional projects (the ESP32-plus-firmware-plus-web case the brief describes — a legitimately good fit), independent parallel review lenses, broad research fan-out, and large mechanical migrations across many independent files.

Note that our motivating example is a *good* fit. That's a reason to build it — and also a reason to be suspicious, because designing from your best-case example is how systems end up narrow.

---

## 8.7 What the brief gets right

To be fair, because these are the load-bearing ideas:

1. **Contracts between agents (§16).** The best idea in the brief. Typed interfaces with deterministic validators turn coordination problems into compile errors. Valuable *even with one agent*, because they encode intent that code cannot.
2. **Verification loops (§18).** "No agent may say Done" is exactly right, and it's the one thing that reliably improves output quality regardless of agent count.
3. **Structured outputs (§13).** Prose between agents is lossy and unauditable. Right call.
4. **Model-agnosticism (§8).** Correct strategically, independent of whether routing gets clever.
5. **Security from the start (§24).** Correct, and the trifecta-breaking decomposition (Doc 5 §5.1) may be the strongest genuine argument for multi-agent that exists.
6. **Don't start from the UI (§31).** Emphatically right; OpenCode's architecture proves it.
7. **Use MCP, don't reinvent (§6).** The single biggest leverage available.

Note that **five of these seven have nothing to do with parallelism.** That's the pattern.

---

## 8.8 The differentiator is not what the brief thinks it is

The brief's stated differentiator is model-agnostic multi-agent orchestration. But:

- Model-agnostic: **OpenCode and Cline already do this.** 20–33+ providers. Not differentiating.
- Multi-agent orchestration: **Claude Code, Codex, Antigravity, and Cursor all ship it.** Not differentiating.
- Parallel agents in isolated workspaces: **Cursor shipped worktrees in 2.0.** Not differentiating.

What **nobody** currently does, based on this research:

1. **Contract-driven development where contracts generate code** and violations are compile errors rather than model-adjudicated debates.
2. **Verification-gated completion enforced by the runtime** — `status: complete` rejected by code when gates haven't passed, not requested in a prompt.
3. **Evidence-graded claims** with automatic demotion of unsupported assertions to assumptions.
4. **Cross-provider independent review** as a routing constraint.
5. **Trifecta-breaking agent decomposition** as an explicit security architecture.
6. **A scheduler that refuses to parallelize** and explains why.

**That's the actual product.** It is less exciting than "orchestrate ten agents across five providers," and it is far more defensible. I'd reframe the pitch around **verifiable, contract-driven development** — with multi-agent as an implementation detail that shows up when it helps.

---

## 8.9 The biggest risk is scope, not architecture

Honest count of what the brief describes: an agent runtime, a model gateway, an MCP client and server, a skills system, a permission system, a sandbox, a workspace manager, an orchestrator, a task scheduler, a knowledge graph, a contract system, a conflict resolver, a verification framework, a browser agent, a document pipeline, embedded tooling, and a UI.

**That is 8–10 separate products.** Each is a serious engineering effort. Several — sandboxing, model gateway, browser automation — are things well-funded teams work on full time.

The mitigation isn't cleverness, it's **subtraction**:
- Reuse aggressively (Doc 7 §7.6). Every "we'll build our own X" should require a written justification.
- Phase gates with **real exit criteria**, especially P5's measured decision point.
- Be willing to ship the 80% (contracts + verification + provider independence) and stop, if the 20% doesn't prove out.

---

## 8.10 Open questions I could not resolve

Stated honestly rather than papered over:

1. **Does adversarial debate beat blackboard coordination for debugging?** Claude Code's docs make a strong case for direct agent-to-agent argument. **[Doc]** Our blackboard design can't reproduce it well. Possible resolution: an explicit `debate` task type with structured rounds. **Needs an experiment, not an argument.**
2. **What's the right decomposition granularity?** Too fine → overhead dominates. Too coarse → no parallelism. I have no principled answer; it's probably empirical per project type.
3. **Who maintains the Brain when humans edit code directly?** Contract-generated code helps. Prose sections have no good answer yet.
4. **Is the human plan-approval gate a bottleneck that kills the value proposition?** If a human must approve every DAG, we've capped throughput at human review speed. But removing it removes our main defense against the largest failure category. Genuine tension, unresolved.
5. **Can we actually measure "quality" well enough to route on it?** §8.5. Unknown until we have gate data.
6. **Does the ESP32 vertical generalize?** We're designing around a domain with unusually crisp contracts (pins, voltages, buses). Web development is much mushier. The architecture may be strongest exactly where it's least general.

---

## 8.11 Bottom line

**Build it — but build it in the opposite order to the brief's enthusiasm, and be honest about which part is the product.**

The contract system, the verification gates, and the structured evidence model are valuable **on day one with a single agent**. They should be built first, and they should be good enough to justify the project by themselves.

Parallel multi-agent orchestration is a **hypothesis to be tested at P5** with a real exit criterion, not a premise to be assumed. The evidence — from academic work and from every vendor shipping the feature — suggests it will help meaningfully in a minority of cases and hurt in others.

A system that knows when *not* to fan out will beat a system that always does. Building that judgment into the scheduler is more valuable than any amount of orchestration machinery.
