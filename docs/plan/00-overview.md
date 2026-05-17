# Pi-Archon — Overview

> Canonical plan. Per-project deep dives live in `01-agent-factory.md`, `02-orchestrator.md`, `03-desktop-ui.md`. Architecture decisions in `04-architecture-decisions/`. Design system, errors, and roadmap in `05`–`07`.

---

## 1. Context

The repo `pi-archon-gui` starts as a bare scaffold. The product is a **planner → executor → reviewer multi-agent coding system** that:

- Wraps **Earendil-Works `pi`** ([github.com/earendil-works/pi](https://github.com/earendil-works/pi)) — a TypeScript agent runtime with JSON-RPC, skills, extensions, multi-provider LLM support — as the per-agent workhorse.
- Borrows **Archon's** ([coleam00/Archon](https://github.com/coleam00/Archon)) DAG-of-nodes workflow idea, reimplemented in **Go** with planner/executor/reviewer roles, incremental "slices," and optional Temporal-backed durability.
- Ships **two remote network backends** — orchestrator and agent factory — designed primarily for **agentic consumption** (other AI agents drive them via MCP + gRPC) with a polished Tauri desktop UI as one of multiple clients.

Three independent, isolated projects share the repo, each with its own root folder, build, configs, and service surface. The backends are **never embedded as desktop sidecars** — they are always reached over the network. The desktop UI is human-facing; the same APIs are simultaneously consumed by AI agents, CI pipelines, and third-party tooling.

The product slot is crowded — Cursor, Windsurf, Cline, Claude Code, Plandex, OpenHands all compete — so we deliberately steal: Cline's **plan/act mode toggle**, Plandex's **rewindable diff sandbox**, Windsurf's **codemap visualization**, Claude Code's **skill marketplace pattern**, CrewAI's **role-based parallelism**, and Linear/Raycast's **dark-first neutral aesthetic**.

---

## 2. Confirmed Decisions

| Decision | Choice |
|---|---|
| Shared "document base" | **Workspace folder only.** No RAG, no separate doc store. The session-mounted workspace is the read/write/share medium. |
| Frontend stack | **SvelteKit 2 + Svelte 5 (runes) + shadcn-svelte + Motion + Tailwind v4** |
| Orchestrator deployment | **Remote network service, multi-user, multi-client from day one.** Auth, RBAC, multi-tenancy, audit. Desktop UI, AI agents, and CI/automation are peer clients. |
| Agent factory deployment | **Remote network service.** Reachable from the orchestrator and (optionally, with scoped tokens) from external agents. Never bundled as a desktop sidecar. |
| API style | **gRPC + gRPC-Web (primary)** for typed clients, **MCP server** on both backends for agentic consumption, **REST gateway** (gRPC-Gateway transpile) for casual HTTP use. |
| Fallback task queue (non-durable mode) | **Custom in-memory queue persisted to SQLite.** Zero external brokers. |
| Durable mode | **Temporal Go SDK**, opt-in per-session via `durable: true` flag. |
| Agent runtime | **`@earendil-works/pi-coding-agent`** in JSON-RPC mode, one pi process per agent slot. |
| Workflow shape | **Slice-based DAG**: Plan → Slice₁ (Exec → Review → Commit) → Slice₂ … → Done. Tasks within a slice are marked parallel/sequential by the planner. |

---

## 3. Top-Level Architecture

```
                       ┌─────────────── CLIENTS (peers) ───────────────┐
                       │                                                │
   ┌───────────────┐   │   ┌─────────────────┐   ┌────────────────┐    │
   │ Tauri Desktop │   │   │ AI Agents       │   │ CI / Automation│    │
   │ UI (Svelte)   │   │   │ (Claude, GPT,   │   │ scripts, bots  │    │
   │ human-facing  │   │   │  Cursor, etc.)  │   │                │    │
   └───────┬───────┘   │   └────────┬────────┘   └────────┬───────┘    │
           │           │            │                     │             │
           │ gRPC-Web  │            │ MCP / gRPC          │ REST / gRPC │
           │  + REST   │            │                     │             │
           └─────┬─────┴────────────┴─────────────────────┘             │
                 │                                                       │
                 ▼   ── TLS, mTLS optional, JWT/OIDC, scoped API tokens ─┘
   ┌───────────────────────────────────────────────────────────────────┐
   │  ORCHESTRATOR (Go) — remote network service                       │
   │  • gRPC server + gRPC-Web + REST gateway + MCP server             │
   │  • Auth/RBAC, multi-tenant, audit                                 │
   │  • Workflow engine (slice DAG)                                    │
   │  • Task queue: in-mem+SQLite (default)  OR  Temporal (durable)    │
   │  • Persistence: SQLite (single-node) or Postgres (scale-out)      │
   │  • Orchestrator's own router LLM (for executor selection)         │
   └───────────────┬───────────────────────────────────┬───────────────┘
                   │ gRPC (mTLS in prod)               │ gRPC
                   ▼                                   ▼
   ┌────────────────────────────────────┐   ┌────────────────────────────┐
   │  AGENT FACTORY #1 (TypeScript)     │   │  AGENT FACTORY #N          │
   │  remote network service            │   │  (horizontal scale-out)    │
   │  • gRPC + MCP server               │   │                            │
   │  • Pool of pi processes            │   │                            │
   │  • Workspace sandbox extension     │   │                            │
   │  • Skill installer (gh/local/url)  │   │                            │
   │  • Model registry, credentials     │   │                            │
   └───────────────┬────────────────────┘   └────────────────────────────┘
                   │ JSON-RPC over stdio
                   ▼
              pi processes (one per agent slot) — sandboxed to workspace dir
                   │
                   ▼
              SHARED NETWORKED WORKSPACE FILESYSTEM
              (NFS / object-store gateway / per-deployment volume)
```

**Key invariants:**

- The backends are **always remote** — even in dev, the desktop app talks to them over TCP/TLS, not stdio. There is no embedded-sidecar fallback.
- **Every public method on every backend is reachable by an AI agent** (via MCP) and by a typed client (via gRPC). Schema parity is enforced.
- Each pi process = one agent (planner, an executor, a reviewer) with **isolated session, isolated context, no cross-talk**.
- All three agent types in a session **share the workspace dir** (read/write boundary enforced by the factory).
- The orchestrator never spawns pi directly — it always goes through the factory's gRPC API.
- The orchestrator is the only source of truth for plan state, slice progress, queue state, and audit log.

---

## 4. Consumption model — designed for agents, not just humans

Both backends expose their full surface through three coordinated channels:

| Channel | Audience | Notes |
|---|---|---|
| **gRPC** (protobuf, streaming) | Typed clients (Tauri UI, Go/Python/TS SDKs, internal services) | Source of truth. Schemas in `proto/`. Reflection enabled. |
| **MCP server** | AI agents (Claude, GPT, any MCP-capable LLM) | Auto-generated from gRPC: every method becomes a tool; streams become resources; long-running ops return correlation IDs to poll. |
| **REST + OpenAPI** | Casual HTTP clients, curl, webhooks, no-code | `grpc-gateway` transpile + generated OpenAPI 3.1. Triggers from Zapier, n8n, GitHub Actions. |

**Agent-friendly API rules** (enforced via lint + code review):

1. **Idempotency keys** on every mutating method — agents retry safely.
2. **Long-running operations return correlation IDs** alongside streams; `GetOperation(id)` for poll-style consumers.
3. **All errors carry actionable text + a stable code + a doc URL.** No bare codes.
4. **Schema reflection** turned on (gRPC reflection, MCP `tools/list`, `/openapi.json`).
5. **Capability discovery endpoint** (`Capabilities()`) describes features, limits, version, supported skills, supported providers.
6. **Natural-language descriptions** in proto comments are surfaced verbatim as MCP tool descriptions — written for LLM readers.
7. **No hidden side-effects.** Every state change is visible through `ListAuditEvents`.
8. **Scoped API tokens** (per-project, per-role) — agents get exactly the permissions they need.

A Claude or Cursor agent can natively use the orchestrator to plan/execute/review a coding task on a remote workspace with the same fidelity as our Tauri UI.

---

## 5. Cross-cutting concerns

### 5.1 Configuration

- All three services use **env-only** config (12-factor), each with `.env.example` checked in.
- No service has compiled-in defaults for ports, paths, or credentials.
- `tools/dev/run-local.sh` composes the dev stack with sensible local defaults.

### 5.2 Secrets

- Never logged. Never returned through gRPC responses.
- Backends: env → OS keychain (`keytar` for factory, `keyring-rs` for Tauri) → Vault.
- LLM API keys live with the **factory**, not the orchestrator — the orchestrator only references them by name.

### 5.3 Observability

- **OpenTelemetry traces** end-to-end (UI → orchestrator → factory → pi process). Trace ID propagated through gRPC metadata.
- **Prometheus metrics** on all services (queue depth, agent utilization, plan-to-commit duration, review-score distribution).
- **Structured JSON logs** with `session_id`, `slice_id`, `task_id`, `agent_id` fields.
- UI dashboard surfaces highlights; advanced users point Grafana at Prometheus.

### 5.4 Build & packaging

- `buf` for proto management across all three services.
- `make` targets at repo root (`make proto`, `make build`, `make e2e`).
- **Desktop installer** (Windows MSI, macOS DMG, Linux AppImage + deb) ships the UI only — no embedded backends. First run prompts for an orchestrator URL.
- **Docker images** for orchestrator and factory: `pi-archon/orchestrator:<tag>` and `pi-archon/factory:<tag>`. Helm charts under `deploy/helm/`.
- **`docker-compose.yml`** at repo root spins up an end-to-end dev stack so a developer running the UI locally just points it at `localhost:50051`.
- GitHub Actions: build matrix, sign macOS bundle, sign Windows installer, publish Docker images on tag.

### 5.5 Versioning

- Independent semver per service; protos use buf breaking-change linting.
- Orchestrator declares its required factory protocol version; mismatch returns `FAILED_PRECONDITION` on health check.

---

## 6. Repo layout (final)

```
pi-archon-gui/
├── AGENTS.md                     # testing charter (anti-patterns)
├── README.md
├── LICENSE
├── Makefile
├── buf.yaml
├── buf.gen.yaml
├── docker-compose.yml            # dev: orchestrator + factory (+ optional Temporal/Postgres)
├── deploy/
│   ├── helm/                     # k8s charts for orchestrator + factory
│   └── compose/                  # variant compose files (durable, postgres)
├── .github/workflows/
│   ├── ci.yml
│   ├── desktop-release.yml
│   └── docker-release.yml
├── docs/
│   ├── plan/                     # canonical plan + per-project deep dives (this folder)
│   ├── manual-tests/             # per-milestone E2E checklists
│   └── api/                      # generated proto docs
├── proto/                        # SHARED proto definitions
│   ├── agent_factory.proto
│   ├── orchestrator.proto
│   └── common.proto
├── apps/
│   └── desktop/                  # Tauri + SvelteKit       (Project 3)
└── services/
    ├── agent-factory/            # TS pi factory            (Project 1)
    └── orchestrator/             # Go orchestrator          (Project 2)
```

---

## 7. Open recommendations — defaults I picked

Explicitly listed so they can be overridden before coding begins:

1. **Sandbox enforcement lives in the factory, not in pi.** Pi has no built-in sandbox; doing it in the factory keeps pi vendor-clean.
2. **Orchestrator uses Haiku 4.5 as its own router LLM** for cheap planning-fallback / executor-selection. Configurable.
3. **Single workspace per session.** No cross-workspace slices. Matches doc-base = workspace.
4. **Git commit per slice, not per task.** Easier review, cleaner history.
5. **Skill GitHub installs are unsigned** at v1. Signed/verified skills is a v2 feature.
6. **No browser-based dashboard at v1** — the Tauri app is the only first-party UI. Orchestrator's gRPC is reachable for third parties.
7. **No embedded backends in the desktop installer.** Always connects to a remote orchestrator. Solo developers use `docker-compose up` for a local backend.
8. **MCP server is co-resident with the gRPC server** on both backends, not a separate process. Single deployable, two ports, shared auth/audit/tenancy.
9. **OpenAPI 3.1 generated from protos** — no hand-written REST schema drift.

---

## See also

- `01-agent-factory.md` — Project 1 (TS, pi pool, gRPC+MCP+REST, skills, sandbox).
- `02-orchestrator.md` — Project 2 (Go, workflow engine, task queue, RBAC, Temporal).
- `03-desktop-ui.md` — Project 3 (Tauri + SvelteKit, screens, design discipline).
- `04-architecture-decisions/` — ADRs for the load-bearing decisions.
- `05-design-system.md` — visual language, components, animations.
- `06-error-catalog.md` — every error code across all three services.
- `07-roadmap.md` — milestones, verification, manual-test plan.
