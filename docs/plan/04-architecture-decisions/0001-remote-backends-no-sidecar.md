# ADR-0001 — Backends are remote network services, never desktop sidecars

**Status:** Accepted
**Date:** 2026-05-17

## Context

The product surface includes a Tauri desktop app. A common shortcut for "desktop + backend" architectures is to bundle the backend as a Tauri sidecar process and ipc to it over stdio. This trades multi-client capability for installer simplicity.

The user explicitly clarified after the first plan draft: *"while the ui is meant to desktop first, the backend services are remote and should be built for agentic consumption."*

## Decision

Both backend services — the agent factory (TS) and the orchestrator (Go) — are **standalone remote network services**. Communication is always over TCP/TLS, never over a stdio sidecar pipe. The Tauri shell ships **without** any embedded backend binary.

For solo developers, a top-level `docker-compose.yml` brings up the full stack against which the UI is pointed at `localhost:50051`.

## Consequences

**Positive**
- Multiple peer clients (UI, AI agents via MCP, CI bots) operate against the same backend with no special casing.
- Backends scale horizontally and deploy as standard container images.
- Auth, multi-tenancy, and observability are designed in from day one — no "is this local or remote?" branches.
- The same code path runs in dev, SaaS, and self-host.

**Negative**
- Solo-developer first-run is harder: they must run `docker compose up` before the UI is useful.
- Installer cannot offer a "zero-network" mode out of the box.

**Mitigations**
- Onboarding wizard detects "no orchestrator at localhost:50051" and offers a one-click "Show me how to start one" with the exact compose command.
- README has a 30-second quickstart.

## Alternatives considered

- **Embedded sidecar for solo users, network for teams.** Rejected: two code paths, ongoing maintenance tax, hides multi-client design from contributors.
- **Bundled local binary spawn on first run.** Rejected for the same reason — operationally indistinguishable from sidecar.
