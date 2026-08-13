# Document 10 — Prior Art: The Spec-Driven Development Landscape

**Date:** 2026-08-12
**Why this exists:** a check for whether Orchestra duplicates something that already ships. The answer is *partly yes*, and it forces a correction to [ADR-0002](../decisions/ADR-0002-contracts-and-gates-are-the-differentiator.md).

---

## 10.1 The finding

**Spec-Driven Development (SDD) is a crowded, mature category that I had not accounted for.** By 2026, essentially every major AI coding tool has shipped a flavour of it: GitHub Spec Kit, AWS Kiro, Tessl, OpenSpec, BMAD, Agent OS, Spec Kitty, GSD — plus SDD workflows inside Claude Code, Cursor and Antigravity. **[2°]**

| Project | What it is | Maturity |
| --- | --- | --- |
| **GitHub Spec Kit** | MIT toolkit; natural-language spec → plan → atomic tasks → code. 30+ agent integrations (Copilot, Claude Code, Cursor, Gemini CLI, Codex, Windsurf…) | **~111k stars**, v0.11.0 Jun 2026 |
| **AWS Kiro** | Agentic IDE built on SDD. EARS requirements syntax; generates `requirements.md`, `design.md`, `tasks.md`; automated hooks | Shipping |
| **Tessl** | "Spec-as-source": reverse-engineers specs from code, maintains **1:1 spec↔file mapping**, marks output `// GENERATED FROM SPEC - DO NOT EDIT`. Founded by Guy Podjarny (Snyk) | Framework in **closed beta ~9 months**; Registry open beta |
| **OpenSpec** (Fission AI) | Change-centric, fewer generated files, lower token use; scored highest in one Feb 2026 independent evaluation | Shipping |
| **agent-gates** | "Quality gates for AI coding agents." Five gates (Spec, Diagnosis, Test, QC, Evidence); structural checks built in, semantic via LLM; Claude Code hooks + CLAUDE.md + Python lib | **0 stars, 3 commits** — preliminary |
| **relari-ai/agent-contracts** | Pre/path/post-conditions verified against agent execution traces; offline verification + runtime certification | Shipping |
| **microsoft/agent-governance-toolkit** | Policy enforcement, zero-trust identity, sandboxing; covers OWASP Agentic Top 10 | Shipping |

**[2°]** throughout except where fetched directly (agent-gates, agent-contracts).

---

## 10.2 What this invalidates

[ADR-0002](../decisions/ADR-0002-contracts-and-gates-are-the-differentiator.md) claimed nothing in the field does contracts-generate-artifacts or runtime-enforced verification. **That claim was wrong.**

- **Tessl does generate code from specs**, with explicit do-not-edit markers — very close to our `authoritative: true`.
- **agent-gates does gate completion**, including a "not done until evidence exists" model.
- **Loki Mode** ships verification gates where "work isn't done until it passes a blind-review completion council and evidence checks." **[2°]**

I over-claimed because Documents 01–09 surveyed *agentic environments* (Claude Code, Codex, Antigravity, Cursor, OpenCode, Cline) and never surveyed the **methodology tooling** that sits on top of them. That was a scoping error in the original research, and it's exactly the kind of thing this document exists to catch.

---

## 10.3 What survives — and it's sharper than before

Third-party analysis of the incumbents identifies a consistent gap, in language that maps almost exactly onto Orchestra's thesis:

> "Spec Kit's prose specs drive the first generation but **do not durably govern the codebase because the authority is convention**. Kiro's specs are more structured, but **EARS phrasing is not an executable contract**." **[2°]**

> "Spec Kit has no verification loop and Kiro's hooks stop at unit tests passing — **neither one checks the running app**." **[2°]**

> "Non-deterministic adherence occurs where AI may ignore or partially follow rules, **with no mechanism to enforce compliance**." **[2°]**

And on Tessl, the closest competitor:

> "**Test linkage is a reference mechanism rather than executable verification.** The spec points to a test file, but the spec itself doesn't prove its own correctness." **[2°]**

