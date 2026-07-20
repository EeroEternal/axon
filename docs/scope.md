# Product Scope — Axon

## One-liner

**Axon** is a thin **Agent Connect Layer**: it flattens the many-to-many relationship between frontend apps and ACP-compatible agents, then grows efficiency, observability, and recording on top of a stable connection surface.

## Problem

Without a connect layer:

```
each App × each Agent = custom integration
```

Cost grows with every new agent or client. Process lifecycle, streaming, cancel, permissions, and session recovery get reimplemented per pair.

## Solution

```
Apps  ──unified API──►  Axon  ──ACP──►  Agents
```

- **Apps** only know: list agents, open session, prompt, subscribe events, resolve permission, cancel.
- **Agents** only speak ACP.
- **Axon** owns: registry, process, initialize, session routing, stream bridge, optional middleware.

## Positioning

| Layer | Owner | Example |
|---|---|---|
| UX / product UI | App | chat workbench, bot, internal console |
| **Connect** | **Axon** | multiplex, route, bridge |
| Agent runtime | Agent binary | Claude Code, Codex, Gemini CLI, custom ACP agent |
| Tools / MCP | Agent (+ config from Axon) | MCP servers passed at `session/new` |

Sibling products in the stack (not Axon):

- LLM gateways (model routing) — different problem
- Policy/sandbox kernels (e.g. action-time enforcement) — can integrate later, not L0
- IDE Agent Panels — consumers of Axon, not competitors if we stay thin

## In scope (by phase)

### L0 — Connect (MVP, must ship first)

- Agent registry (config-driven: command, args, env, labels)
- Connection lifecycle: spawn, `initialize`, capabilities, kill, crash signal
- Session lifecycle: `new` / optional `close`; route by session id
- Prompt turn: `session/prompt` + stream `session/update` to clients
- Cancel: `session/cancel`
- Permission loop: forward `request_permission` → app resolves option
- Minimal workspace FS for ACP client methods (`read` / `write` with cwd sandbox) when advertised
- Normalized event schema over WebSocket (or equivalent)
- Multi-app / multi-session isolation (no cross-talk)

### L1 — Efficiency

- Connection reuse / pool (same agent definition)
- Warm initialize for hot agents
- Concurrency limits (per app / per agent)
- Simple routing: sticky, least-load, label match
- Idle GC for connections and sessions
- Health: dead process → mark sessions disconnected

### L2 — Observability & recording

- Trace id per prompt turn (tool / permission / stopReason)
- Metrics: connections, sessions, latency, error/crash rates
- Transcript store per session (chunks, tools, permissions)
- Audit: who approved what on which cwd
- Optional raw ACP NDJSON debug backlog

## Out of scope (explicit non-goals for early phases)

Do **not** build these into the core path until L0 is proven:

| Non-goal | Why |
|---|---|
| Full multi-agent orchestration graphs | Different product |
| IDE-grade buffer sync / inline diff UX | Belongs in the app |
| Replacing ACP with a new agent protocol | Stand on official ACP + SDK |
| Terminal full surface (L0 optional later) | Increases client burden; add when agents need it |
| Heavy multi-tenant IAM / billing | `appId` + simple auth is enough first |
| Built-in chat UI as the product | Demo UI ok; product is the layer |
| MCP server implementation catalog | Pass-through config only |
| Remote ACP transport R&D | Use stdio until protocol stabilizes remote |

## Success criteria

### L0 done when

1. One app can switch `agentId` without code changes beyond config.
2. Two apps × two agents run concurrent sessions with no event mix-ups.
3. Permission request round-trips correctly.
4. Agent process crash is visible to the app as a clear session/connection error.
5. A new ACP agent is added by **config only** (command/args/env).

### Product-market fit signals (later)

- Second independent app integrates Axon without forking.
- Operators use transcript/metrics to debug a real incident.
- Connection reuse measurably cuts cold-start cost.

## Design principles

1. **Thin core** — L1/L2 are middleware; turning them off still leaves a working connect layer.
2. **Pass-through default** — do not reinterpret agent semantics unless needed for safety (path sandbox).
3. **Stable app-facing schema** — ACP can evolve behind `packages/acp-client`; apps depend on Axon events/API.
4. **Config over code** for agents — registry YAML/JSON, not hard-coded binaries.
5. **Local-first path** — stdio agents on the machine (or host) running Axon; remote is later.

## Primary users

| User | Need |
|---|---|
| App developer | One SDK/API for any ACP agent |
| Platform owner | Register agents, run a gateway, observe later |
| Agent author | Only implement ACP; Axon is just another client |

## Non-users (for now)

- End users of coding UIs (they use the app, not Axon directly)
- Teams that only ever bind one vendor SDK and never switch agents
