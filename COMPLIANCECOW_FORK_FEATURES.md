# ComplianceCow Fork — Feature Registry

This document tracks every ComplianceCow-specific customization layered on top of
upstream `goose`, plus the companion changes in **CowGooseService** (the Go
WebSocket↔SSE bridge). Use it as the single source of truth when rebasing onto a
new upstream, onboarding, or restoring a feature.

- **goose fork (Rust):** `/Users/Arul/Documents/rust/goose/` — branch `dev_2`
- **CowGooseService (Go):** `/Users/Arul/Documents/projects/continube/ComplianceCow/src/cowgooseservice/`
- **cow-mcp (Python):** `http://0.0.0.0:45678/mcp`

> **Rebase tip:** `git log --oneline upstream/main..dev_2` lists the fork commits.
> The features below map to those commits plus in-flight (uncommitted) work noted
> per section.

---

## Part 1 — goose fork (Rust) features

### 1. Dynamic per-session LLM provider switching
**Why:** One goose server serves many tenants; each WebSocket session can use a
different provider/model/API key supplied at runtime (not from global config).

- **Headers:** `X-Provider-Name`, `X-Model-Name`, `X-Api-Key` (forwarded by
  CowGooseService into the session's `websocket_headers.v0` extension data).
- **Entry point:** `POST /agent/update_provider` → `update_agent_provider()`
  in `crates/goose-server/src/routes/agent.rs`.
- **Core fn:** `create_with_api_key()` in `crates/goose/src/providers/init.rs`
  (supports `anthropic` and `openai` providers).
- **Provider constructors:** `from_api_key()` in
  `crates/goose/src/providers/openai.rs` and `.../anthropic.rs`.
- **Session cache:** `Agent::ensure_session_provider()` /
  `provider_for_session()` + the `session_providers` map in
  `crates/goose/src/agents/agent.rs`.
- **Commits:** `193bdcd36` (dynamic provider creation w/ explicit API keys),
  `13772b5b7` (MCP integration).

### 2. Provider host passthrough  *(this session — uncommitted)*
**Why:** Route the `openai`/`anthropic` provider to an alternate base URL (e.g.
DeepSeek at `https://api.deepseek.com`) per session.

- **`UpdateProviderRequest.host: Option<String>`** in
  `crates/goose-server/src/routes/agent.rs`.
- `from_api_key(model, api_key, host_override)` uses the override instead of the
  global `OPENAI_HOST`/`ANTHROPIC_HOST`; `preserve_thinking_context` is derived
  from whether the resolved host is OpenAI's own API.
- **Note:** Today this is driven by goose's config (`OPENAI_HOST`); CowGooseService
  does **not** yet send `host` in the request body. The field exists for when it does.

### 3. `create_with_api_key` accepts a pre-built `ModelConfig`  *(this session — uncommitted)*
**Why:** Previously `create_with_api_key` rebuilt `ModelConfig` from just the model
name and **discarded** `request_params` the route had merged in. Now the route's
fully-prepared `model_config` (canonical limits, context limit, `request_params`)
flows through to the per-session provider.

- `crates/goose/src/providers/init.rs` — signature is
  `create_with_api_key(provider_name, model_config: ModelConfig, api_key, host)`.
- Call sites: `crates/goose-server/src/routes/agent.rs` (passes `model_config`)
  and `crates/goose/src/agents/agent.rs` (builds a `ModelConfig` first).
- **This is the plumbing that lets CowGooseService disable DeepSeek thinking.**

### 4. Dynamic header injection into MCP tool calls
**Why:** Forward tenant auth/security headers to the cow-mcp server on every tool
call, read fresh per-request (tokens rotate).

- `DynamicHeaderClient` reads `websocket_headers.v0` and injects allowed headers
  as HTTP headers on Streamable-HTTP MCP requests.
- Only headers in goose YAML `allowed_headers` are forwarded (case-insensitive).
- `allowed_headers` threaded through `call_tool` / extension constructors
  (`crates/goose/src/agents/extension_manager.rs`, `acp/server*.rs`,
  `skills/client.rs`).
- **Commits:** `c7d239844`, `e17dddfd4` (missed HashMap import).

### 5. Custom provider configuration header (Anthropic)
**Why:** Include ComplianceCow provider-config header in Anthropic API requests.
- **Commit:** `a4a95e003`.
- `crates/goose/src/providers/anthropic.rs` — `custom_headers` support.

### 6. Caching configuration + tracking metadata
**Why:** Prompt caching configuration and per-request metadata for tracking.
- **Commit:** `1d2dec92f`.

### 7. `available_tools` filtering
**Why:** Restrict which MCP tools are exposed per session.
- **Commit:** `13772b5b7`.

### 8. Reasoning-content accumulation fix (upstream issue #9675)  *(this session — uncommitted)*
**Why:** In multi-turn streaming, DeepSeek streams reasoning in early chunks and
tool calls later. The old code took thinking from the *last* chunk only, silently
losing `reasoning_content` on thinking-only assistant turns.

- `crates/goose/src/agents/agent.rs` — `accumulated_thinking` collects
  `MessageContent::Thinking` across **all** chunks in the stream loop.
  - No-tool-call path pushes only non-thinking content as standalone messages.
  - Tool-call path emits a thinking message built from `accumulated_thinking`.

### 9. DeepSeek v4 models + thinking disable  *(this session — uncommitted)*
**Why:** Disable DeepSeek reasoning to save output-token cost; add v4 models.

- `crates/goose/src/providers/declarative/deepseek.json`:
  - Added models `deepseek-v4-flash`, `deepseek-v4-pro` (alongside
    `deepseek-chat`, `deepseek-reasoner`).
  - Added `"default_request_params": {"thinking": {"type": "disabled"}}`.
- `crates/goose/src/config/declarative_providers.rs` — new
  `default_request_params: Option<HashMap<String, Value>>` field on
  `DeclarativeProviderConfig`.
- `crates/goose/src/providers/openai.rs` — `from_custom_config()` merges
  `default_request_params` into the model config via
  `with_merged_request_params`.
- `custom_deepseek` is in `PROVIDERS_NEEDING_MAX_TOKENS_REMAP`.
- **Applies to:** the declarative `custom_deepseek` provider path. The
  CowGooseService path (provider `openai` + `OPENAI_HOST`=DeepSeek) is handled by
  the Go side (Part 2 §4), since the provider name there is `openai`, not
  `custom_deepseek`.

### 10. Recipe instruction in agent initialization
**Why:** Inject recipe-specific instruction during agent init.
- **Commit:** `839655550`.

### 11. Anthropic safety-refusal surfacing  *(this session — uncommitted)*
**Why:** When Anthropic returns `stop_reason: "refusal"` (model declines for
safety/policy), the stream otherwise yielded an empty/cryptic result and the
agent loop could stall. Surface it as a readable assistant message instead.
**Note:** this does **not** prevent refusals — it makes them visible/graceful.

- `crates/goose/src/providers/formats/anthropic.rs`:
  - `STOP_REASON_REFUSAL = "refusal"` constant.
  - `refusal_message(delta)` — formats `stop_details.category` /
    `stop_details.explanation` into a readable string.
  - In the `message_delta` stream handler: on a `refusal` stop reason, yield an
    assistant message once (`refusal_sent` guard) before usage extraction.
- Ported from teammate PR `opensecuritycompliance/goose#4`
  (commit `80a70bae`). Relevant because our `anthropic` provider is routed to
  DeepSeek's Anthropic-compatible endpoint (`ANTHROPIC_HOST`) and to real Claude.

---

## Part 2 — CowGooseService (Go) features

Bridge architecture details: see `ARCHITECTURE.md` and `CLAUDE.md` in the
CowGooseService repo. ComplianceCow-glue features:

### 1. WebSocket ↔ SSE bridge (core)
Browser WS ⇄ goose HTTP+SSE; multiplexes many goose sessions over one WS.
- `bridge/connection_manager.go`, `bridge/session_worker.go`, `bridge/sse_ws_bridge.go`.

### 2. Header forwarding
Captures whitelisted WS-handshake headers and stores them in the goose session as
`websocket_headers.v0` (consumed by goose feature §4 above).
- `bridge/sse_ws_bridge.go` — `HeaderWhitelist`, `ExtractWhitelistedHeaders`.

### 3. Per-session provider resolution
Resolves provider/model/API-key per session: handshake headers first, then
vault/prompty fallback by session type.
- `main.go` — `resolveProvider` closure in `HandleWebSocketBridge`.
- `utils/llm_tools_vault.go` — vault key lookup (`openai`/`anthropic`).

### 4. DeepSeek thinking control (request_params injection)  *(this session)*
**Why:** Turn DeepSeek reasoning on/off per deployment to save output tokens.
DeepSeek is reached via the `openai` provider (`OPENAI_HOST`=DeepSeek), so the
`thinking` param must be injected per session — the declarative
`custom_deepseek` default does **not** apply here.

- **`utils/deepseek_thinking.go`** — `BuildDeepSeekThinkingParams(model)`:
  - Matches when model name contains `deepseek` (safety guard, so genuine
    OpenAI/Anthropic sessions are never sent the DeepSeek-only field).
  - **Env `DEEPSEEK_THINKING_MODE`** (default `disabled`):
    - `disabled`/`off`/`false` → `{"thinking": {"type": "disabled"}}`
    - `enabled`/`on`/`true` → `{"thinking": {"type": "enabled"}}` (+ effort)
    - `passthrough`/`none`/empty → inject nothing
  - **Env `DEEPSEEK_REASONING_EFFORT`** (optional, e.g. `high`/`max`) — used
    only when mode is `enabled`.
- **Threading:**
  - `gooseclient/client.go` — `UpdateProviderRequest.RequestParams` (JSON
    `request_params,omitempty`); `UpdateProvider(..., requestParams)`.
  - `bridge/connection_manager.go` — `ProviderInfo.RequestParams`, passed to
    `UpdateProvider`.
  - `main.go` — `resolveProvider` sets
    `RequestParams: gooseutils.BuildDeepSeekThinkingParams(modelName)`.

### 5. Deferred — generic per-provider request_params factory
Planned: refactor §4 into a builder registry (strategy pattern) so any provider
can contribute request_params, matched by provider name and/or model. Paused by
request; tracked as a background task. Caveat: goose already handles OpenAI
`reasoning_effort` and Anthropic extended-thinking internally — only DeepSeek
needs external injection today.

---

## End-to-end: how DeepSeek thinking-disable flows
1. CowGooseService `resolveProvider` → provider `openai`, model
   `deepseek-v4-pro[1m]` → `BuildDeepSeekThinkingParams` returns
   `{"thinking": {"type": "disabled"}}` (default).
2. `POST /agent/update_provider` body carries `request_params`.
3. goose `update_agent_provider` merges them into `model_config`, passes it to
   `create_with_api_key` (Part 1 §3 — previously dropped).
4. `OpenAiProvider` (host = DeepSeek via `OPENAI_HOST`) injects `thinking` into
   the chat-completions payload → DeepSeek skips reasoning, saving output tokens.

## Relevant config / env reference
| Where | Key | Purpose |
|---|---|---|
| goose `config.yaml` | `OPENAI_HOST=https://api.deepseek.com` | Routes `openai` provider to DeepSeek |
| goose `config.yaml` | `ANTHROPIC_HOST=https://api.deepseek.com/anthropic` | DeepSeek's Anthropic-compatible endpoint |
| goose `config.yaml` | `GOOSE_MODEL=deepseek-v4-pro[1m]` | Default model |
| CowGooseService env | `DEEPSEEK_THINKING_MODE` | `disabled` (default) / `enabled` / `passthrough` |
| CowGooseService env | `DEEPSEEK_REASONING_EFFORT` | optional effort when enabled |
| goose YAML | `allowed_headers` | Headers forwarded to MCP server |

## Build / verify
- **goose:** `source bin/activate-hermit && cargo build && cargo clippy --all-targets -- -D warnings && cargo fmt`
- **CowGooseService:** `/usr/local/go/bin/go build ./... && /usr/local/go/bin/go vet ./... && /usr/local/go/bin/gofmt -l .`
- After goose server-route changes: `just generate-openapi`.
