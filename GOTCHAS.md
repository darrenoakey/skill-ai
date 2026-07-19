# daz-agent-sdk — Gotchas & Known Pitfalls

## Anthropic OAuth
- Tokens (sk-ant-oat01-*) require `anthropic-beta: oauth-2025-04-20` header. Refresh via `POST https://platform.claude.com/v1/oauth/token` with `{"grant_type":"refresh_token","refresh_token":"<rt>","client_id":"9d1c250a-e61b-44d9-88ed-5944d1962f5e"}`. Rate limit timestamps are Unix epoch integers.

## claude_agent_sdk Inside Claude Code
- `ClaudeAgentOptions.env` CANNOT unset env vars (SDK merges `{**os.environ, **options.env}`). Pop `CLAUDECODE` from `os.environ` before connecting, restore in finally.
- `claude_agent_sdk` inside Claude Code produces `tool_use ids must be unique` errors — fundamental incompatibility. Use Go MCP binary over stdio JSON-RPC instead.
- Streaming mode: `stream_input()` closes stdin immediately when no hooks/MCP servers configured. Workaround: pass dummy hooks or use `claude --print` subprocess.
- `query()` yields 5 message types. Final answer is in `ResultMessage.result` — NOT in `AssistantMessage` TextBlocks.

## claude_agent_sdk — Python Runtime
- `async for msg in query()` crashes on unknown message types (e.g., `rate_limit_event`). Use manual `__anext__()` with try/except.
- `claude_agent_sdk` from sync code: wrap with `asyncio.run()`. Never use `subprocess.run(["claude", ...])` inside Claude Code sessions.
- `ProcessError` on non-zero exit can happen during teardown after success. Verify via DB/API rather than treating as failure.
- `query()` from FastAPI background thread: use `ThreadPoolExecutor` + `asyncio.run()` inside the thread.

## daz_agent_sdk
- In v0.2.13, `agent.conversation()` accepts `provider` and `model` but not `base_url`. A provider name in `~/.daz-agent-sdk/config.yaml` is not usable unless it is registered by the installed SDK; unknown names fail at `chat.say()` with `Provider '<name>' is not available`. Route through a registered provider (for example `ollama`) whose configured endpoint matches the URL, and prove it with one real request—not only a signature test.
- `agent.ask()` prompts saying "Visit this URL" trigger tool calls instead of text blocks → `response.text == ""`. Phrase prompts to avoid tool use.
- Pydantic `schema=` parameter: response has `.parsed` attribute. `resp.text` may be empty — always use `resp.parsed`.
- `schema=` with Claude provider triggers agentic tool use (file writing) which is slow. Provider now sets `max_turns>=2` automatically so the file-write tool can execute, and falls back to parsing JSON from response text. For autonomous loops or high-throughput structured output: skip `schema=`, append JSON schema instructions to the prompt manually, and parse JSON from `result.text` using `parse_json_from_llm()` from `daz_agent_sdk.types`. This is 5-10x faster and avoids the agentic overhead.
- Claude provider now raises `AgentError(kind=INTERNAL)` on empty non-structured responses instead of returning empty string. This triggers the fallback retry system automatically.
- Still-image generation/editing is unconditionally disabled through daz-agent-sdk, regardless of legacy provider support. Use `/Users/darrenoakey/bin/generate_image` through Mac mini IGS. Arbiter background removal remains permitted only as non-generative processing of an existing image.
