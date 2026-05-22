# Gateway API — architecture view

This document describes the **HTTP/WebSocket gateway** from the workspace boundary inward: how [`zeroclaw-api`](../../../../crates/zeroclaw-api/) defines shared contracts, how [`zeroclaw-gateway`](../../../../crates/zeroclaw-gateway/) exposes them over the network, and where to find deeper HTTP semantics.

For field-level config CRUD, JSON Patch rules, secrets behavior, and Scalar/OpenAPI details, see [Gateway HTTP API](./api.md).

## Role in the architecture

```text
zeroclaw-api          ← trait & shared types only (no I/O implementations)
       ↑
zeroclaw-runtime, zeroclaw-channels, zeroclaw-config, zeroclaw-memory, …
       ↑
zeroclaw-gateway    ← Axum server: REST, SSE, WebSockets, static dashboard
```

- **`zeroclaw-api`** is the narrowest crate: traits (`Channel`, `Tool`, `Memory`, …), wire-oriented structs (`ChannelMessage`, `SendMessage`, `ToolSpec`), streaming shapes (`TurnEvent`), and **task-local context** used across the stack (`TOOL_LOOP_SESSION_KEY`, `TOOL_LOOP_THREAD_ID`, `TOOL_CHOICE_OVERRIDE`) — see [`lib.rs`](../../../../crates/zeroclaw-api/src/lib.rs).
- **`zeroclaw-gateway`** depends on those types where it bridges HTTP to the rest of the system: webhook handlers normalize inbound traffic into `ChannelMessage`; the tools registry surfaces `ToolSpec` over REST; the chat WebSocket translates runtime `TurnEvent` frames into JSON for clients (the runtime re-exports `TurnEvent` from `zeroclaw_api::agent`).

The gateway does **not** implement `zeroclaw-api` traits itself; it orchestrates implementations from other crates behind `AppState` (config, pairing, memory, agent loop, channels).

## Contracts from `zeroclaw-api` used at the gateway

