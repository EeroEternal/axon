# Roadmap & Development Cadence — Axon

## North star

Ship a **reliable L0 connect layer** quickly, then thicken only where real multi-agent usage hurts (cost, chaos, blindness).

```
Phase 0 foundation → Phase 1 L0 connect → Phase 2 multi-tenant usage
       → Phase 3 L1 efficiency → Phase 4 L2 observe/record → Phase 5 harden
```

Cadence is **weekly slices**, each slice demoable. Prefer vertical slices (one path works end-to-end) over horizontal “framework first.”

---

## Phase 0 — Foundation (this commit)

**Goal:** shared product truth so implementation does not drift.

| Deliverable | Status |
|---|---|
| Name, positioning, scope | Done (`docs/scope.md`) |
| Roadmap & cadence | Done (this doc) |
| Architecture sketch | Done (`docs/architecture.md`) |
| L0 API sketch | Done (`docs/api-l0.md`) |
| Repo skeleton | In progress |

**Exit:** contributors agree what is in/out and what “L0 done” means.

---

## Phase 1 — L0 Connect (MVP)

**Goal:** one app, two agents (or one real + one fake), full prompt stream + permission.

### Week 1 — Wire

- Scaffold monorepo (`apps/gateway`, `packages/core`, `packages/acp-client`, `packages/protocol`)
- Agent registry load from YAML
- Spawn + ACP `initialize` via official SDK
- In-memory `Connection` + `Session` maps
- `POST /sessions` + `POST /sessions/:id/prompt` (blocking or async turn id)
- WS (or SSE) push of normalized `session/update` events
- Graceful process kill on connection delete

**Demo:** CLI or minimal HTML page: send “hello”, see agent chunks.

### Week 2 — Client surface + safety

- Implement ACP client handlers needed for target agents:
  - `session/request_permission` → event + resolve API
  - `fs/read_text_file` / `fs/write_text_file` with **cwd sandbox**
- `session/cancel`
- Crash / exit → error event to subscribers
- stderr capture for load failures
- Example `agents.example.yaml` + docs to run a public sample agent

**Demo:** permission prompt in client; reject/allow; cancel mid-turn.

### Week 3 — Multiplicity

- Concurrent sessions on one connection (if agent supports) and across agents
- `appId` tagging on sessions
- List agents + connection/session status endpoints
- Stable error model (`agent_exited`, `permission_timeout`, …)
- Golden path e2e test with fake ACP agent process

**Exit (L0 done):** criteria in `docs/scope.md` all green.

---

## Phase 2 — Real usage hardening

**Goal:** second app or second agent without rewriting the core.

- Second real ACP agent in registry (config-only)
- Optional `session/load` / `session/close` when capabilities allow
- MCP server list pass-through on `session/new`
- Auth stub: static token or local unix socket mode
- Structured logging (json lines)
- Dockerfile / single-command dev run

**Exit:** “add agent = edit YAML + restart” is true for at least two agents.

---

## Phase 3 — L1 Efficiency

**Goal:** less spawn thrash, controlled concurrency.

| Work item | Notes |
|---|---|
| Connection pool | Reuse idle connections per `agentId` (+ env hash) |
| Warm pool | Optional min-idle |
| Limits | max sessions / concurrent prompts |
| Idle GC | TTL for session & connection |
| Routing hooks | sticky `appId+agentId` first |

**Exit:** metrics show reuse; cold start path still works when pool empty.

---

## Phase 4 — L2 Observability & recording

**Goal:** see and replay without changing connect semantics.

| Work item | Notes |
|---|---|
| EventBus subscribers | metrics, transcript, audit as plugins |
| Trace id | inject at prompt, echo on events |
| Transcript store | file or sqlite first |
| Debug ACP tap | optional NDJSON ring buffer per connection |
| Read APIs | `GET /sessions/:id/transcript` |

**Exit:** one real debugging story uses transcript + metrics.

---

## Phase 5 — Harden & publish

- Versioned app-facing protocol (`protocol` package semver)
- SDK client for apps (TS first)
- CI, release tags, smoke tests against pinned agent versions
- Public README quickstart
- Decide license

---

## Cadence rules

1. **One vertical slice per week** — runnable path > perfect abstractions.
2. **No L1/L2 in the hot path until L0 criteria pass** — middleware hooks may exist early, implementations stay empty.
3. **Fake agent first, real agent second** — keep a tiny in-repo ACP agent fixture for CI.
4. **Config over code** — new agent must not require a new package.
5. **Write down breaks** — if ACP version or agent adapter changes, note in `docs/compat.md` (create when first needed).
6. **API freeze windows** — after L0 exit, app-facing events/API need changelog for breaking changes.

## Suggested issue labels

- `phase/0` … `phase/5`
- `area/registry` `area/connection` `area/session` `area/events` `area/permission` `area/fs` `area/pool` `area/obs`
- `type/bug` `type/doc` `type/spike`

## Decision log (short)

| Decision | Choice | Rationale |
|---|---|---|
| Protocol | ACP v1 stdio | Ecosystem + official SDKs |
| Product name | Axon | Thin signal path, not a “brain” |
| Thickness | L0 thin | Multiplex first |
| App transport | HTTP + WebSocket | Simple for web/desktop hosts |
| Implementation language (planned) | TypeScript | Official `@agentclientprotocol/sdk`, fast spawn/WS iteration |

Language can be revisited if a single-binary Rust gateway becomes a hard requirement; architecture docs stay language-agnostic.
