# Document 1 — Competitive Architecture Study

Evidence grades: **[Doc]** vendor docs read directly · **[Src]** source/spec · **[2°]** secondary source · **[Inf]** my inference · **[Unk]** not public.
See [Document 0](00-research-method-and-sources.md) for the egress caveat — Cursor, Antigravity, OpenCode and Codex vendor domains were blocked, so those sections lean on [2°].

---

## 1.1 The single most useful finding

Before the per-product detail, the thing that reframed my thinking:

> **Every mature system in this space has converged on the same four-layer shape, and they differ mainly in where they put the isolation boundary.**

```
   Presentation      TUI / IDE panel / web / agent manager
        │
   Session+Loop      agent loop, context assembly, compaction, permissions
        │
   Tool Layer        built-in file/shell tools + MCP clients + skills
        │
   Model Layer       one or many providers
```

Claude Code, Codex, OpenCode, and Cline are all recognizably this. Antigravity and Cursor add a **workspace/orchestration layer** above the session, which is exactly where our brief wants to innovate. That is genuinely the open ground — but see [Document 8](08-critique.md) before getting excited, because the vendors who built that layer are also the ones publishing warnings about it.

---

## 1.2 Claude Code

Best-evidenced section — `code.claude.com` was reachable.

### Agent loop and context
- Single agent loop per session, with its own context window. Subagents get **their own separate context windows**; results return as a summary to the caller. **[Doc]**
- The documented purpose of a subagent is explicitly **context economy**: "the subagent does that work in its own context and returns only the summary." **[Doc]**
- Compaction/summarization exists as a user-facing feature (`/compact`); the internal heuristics are **[Unk]**.

### Subagents
Defined as markdown files with YAML frontmatter, resolvable at project / user / plugin / CLI scope. Documented fields include `name`, `description`, `tools` (allowlist), `model`, `skills`, `mcpServers`, and `background`. **[Doc]**

Key documented constraints, which are design lessons for us:
- Subagents **cannot talk to each other**. They report only to the caller. **[Doc]**
- Built-in subagent types exist (e.g. `Explore`, `Plan`). **[Doc]**

### Agent teams (experimental)
This is the closest shipping analogue to our orchestrator, and it is unusually well documented. **[Doc]**

- Gated behind `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`; **disabled by default**.
- Architecture: **team lead** (the main session) + **teammates** (full independent Claude Code sessions) + **shared task list** + **mailbox**.
- Mailboxes are **JSON files** at `~/.claude/teams/{team}/inboxes/{agent}.json`. Task list at `~/.claude/tasks/{team}/`.
- Task claiming uses **file locking** to prevent races.
- Tasks support **dependencies**; completing a task auto-unblocks dependents.
- Teammates load CLAUDE.md, MCP servers, and skills fresh — **the lead's conversation history does not carry over**.
- Plan-approval protocol: teammate works read-only until the lead approves; rejection returns feedback and the teammate revises.
- Hooks `TeammateIdle`, `TaskCreated`, `TaskCompleted` can **block** transitions via exit code 2.

Documented limitations we should treat as hard-won lessons:
- **No nested teams** — teammates cannot spawn teammates.
- **Lead is fixed** for the session's lifetime.
- **Permissions set at spawn**; teammates inherit the lead's mode.
- Task status **lags** — teammates fail to mark tasks complete, blocking dependents.
- Teammates **stop early** on errors.
- **No session resumption** for in-process teammates.

And the cost warning, verbatim in the docs: agent teams "use significantly more tokens than a single session… For sequential tasks, same-file edits, or work with many dependencies, a single session or subagents are more effective." **[Doc]**

> That sentence is the vendor of the most-hyped multi-agent feature telling you when not to use it. Document 8 builds on it.

### Skills
`SKILL.md` + frontmatter; custom commands were **merged into** skills. Documented frontmatter includes `name`, `description`, `allowed-tools`, `disable-model-invocation`, `context: fork`, `agent:`, `argument-hint`. **[Doc]**
- **Progressive disclosure** is the core idea: "a skill's body loads only when it's used, so long reference material costs almost nothing until you need it."
- Skills can bundle scripts and reference them via `${CLAUDE_SKILL_DIR}`, with tool permissions scoped to those scripts.
- `context: fork` runs the skill in a forked context, optionally with a specific `agent:` — i.e. **a skill can itself be a delegated subagent invocation**. That's an elegant unification worth stealing (class D).

