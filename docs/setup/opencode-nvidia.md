# Local setup — OpenCode + NVIDIA Build

**Verified against the live catalogue on 2026-08-13.** Windows paths; Linux/WSL noted where they differ.

This gets you a working agent on free cloud inference. It is M0's first deliverable — everything else in this project is theory until you're coding with an agent daily.

---

## 1. NVIDIA API key

1. https://build.nvidia.com → sign up (free, no card)
2. Generate an API key — starts with `nvapi-`
3. **The key is shown once.** Store it in a password manager, never in the repo.

> ⚠️ **Never paste a key into a screenshot, issue, or config you might commit.** If one leaks, revoke it immediately in your account's API Keys page and generate a new one. `.gitignore` already excludes `auth.json` and `.env*`, but the reliable protection is not putting keys in files at all — see step 4.

## 2. Install OpenCode

Requires Node 18+.

```powershell
npm i -g opencode-ai
opencode --version
```

## 3. Config

Create `C:\Users\<YourName>\.config\opencode\opencode.json`

That's `.config` in your **user folder**, not `%APPDATA%`. OpenCode's docs and some plugins disagree on this — it's a known reported discrepancy. If the config isn't picked up, try `%APPDATA%\opencode\`. On Linux/WSL it's `~/.config/opencode/opencode.json`.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "nvidia": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "NVIDIA NIM",
      "options": {
        "baseURL": "https://integrate.api.nvidia.com/v1"
      },
      "models": {
        "z-ai/glm-5.2": { "name": "GLM-5.2" },
        "openai/gpt-oss-120b": { "name": "gpt-oss-120b" },
        "nvidia/nvidia-nemotron-nano-9b-v2": { "name": "Nemotron Nano 9B" }
      }
    }
  }
}
```

**Confirm each model ID against its page's code sample** (`model="..."`). The IDs above were derived from catalogue URL slugs, and NVIDIA's slugs don't always match the API string exactly.

## 4. Add the key — use the CLI

```powershell
opencode auth login     # choose nvidia, paste the key
```

**Do this instead of hand-writing `auth.json`.** On Linux it lives at `~/.local/share/opencode/auth.json`, but the Windows location is exactly where the path discrepancy above bites. `auth login` writes it correctly and keeps the key out of any file you might commit.

## 5. Run

```powershell
cd C:\path\to\a\project
opencode
```

```
/models
/model nvidia/z-ai/glm-5.2
```

## 6. Verify — tool calling, not chat

**Chat proves nothing.** What Orchestra depends on is tool calling inside a loop. Give it a real task in a real repo:

> "Read the files in this project, find where X is defined, add a function that does Y, and run the tests."

Three things must happen:

- [ ] it **reads files** without being given paths
- [ ] it **edits** and the edit lands correctly
- [ ] it **runs a command** and reacts to the output

If it chats but never calls a tool, the model isn't doing function calling properly — switch models before concluding anything about OpenCode.

---

## Model notes (verified 2026-08-13)

| Model | Why |
| --- | --- |
| **`z-ai/glm-5.2`** ← driver | *"flagship LLM for agentic workflows, coding, and long-horizon reasoning."* Tagged `agentic ai`. Free endpoint. Released ~1 month prior. |
| `openai/gpt-oss-120b` | MoE reasoning, free endpoint. Alternate. |
| `nvidia/nvidia-nemotron-nano-9b-v2` | *"excelling in reasoning and agentic tasks."* Small and fast — candidate for the cheap tier. |

> **The catalogue's `Code Generation` filter is under-curated** — it returns only 5 old Llama models and reports `Publisher: NVIDIA 0`, which is obviously wrong on NVIDIA's own site. **Search unfiltered.** GLM-5.2 does not appear under that filter but is present and free.

## Limits

Free tier: **1,000 credits** on signup (up to 5,000 on request), **40 requests/minute**.

Fine for single-agent work. Agentic loops are request-heavy, so you will feel 40 RPM if you push hard — that's the failover case in [ADR-0004](../decisions/ADR-0004-cloud-inference-only.md). Add a second provider as another `provider` block when needed.

`nemotron-nano-9b-v2` may cover the cheap tier well enough that Groq isn't needed. Worth testing before adding a second provider.
