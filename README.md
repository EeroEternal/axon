# Axon

**Agent Connect Layer** — multiplex frontend apps to ACP agents.

Axon sits between applications and [Agent Client Protocol](https://agentclientprotocol.com/) agents. Apps talk to one stable API; agents stay on ACP (stdio JSON-RPC). Many-to-many wiring lives here, not in every client.

```
App A ─┐                    ┌─► Claude Code (ACP)
App B ─┼─► Axon ────────────┼─► Codex (ACP)
App C ─┘   connect layer    └─► Gemini / custom
```

## What it is

| Is | Is not |
|---|---|
| Connection + session multiplex | Full agent OS / orchestrator |
| Thin API over ACP | IDE / Agent Panel UX |
| Process + stream + permission bridge | LLM provider gateway |
| Extensible (efficiency, observability later) | Thick control plane on day one |

## Status

**Phase 0 — foundation.** Product scope and roadmap are defined; L0 connect surface is next.

See:

- [Product scope](docs/scope.md)
- [Roadmap & cadence](docs/roadmap.md)
- [Architecture](docs/architecture.md)
- [API sketch (L0)](docs/api-l0.md)

## Quick mental model

```
AgentDefinition  →  Connection  →  Session  →  Events (WS)
     (config)         (process)     (ACP)         (normalized)
```

## Repo layout (planned)

```
axon/
  apps/
    gateway/          # HTTP + WebSocket entrypoint
  packages/
    core/             # registry, connection, session, event bus
    acp-client/       # ACP SDK adapter (spawn, initialize, RPC)
    protocol/         # shared types & event schema for frontends
  docs/
  agents.example.yaml
```

## License

TBD.