### Permissions
- Rules evaluated **deny → ask → allow**; first match wins; **specificity does not override order**. **[Doc]**
- Syntax: `Bash(npm run test *)`, `Read(//abs/path/**)`, `Edit(...)`, `WebFetch(domain:example.com)`, `mcp__server__tool`.
- The implementation is **shell-operator aware** (`&&`, `||`, `;`, `|`, `&`, newlines each matched independently) and **strips wrappers** (`timeout`, `nice`, `nohup`, `xargs` without flags, safe env assignments). **[Doc]**
- The docs openly admit **pattern-matching permissions are fragile** for argument constraints and recommend deny-listing `curl`/`wget` and routing through `WebFetch(domain:…)` instead. **[Doc]**
- OS-level enforcement is a separate **sandbox** feature; permission rules alone don't stop a subprocess that opens files itself. **[Doc]**

### Everything else
Hooks, MCP client support, git integration, background tasks, `/resume`, cross-session messaging, an Agent SDK, and a plugin system are all documented. Model selection is user-facing (`/model`, per-subagent `model:`, org `availableModels` allowlist with documented substitution rules) — **there is no documented automatic quality-based router**. **[Doc]** Any claim otherwise is **[Unk]**.

---

## 1.3 OpenAI Codex

⚠️ `developers.openai.com` was blocked; this is **[2°]** throughout and should be re-verified.

- **GPT-5.5** default backing model as of 2026; native subagents, MCP, hooks, auto-review, and remote Codex Cloud tasks. **[2°]**
- **Subagents are declarative TOML** under `.codex/agents/` (project) or `~/.codex/agents/` (user). Each requires `name`, `description`, `developer_instructions`; optionally pins its own **model, reasoning effort, sandbox mode, and MCP servers**. **[2°]**
- Concurrency is **explicitly capped in config**: `[agents]` with `max_threads` (default **6**), `max_depth` (prevents recursive delegation), `job_max_runtime_seconds`. **[2°]**
- Built-in agents: `explorer`, `worker`, `default` — `worker` intended for many small parallel tasks. **[2°]**
- **Sandboxing is the strongest of any major CLI agent and is on by default.** Three profiles: `read-only`, `workspace-write` (default), `danger-full-access`. Uses **Landlock + seccomp** on Linux. Approval policy is a *separate* axis: `untrusted`, `on-request`, `never`. Network access is off by default under `workspace-write`. **[2°]**
- MCP config in TOML; v0.147.0 ships opt-in **MCP 2026-07-28** support with paginated discovery, multi-round requests, non-blocking server startup. **[2°]**
- `AGENTS.md` is the instruction file convention, also used to drive **custom code-review rules**. **[2°]**
- Hooks inject scripts into the agentic loop (logging, secret-scanning prompts, memory summarization, stop-turn validation), with `/hooks` for trust management. **[2°]**

**What to steal:** the two-axis separation of **sandbox mode** and **approval policy**. It is cleaner than conflating them, and it's the design I recommend in [Document 5](05-security-architecture.md). Also `max_depth` — a hard cap on delegation recursion is a cheap safeguard we should have from day one.

---

## 1.4 Google Antigravity

⚠️ `antigravity.google` blocked. **[2°]** throughout.

- **Agent Manager is the primary surface**, with the editor as a *peer*, not the center. Explicitly an orchestration dashboard. **[2°]**
- Multiple agents run **in parallel across workspaces**; one refactors while another writes tests. **[2°]**
- **Artifacts** are the headline concept: structured deliverables — implementation plans, task lists, screenshots, browser recordings, diffs, test reports — created so a human can **review at milestones instead of watching every step**. Artifacts are **commentable**; feedback flows back to the agent. **[2°]**
- **Implementation Plan** is a pre-flight artifact reviewed before code changes. **[2°]**
- **Browser subagent** runs a *different, browser-specialized model* from the main agent, in a **separate Chrome profile** so the user's cookies/sign-ins are not shared. **[2°]**
- **Knowledge base**: agents both retrieve from and contribute to a store of learnings across tasks. **[2°]**
- Agents operate across three surfaces: editor, terminal, browser. **[2°]**

