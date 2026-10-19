# Embedding route per provider: Agent SDK vs spawning the CLIs

Status: research note. Resolves [wayfinder ticket #45](https://github.com/alilloig/trade-wars/issues/45). Date: 2026-10-19.

Scope: a desktop app (TypeScript, pnpm) that runs a game harness for the cheapest models: Claude Haiku 5.5 / Sonnet 5.5 and OpenAI `gpt-6.1-sol`. Five routes are compared: Anthropic (A1) the Claude Agent SDK for TypeScript, (A2) the `claude` CLI headless with `stream-json`; OpenAI (O1) `codex exec`, (O2) the Codex app-server protocol, (O3) the Codex TypeScript SDK.

The lever list comes from section 7 of `docs/trade-wars-in-claude-code.html` at commit `c7724ae`. `docs/README.md` at commit `6cee6ee` (branch `feat/revisit-trade-wars-design`) marks that document superseded on 2026-10-19. The levers stay valid as targets. The mod API names in its "How" column (`omitClaudeMd`, `$.model.complete`, `$.model.classify`) exist only inside the Claude Code mod runtime, so this note maps each lever by intent.

Every claim carries a source and a date. `measured` = a local probe run on 2026-10-19, single sample, list prices. UNVERIFIED marks a claim that no primary source confirmed.

## TL;DR

- Both Anthropic routes run the same binary. The Agent SDK spawns a bundled 256 MB `claude` executable and drives it with `--input-format stream-json --output-format stream-json` (`measured`; [TS options reference](https://code.claude.com/docs/en/agent-sdk/typescript), 2026-10-19). The SDK adds in-process tool handlers, hook callbacks and a pinned CLI version. Model pin, allowlist, `CLAUDE.md` omission and per-token cost are identical on A1 and A2.
- Recommended Anthropic route: **A1, the Agent SDK, with API-key auth**, for the agentic tier. For the Tier 1 decider, no listed route keeps the "no agent for simple forks" lever; only the raw Messages API does (see section 7).
- Recommended OpenAI route: **O3, the Codex SDK**, with the app's tools exposed as a local MCP server. O2 (app-server) is the only OpenAI route with in-process tools and no respawn, and it is the only route OpenAI names for "Sign in with ChatGPT", but OpenAI documents it as experimental and "not supported for production workloads" ([app-server docs](https://developers.openai.com/codex/app-server), 2026-10-19). For the Tier 1 decider the raw Responses API is again the only route that keeps every lever.
- Levers: all five routes keep the four harness-level levers (preloaded state, fixed-format state, gate in code, two RPC round-trips) and "bounded answers". A1 and A2 keep "no project instructions" and "minimal tool list" fully. O1, O2 and O3 keep "no project instructions" with config overrides and keep "minimal tool list" only partially: a 9.5k to 23k input-token floor remains after every documented switch is off (`measured`). No route keeps "no agent for simple forks".
- Licence: Codex CLI and SDK are Apache-2.0 and redistributable with the licence text and NOTICE. Claude Code and the Agent SDK are proprietary ("© Anthropic PBC. All rights reserved."); redistribution inside a product is granted under the Commercial Terms with conditions: unmodified binary, no built-in auth method removed, no paying for or intermediating usage on behalf of users, each user on their own credentials, no "Claude Code" branding. Auth: Anthropic tells developers, "including those using the Agent SDK", to use API keys and forbids offering claude.ai login or routing through plan credentials "on behalf of their users"; a user may still sign in to the unmodified binary with their own subscription. OpenAI states app-server authentication "has never been permitted for commercial or hosted services" and offers "Sign in with ChatGPT" plan usage for "open-source and locally hosted apps"; paid or hosted apps need the partner form. OpenAI's consumer Terms of Use could not be fetched (HTTP 403): UNVERIFIED.

## 1. Method

Local versions on 2026-10-19: Claude Code CLI `2.1.303`, `@anthropic-ai/claude-agent-sdk` `0.3.303`, Codex CLI `codex-cli 0.163.0`, `@openai/codex-sdk` `0.163.0`, pnpm `11.22.0`, Node from the user's shell.

Primary sources: the SDK type declarations and package files installed from npm; the local `--help` output of both CLIs; the app-server JSON schema generated locally with `codex app-server generate-json-schema`; the official docs at `code.claude.com`, `developers.openai.com` (the `platform.openai.com` and `developers.openai.com/codex` URLs redirect there) and `github.com/openai/codex` at tag `rust-v0.163.0`; the legal pages of both vendors; and the `claude-api` skill model table (cached 2026-10-06) for Anthropic list prices.

Probes: one headless decision per route with the same prompt shape and the same JSON schema (`move` enum `sell|hold|ship`, `reason` string, `additionalProperties: false`). Scripts and raw outputs are in Appendix A. Each number is one sample and is not a benchmark.

## 2. The section 7 levers

| Lever | Intent, mod-independent | Route-dependent? |
| --- | --- | --- |
| No project instructions | The decider never sees `CLAUDE.md` / `AGENTS.md`, user settings, hooks or plugins. | Yes |
| Minimal tool list | Only the game tools are visible. Every visible tool description costs tokens each turn. | Yes |
| No agent for simple forks | A plain completion: identity block, state line, answer. No harness system prompt, no history, no tools. | Yes |
| Bounded answers | A label set or a JSON schema. No parsing, no retry on malformed output. | Yes |
| Preloaded state | The state line is injected into the prompt. No discovery round-trips. | No, harness-level |
| Fixed-format state | One record per planet, fixed field order, format explained once. | No, prompt-level |
| Gate in code | Preflight quotes and policy guards run in code. A tick with nothing to do calls no model. | No, harness-level |
| Two RPC round-trips | Snapshot views on chain (spec Phase 1). | No, on-chain |

The last four levers live in the harness code, the prompt text or the chain, not in the embedding layer. Every route keeps them. The rest of this note scores the first four.

## 3. Anthropic routes

### 3.1 What the Agent SDK is

- [Hosting docs](https://code.claude.com/docs/en/agent-sdk/hosting) (2026-10-19): "When your code calls `query()`, the SDK spawns a separate `claude` CLI process and talks to it over stdio." and "One agent session maps to one subprocess."
- [TS reference](https://code.claude.com/docs/en/agent-sdk/typescript) (2026-10-19): "The SDK bundles a native Claude Code binary for your platform as an optional dependency such as `@anthropic-ai/claude-agent-sdk-darwin-arm64`. Most installs need no separate Claude Code install. The SDK version tracks the bundled Claude Code version." Options row: `pathToClaudeCodeExecutable` "Auto-resolved from bundled native binary".
- `measured`: the child process was `.../@anthropic-ai/claude-agent-sdk-darwin-arm64/claude --output-format stream-json --verbose --input-format stream-json --effort=low --max-turns=4 --max-budget-usd=0.05 --model=claude-haiku-5-5 --json-schema={...} --allowedTools=mcp__tw__tw_state --tools= --setting-sources= --strict-mcp-config --no-session-persistence`.
- `package.json` 0.3.303: `optionalDependencies` list eight platform packages. The darwin-arm64 package holds one file `claude`, a Mach-O arm64 executable of 267,973,136 bytes, `"description": "Native CLI binary for @anthropic-ai/claude-agent-sdk on darwin-arm64"`, `"license": "SEE LICENSE IN LICENSE.md"`. The SDK package itself has no `cli.js`; [CHANGELOG 0.2.113](https://github.com/anthropics/claude-agent-sdk-typescript) (2026-10-19): "Changed the SDK to spawn a native Claude Code binary (via a per-platform optional dependency) instead of bundled JavaScript".
- `sdk.d.ts` 0.3.303 also exposes `spawnClaudeCodeProcess?: (options: SpawnOptions) => SpawnedProcess` "Custom function to spawn the Claude Code process", `startup()` ([CHANGELOG 0.2.89](https://github.com/anthropics/claude-agent-sdk-typescript): "pre-warm the CLI subprocess before `query()`, making the first query ~20x faster") and `prewarm()` ("A spare holds roughly 230 to 260 MB of memory while it waits", TS reference).

Consequence: A1 and A2 are the same binary with the same flags. A1 adds three things: tool handlers that run in the host process (`measured`: the `tw_state` handler ran in the Node process), hook callbacks as functions (`measured`: `PreToolUse`, `PostToolUse`, `Stop` fired in-process), and a CLI version pinned to the app instead of whatever the user installed. The SDK's in-process MCP server and hooks travel over the same stdio stream as `control_request` / `control_response` lines; that control protocol is not documented as a supported interface for a non-SDK host (UNVERIFIED).

### 3.2 Comparison table, Anthropic

| Criterion | A1. Agent SDK (TypeScript) | A2. `claude -p` headless, `stream-json` |
| --- | --- | --- |
| Model pin | `model: 'claude-haiku-5-5'` accepted as a full ID (`measured`: `init.model`). [Model config](https://code.claude.com/docs/en/model-config) (2026-10-19): "Its model ID is `claude-haiku-5-5`"; "Sonnet 5.5 requires Claude Code v2.1.284 or later ... Use v2.1.293 or later with Haiku 5.5." The exact Sonnet 5.5 ID string is not printed on that page (UNVERIFIED there; the `claude-api` skill table, cached 2026-10-06, gives `claude-sonnet-5-5`). `fallbackModel`, `effort: 'low'..'max'`, `thinking`, `betas` in `sdk.d.ts`. | `--model claude-haiku-5-5` ("an alias ... or a model's full name", `--help` 2.1.303), `--fallback-model`, `--effort`, `--betas` ("API key users only"). |
| Custom tools | `createSdkMcpServer({ name, tools: [tool(name, description, zodShape, handler)] })` in `mcpServers: { tw }`. [Custom tools](https://code.claude.com/docs/en/agent-sdk/custom-tools) (2026-10-19): "The server runs in-process inside your application, not as a separate process." Name: `mcp__tw__tw_state` (`measured`). Tool search defers SDK MCP tool schemas by default; `alwaysLoad: true` or `ENABLE_TOOL_SEARCH=false` keeps full schemas in the prompt ([tool search](https://code.claude.com/docs/en/agent-sdk/tool-search), 2026-10-19: "With fewer than ~10 tools ... loading everything upfront is typically faster"). | External MCP server only: `--mcp-config <file-or-json>` + `--strict-mcp-config` ([CLI reference](https://code.claude.com/docs/en/cli-reference), 2026-10-19). The desktop app must expose its tools as a stdio or HTTP MCP server. |
| Tool allowlist | `tools: []` "All built-ins are removed. Claude can only use your MCP tools." `allowedTools` is a permission list, not a visibility list: "Listed tools run without a permission prompt. Other unlisted tools remain available" ([custom tools](https://code.claude.com/docs/en/agent-sdk/custom-tools), 2026-10-19). `measured`: `init.tools` = `["StructuredOutput","mcp__tw__tw_state"]`. Set `permissionMode` explicitly; since v0.3.286 the omitted default can be auto mode ([permissions](https://code.claude.com/docs/en/agent-sdk/permissions), 2026-10-19). | `--tools ""` ("Use \"\" to disable all tools"; "doesn't affect MCP tools; to deny those too, use `--disallowedTools \"mcp__*\"`"), `--allowedTools`, `--disallowedTools` (CLI reference, `--help` 2.1.303). `measured`: `init.tools` = `[]`. |
| Omit project instructions | `settingSources: []`. `sdk.d.ts`: "Pass `[]` to disable filesystem settings (SDK isolation mode). Must include `'project'` to load CLAUDE.md files." Default when omitted: "all sources are loaded (matches CLI defaults)" ([Claude Code features](https://code.claude.com/docs/en/agent-sdk/claude-code-features), 2026-10-19). Not covered by `settingSources: []`: managed policy, `~/.claude.json`, auto memory (`CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`), claude.ai connectors (`strictMcpConfig: true`). `CLAUDE_CODE_DISABLE_CLAUDE_MDS=1` removes every CLAUDE.md ([system prompts](https://code.claude.com/docs/en/agent-sdk/modifying-system-prompts), 2026-10-19). | `--setting-sources ""` (`measured`, works on subscription auth) or `--bare`: "skip hooks ..., LSP, plugin sync, attribution, auto-memory, background prefetches, keychain reads, and CLAUDE.md auto-discovery ... Anthropic auth is strictly ANTHROPIC_API_KEY or apiKeyHelper" (`--help` 2.1.303). [Headless docs](https://code.claude.com/docs/en/headless) (2026-10-19): "`--bare` is the recommended mode for scripted and SDK calls, and will become the default for `-p` in a future release." |
| System prompt | `systemPrompt: string` "The SDK sends only what you provide". Omitted: "a minimal prompt that covers tool calling but omits the rest of the `claude_code` preset's content ... This differs from `claude -p`" (system prompts page). Preset form `{ type:'preset', preset:'claude_code', append, excludeDynamicSections, snapshot }`; `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` splits a custom prompt into a cacheable prefix and a dynamic suffix (`sdk.d.ts`). | `--system-prompt` "Replace the entire system prompt", `--append-system-prompt`, `--system-prompt-snapshot on\|off`, `--exclude-dynamic-system-prompt-sections` (CLI reference). |
| Bounded answers | `outputFormat: { type:'json_schema', schema }`; validated data in `result.structured_output`; error subtype `error_max_structured_output_retries`; a `success` result with no `structured_output` is a failure ([structured outputs](https://code.claude.com/docs/en/agent-sdk/structured-outputs), 2026-10-19). Implemented as a tool, not as API `output_config`: `sdk.d.ts` calls these "End-turn tool sessions (`outputFormat: {type: 'json_schema'}` ...)"; `measured`: `init.tools` includes `StructuredOutput`, the model emits `tool_use:StructuredOutput`, `num_turns` 2-3. A label set is an `enum` in the schema. No classify primitive. | `--json-schema '<schema>'` with `--output-format json`; same `StructuredOutput` tool (`measured`). |
| Hooks / events for a UI | `hooks: { PreToolUse: [{ matcher, hooks: [cb] }], ... }`; 33 events in `HOOK_EVENTS` (`sdk.d.ts`); "Hooks run in your application process" ([agent loop](https://code.claude.com/docs/en/agent-sdk/agent-loop), 2026-10-19). `canUseTool` callback; `includePartialMessages` yields `stream_event` with raw API stream events; `SDKMessage` union covers `system/init`, `assistant`, `user`, `result`, `rate_limit_event`, status and task messages. | Same NDJSON lines on stdout (`measured`: `system/init`, `assistant`, `rate_limit_event`, `result`). Hooks only as settings-file commands (`--include-hook-events` to see them). `--permission-prompts host\|none`, `--permission-prompt-tool <mcp tool>`, `--include-partial-messages`, `--replay-user-messages` (CLI reference). |
| Cost / latency fields | `result`: `total_cost_usd`, `duration_ms`, `duration_api_ms`, `num_turns`, `usage`, `modelUsage[model].{inputTokens, outputTokens, thinkingTokens, cacheReadInputTokens, cacheCreationInputTokens, costUSD, costBasis}`, `ttft_ms` (TS reference). [Cost tracking](https://code.claude.com/docs/en/agent-sdk/cost-tracking) (2026-10-19): "The `total_cost_usd` and `costUSD` fields are client-side estimates, not authoritative billing data." Budget: `maxBudgetUsd`, `maxTurns`. | Same fields in the `result` line; `--max-budget-usd`, `--max-turns` (print mode). |
| Cost / latency `measured` | Haiku 5.5, custom prompt, `tools: []`, one MCP tool, JSON schema: 0.00053 USD, 3 turns, 2,609 input tokens (1,366 cache write + 1,243 cache read), 245 output (94 thinking), `duration_api_ms` 3,745, wall 7,444 ms including spawn and MCP init. | Haiku 5.5, custom prompt, `--tools ""`, JSON schema: 0.00079 USD, 3 turns, `duration_ms` 4,014, wall 5,463 ms. One-word answer, no schema: 0.00024-0.00035 USD, ~1,120 input tokens, `duration_ms` ~1,000. |
| Harness floor `measured` | ~1,120 input tokens with a custom system prompt and no tools; ~4,350 with the default `claude -p` prompt and no tools; ~18,500 with the default tool set (cache write + cache read, Haiku 5.5). `--bare` was not measured: it needs an API key. | Same binary, same numbers. |
| Session model | One subprocess per `query()`. `persistSession: false` (default `true`, writes `~/.claude/projects/`). Streaming input (`prompt` as an AsyncIterable, `streamInput()`) keeps one process for many turns; each turn emits its own `result` (cost tracking page). | One process per invocation, or one persistent process with `--input-format stream-json`. `measured`: two user turns in one process, each with its own `system/init` and `result`; turn 2 carried turn 1 history (1,278 vs 1,166 input tokens); 3,309 ms wall for both turns, ~1,000 ms API each. |
| Prompt caching | Automatic. Default TTL 5 minutes on an API key; `ENABLE_PROMPT_CACHING_1H=1` or `CLAUDE_CODE_PROMPT_CACHE_TTL=1h` in `env` (cost tracking page). `measured` on subscription auth: `cache_creation.ephemeral_1h_input_tokens` was used. | Same. |
| Auth mode | `ApiKeySource` includes `'oauth'` and `'none'` (`sdk.d.ts`). `measured`: `apiKeySource: "none"` with no `ANTHROPIC_API_KEY`, a `rate_limit_event` with `rateLimitType: five_hour`, `costBasis: "list"`: the probe ran on the logged-in subscription. Policy: see section 6. | Same. `--bare` forces `ANTHROPIC_API_KEY` or `apiKeyHelper`; "Bare mode does not read `CLAUDE_CODE_OAUTH_TOKEN`" ([authentication](https://code.claude.com/docs/en/authentication), 2026-10-19). |
| Licence | `LICENSE.md` in `@anthropic-ai/claude-agent-sdk@0.3.303` and in the platform binary package, verbatim: "© Anthropic PBC. All rights reserved. Use is subject to the Legal Agreements outlined here: https://code.claude.com/docs/en/legal-and-compliance." npm `license`: `SEE LICENSE IN README.md`. See section 6. | `@anthropic-ai/claude-code@2.1.303` npm `license`: `SEE LICENSE IN README.md`; same legal page. |

## 4. OpenAI routes

### 4.1 What each route is

- O1 `codex exec`: one process per call; `--json` prints JSONL events (`thread.started`, `turn.started`, `item.started|updated|completed`, `turn.completed {usage}`, `turn.failed`, `error`), `--output-schema <FILE>`, `-o/--output-last-message`, `--ephemeral`, `--ignore-user-config`, `--ignore-rules`, `-c key=value`, `--disable <FEATURE>` (`codex exec --help` 0.163.0; [non-interactive docs](https://developers.openai.com/codex/noninteractive), 2026-10-19).
- O2 app-server: `codex app-server` speaks JSON-RPC over stdio (`--listen stdio://|unix://|ws://`), one process hosts many threads ([app-server docs](https://developers.openai.com/codex/app-server), 2026-10-19: "`thread/loaded/list` returns thread IDs currently loaded in memory"). Marked `[experimental]` in `--help`; the docs say it is "experimental and not supported for production workloads" and "If you are automating jobs or running Codex in CI, use the Codex SDK instead." The Python SDK drives the app-server; the TypeScript SDK does not.
- O3 `@openai/codex-sdk`: "wraps the `codex` CLI from `@openai/codex`. It spawns the CLI and exchanges JSONL events over stdin/stdout" ([sdk/typescript/README.md](https://github.com/openai/codex/blob/main/sdk/typescript/README.md), 2026-10-19). `exec.ts`: spawns `["exec", "--experimental-json", ...]` per `run()`, resolves the binary from `@openai/codex` platform packages (`vendor/<triple>/bin/codex`), never searches `PATH`; `codexPathOverride` bypasses that. `@openai/codex-sdk@0.163.0` depends on `@openai/codex@0.163.0`, which pulls `@openai/codex-{linux,win32,darwin}-{x64,arm64}` as optional dependencies (`npm view`, 2026-10-19).

### 4.2 Comparison table, OpenAI

| Criterion | O1. `codex exec --json` | O2. app-server (JSON-RPC) | O3. Codex SDK (TypeScript) |
| --- | --- | --- | --- |
| Model pin | `-m gpt-6.1-sol` or `-c model="gpt-6.1-sol"`; `-c model_reasoning_effort="low"` ([config reference](https://developers.openai.com/codex/config-reference), 2026-10-19). `codex debug models` lists `gpt-6.1-sol` with `default_reasoning_level: "low"`, efforts `low..ultra` (local, 2026-10-19). | `thread/start.model`, `turn/start.model`, `turn/start.effort` (generated schema v2, 2026-10-19; `measured`: `thread/start` with `model: 'gpt-6.1-sol'`, `turn/start` with `effort: 'low'`). `model/list` returns models and efforts. | `ThreadOptions.model` → `--model`; `modelReasoningEffort` → `--config model_reasoning_effort="..."` (`threadOptions.ts`, `exec.ts`). The SDK type lists `minimal`, which the `gpt-6.1-sol` model page rejects. |
| Custom tools | MCP servers from `config.toml` or `-c mcp_servers...` (stdio `command`/`args`/`env`, streamable HTTP `url`); per server `enabled_tools` "Tool allow list.", `disabled_tools` "Tool deny list (applied after `enabled_tools`)." (config reference). No in-process tools. | `thread/start.dynamicTools` (experimental, needs `initialize.capabilities.experimentalApi: true`): `{ type:'function', name, description, inputSchema, deferLoading? }`; the server sends request `item/tool/call {threadId, turnId, callId, tool, namespace?, arguments}`; the client answers `{ contentItems:[{type:'inputText', text}], success }` (generated schema; `codex-rs/protocol/src/dynamic_tools.rs` at `rust-v0.163.0`). `measured`: `tw_state` was called back and answered in the host process on both decisions. | No tool API. `CodexOptions` = `codexPathOverride, baseUrl, apiKey, config, configOverrides, env`; tools only through `config: { mcp_servers: {...} }` flattened to `--config` (`codexOptions.ts`, README). |
| Tool allowlist / shell removal | `features.shell_tool` "Enable command execution tools (stable; on by default)." `codex-rs/core/src/tools/spec_plan.rs` registers no shell tool when the feature is off. `measured`: `--disable shell_tool --disable unified_exec ...` plus `model_instructions_file` dropped input from 22,966 to 13,483 tokens; `--ignore-user-config` then to 9,521. Web search is on by default ("cached"); `web_search = "disabled"` turns it off ([web search docs](https://developers.openai.com/codex/web-search), 2026-10-19). | Same config keys via `thread/start.config` (type verified in `v2/thread.rs`; semantics undocumented, UNVERIFIED) or the daemon's `-c` flags. `measured`: with `project_doc_max_bytes=0` only, 18,853 input tokens per decision, of which 18,560 cache hits; user MCP servers were started (`mcpServer/startupStatus/updated` events). | `config: { features: { shell_tool: false }, web_search: 'disabled', ... }`; `sandboxMode: 'read-only'`; `webSearchMode: 'disabled'`; `approvalPolicy` (`threadOptions.ts`). No `--ignore-user-config` exposure. |
| Omit project instructions | `-c project_doc_max_bytes=0`: `agents_md.rs` line 133 returns `None` when the budget is 0 (source, `rust-v0.163.0`). `measured`: the user-level `~/.codex/AGENTS.md` still appeared in `codex debug prompt-input` with that override; `--ignore-user-config` removed it (mechanism UNVERIFIED). `model_instructions_file` "Replacement for built-in instructions instead of `AGENTS.md`." `developer_instructions` adds a separate developer message (config reference). | `thread/start.baseInstructions`, `developerInstructions`, `config` (schema; `measured`: `baseInstructions` accepted). `ThreadStartResponse.instructionSources` lists the instruction files loaded. | Via `config` overrides only (`project_doc_max_bytes: 0`, `model_instructions_file`). |
| Bounded answers | `--output-schema <FILE>`. Enforced with Responses API `text.format = { type: json_schema, strict: true, schema, name: "codex_output_schema" }` (`codex-rs/codex-api/src/common.rs` lines 392-410, `rust-v0.163.0`). `measured`: the final `agent_message` parsed against the schema in one model call on every run. | `turn/start.outputSchema` "Optional JSON Schema used to constrain the final assistant message for this turn." Final text in `item/completed` (`agentMessage`), streamed as `item/agentMessage/delta` (`measured`). | `thread.run(prompt, { outputSchema })`; the SDK writes a temp file and passes `--output-schema`; `finalResponse` is the last `agent_message` text (`thread.ts`). |
| Hooks / events for a UI | JSONL item types `agent_message`, `reasoning`, `command_execution`, `file_change`, `mcp_tool_call`, `collab_tool_call`, `web_search`, `todo_list`, `error` (`codex-rs/exec/src/exec_events.rs`). Config `hooks` (`PreToolUse, PermissionRequest, PostToolUse, PreCompact, PostCompact, UserPromptSubmit, SubagentStop, Stop, Interrupt, SessionStart, SubagentStart, SessionEnd`) run as commands and need trust ([hooks docs](https://developers.openai.com/codex/hooks), 2026-10-19); `notify` fires `agent-turn-complete`. | Notifications `thread/started`, `turn/started`, `item/started`, `item/completed`, `item/agentMessage/delta`, `item/reasoning/*Delta`, `thread/tokenUsage/updated`, `turn/completed`, `hook/started`, `hook/completed`, `account/rateLimits/updated`; server requests `item/commandExecution/requestApproval`, `item/tool/call` (generated schema; `measured` event list in Appendix A). `initialize.capabilities.optOutNotificationMethods` silences named notifications. | `runStreamed()` yields the same `ThreadEvent` union as O1 (`events.ts`). |
| Cost / latency fields | `turn.completed.usage = { input_tokens, cached_input_tokens, cache_write_input_tokens, output_tokens, reasoning_output_tokens }`. No duration field. | `thread/tokenUsage/updated { tokenUsage: { last, total, modelContextWindow } }` with the same breakdown; `turn/completed.turn.durationMs`. There is no `thread/tokenCount` in 0.163.0. | `Turn.usage` (same shape). |
| Cost / latency `measured` | 22,966 in / 79 out, wall 7,506 ms (user config loaded); 13,483 in / 49 out, 5,117 ms (tools stripped); 9,521 in / 92 out, 6,142 ms (+ `--ignore-user-config`). `cached_input_tokens` 0 on all three. | 18,853 in (18,560 cached) / 96 out, `durationMs` 8,647, wall 9,355 ms; second thread 18,853 in (18,560 cached) / 54 out, 8,817 / 8,905 ms. Each decision made two model calls (tool call, then answer). `initialize` took 185 ms. | Not run. Same path as O1 (`codex exec --experimental-json`), so the O1 numbers apply plus Node spawn overhead. |
| Process model | New process per call. | One daemon; `thread/start` per decision gives a fresh context without respawn; `ephemeral: true` skips rollout files. | New `codex exec` process per `run()` (`exec.ts`). |
| Auth mode | `codex login` (ChatGPT plan) or `codex login --with-api-key` / `CODEX_API_KEY`; "`codex exec` reuses saved CLI authentication by default." "API keys are still the recommended default for automation." ([auth docs](https://developers.openai.com/codex/auth), 2026-10-19). `--ignore-user-config` keeps auth on `CODEX_HOME`. `measured`: `codex login status` = "Logged in using ChatGPT". | `account/login/start` with `apiKey`, `chatgpt`, `chatgptDeviceCode` or experimental `chatgptAuthTokens`; "Codex owns the ChatGPT OAuth flow, persists tokens, and refreshes them automatically." (app-server docs). | `apiKey` option sets `CODEX_API_KEY`; otherwise inherits the environment and `CODEX_HOME` auth (`exec.ts`). |
| Licence | Repo `LICENSE`: Apache License 2.0; `NOTICE`: "OpenAI Codex / Copyright 2025 OpenAI" plus a Ratatui MIT attribution (tag `rust-v0.163.0`, 2026-10-19). `@openai/codex@0.163.0` `license: Apache-2.0`. | Same binary. | `@openai/codex-sdk@0.163.0` `license: Apache-2.0` (`npm view`, 2026-10-19). |

### 4.3 `gpt-6.1-sol` price and the per-decision cost

[Model page](https://developers.openai.com/api/docs/models/gpt-6.1-sol) (2026-10-19): context window 1,050,000, max output 128,000; structured outputs, function calling, streaming and prompt caching supported; reasoning effort `low`, `medium` (API default), `high`, `xhigh`, `max`; `none` and `minimal` not supported. [Pricing page](https://developers.openai.com/api/docs/pricing) (2026-10-19), Standard tier, per 1M tokens, "short context (≤272K input tokens)": input 2.00 USD, cached input 0.10 USD, output 10.00 USD. The cheapest documented neighbour on the same page is `gpt-6-luna` at 0.10 / 0.01 / 0.50 USD.

Anthropic list prices from the `claude-api` skill table (cached 2026-10-06), per 1M tokens: Haiku 5.5 0.10 / 0.50 USD (prompts up to 100K); Sonnet 5.5 2.00 / 10.00 USD, cache reads 0.20 USD.

Per-decision list cost computed from the measured token counts:

| Route and configuration | Input (cached) | Output | List cost per decision |
| --- | --- | --- | --- |
| A1/A2 Haiku 5.5, custom prompt, no tools, one word | ~1,120 (0) | 4 | 0.00024-0.00035 USD (CLI estimate) |
| A1 Haiku 5.5, one MCP tool + schema, 3 turns | 2,609 (1,243) | 245 | 0.00053 USD (CLI estimate) |
| A1 Sonnet 5.5, same shape, computed (cache writes priced at the input rate, cache reads at 0.20 USD/M) | 2,609 (1,243) | 245 | ~0.0055 USD |
| O1 `codex exec`, user config, uncached | 22,966 (0) | 79 | ~0.047 USD |
| O1 `codex exec`, stripped, uncached | 9,521 (0) | 92 | ~0.020 USD |
| O2 app-server, cache hit | 18,853 (18,560) | 96 | ~0.0034 USD |

Reading: the Codex harness carries a 9.5k to 23k token floor that the Claude harness does not. Prompt caching cuts an O2 decision to roughly 7x a Haiku decision; an uncached `codex exec` decision costs 40x to 100x. On subscription logins neither vendor bills per token, but both apply plan rate limits (`measured`: Claude `rate_limit_event` `five_hour`; Codex `account/rateLimits/updated`).

## 5. Lever map

| Lever | A1 Agent SDK | A2 `claude -p` | O1 `codex exec` | O2 app-server | O3 Codex SDK |
| --- | --- | --- | --- | --- | --- |
| No project instructions | Kept: `settingSources: []` (must be set; default loads all), `strictMcpConfig`, `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`. | Kept: `--setting-sources ""` + `--strict-mcp-config`, or `--bare` (API key only). | Kept: `-c project_doc_max_bytes=0 --ignore-user-config --ignore-rules`, `model_instructions_file`. | Kept with caveat: `baseInstructions` + `config` on `thread/start`; the daemon still reads `~/.codex/config.toml` (user MCP servers started, `measured`). | Partial: `config` overrides only; no `--ignore-user-config` exposure, so user-level `AGENTS.md` and MCP servers load unless the user config is clean. |
| Minimal tool list | Kept: `tools: []`, in-process MCP tools, `alwaysLoad` for full schemas. Floor ~1.1k tokens. | Kept: `--tools ""`, external MCP server. Floor ~1.1k tokens. | Partial: shell removable (`features.shell_tool=false`), MCP tools only, floor ~9.5k tokens. | Partial: in-process dynamic tools (`measured`), same base prompt and floor (~18.9k with user config). | Partial: MCP tools only, same floor as O1. |
| No agent for simple forks | Lost: harness prompt, agent loop, `StructuredOutput` round trip. | Lost, same. | Lost: base instructions, tool specs, multi-agent and skills blocks. | Lost, same. | Lost, same. |
| Bounded answers | Kept: `outputFormat` json_schema with `enum`; validated; costs one extra model turn. | Kept: `--json-schema`; same mechanism. | Kept: `--output-schema`, Responses API `strict: true`, one model call. | Kept: `turn/start.outputSchema`, same. | Kept: `outputSchema`, same. |
| Preloaded state | Kept | Kept | Kept | Kept | Kept |
| Fixed-format state | Kept | Kept | Kept | Kept | Kept |
| Gate in code | Kept | Kept | Kept | Kept | Kept |
| Two RPC round-trips | Kept | Kept | Kept | Kept | Kept |

The "no agent for simple forks" lever is lost on every listed route. The only embedding that keeps it is the raw model API: Anthropic Messages API (`@anthropic-ai/sdk`, `messages.create` with `output_config.format` for structured output, per the `claude-api` skill, 2026-10-06) and the OpenAI Responses API (`text.format` json_schema, per the `gpt-6.1-sol` model page). Neither was in the ticket's list; section 7 treats this as a scope finding.

## 6. Licence and auth verdicts

### 6.1 Anthropic

Licence text, verbatim from `LICENSE.md` in both `@anthropic-ai/claude-agent-sdk@0.3.303` and `@anthropic-ai/claude-agent-sdk-darwin-arm64@0.3.303` (2026-10-19): "© Anthropic PBC. All rights reserved. Use is subject to the Legal Agreements outlined here: https://code.claude.com/docs/en/legal-and-compliance."

[Legal and compliance page](https://code.claude.com/docs/en/legal-and-compliance) (2026-10-19), verbatim:
- "OAuth authentication is intended exclusively for purchasers of Claude Free, Pro, Max, Team, and Enterprise subscription plans and is designed to support ordinary use of Claude Code and other native Anthropic applications."
- "Developers building products or services that interact with Claude's capabilities, including those using the Agent SDK, should use API key authentication through Claude Console or a supported cloud provider. Anthropic does not permit third-party developers to offer Claude.ai login into their own applications, or to route requests through Free, Pro, or Max plan credentials on behalf of their users. Moreover, developers may not collect, store, or intermediate Claude.ai credentials or session tokens".
- "Nor does it prevent an end user from signing in to the unmodified Claude Code binary with their own Claude subscription". Conditions on the same page: "The Claude Code binary must not be modified" and each end user authenticates with their own key or subscription, with billing to the end user.
- "Advertised usage limits for Pro and Max plans assume ordinary, individual usage of Claude Code and the Agent SDK."

[Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview) (2026-10-19): "Unless previously approved, Anthropic does not allow third party developers to offer claude.ai login or rate limits for their products, including agents built on the Claude Agent SDK. Use the API key authentication methods described in the Quickstart instead." Branding: "Not permitted: 'Claude Code' or 'Claude Code Agent'".

Redistribution grant, same legal page (2026-10-19), verbatim: "Can customers offer Claude Code in their products? Unless we've mutually agreed otherwise, preinstalling or running Claude Code in your products or services (e.g. in hosted sandboxes or other agent infrastructure) requires agreeing to our Commercial Terms of Service and complying with the conditions below:" "The Claude Code binary must not be modified. Claude Code must be installed and run as published by Anthropic, and customers may not remove, disable, or restrict any authentication method built into it (including methods that permit signing in with a Claude account or the user's own API key)." "Customers may not pay for, resell, or intermediate Claude usage on their end users' behalf. Each end user must authenticate with their own Anthropic API key, Claude subscription plan credentials, or 3P inference provider credential". Branding: "You can accurately say, in plain text, that your product has Claude Code preinstalled or that it runs Claude Code. But you can't use the Claude Code or Anthropic names or logos as part of your own product, feature, or company name". SDK overview "License and terms": "Use of the Claude Agent SDK is governed by Anthropic's Commercial Terms of Service, including when you use it to power products and services that you make available to your own customers and end users". [Commercial Terms](https://www.anthropic.com/legal/commercial-terms) (effective 2025-06-17, fetched 2026-10-19) A.1: "Anthropic gives Customer permission to use the Services, including to power products and services Customer makes available to its own customers and end users". The SDK README documents embedding the platform binary in a compiled executable (`import binPath from '@anthropic-ai/claude-agent-sdk-darwin-arm64/claude' with { type: 'file' }` ... `pathToClaudeCodeExecutable: cliPath`). The npm `license` pointer "SEE LICENSE IN README.md" is dangling: neither README has a licence section; the GitHub `LICENSE.md` files read "Use is subject to Anthropic's Commercial Terms of Service."

Subscriber-side statements: [support article 15036540](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan) (2026-10-19): "Update October 7, 2026: Claude Max and Team plans now include monthly API credits, which cover the Claude Agent SDK, claude -p, the Claude API, and Claude Managed Agents. You can still use the Claude Agent SDK, claude -p, and third-party apps with your subscription limits." [Support article 17154008](https://support.claude.com/en/articles/17154008-monthly-api-credits-for-max-and-team-plans): "When you're signed in with your Claude plan instead, claude -p and Agent SDK usage still draw from your plan's usage limits and don't use your API credits." [Consumer Terms](https://www.anthropic.com/legal/consumer-terms) (effective 2025-10-08) §3.7 bars access "through automated or non-human means" except via an API key or where explicitly permitted. [AUP](https://www.anthropic.com/legal/aup) dated "Effective November 12, 2026" adds "Resell, proxy, or otherwise provide access to Claude through unauthorized means, including services that route requests through consumer subscriptions or misrepresent the product or client being used"; the in-force 2025-09-15 version has no such bullet.

Verdicts:
- Licence identifier: proprietary, all rights reserved. Redistribution of the unmodified `claude` binary (bundled by the SDK or by `@anthropic-ai/claude-code`) inside a desktop installer: granted under the Commercial Terms with the conditions quoted above. The app must not strip or block any built-in auth method, must not intermediate usage or credentials, must keep its own branding, and the developer agrees to the Commercial Terms.
- Subscription auth: the developer side and the subscriber side read differently. Developers "including those using the Agent SDK, should use API key authentication" and may not "offer Claude.ai login into their own applications" or "route requests through Free, Pro, or Max plan credentials on behalf of their users"; "Unless previously approved". The end user may sign in to the unmodified binary through Anthropic's own flow, and the subscriber articles say third-party apps draw on plan limits. Whether a desktop harness that spawns the unmodified binary and relies on the user's existing login falls on the permitted side is UNVERIFIED and needs Anthropic's prior approval; the November 2026 AUP bullet narrows it further. Sanctioned mode for A1 and A2: the user's own API key (Max and Team plans now include monthly API credits), or Bedrock / Vertex / Foundry.

### 6.2 OpenAI

Licence: `LICENSE` at `github.com/openai/codex` tag `rust-v0.163.0` is the Apache License, Version 2.0; README: "This repository is licensed under the Apache-2.0 License"; `NOTICE`: "OpenAI Codex / Copyright 2025 OpenAI" plus a Ratatui MIT attribution (2026-10-19). Apache 2.0 §6: "This License does not grant permission to use the trade names, trademarks, service marks, or product names of the Licensor, except as required for reasonable and customary use in describing the origin of the Work and reproducing the content of the NOTICE file." npm: `@openai/codex@0.163.0` and `@openai/codex-sdk@0.163.0` both `license: Apache-2.0` (2026-10-19).

Auth terms, [app-server docs](https://developers.openai.com/codex/app-server) "Auth endpoints" (2026-10-19), verbatim: "If you've built a local or open-source application using Codex app-server authentication, you can continue using it, though we recommend migrating to Sign in with ChatGPT so users have greater control over and visibility into their usage." "App-server authentication has never been permitted for commercial or hosted services." "We launched Sign in with ChatGPT to support these use cases". [Auth docs](https://developers.openai.com/codex/auth) (2026-10-19): "When you sign in with an API key, Codex uses standard API pricing instead of included ChatGPT plan credits." "API keys are still the recommended default for automation."

Packaging facts (`npm pack`, 2026-10-19): `@openai/codex-sdk@0.163.0` ships `LICENSE` (Apache 2.0, "Copyright 2025 OpenAI"); `@openai/codex@0.163.0` ships `bin/codex.js`, `package.json`, `README.md` and no LICENSE file; its six platform packages are npm aliases (`npm:@openai/codex@0.163.0-darwin-arm64`, 340,793,950 bytes unpacked), all `Apache-2.0` on the registry. The Codex SDK README notes the SDK "injects its required variables (such as `CODEX_API_KEY`) on top of the environment you provide" and is "useful for sandboxed hosts like Electron apps."

"Sign in with ChatGPT" (SIWC) pages (2026-10-19): [token-sharing overview](https://developers.openai.com/siwc/token-sharing-open-source): "These docs explain ChatGPT plan usage for open-source and locally hosted apps. If you're interested in offering it in a paid or remotely hosted app, complete the interest form." [Preview limitations](https://developers.openai.com/siwc/token-sharing-open-source/preview-limitations): "These limits apply to ChatGPT plan usage through Sign in with ChatGPT and https://api.openai.com/v1, both directly and through Codex app-server." [User-facing page](https://learn.chatgpt.com/docs/sign-in-with-chatgpt): "App usage counts toward your existing plan limits. Connecting an app does not add a new allowance, and the app may have its own fees." App-server docs: "ChatGPT external tokens (chatgptAuthTokens) - experimental and intended for host apps that already own the user's ChatGPT auth lifecycle."

Verdicts:
- Licence identifier: Apache-2.0. Redistribution of the `codex` binary and the SDK inside a desktop binary: granted (Apache 2.0 §4), with the obligations: ship the licence text (copy it from the SDK tarball or the repo, the CLI wrapper tarball has none), reproduce NOTICE, no use of the OpenAI or Codex marks beyond origin attribution (§6).
- Subscription auth: for an open-source, locally run app, OpenAI documents "Sign in with ChatGPT" plan usage and the app-server as its carrier; app-server's older built-in auth "has never been permitted for commercial or hosted services"; paid or remotely hosted apps need the partner form. No statement covers `codex exec` or the SDK driving a ChatGPT-plan login from a third-party app (UNVERIFIED); the auth docs say "API keys are still the recommended default for automation."

### 6.3 Consumer terms

OpenAI: https://openai.com/policies/terms-of-use, `/usage-policies`, `/services-agreement`, `/business-terms` and the help-center articles all returned HTTP 403 (Cloudflare interstitial) to WebFetch, curl and a headless browser on 2026-10-19. No clause from those documents is quoted here. UNVERIFIED.

Anthropic: Consumer Terms §3.7 and the November 2026 AUP bullet are quoted in 6.1. The Commercial Terms contain no clause with the words "redistribute", "sublicense", "embed" or "bundle"; D.4 restricts building a competing product, reverse engineering and reselling "except as expressly approved by Anthropic" (2026-10-19).

## 7. Recommendation

### 7.1 Anthropic: A1, the Agent SDK, on API-key auth, for the agentic tier only

Reasons: same binary and same levers as A2; adds in-process tools and hook callbacks (`measured`); pins the CLI version to the app (`sdk.d.ts`, bundled binary); `startup()` and `prewarm()` remove the spawn cost from the decision path. Configuration that keeps the levers: `settingSources: []`, `strictMcpConfig: true`, `tools: []`, `mcpServers: { tw: createSdkMcpServer(...) }` with `alwaysLoad: true` on each tool, `allowedTools: ['mcp__tw__*']`, `permissionMode: 'dontAsk'`, `systemPrompt: [staticPrompt, SYSTEM_PROMPT_DYNAMIC_BOUNDARY, stateLine]`, `outputFormat: { type: 'json_schema', schema }`, `persistSession: false`, `maxTurns`, `maxBudgetUsd`, `effort: 'low'`, `env: { ...process.env, CLAUDE_CODE_DISABLE_AUTO_MEMORY: '1', ENABLE_PROMPT_CACHING_1H: '1', CLAUDE_AGENT_SDK_CLIENT_APP: 'trade-wars/x.y' }`.

Auth shape: the app asks the user for their own Anthropic API key (Max and Team plans include monthly API credits since 2026-10-07) and passes it in `env`. The app must not build a Claude login screen, must not read or store OAuth tokens, and must not disable the binary's own login. Whether the harness may additionally rely on a user's existing subscription login is an open legal item (section 6.1); ask Anthropic before shipping that.

Choose A2 only when the product must run on the user's own installed `claude`. Even then, use the SDK with `pathToClaudeCodeExecutable` pointed at that binary rather than a hand-written stream-json client, so the control protocol stays Anthropic's and the in-process tools keep working.

For the Tier 1 decider (section 4 of the mod doc: "No files, no tools, no history ... ~200 tokens per decision"), neither route reaches the target: the harness floor is ~1,100 tokens plus one `StructuredOutput` round trip. Use the Messages API directly (`@anthropic-ai/sdk`, `messages.create`, `output_config.format` json_schema, `claude-haiku-5-5`), which keeps all eight levers. The ticket's route list did not include it; this is the scope finding to carry into the desktop client map (#41).

### 7.2 OpenAI: O3, the Codex SDK, with the app's tools as a local MCP server

Reasons: Apache-2.0 and bundled binary make it the only OpenAI route that is clean to ship; OpenAI's own docs point automation at the SDK and mark the app-server experimental; the output schema is enforced in one model call. Costs: a new `codex exec` process per decision and a 9.5k+ token floor, so prompt caching must stay warm (same prefix, same config) and the decision cadence should stay above the cache TTL. Configuration: `config: { project_doc_max_bytes: 0, model_instructions_file, features: { shell_tool: false, unified_exec: false, apps: false, plugins: false, skill_search: false, tool_suggest: false, sleep_tool: false, computer_use: false, browser_use: false }, web_search: 'disabled', mcp_servers: { tw: { command, args } } }`, `sandboxMode: 'read-only'`, `approvalPolicy: 'never'`, `modelReasoningEffort: 'low'`, `outputSchema` per `run()`. A clean `CODEX_HOME` (set in `env`) is the only way to exclude the user's own `config.toml` and `AGENTS.md` through the SDK. That also drops the saved ChatGPT login (`--ignore-user-config` help: "auth still uses `CODEX_HOME`"), so the SDK route then needs `apiKey` / `CODEX_API_KEY`, which matches the auth verdict in section 6.2.

Choose O2 when the product needs in-process tools without an MCP sidecar, or many decisions per minute without respawn, and accepts "experimental and not supported for production workloads". It is also the route OpenAI names for "Sign in with ChatGPT" plan usage in "open-source and locally hosted apps", which this project is; a paid or hosted edition would need OpenAI's partner form. Ship the Apache 2.0 licence text and NOTICE with either route.

For the Tier 1 decider use the Responses API directly (`openai` npm, `text.format` json_schema, `gpt-6.1-sol` or `gpt-6-luna`), same reasoning as 7.1.

### 7.3 Provider-agnostic harness interface

The interface follows the levers, not the vendors. Tool handlers live in the host process; each adapter bridges them (SDK MCP server, app-server dynamic tools, or a local stdio MCP server that the app spawns from its own binary).

```ts
export type JsonSchema = Record<string, unknown>;

export interface ToolDef {
  name: string;                       // 'tw_state', 'tw_sell', ...
  description: string;
  inputSchema: JsonSchema;
  handler: (input: unknown) => Promise<string>;   // runs in the host process
}

export interface DecisionRequest {
  model: string;                      // 'claude-haiku-5-5' | 'gpt-6.1-sol'
  system: string;                     // identity block, static, cacheable
  state: string;                      // fixed-format state line, preloaded
  schema: JsonSchema;                 // bounded answer; label sets as enum
  tools: ToolDef[];                   // [] for Tier 1
  limits: { maxTurns: number; maxUsd?: number; effort: 'low' | 'medium' | 'high' };
  signal?: AbortSignal;
}

export type HarnessEvent =
  | { type: 'started'; sessionId: string }
  | { type: 'tool_call'; name: string; input: unknown }
  | { type: 'tool_result'; name: string; output: string }
  | { type: 'text_delta'; text: string }
  | { type: 'usage'; input: number; cachedInput: number; output: number; reasoning: number }
  | { type: 'finished'; reason: 'ok' | 'max_turns' | 'budget' | 'error' | 'refusal' };

export interface DecisionResult<T> {
  output: T | null;                   // schema-validated or null
  usage: { input: number; cachedInput: number; output: number; reasoning: number; costUsd?: number };
  latencyMs: number;
  turns: number;
}

export interface Decider {
  readonly id: 'anthropic-messages' | 'anthropic-agent-sdk' | 'openai-responses' | 'codex-sdk' | 'codex-app-server';
  start(): Promise<void>;             // pre-warm: SDK startup(), app-server initialize, no-op for raw APIs
  decide<T>(req: DecisionRequest, onEvent?: (e: HarnessEvent) => void): Promise<DecisionResult<T>>;
  stop(): Promise<void>;
}
```

Rules the adapters must share: no history between decisions (new session, thread or request each time); the system prompt is byte-stable across decisions so the prompt cache hits; `costUsd` is an estimate on every route (Anthropic says so; OpenAI gives tokens only); a decision that returns no schema-valid output is a failure, never a parse attempt on free text.

## 8. Open items

1. Whether a desktop harness that spawns the unmodified `claude` binary and relies on the user's existing subscription login is on the permitted side of the legal page (developer rule vs end-user carve-out), and how the AUP bullet dated 2026-11-12 applies. UNVERIFIED; Anthropic requires prior approval for third-party products. Also UNVERIFIED: any Anthropic statement on `CLAUDE_CODE_OAUTH_TOKEN` in third-party apps (GitHub issues anthropics/claude-code#42106 and claude-agent-sdk-python#559 have no staff reply).
2. OpenAI consumer Terms of Use, Usage Policies and Services Agreement: not fetched (HTTP 403). UNVERIFIED.
3. Whether ChatGPT-plan auth may be used from `codex exec` or the Codex SDK inside a third-party app; the only explicit statements cover the app-server and "Sign in with ChatGPT" for open-source, locally hosted apps. UNVERIFIED.
4. The Sonnet 5.5 full model ID string is not printed on the Claude Code model page; the `claude-api` skill table gives `claude-sonnet-5-5`.
5. `project_doc_max_bytes=0` versus the user-level `~/.codex/AGENTS.md`: the source gate skips project docs, the `debug prompt-input` dump still showed the global file; `--ignore-user-config` removed it. Mechanism UNVERIFIED.
6. `thread/start.config` semantics on the app-server are undocumented. UNVERIFIED.
7. All latency and cost numbers are single samples on one machine on 2026-10-19.

## Appendix A. Probes

All scripts ran on 2026-10-19 on macOS (Darwin 25.6.0, arm64). Prices are list prices; both CLIs were logged in with subscriptions.

### A.1 `claude -p`, Haiku 5.5, custom prompt, no tools, JSON schema

```
claude -p --no-session-persistence --setting-sources "" --strict-mcp-config --tools "" \
  --model claude-haiku-5-5 --system-prompt "You are a game decider. State: E=120 L=40 T=5 nova=300 inbound=0. Pick one move." \
  --output-format json --json-schema "$SCHEMA" --max-budget-usd 0.05 "Decide now."
```
Result: `num_turns` 3, `total_cost_usd` 0.00079, `duration_ms` 4014, `duration_api_ms` 4571, usage `cache_creation_input_tokens` 2134, `cache_read_input_tokens` 1685, `output_tokens` 454 (243 thinking), `structured_output` `{"move":"sell","reason":"..."}`, wall 5,463 ms. Stream-json view of the same call: `init.tools` = `["StructuredOutput"]`, `apiKeySource` `none`, assistant blocks `thinking`, `tool_use:StructuredOutput`, then `user tool_result`, `num_turns` 2, cost 0.00031.

### A.2 `claude -p`, harness floor

Same flags, prompt "Reply with exactly one word: hold.", one turn each:
- default system prompt, `--tools ""`: cache write 3,705 + cache read 649 input, 4 output, 0.00075 USD, `duration_ms` 1,085.
- default system prompt, `--tools default`: cache write 7,276 + cache read 11,228, 4 output, 0.00169 USD, `duration_ms` 1,144.
- `--system-prompt "Reply with one word."`, `--tools ""`: cache write 1,120 + cache read 0, 4 output, 0.00035 USD, `duration_ms` 1,171.

### A.3 `claude -p`, one persistent process, two turns

`--input-format stream-json --output-format stream-json --verbose`, two `{"type":"user","message":{"role":"user","content":"..."}}` lines on stdin. Output: `system/init`, `assistant`, `rate_limit_event`, `result` (turn 1: 1,164 cache write + 2 input, 4 output, 0.000235 USD, `duration_ms` 1,050), then `system/init`, `assistant`, `result` (turn 2: 734 cache write + 542 cache read + 2 input, 4 output, 0.00039 USD, `duration_ms` 985). Wall for the whole process 3,309 ms.

### A.4 Agent SDK 0.3.303, in-process tool, JSON schema

`query({ prompt, options: { model: 'claude-haiku-5-5', systemPrompt, settingSources: [], tools: [], mcpServers: { tw }, allowedTools: ['mcp__tw__tw_state'], strictMcpConfig: true, persistSession: false, maxTurns: 4, maxBudgetUsd: 0.05, effort: 'low', outputFormat: { type: 'json_schema', schema }, hooks: { PreToolUse, PostToolUse, Stop } } })` with `tw = createSdkMcpServer({ name: 'tw', tools: [tool('tw_state', ..., {}, handler)] })`.

Output: `init` `{ model: 'claude-haiku-5-5', tools: ['StructuredOutput','mcp__tw__tw_state'], mcp_servers: [{ name: 'tw', status: 'connected', source: 'sdk' }], apiKeySource: 'none', claude_code_version: '2.1.303' }`; assistant blocks `tool_use:mcp__tw__tw_state`, `thinking`, `tool_use:StructuredOutput`; result `num_turns` 3, `duration_ms` 2,966, `duration_api_ms` 3,745, `total_cost_usd` 0.00053, usage 1,366 cache write + 1,243 cache read + 4 input, 228 output, `modelUsage['claude-haiku-5-5'].thinkingTokens` 94; `structured_output` `{"move":"hold","reason":"..."}`; wall 7,444 ms. The tool handler and the three hooks ran in the host process (pid match). Child process: the bundled `claude` binary with the flags quoted in section 3.1.

### A.5 `codex exec`, `gpt-6.1-sol`, output schema

```
codex exec --json --ephemeral -s read-only --skip-git-repo-check -C "$OUT" -m gpt-6.1-sol \
  --output-schema "$OUT/schema.json" -c 'project_doc_max_bytes=0' "...Pick one move. Do not run any commands."
```
Events: `thread.started`, `turn.started`, `item.completed` (`agent_message` `{"move":"hold","reason":"..."}`), `turn.completed` usage `{ input_tokens: 22966, cached_input_tokens: 0, cache_write_input_tokens: 0, output_tokens: 79, reasoning_output_tokens: 48 }`; wall 7,506 ms.

Stripped variant (`+ -c model_instructions_file=... --disable shell_tool --disable unified_exec --disable apps --disable plugins --disable skill_search --disable tool_suggest --disable sleep_tool --disable computer_use --disable browser_use`): usage 13,483 / 49 (14 reasoning), wall 5,117 ms. One earlier attempt failed with `{"type":"error","message":"Selected model is at capacity. Please try a different model."}` and `turn.failed`.

Stripped + `--ignore-user-config --ignore-rules`: usage 9,521 / 92 (60 reasoning), wall 6,142 ms.

`codex debug prompt-input` under the stripped config (no `--ignore-user-config` flag on this subcommand) listed user-side items: a `<skills_instructions>` block (1,977 chars), `<permissions instructions>` (362), `<collaboration_mode>` (1,056), `<multi_agent_role>` (2,400), `<multi_agent_mode>` (271), `# AGENTS.md instructions` with the user's global `~/.codex/AGENTS.md` (2,759), `<environment_context>` (478). Tool definitions and base instructions are not in this dump.

### A.6 `codex app-server`, dynamic tool, output schema, two threads

Node script over stdio: `initialize { clientInfo, capabilities: { experimentalApi: true } }` (185 ms), then per decision `thread/start { model: 'gpt-6.1-sol', ephemeral: true, cwd, approvalPolicy: 'never', sandbox: 'read-only', baseInstructions, dynamicTools: [{ type: 'function', name: 'tw_state', description, inputSchema }] }` and `turn/start { threadId, input: [{ type: 'text', text }], outputSchema, effort: 'low' }`; the script answered `item/tool/call` with `{ contentItems: [{ type: 'inputText', text }], success: true }`.

Decision 1: `status` completed, `durationMs` 8,647, wall 9,355 ms, usage last `{ inputTokens: 18853, cachedInputTokens: 18560, cacheWriteInputTokens: 0, outputTokens: 96, reasoningOutputTokens: 61 }`, `modelContextWindow` 475,000. Decision 2: 8,817 / 8,905 ms, 18,853 (18,560 cached) / 54 (18). Final `agentMessage` text: `{"move":"sell","reason":"..."}`. Event sequence per decision: `thread/started`, nine `mcpServer/startupStatus/updated` (user config MCP servers), `thread/status/changed`, `turn/started`, `hook/started` ×2, `hook/completed` ×2, `item/started`, `item/completed`, `turn/moderationMetadata`, `item/started`, server request `item/tool/call`, `item/completed`, `thread/tokenUsage/updated`, `account/rateLimits/updated`, reasoning item, `item/agentMessage/delta` ×N, `item/completed`, `thread/tokenUsage/updated`, `hook/started`, `hook/completed`, `thread/status/changed`, `turn/completed`.
