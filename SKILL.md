---
name: ai
description: daz-agent-sdk — provider-agnostic AI for text, structured output, agentic work, TTS, STT, and programmatic still images. Use its public image API in version 0.2.17 or newer only as a client for the hardwired Mac mini IGS route; agents use the generate_image CLI.
---

# daz-agent-sdk — Programmatic AI Integration

Provider-agnostic AI library with tier-based routing and automatic fallback across Claude, Gemini, Codex, and Ollama. One import, one line per query. No API keys needed for Claude (ambient auth from Claude Code).

**If writing AI-powered code**, read `EXAMPLES.md` in this directory.
**If configuring, debugging errors, or using the CLI**, read `REFERENCE.md` in this directory.
**If hitting unexpected behavior**, read `GOTCHAS.md` in this directory.

## When This Skill Applies

Any time code needs programmatic AI: text generation, structured output, classification, summarization, agentic tool use, TTS, STT, or still images. For programmatic still generation/editing, require `daz-agent-sdk>=0.2.17` and use its public image API. That API is only a client for the same hardwired Mac mini IGS durable queue and logged-in Codex session; image provider, model, and server selectors are disabled. Agent-facing image work uses `/Users/darrenoakey/bin/generate_image` directly.

---

## Installation

```bash
# From the project directory
cd /Volumes/T9/darrenoakey/src/daz-agent-sdk
./run install          # installs system-wide as editable package

# Or via pip
pip install -e /Volumes/T9/darrenoakey/src/daz-agent-sdk
```

Requires Python 3.11+.

### Go

```bash
go get github.com/darrenoakey/daz-agent-sdk/go
```

Full Go port with text providers and TTS/STT subprocess wrappers. Use a public image API only if the installed Go SDK exposes the canonical `>=0.2.17` IGS client; otherwise route agent-facing work to `/Users/darrenoakey/bin/generate_image` and do not invent a direct provider integration.

---

## Imports

```python
from daz_agent_sdk import agent                          # ready-to-use singleton
from daz_agent_sdk import Agent, Conversation            # classes
from daz_agent_sdk import Tier, Capability, ErrorKind    # enums
from daz_agent_sdk import Response, StructuredResponse   # response types
from daz_agent_sdk import ImageResult, AudioResult       # capability results
from daz_agent_sdk import Message, ModelInfo, AgentError # supporting types
```

---

## Tiers

Tiers control which providers and models are used. The fallback engine cascades through the tier chain on transient errors.

| Tier | Models (in fallback order) | Use Case |
|------|---------------------------|----------|
| `Tier.VERY_HIGH` | claude-opus-4-6 → gpt-5.3-codex → gemini-2.5-pro | Critical, highest quality |
| `Tier.HIGH` | claude-opus-4-6 → gpt-5.3-codex → gemini-2.5-pro | Default — general purpose |
| `Tier.MEDIUM` | claude-sonnet-4-6 → gpt-5.3-codex → gemini-2.5-flash | Balanced speed/quality |
| `Tier.LOW` | claude-haiku-4-5 → gemini-2.5-flash-lite → ollama:qwen3-8b | Fast, cheap |
| `Tier.FREE_FAST` | ollama:qwen3-8b | Local only, zero cost |
| `Tier.FREE_THINKING` | ollama:qwen3-30b-32k → ollama:deepseek-r1:14b | Local reasoning, zero cost |

---

## Simple Ask (One-Shot)

```python
import asyncio
from daz_agent_sdk import agent, Tier

async def main():
    # Default tier (HIGH) — uses Claude, falls back to Gemini/Codex
    answer = await agent.ask("Explain quantum tunnelling in one paragraph")
    print(answer.text)

    # Cheap/fast tier
    answer = await agent.ask("Summarise this text: ...", tier=Tier.LOW)
    print(answer.text)

    # Free local model
    answer = await agent.ask("What is 2+2?", tier=Tier.FREE_FAST)
    print(answer.text)

    # With system prompt
    answer = await agent.ask(
        "Review this function for bugs",
        system="You are a senior code reviewer. Be concise.",
    )
    print(answer.text)

asyncio.run(main())
```

### Response Object

```python
answer = await agent.ask("Hello")
answer.text              # str — the response text
answer.model_used        # ModelInfo — which model actually responded
answer.model_used.provider        # "claude", "gemini", etc.
answer.model_used.qualified_name  # "claude:claude-opus-4-6"
answer.conversation_id   # UUID
answer.turn_id           # UUID
answer.usage             # dict — token counts etc.
```

---

## Do NOT Use

- **`claude_agent_sdk` directly** — daz-agent-sdk wraps it with fallback, tiers, and a cleaner API
- **Claude API / `anthropic` SDK** — no direct API calls; use daz-agent-sdk
- **`openai` SDK directly** — use daz-agent-sdk with Codex provider
- **`google.genai` directly** — use daz-agent-sdk with Gemini provider
- **Manual JSON parsing from AI responses** — use `schema=` with Pydantic models
- **Manual markdown stripping** — structured output handles this automatically

