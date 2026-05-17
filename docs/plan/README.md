# Pi-Archon — Plan Index

This folder is the canonical plan for the Pi-Archon multi-agent coding system.

## Read in order

1. [`00-overview.md`](./00-overview.md) — Context, decisions, top-level architecture, consumption model, cross-cutting concerns, repo layout.
2. [`01-agent-factory.md`](./01-agent-factory.md) — Project 1: TypeScript pi-process pool, gRPC + MCP + REST, skills, sandbox.
3. [`02-orchestrator.md`](./02-orchestrator.md) — Project 2: Go workflow engine, task queue, multi-tenancy, Temporal.
4. [`03-desktop-ui.md`](./03-desktop-ui.md) — Project 3: Tauri + SvelteKit thin client.
5. [`04-architecture-decisions/`](./04-architecture-decisions/) — ADRs for the load-bearing decisions.
6. [`05-design-system.md`](./05-design-system.md) — Visual language, components, motion discipline.
7. [`06-error-catalog.md`](./06-error-catalog.md) — Every error code across all three services.
8. [`07-roadmap.md`](./07-roadmap.md) — Milestones, manual-test plan, end-to-end verification.

## See also

- [`../../AGENTS.md`](../../AGENTS.md) — Testing charter. Manual E2E is the source of truth. Anti-patterns are listed.
- [`../manual-tests/`](../manual-tests/) — Per-milestone manual test checklists (added as milestones begin).
- [`../prototypes/`](../prototypes/) — Interactive HTML prototype (`ui-end-state.standalone.html` for offline) and pre-rendered `screens/` PNGs of every screen. These are embedded throughout `00-overview.md`, `03-desktop-ui.md`, and `05-design-system.md` and are the visual contract for the build.
