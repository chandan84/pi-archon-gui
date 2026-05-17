# 07 — Roadmap & Verification

> Each milestone ends with a **manual E2E pass** against real services. No milestone is "done" until that passes. Per `AGENTS.md`, manual user testing is the source of truth.

---

## Milestones

### M0 — Repo scaffolding & protos (1 week)

- Three project folders, `buf`, `Makefile`, `.env.example`s, CI skeleton, `AGENTS.md` (already in place), this plan in `docs/plan/`.
- `proto/agent_factory.proto`, `proto/orchestrator.proto`, `proto/common.proto` with the service signatures from `01-` and `02-`.
- `docker-compose.yml` at repo root with placeholder service definitions.
- CI: lint, buf-breaking, build-only.

**Exit:** `make build` succeeds for all three projects; `buf lint` passes; CI green.

---

### M1 — Factory MVP (2.5 weeks)

- Pi process pool, JSON-RPC bridge, `StartAgent` / `Prompt` / `Abort`.
- Sandbox extension (`workspace-guard`).
- Skill installer (local + GitHub).
- Model registry, `SetModel`, `ListModels`.
- **gRPC + MCP + REST** surfaces co-resident.
- TLS optional, JWT verification.
- OpenTelemetry traces.

**Manual test:**
- Start/stop agents from a grpcurl client AND from a Claude session via MCP.
- Install one skill from GitHub, one from a local folder.
- Verify sandbox: a skill that writes to `/tmp` outside the workspace hard-fails with `SANDBOX_VIOLATION`.
- `Capabilities()` returns a sane shape consumable by an LLM agent.

---

### M2 — Orchestrator MVP, non-durable (3.5 weeks)

- **gRPC + MCP + REST** co-resident.
- SQLite store with sqlc-generated queries.
- JWT auth, single-org single-user seed.
- Slice state machine end-to-end.
- In-mem + SQLite-WAL task queue.
- Planner → executor → reviewer happy path.
- Git commit gate.
- `GetOperation(id)` polling + `Capabilities()` introspection.

**Manual test:**
- Drive a real "build a TODO CLI" workspace end-to-end via grpcurl.
- Drive the same end-to-end from a Claude session using the MCP server.
- Inspect SQLite directly to verify state transitions.
- Verify git log shows commits with the `Pi-Archon-Slice` trailer.

---

### M3 — Tauri UI MVP (3 weeks)

- Onboarding (server profile picker → orchestrator URL → OIDC/password sign-in).
- Sessions, Plan Review, Live Run, Skills, Models, Settings.
- Real gRPC-Web streams against the remote orchestrator.
- Linear-tier polish on every screen per `05-design-system.md`.

**Manual test:**
- Full UI walkthrough against a `docker-compose`-spawned local backend.
- Same walkthrough against a remote staging deployment.
- Verify reconnect-on-network-blip behavior.

---

### M4 — Multi-tenant & RBAC (2 weeks)

- Orgs, Projects, Users, Roles, Invites, API tokens.
- Audit log surfaces in UI.
- OIDC bridge.
- Tenant-resolution interceptor on every gRPC call.

**Manual test:**
- 3 roles (Owner, Developer, Viewer) across 2 orgs.
- Cross-tenant access attempts fail with `TENANCY_MISMATCH`.
- Audit log entries attribute every action correctly.

---

### M5 — Durable execution (2 weeks)

- Temporal worker, dynamic activities, retry policies, signals for user approvals.
- Per-session `durable: true` flag wires to the Temporal backend.

**Manual test:**
- Kill the orchestrator mid-slice with `durable: false` → SQLite WAL replay resumes the queue.
- Same with `durable: true` → Temporal resumes the workflow on orchestrator restart.

---

### M6 — Dashboards & monitoring (1.5 weeks)

- Org dashboard: active sessions, queue health, factory capacity, cost rollup, audit feed.
- Runtime controls: pause / resume / cancel / delete-files / hot-swap model.

**Manual test:**
- Long-running session with pause/resume/cancel; UI stays consistent.
- Delete-agent-files flow with typed-confirm.

---

### M7 — Polish & competitor-feature parity (2 weeks)

- Plan Codemap canvas.
- Plandex-style cumulative rewindable diff.
- Command palette (⌘K).
- Skill marketplace install-from-GitHub UX.
- Animation pass — every transition obeys `05-design-system.md` §5.
- Accessibility audit (axe + keyboard-only run).

---

### M8 — Packaging & release (1 week)

- Signed installers (macOS DMG with notarization, Windows MSI with Authenticode, Linux AppImage + deb).
- Docker images published on tag.
- Helm charts.
- Docs site (generated from this folder + proto docs).
- Demo video.

**Total: ~18.5 weeks.**

---

## End-to-end verification (re-run after every milestone from M3 onwards)

1. **Start everything:** `docker compose up` boots SQLite-backed orchestrator and a factory with `POOL_MAX=4`. Launch Tauri in dev mode.
2. **Onboarding:** complete first-run wizard, point at `localhost:50051`, sign in as the seed admin.
3. **New session:** point at a fresh empty workspace dir; pick Claude Sonnet for planner, Claude Haiku for executor, Claude Opus for reviewer; threshold = 7.0; durable = off.
4. **Prompt:** "Build me a simple Rust CLI that counts lines in stdin, with tests."
5. **Plan review:** confirm slices appear (expect 2–3 small slices), parallel/sequential markers visible, edit a task title, approve.
6. **Execution:** open Live Run — verify each agent's stream is independent, artifacts populate the tree, parallel tasks run truly in parallel (factory has spare slots).
7. **Review:** first slice scores ≥7 → auto-commits; verify `git log` shows the commit with the `Pi-Archon-Slice` trailer.
8. **Force low score:** second slice — manually tamper with reviewer prompt to under-score → UI prompts; accept-lower → commits with audit entry; or reject → re-executes.
9. **Sandbox proof:** install a known-bad skill that writes `/etc/passwd` → execution hard-fails with `SANDBOX_VIOLATION`; reviewer score = 0; audit logged.
10. **Resilience (non-durable):** `kill -9` the orchestrator mid-slice → restart → in-mem queue restores open tasks from SQLite; session resumes.
11. **Resilience (durable):** repeat (10) with `durable=on` → kill orchestrator → restart → Temporal resumes the workflow.
12. **RBAC:** sign in as Viewer → confirm all mutating actions are disabled in UI and rejected at gRPC.
13. **UI polish:** keyboard-only navigation works end-to-end; dark/light theme toggle clean; no jank in animations on a mid-range laptop.
14. **Agentic-consumption proof:** point a Claude/Cursor session at the orchestrator's MCP endpoint with a scoped API token; it discovers tools via `Capabilities()`, creates a session, drives a full plan→execute→review→commit cycle without touching the UI; audit log shows the agent's actions clearly attributed to the token.
15. **Multi-client concurrency:** the UI and an MCP-driven agent operate on different sessions in the same project simultaneously; no cross-contamination; both see consistent dashboard state.

---

## Per-milestone checklists

Live in `docs/manual-tests/Mxx.md` and are created at the start of each milestone. Each is a literal step-by-step list a tester (or an automation that drives the UI like a tester) walks through. They are explicitly **not** automated unit tests — see `AGENTS.md`.
