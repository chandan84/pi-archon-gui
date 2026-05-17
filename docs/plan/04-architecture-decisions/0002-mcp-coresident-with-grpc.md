# ADR-0002 — MCP server is co-resident with gRPC on both backends

**Status:** Accepted
**Date:** 2026-05-17

## Context

Backends must be **agent-consumable first** (per ADR-0001). The Model Context Protocol (MCP) is the de-facto standard for exposing tools to LLM agents (Claude, Cursor, and increasingly OpenAI). gRPC remains the primary typed-client API.

We could deploy MCP and gRPC as separate processes (clean separation) or co-resident in the same process (shared everything).

## Decision

MCP and gRPC live in the **same process** on each backend, on **separate ports**:

- Agent factory: gRPC `50061`, MCP `50062`, REST `50063`
- Orchestrator: gRPC `50051`, gRPC-Web `50052`, MCP `50053`, REST `50054`

The MCP server is **auto-generated from gRPC service definitions** — each gRPC method becomes one MCP tool. Long-running streams are mapped to "kick off + poll" via the `GetOperation(id)` endpoint, because agents typically can't hold streams open across LLM turns.

Both surfaces share the same:
- Authentication (JWT / OIDC / scoped API token)
- Tenancy resolution (Org → Project)
- Audit logging
- Idempotency-key handling
- Telemetry (OTel traces span all three surfaces with the same trace ID)
- Business logic (one implementation, two transports)

## Consequences

**Positive**
- Zero schema drift — proto is the source of truth, MCP descriptions live in proto comments.
- Half the deployable artifacts, half the secrets to rotate, half the network policy.
- An agent and a UI client see the same auditable actions.

**Negative**
- Cross-cutting concerns must be designed once to work for both transports (e.g., streaming responses).

**Mitigations**
- `mcp/bridge.go` (orchestrator) and `src/mcp/tool-bridge.ts` (factory) own the mapping rules. Any new gRPC method automatically gets an MCP tool unless explicitly marked `(mcp_expose) = false` in the proto.