---

## Gotchas & Known Pitfalls

- **Models asked to "return valid JSON only" still emit invalid JSON two common ways**: (1) markdown fences around the object, and (2) literal control characters (real newlines/tabs) INSIDE string values — e.g. a two-paragraph value with a real blank line in it. Strict `JSON.parse`/`json.loads` rejects (2) ("Invalid control character"), so a naive parse-or-fallback returns the raw blob and it leaks downstream (beezle3 daily digest rendered `{"commentary":...}` verbatim and narrated it into a video). Every parse of model text needs BOTH fence-stripping and a repair pass that escapes control chars found inside string literals before giving up.
- **Still images have one route.** Agents use `/Users/darrenoakey/bin/generate_image`; programmatic applications may use the public image API in `daz-agent-sdk>=0.2.17`. Both target the same hardwired Mac mini IGS durable queue and logged-in Codex session. Image provider/model/server selectors, built-in/cloud image tools, direct provider APIs, Arbiter/Spark still jobs, and all backend fallbacks are disabled. Preserve one request/job identity through service failures, rate limits, and transient faults, waiting/retrying indefinitely by default. IGS handles prompt rewrites and internal-fault repair; callers never switch backend.
- **`boringstack` provider** (10.0.0.237 Ollama, hosts `qwen3.6:35b-a3b`) is a first-class provider in the SDK — use `boringstack:qwen3.6:35b-a3b` in tier chains. Tier-chain entries split on the FIRST colon, so a model id with its own colon is fine. A provider named `foo` must expose class `FooProvider`.
- **Go SDK provider registration is manual**: importing `github.com/darrenoakey/daz-agent-sdk/go` alone does not register concrete providers, so `agent.Ask` can fail with "All providers in chain failed" even when config and services are healthy. Register factories from `github.com/darrenoakey/daz-agent-sdk/go/provider` before asking, and for local Ollama tiers explicitly set `cfg.Providers["ollama"]["base_url"]` to a reachable host such as `http://10.0.0.42:11434`; the Go SDK may not support Python-only providers like `arbiter`.
- **Daemon/background Go SDK calls should pin a verified provider when fallback providers are not configured**: tier fallback can reach Gemini/OpenAI and then stop on missing API-key config even if Claude CLI auth works. For local daemons that should use subscription CLI auth, call `agent.Ask(..., sdk.WithAskProvider("claude"), sdk.WithAskModel("claude-sonnet-4-6"))` after registering the Claude provider instead of relying on the tier chain.
- **Claude Agent SDK may run its bundled CLI instead of the installed one**: the Python SDK resolves its bundled `claude` before `PATH`, so long-running daemons that need the user's current Claude Code subscription behavior should pass `ClaudeAgentOptions(cli_path=...)` to the installed `claude` executable. Also treat `ResultMessage.is_error` as failure even when the SDK subprocess exits 0.
- **Claude Agent SDK can emit raw `rate_limit_event` stream messages that its Python parser does not know about**: subscription CLI turns may succeed after this event, but `claude_agent_sdk` can raise `MessageParseError: Unknown message type: rate_limit_event` before yielding the final `ResultMessage`. Patch `claude_agent_sdk._internal.client.parse_message` to convert that event into a `SystemMessage` when building long-running daemon integrations.
- **Editable install of the SDK into a relocated venv**: if a venv's `bin/pip` shebang is stale (venv built on a different volume path), run `<venv>/bin/python -m pip install -e ~/src/daz-agent-sdk --no-deps` (the SDK declares a local-only `arbiter-client` dep that isn't on PyPI; `--no-deps` avoids the resolver failure).
- **Reasoning-mode local models (e.g. qwen3.6 thinking tier) fail long-form generation two ways that think-tag stripping does NOT catch**: (1) the entire response is the model's own planning notes (numbered, markdown-bold "1. **Analyze User Input:** - **POV/Tense:**..." restating the prompt) with NO `<think>` wrapper at all; (2) the visible answer is truncated because the thinking budget crowded it out — either drastically short (94 words vs a 1500-word target) or full-length but stopping mid-clause on a bare comma. Detect both content-wise (meta-marker/numbered-bold-header regex for (1); word-count floor + last-char-not-in-`.!?"”’)—…` for (2)) and retry: once same-tier with a reinforced instruction, then once on a non-reasoning tier, then hard-fail — never silently keep the broken text. Observed on noveliser2 write_section (src/write_section.py `_invalid_prose_reason`); regression tests must use the ACTUAL captured bad payloads.
- **Two prompt constraints where one is a blanket ban and the other needs an exception: the ban silently wins.** A revise-prompt said "do NOT change the scene's events or alter its length" AND "weave in these missing planned beats" — the model obeyed the ban and never restored the beats (measured on real runs: beats stayed missing after every revision pass). When an instruction needs to override a general rule, state the carve-out explicitly inside the rule itself ("the ONE exception: the missing beats below MUST be added, even though the section will grow"), don't rely on the model to reconcile the contradiction.
