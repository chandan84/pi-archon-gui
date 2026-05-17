# ADR-0005 — SQLite is the default orchestrator store; Postgres is optional

**Status:** Accepted
**Date:** 2026-05-17

## Context

The orchestrator needs persistent state for sessions, plans, slices, tasks, audit, queue WAL, users, tokens. A multi-tenant remote service traditionally implies Postgres or similar. But the requirement to be easy to self-host (and the task-queue ADR-0003 keeping zero infra) push toward an embedded store.

## Decision

Default DSN is SQLite (libSQL, WAL mode), running embedded in the orchestrator process. Schema and queries are managed via `sqlc` so the data layer is database-portable. A Postgres driver is wired up and selectable via `ORCH_DB_DSN=postgres://...`.

SQLite handles single-node deployments — solo developers, small teams, demo SaaS instances. Postgres is the upgrade path for multi-node deployments (HA orchestrator behind a load balancer) and for shared analytics warehouses.

## Consequences

**Positive**
- One binary, no infra dependency, instant `docker compose up`.
- Backup is a file copy.
- Tests use real SQLite — fast, no test container needed.
- Migration story to Postgres is `pg_loader` or a one-off dump+restore.

**Negative**
- SQLite has limited write concurrency. With many parallel sessions on a single orchestrator, the WAL can become contended.
- Horizontal scale-out of the orchestrator requires Postgres (or libSQL turso replication).

**Mitigations**
- Document the rough scale ceiling (~50 concurrent active sessions per SQLite orchestrator).
- Provide a `make migrate-to-postgres` target.
- The queue WAL writes are append-only and small — main hot path is well-suited to SQLite.