### The distinction that matters

**Every SDD tool's spec is prose that an LLM interprets. None of them contain executable constraints.**

| | SDD tools | Orchestra |
| --- | --- | --- |
| Spec content | natural language / EARS / markdown | YAML with **named rule functions** |
| Who enforces | the LLM, by convention | **the validator and the compiler** |
| Verification | LLM-judged, or "tests passed" | deterministic checks + declared gates |
| Cost per check | a model call | free, instant |
| Determinism | non-deterministic adherence | binary |
| Coverage | all code (Tessl: 1:1 per file) | **interfaces only, ~20–30%** |

`unique_pins` is a function. `adc2_unusable_with_wifi` is a function. `fk_integrity` is a function. They run in milliseconds, cost nothing, and cannot be partially followed. No SDD tool has an equivalent — their constraints are sentences.

### Where the incumbents are weak

- **Tessl's 1:1 spec-per-file mapping is criticised as too rigid**, and the Martin Fowler analysis draws a direct parallel to **Model-Driven Development** from the early 2000s — with the warning that spec-as-source "risks combining the worst of both: inflexibility *and* non-determinism." **[2°]** Observed output language was JavaScript only, despite the language-agnostic claim. Still not GA after ~9 months of beta.
- **agent-gates** is the right idea but is 3 commits old, Claude Code-specific, and leans on LLM calls for semantic gates.
- **relari agent-contracts solves a different problem** — verifying *agent runtime behaviour* from execution traces, not validating software artifacts. Not a competitor.

Orchestra deliberately covers **interfaces only**, which sidesteps the rigidity criticism that lands on Tessl. Generating every file from a spec is Model-Driven Development with a new coat of paint. Generating `pins.h` and a typed API client is just codegen — a boring, proven technique applied where coordination actually breaks.

---

## 10.4 Revised positioning

**Old (wrong):** "Nobody does contracts that generate code or enforced verification."

**New (defensible):**

> Every SDD tool makes specs *authoritative by convention* — an LLM is asked to follow prose. Orchestra makes a narrower class of decisions *authoritative by construction*: interface constraints are executable functions, and the artifacts they generate are enforced by the compiler. **Their authority is convention. Ours is the toolchain.**

Two consequences:

1. **We should not compete with Spec Kit on specs.** It has 111k stars and 30+ agent integrations. Writing another prose-spec workflow is a losing move.
2. **We should consider composing with it.** Spec Kit answers *what to build*. Orchestra answers *this interface decision is now mechanically enforced*. Those layer cleanly, and since Orchestra ships as an MCP server, coexistence is the default rather than an integration project.

---

## 10.5 Risks this raises

| Risk | Assessment |
| --- | --- |
| **Category is crowded** | Real. Being late to a crowded category is a genuine disadvantage — but the crowd validates that the problem is worth solving, and every incumbent has the same named gap. |
| **Tessl closes the gap** | They're VC-backed with a strong founder. If they add executable constraints, our advantage narrows sharply. **Mitigation: ship fast; L2 is weeks.** Their 9-month closed beta suggests it's harder than it looks. |
| **Spec Kit adds validators** | Possible. 111k stars means resources. But its design centre is prose specs across 30+ agents; deterministic domain validators are a different shape of thing. |
| **We're solving a niche** | Interfaces are ~20–30% of coding work. That's the honest scope, and narrower than the original vision implied. |

---

## 10.6 What to do about it

1. **Correct ADR-0002.** Done — see its Update section.
2. **Re-run the M1 definition of done against this framing.** Our test — change an OpenAPI field, watch the frontend typecheck fail — is still the right one, and notably *not* something Spec Kit or Kiro do.
3. **Evaluate Spec Kit hands-on during M0.** It's MIT, 111k stars, and integrates with OpenCode-class agents. If it does the "what to build" half well, adopt it rather than rebuild it — the same reuse principle that gave us ADR-0001.
4. **Watch Tessl's GA.** If the Framework ships with executable constraints, revisit the whole positioning.

**Action added to M0.**
