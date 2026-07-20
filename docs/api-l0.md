# L0 API Sketch — Axon

App-facing surface for Phase 1. Stable enough to implement against; not a frozen contract until L0 exit (then semver via `packages/protocol`).

Base URL example: `http://127.0.0.1:8787`  
Events: `ws://127.0.0.1:8787/sessions/:sessionId/events`

---

## Agents

### `GET /agents`

List registered agents and optional live connection hints.

```json
{
  "agents": [
    {
      "id": "example",
      "labels": ["dev"],
      "status": "idle"
    }
  ]
}
```

---

## Sessions

### `POST /sessions`

Create a session (Axon may spawn/reuse a connection internally).

```json
// request
{
  "agentId": "example",
  "cwd": "/absolute/path/to/project",
  "appId": "web-workbench",
  "mcpServers": []
}

// response
{
  "sessionId": "axn_sess_01H...",
  "agentId": "example",
  "connectionId": "axn_conn_01H...",
  "agentSessionId": "sess_abc",
  "cwd": "/absolute/path/to/project"
}
```

### `GET /sessions/:sessionId`

```json
{
  "sessionId": "axn_sess_01H...",
  "agentId": "example",
  "connectionId": "axn_conn_01H...",
  "state": "idle",
  "cwd": "/absolute/path/to/project",
  "appId": "web-workbench"
}
```

### `DELETE /sessions/:sessionId`

Close session (ACP `session/close` if supported; always drop local state).

---

## Prompt & cancel

### `POST /sessions/:sessionId/prompt`

```json
// request
{
  "prompt": "Explain this repo",
  // or structured:
  "content": [{ "type": "text", "text": "Explain this repo" }]
}

// response (when turn completes)
{
  "stopReason": "end_turn"
}
```

Streaming content is **not** in this response body; use events.

Alternative (if we adopt async turns later):

```json
{ "turnId": "axn_turn_01H..." }
```

L0 default: await turn completion on the HTTP request **or** return immediately and only use WS — pick one in implementation (prefer **HTTP awaits stopReason**, WS carries chunks).

### `POST /sessions/:sessionId/cancel`

```json
{}
```

---

## Permissions

### `POST /sessions/:sessionId/permissions/:requestId`

```json
// request
{
  "optionId": "allow-once"
}

// or cancel
{
  "outcome": "cancelled"
}
```

---

## Events (WebSocket)

Connect: `WS /sessions/:sessionId/events`

All messages:

```json
{
  "type": "agent_message_chunk",
  "sessionId": "axn_sess_01H...",
  "ts": "2026-07-19T12:00:00.000Z",
  "payload": {}
}
```

### Event types (L0)

| type | payload (sketch) |
|---|---|
| `agent_message_chunk` | `{ text, messageId? }` |
| `agent_thought_chunk` | `{ text, messageId? }` |
| `user_message_chunk` | `{ text, messageId? }` |
| `tool_call` | `{ toolCallId, title, status, kind? }` |
| `tool_call_update` | `{ toolCallId, status?, content? }` |
| `plan` | `{ entries: [...] }` |
| `permission_request` | `{ requestId, toolCall, options }` |
| `stop` | `{ stopReason }` |
| `error` | `{ code, message }` |
| `connection_lost` | `{ reason }` |

Unknown ACP updates: emit `raw_update` with passthrough JSON **or** ignore until needed — prefer explicit allow-list in L0, log the rest.

---

## Optional L0 (if needed by first real agent)

### Connections (explicit)

Only if apps need to pin a connection:

- `POST /connections` `{ agentId }`
- `DELETE /connections/:connectionId`
- `POST /sessions` may accept `connectionId` instead of `agentId`

Default path: apps only pass `agentId`.

---

## Error response shape

```json
{
  "error": {
    "code": "agent_spawn_failed",
    "message": "spawn EACCES ...",
    "details": {}
  }
}
```

HTTP status: `4xx` for caller faults, `5xx` / `503` for agent/process faults.
