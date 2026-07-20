# Decision Log

Record durable product/tech decisions. Newest first.

## 2026-07-19 — Name: Axon

Agent Connect Layer brand. Emphasizes signal path (apps ↔ agents), not an “agent brain.”

## 2026-07-19 — Thin L0 first

Multiplex + session + stream + permission before pool/obs/transcript. L1/L2 attach via EventBus.

## 2026-07-19 — Protocol: ACP v1 stdio

Stand on Agent Client Protocol and official SDKs. Do not invent a parallel agent wire protocol.

## 2026-07-19 — App transport: HTTP + WebSocket

Simple for web and desktop hosts. Normalize events; do not expose raw JSON-RPC to apps.

## 2026-07-19 — Implementation language (planned): TypeScript

Official TS ACP SDK, fast iteration for spawn/WS. Revisit single-binary Rust only if packaging/ops demand it.

## 2026-07-19 — Apps address agents by `agentId`

Connection pooling is an internal optimization (L1). Public API defaults to `agentId` + session, not mandatory connection handles.