**What to steal:** artifacts-as-review-surface is the best idea in this entire competitive set, and it's class **D** (a pattern, reproducible). It solves a real problem — a human cannot supervise six parallel agents by reading six transcripts. It maps directly onto our structured-output requirement (brief §13). The separate browser profile is a security pattern we should copy verbatim.

---

## 1.5 Cursor

⚠️ `cursor.com` blocked. **[2°]** throughout.

- Since **Cursor 2.0**, agents run in **isolated git worktrees** — own branch, own filesystem — so parallel agents don't collide. **[2°]**
- Three execution modes: local, local-worktree, and **Cursor-hosted cloud agents** that clone into an isolated VM with a full dev environment. **[2°]**
- Up to **8 simultaneous parallel agents**, but the practical guidance is **start with 2–3** because "merge conflict complexity scales non-linearly with agent count." **[2°]**
- `worktree.json` in project root enables "Build in Parallel"; `/worktree`, `/multitask`, and **`/best-of-n`** commands. **[2°]**
- Rules system (`.cursor/rules`), MCP support, background agent API. **[2°]**

**What to steal:** git worktrees as the default isolation primitive. It is nearly free, it's class **A/B** (git is a standard; libgit2/CLI is reusable), and it gives you a real merge story instead of a bespoke conflict resolver. **`/best-of-n` is also a warning label** — Cursor shipped N-parallel-attempts as an *explicit, user-invoked* mode, not as the default. That is a considered choice, and we should make the same one.

---

## 1.6 OpenCode

MIT licensed, `sst/opencode`. `opencode.ai` blocked → **[2°]**, but the repo is open (class **B**).

- **Client–server architecture.** A backend server owns sessions, context prep, agent execution, tool orchestration, provider integration. TUI/desktop/web/IDE/SDK clients are *only* presentation. `opencode serve` runs standalone; clients talk HTTP + event streams. **[2°]**
- "The TUI never executes AI tasks itself." **[2°]**
- Provider-agnostic by design: **20+ providers** via the **Vercel AI SDK**, with the provider/model catalogue driven by **models.dev**. Any OpenAI-compatible endpoint works, including local models. **[2°]**
- `SessionPrompt.loop()` drives the agent; `Provider.getModel()` resolves the model. **[2°]**
- Plugins, skills, MCP servers, custom tools, LSP integration. **[2°]**

**What to steal — and this is the biggest architectural decision in the whole project:** the **server-first, headless-core split**. It is the reason OpenCode can have a TUI, a desktop app, a web UI, and an SDK without forking logic. Our brief already says "do not start from the UI" (§31); OpenCode proves the shape works. `models.dev` as a **capability catalogue** is directly relevant to our capability router ([Document 4](04-model-gateway.md)).

---

## 1.7 Cline

Open source, `cline/cline`. **[2°]**

- VS Code extension (plus CLI and SDK) over a shared **Cline SDK session layer** handling agent turns, tools, mode coordination, MCP, checkpoints, telemetry, compaction, **mistake limits**, task history. **[2°]**
- **Plan / Act modes**: a global state-managed duality — explore and propose, then execute with privileges. **[2°]**
- **33+ providers** including OpenAI-compatible, LM Studio, Ollama. **[2°]**
- **MCP Marketplace** in-extension: discover/install/enable Skills, MCP servers, and Plugins. **[2°]**
- Browser use and terminal use as first-class tools; approval-gated autonomous loop. **[2°]**

**What to steal:** **checkpoints** and **mistake limits**. Checkpointing agent state so a human can rewind is underrated and much cheaper than trying to make agents never fail. A mistake limit — halt after N consecutive failures — is a trivially implementable circuit breaker that most designs forget.

---

## 1.8 The capability matrix (brief §28)

Legend: ● full / documented · ◐ partial or limited · ○ absent or not documented · **?** unverified due to blocked source.

