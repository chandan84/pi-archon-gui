# ADR-0003 — Durable execution is per-session opt-in via a Queue interface

**Status:** Accepted
**Date:** 2026-05-17

## Context

Some users want strict durable execution (survive orchestrator crashes, network blips, host reboots). Others want zero infrastructure overhead — no Temporal server, no Postgres, no Redis. The requirements explicitly say durability should be optional, chosen per session at submission time.

## Decision

The orchestrator owns a `Queue` interface (`internal/queue/queue.go`). Two implementations:

1. **`inmem_sqlite` (default)** — in-memory channel + write-ahead to a SQLite `task_log` table. Survives orchestrator restart by replay. Zero external dependencies. Used by default and when `session.durable = false`.
2. **`temporal` (opt-in)** — Temporal Go SDK workflow per session, dynamic activities for agent calls. Used when `session.durable = true`. Requires a reachable Temporal server (config: `ORCH_TEMPORAL_HOST`).

Workflow logic, slice progression, and retry semantics live **above** the `Queue` interface — identical code regardless of backend. The implementation difference is purely "where does state live and what does recovery look like."

Retry policies (initial interval, backoff coefficient, max interval, max attempts) and per-task timeouts come from session config in both modes.

## Consequences

**Positive**
- Solo devs and SaaS users both get a working system with one binary.
- Production teams can flip `durable = true` per critical session without changing code.
- Switching backends is a deployment decision, not a code change.

**Negative**
- Two implementations to keep in sync — but the surface is small (~6 methods).
- The in-mem-SQLite backend cannot guarantee exactly-once across a host crash mid-write; Temporal can. Documented limitation.

**Mitigations**
- Single end-to-end test suite runs against both backends.
- The `Queue` interface is intentionally small to keep parity feasible.
