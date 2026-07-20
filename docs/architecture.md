# Architecture — Axon

## Layers

```
┌──────────────────────────────────────────────────────────┐
│  Apps (many)                                             │
│  HTTP / WebSocket  ·  optional @axon/client later        │
└────────────────────────────┬─────────────────────────────┘
                             │
┌────────────────────────────▼─────────────────────────────┐
│  apps/gateway                                            │
│  REST routes · WS upgrade · auth stub · request IDs      │
└────────────────────────────┬─────────────────────────────┘
                             │
┌────────────────────────────▼─────────────────────────────┐
│  packages/core                                           │
│  Registry · ConnectionManager · SessionManager · EventBus│
│  PermissionBroker · WorkspaceProvider (sandbox)          │
└───────────────┬────────────────────────────▲─────────────┘
                │                            │
                │ commands                   │ updates /
                │                            │ client RPCs
┌───────────────▼────────────────────────────┴─────────────┐
│  packages/acp-client                                     │
│  spawn · NDJSON transport · ACP SDK · capability map     │
└────────────────────────────┬─────────────────────────────┘
                             │ ACP stdio
┌────────────────────────────▼─────────────────────────────┐
│  Agents (many)                                           │
└──────────────────────────────────────────────────────────┘
```

`packages/protocol` holds **app-facing** types (events, REST DTOs). Apps should depend on this schema, not raw ACP types.

## Core objects

```
AgentDefinition
  id, command, args, env, labels, options

Connection
  id, agentId, process, acp handle, capabilities, state
  sessions: Set<sessionId>

Session
  id              # Axon-stable id (public)
  agentSessionId  # ACP session id
  connectionId
  appId?
  cwd
  state           # idle | prompting | closed | error
  createdAt

PermissionRequest
  id, sessionId, toolCall, options, createdAt
```

### Relationship

```
AgentDefinition 1 ── * Connection 1 ── * Session
App (appId)     * ── * Session
```

L0 may use **1 connection per session** (simplest).  
L1 introduces **connection reuse** (1 connection → many sessions) when the agent allows concurrent sessions.

## Data flow

### Prompt turn

```
App  POST /sessions/:id/prompt
  → SessionManager
  → acp-client session/prompt
  → Agent
  → session/update (many) → EventBus → WS subscribers
  → session/prompt result (stopReason) → EventBus (stop) + HTTP response / turn complete
```

### Permission

```
Agent  session/request_permission
  → acp-client handler
  → PermissionBroker (park promise)
  → EventBus  permission_request
  → App  POST .../permissions/:reqId  { optionId }
  → resolve promise → ACP response
```

### Crash

```
process exit
  → Connection state=dead
  → all sessions error event
  → EventBus connection_lost
```

## EventBus (middleware seam)

All normalized session events pass through a bus:

```
emit(event) → [ WS fanout | metrics? | transcript? | audit? ]
```

L0: only WS (and logs).  
L1/L2: attach subscribers without changing producers.

## WorkspaceProvider

Minimal interface for ACP client FS methods:

```ts
interface WorkspaceProvider {
  resolve(path: string): Result<AbsolutePath>; // must stay in allowed roots
  readText(path: string, range?: LineRange): Promise<string>;
  writeText(path: string, content: string): Promise<void>;
}
```

**Allowed roots** default to session `cwd` (+ future `additionalDirectories`).  
Writes are allowed only if client advertised `writeTextFile` and policy allows (L0: allow inside roots).

Terminal support is intentionally deferred; advertise `terminal: false` until Phase 2+.

## Configuration

```yaml
# agents.example.yaml
agents:
  - id: example
    command: npx
    args: ["tsx", "path/to/fake-agent.ts"]
    env: {}
    labels: [dev, fixture]

server:
  host: 127.0.0.1
  port: 8787
```

No agent-specific code paths in core. Adapters belong to the agent binaries themselves (ACP).

## Failure model (app-visible)

| Code | Meaning |
|---|---|
| `agent_not_found` | Unknown agent id |
| `agent_spawn_failed` | Process failed to start |
| `agent_initialize_failed` | ACP initialize error / unsupported version |
| `agent_exited` | Process died under us |
| `session_not_found` | Bad session id |
| `session_busy` | Prompt already in flight (if we disallow concurrent prompts) |
| `permission_timeout` | App did not resolve in time |
| `cancelled` | Turn cancelled (not an unexpected error) |
| `fs_denied` | Path outside sandbox |

## Trust boundary

- Axon runs **with the privileges of its process user**.
- Sandbox is **path policy**, not a full OS sandbox (compose with Keel/boxlite later if needed).
- Apps that can call Axon can trigger agent tool use subject to permission prompts — treat gateway bind address and auth as security-sensitive from day one (even if auth is a stub).

## Non-architecture (avoid)

- Sharing one global ACP connection across unrelated security domains without isolation
- Leaking raw JSON-RPC ids to apps
- Blocking the gateway event loop on long agent turns without async session state
- Encoding agent-specific hacks in `core` (use `_meta` pass-through or registry options only when unavoidable)