| Capability | Claude Code | Codex | Antigravity | Cursor | OpenCode | Cline | **Our approach** | **Class** |
| --- | :-: | :-: | :-: | :-: | :-: | :-: | --- | :-: |
| **Terminal** | ● | ● | ● | ● | ● | ● | Own tool; sandboxed by default | D |
| **File editing** | ● | ● (`apply_patch`) | ● | ● | ● | ● | Own tools + structured patch format | D |
| **MCP client** | ● | ● | ● | ● | ● | ● | **Full client, 2026-07-28 spec** | A |
| **MCP server (expose self)** | ◐ | ◐ | ? | ◐ | ◐ | ◐ | **Yes — expose orchestrator as MCP server** | A |
| **Skills** | ● | ● | ● | ◐ | ● | ● | Adopt `SKILL.md` convention | A/D |
| **Subagents** | ● | ● | ● | ● | ◐ | ○ | Core primitive, typed contracts | D |
| **Parallel agents** | ◐ (experimental) | ● (cap 6) | ● | ● (cap 8) | ◐ | ○ | **Capped, DAG-scheduled, conflict-aware** | D |
| **Inter-agent messaging** | ◐ (teams only) | ○ | ? | ○ | ○ | ○ | **Via blackboard, not chat** | D |
| **Browser** | ◐ (via MCP) | ◐ | ● (subagent + own profile) | ◐ | ◐ | ● | MCP (Playwright + Chrome DevTools) | B/C |
| **Worktrees** | ◐ (manual) | ○ | ? | ● | ○ | ○ | **Default isolation primitive** | A/B |
| **Sandbox** | ◐ (opt-in) | ● (**default on**) | ● | ● (cloud VM) | ◐ | ◐ | **Default on**, Landlock/seccomp → microVM | B |
| **Long-running agents** | ◐ | ● (cloud) | ● | ● (cloud) | ◐ | ◐ | Durable task queue | D |
| **Background tasks** | ● | ● | ● | ● | ◐ | ◐ | Server-side job runner | D |
| **Model routing** | ○ (manual) | ◐ (per-agent pin) | ◐ (browser model) | ◐ | ● (multi-provider) | ● | **Capability-based router** | D |
| **LSP** | ○ | ○ | ? | ● | ● | ◐ | Reuse existing LSP clients | B |
| **Git** | ● | ● | ● | ● | ● | ● | Worktree-native, merge-queue | B/D |
| **Artifacts** | ◐ (files) | ◐ | ● (**central**) | ◐ | ○ | ◐ | **Central review surface** | D |
| **Computer use** | ○ | ◐ | ● | ◐ | ○ | ◐ | Deferred; via MCP when needed | C |
| **Memory** | ● (CLAUDE.md) | ● (AGENTS.md) | ● (knowledge base) | ● (rules) | ◐ | ● | **Project Brain** (structured) | D |
| **Hooks** | ● | ● | ? | ◐ | ● (plugins) | ◐ | Lifecycle hooks + policy gates | D |
| **Checkpoints/rewind** | ● | ◐ | ? | ◐ | ◐ | ● | Git-backed checkpoints | B/D |
| **Public API/SDK** | ● (Agent SDK) | ● | ? | ● (BG agent API) | ● (server+SDK) | ● (SDK) | **Server-first, API is primary** | D |

### Reading the matrix

Three observations that matter more than the individual cells:

1. **The MCP column is solid ● across the board.** This is the strongest possible argument for the brief's §6 instruction: consume MCP, don't reinvent it. Every competitor has already paid the integration cost and the server ecosystem is in the thousands. This is our single biggest free lunch.

2. **Nobody has a real capability-based model router.** Every system either pins models manually or per-agent. OpenCode and Cline are *multi-provider* but that's provider abstraction, not routing. This is genuine open ground — though see [Document 8](08-critique.md) §8.5 for why it's harder than it looks.

3. **Sandboxing default-on is rare.** Only Codex. Claude Code's own docs say its sandbox is a separate opt-in feature. If we ship default-on isolation we are ahead of most of the field on the axis that will matter most when something goes wrong.

---

## 1.9 What each competitor teaches us

| System | The one lesson |
| --- | --- |
| **Claude Code** | Context economy is the *point* of subagents. Also: its docs honestly enumerate multi-agent failure modes — free QA for our design. |
| **Codex** | Separate the sandbox axis from the approval axis. Cap delegation depth and thread count in config, not in prompts. |
| **Antigravity** | Humans cannot supervise N transcripts. Give them **artifacts** to review instead. |
| **Cursor** | Git worktrees are the cheapest real isolation. And parallelism should be opt-in, not default. |
| **OpenCode** | Build a **headless server** first. UIs are clients. Provider independence via a normalized SDK + capability catalogue. |
| **Cline** | Checkpoints and mistake limits. Plan/Act as an explicit state machine, not a vibe. |