| API item | Gateway usage |
|----------|----------------|
| [`channel::ChannelMessage`](../../../../crates/zeroclaw-api/src/channel.rs), [`SendMessage`](../../../../crates/zeroclaw-api/src/channel.rs) | Webhook paths (`/webhook`, `/whatsapp`, …) deserialize platform payloads and drive the agent using the same message model as every channel integration. Inbound messages may carry `thread_ts`, `interruption_scope_id`, and [`media::MediaAttachment`](../../../../crates/zeroclaw-api/src/media.rs) lists for the media pipeline. |
| [`tool::ToolSpec`](../../../../crates/zeroclaw-api/src/tool.rs) | `GET /api/tools` returns registered tools as JSON derived from specs (`name`, `description`, `parameters`). |
| [`agent::TurnEvent`](../../../../crates/zeroclaw-api/src/agent.rs) | Chat WebSocket streams chunks, [`TurnEvent::Thinking`](../../../../crates/zeroclaw-api/src/agent.rs) deltas, tool calls/results, etc.; runtime emits `TurnEvent`, gateway maps to JSON `type` fields (see [Real-time transports](#real-time-transports)). |
| Task locals in [`zeroclaw_api`](../../../../crates/zeroclaw-api/src/lib.rs) ([`TOOL_LOOP_SESSION_KEY`](../../../../crates/zeroclaw-api/src/lib.rs), [`TOOL_LOOP_THREAD_ID`](../../../../crates/zeroclaw-api/src/lib.rs), [`TOOL_CHOICE_OVERRIDE`](../../../../crates/zeroclaw-api/src/lib.rs)) | Session key is set during WebSocket handling so session-scoped tools see the active session; thread ID and tool-choice override are consumed by the agent loop / providers (declared in `zeroclaw-api`, not gateway-local). |

Other gateway modules (e.g. `node_tool.rs`) import [`Tool`](../../../../crates/zeroclaw-api/src/tool.rs) / [`ToolResult`](../../../../crates/zeroclaw-api/src/tool.rs) for node-side execution helpers.

[`vad::Vad`](../../../../crates/zeroclaw-api/src/vad.rs) / [`VadEvent`](../../../../crates/zeroclaw-api/src/vad.rs) are API-layer hooks for future voice pipelines; the gateway’s optional duplex path uses its own JSON [`VoiceEvent`](../../../../crates/zeroclaw-gateway/src/voice_duplex.rs) frames on `/ws/chat` (see below), not these trait types directly yet.

## Authentication model

- **Dashboard REST (`/api/*`)** — When pairing is required, handlers use bearer validation (`Authorization: Bearer <token>`) via shared helpers in [`api.rs`](../../../../crates/zeroclaw-gateway/src/api.rs). Pairing flows expose tokens out-of-band (startup logs, `/pair`, enhanced pairing APIs).
- **WebSockets** — Token may appear as `Authorization`, `Sec-WebSocket-Protocol: bearer.<token>`, or `?token=` (browsers cannot set arbitrary headers on `WebSocket`). Chat protocol advertises `zeroclaw.v1`; ACP uses `zeroclaw.acp.v1`.
- **Webhooks** — Channel-specific verification (e.g. WhatsApp, WATI) and optional `X-Session-Id` for session affinity; not the same bearer model as `/api/*`.

Treat `/api/*` as **local-trust by default**: exposing it on a network requires TLS and pairing discipline.

## HTTP route map

Routes are registered in [`zeroclaw-gateway/src/lib.rs`](../../../../crates/zeroclaw-gateway/src/lib.rs). Optional **path prefix** nests the entire app when configured.

### Health, metrics, admin

| Method | Path | Notes |
|--------|------|--------|
| GET | `/health` | Liveness |
| GET | `/metrics` | Metrics scrape |
| POST | `/admin/shutdown`, `/admin/reload` | Operator control |
| GET | `/admin/paircode`, POST `/admin/paircode/new` | Pairing administration |

### Pairing (legacy + enhanced)

| Method | Path |
|--------|------|
| POST | `/pair`, GET `/pair/code` |
| POST | `/api/pairing/initiate`, `/api/pair` |
| GET | `/api/devices` |
| DELETE | `/api/devices/{id}` |
| POST | `/api/devices/{id}/token/rotate` |

### Config & docs

| Method | Path |
|--------|------|
| PATCH, OPTIONS | `/api/config` |
| GET, PUT, DELETE, OPTIONS | `/api/config/prop` |
| GET | `/api/config/list`, `/api/config/drift`, `/api/config/templates` |
| POST | `/api/config/map-key`, `/api/config/init`, `/api/config/migrate` |
| GET | `/api/openapi.json`, `/api/docs` |

OpenAPI generation (`openapi.rs`) currently documents the **config surface** in depth; other REST handlers are described here and in source. See [Gateway HTTP API](./api.md).

### Onboarding & personality

| Method | Path |
|--------|------|
| GET | `/api/onboard/catalog`, `/api/onboard/catalog/models`, `/api/onboard/status`, `/api/onboard/sections` |
| GET | `/api/onboard/sections/{section}` |
| POST | `/api/onboard/sections/{section}/items/{key}` |
| GET | `/api/personality`, `/api/personality/templates`, `/api/personality/{filename}` |
| PUT | `/api/personality/{filename}` |

### Agent operations & observability

| Method | Path |
|--------|------|
| GET | `/api/status`, `/api/tools`, `/api/health`, `/api/channels`, `/api/cli-tools`, `/api/cost`, `/api/integrations`, `/api/integrations/settings` |
| GET, POST | `/api/doctor` |
| GET, POST | `/api/cron`; GET, PATCH `/api/cron/settings`; PATCH, DELETE `/api/cron/{id}` |
| GET | `/api/cron/{id}/runs` |
| POST | `/api/cron/{id}/run` — **long timeout** (separate router; default 10 minutes via env) |

### Memory & sessions

| Method | Path |
|--------|------|
| GET, POST | `/api/memory` |
| DELETE | `/api/memory/{key}` |
| GET | `/api/sessions`, `/api/sessions/running`, `/api/sessions/{id}/messages`, `/api/sessions/{id}/state` |
| PUT, DELETE | `/api/sessions/{id}` |
| POST | `/api/sessions/{id}/abort` |

### Canvas (A2UI)

| Method | Path |
|--------|------|
| GET, POST, DELETE | `/api/canvas`, `/api/canvas/{id}` |
| GET | `/api/canvas/{id}/history` |

### Optional features

| Feature | Routes |
|---------|--------|
| `webauthn` | `/api/webauthn/register/start`, `/finish`, `/api/webauthn/auth/start`, `/finish`, `/api/webauthn/credentials`, DELETE `/api/webauthn/credentials/{id}` |
| `plugins-wasm` | GET `/api/plugins` |

### Integrations & hooks

| Method | Path |
|--------|------|
| POST | `/webhook`, `/hooks/claude-code`, `/linq`, `/nextcloud-talk`, `/webhook/gmail` |
| GET, POST | `/whatsapp`, `/wati` (verification + message handlers) |

### Static dashboard

| Method | Path |
|--------|------|
| GET | `/_app/{*path}`, SPA fallback | Non-API GET serves bundled UI assets |

## Real-time transports

| Transport | Path | Purpose |
|-----------|------|---------|
| SSE | `GET /api/events`, `GET /api/events/history` | Dashboard event firehose + replay buffer |
| WebSocket | `/ws/chat` | Agent chat; JSON protocol in [`ws.rs`](../../../../crates/zeroclaw-gateway/src/ws.rs). Client→server: `message`, optional first-frame `connect`. Server→client: `session_start`, `connected`, `chunk`, `thinking` (maps from [`TurnEvent::Thinking`](../../../../crates/zeroclaw-api/src/agent.rs)), `tool_call` / `tool_result` (stable `id`), `agent_start` / `agent_end`, `chunk_reset`, `done`, `aborted`, `error`. With **`gateway-voice-duplex`**, the same socket may carry voice duplex JSON (`speech_start`, `speech_end`, `barge_in`, `tts_chunk`, …) parsed in [`voice_duplex.rs`](../../../../crates/zeroclaw-gateway/src/voice_duplex.rs). |
| WebSocket | `/ws/canvas/{id}` | Live canvas sync |
| WebSocket | `/ws/nodes` | Node discovery / tooling |
| WebSocket | `/acp` | ACP bridge (`zeroclaw.acp.v1`) |

## Operational limits

Defined in [`lib.rs`](../../../../crates/zeroclaw-gateway/src/lib.rs):

- **Body limit**: `MAX_BODY_SIZE` (64 KiB) on the main router.
- **Timeout**: default 30 s per request; override with `ZEROCLAW_GATEWAY_TIMEOUT_SECS`.
- **Cron run**: `ZEROCLAW_GATEWAY_LONG_RUNNING_REQUEST_TIMEOUT_SECS` (default 600 s) for `POST /api/cron/{id}/run` only.
- **Rate limiting**: pairing and webhook buckets (`RATE_LIMIT_*` constants); sliding-window implementation in the same module.

## Related reading

- [Gateway HTTP API](./api.md) — config CRUD, JSON Patch, secrets, stable error codes, Scalar.
- [Architecture overview](../architecture/overview.md), [Crates](../architecture/crates.md).
- Issue **#6175** — gateway HTTP API / OpenAPI tracking (referenced in code comments).
