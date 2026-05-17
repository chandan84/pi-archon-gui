# 02 — Orchestrator (Go)

**Path:** `services/orchestrator/`
**Language:** Go 1.23, modules, `buf` for protos.
**Deployment:** remote network service. Single static binary; Docker image; horizontally scalable behind gRPC LB; SQLite for single-node, Postgres for multi-node.
**Why Go:** performance, single static binary, first-class concurrency, mature Temporal SDK, ideal for a network service.

---

## 1. Responsibilities

1. **gRPC + MCP + REST API** for all clients — Tauri UI, AI agents (Claude/GPT/Cursor/etc.), CI bots, custom SDKs.
2. **Multi-tenant + RBAC** (Org → Project → Workspace; roles: Owner, Admin, Developer, Viewer; plus scoped machine tokens for agents).
3. **Auth**: JWT bearer (HS256/RS256), optional OIDC bridge (Dex / Auth0 / Keycloak), scoped API tokens.
4. **Workflow engine**: planner→executor→reviewer orchestration with slice progression.
5. **Task queue**: pluggable backend — `inmem-sqlite` (default) or `temporal` (durable opt-in).
6. **Persistence**: SQLite (libSQL/sqlc) by default, Postgres optional for shared/remote deployments.
7. **Audit log** of every plan, slice, task, model call, skill change, commit, agent-driven action.
8. **LLM-side intelligence**: the orchestrator itself uses a small/cheap LLM to (a) decompose user instructions if planner not yet involved, (b) decide which executor specialization to assign per task.
9. **Capability discovery + operation polling**: `Capabilities()` and `GetOperation(id)` endpoints purpose-built for agent consumers that can't hold streams open.

---

## 2. Folder layout

```
services/orchestrator/
├── .env.example
├── go.mod
├── go.sum
├── cmd/
│   └── orchestrator/
│       └── main.go
├── proto/
│   └── orchestrator.proto
├── internal/
│   ├── config/                # envconfig-loaded, validated
│   ├── grpc/
│   │   ├── server.go
│   │   ├── interceptors/      # auth, tenant resolve, audit, otel, idempotency
│   │   ├── sessions.go
│   │   ├── plans.go
│   │   ├── slices.go
│   │   ├── tasks.go
│   │   ├── skills.go          # proxies to agent-factory
│   │   ├── models.go
│   │   ├── users.go
│   │   ├── operations.go      # GetOperation for poll-style agent clients
│   │   ├── capabilities.go    # introspection for agent clients
│   │   └── audit.go
│   ├── mcp/
│   │   ├── server.go          # MCP server: every gRPC method → MCP tool
│   │   ├── bridge.go          # gRPC ↔ MCP adapter, idempotency, error mapping
│   │   └── descriptions.go    # LLM-readable tool descriptions
│   ├── rest/
│   │   └── gateway.go         # grpc-gateway transpile + OpenAPI 3.1
│   ├── auth/
│   │   ├── jwt.go
│   │   ├── oidc.go
│   │   └── rbac.go            # role matrix
│   ├── tenancy/
│   │   └── org_project.go
│   ├── workflow/
│   │   ├── engine.go          # the brain
│   │   ├── slice.go           # slice progression
│   │   ├── planner_call.go
│   │   ├── executor_dispatch.go
│   │   ├── reviewer_call.go
│   │   └── archon_dag.go      # reusable DAG primitives (nodes, edges, gates)
│   ├── queue/
│   │   ├── queue.go           # interface
│   │   ├── inmem_sqlite.go    # default
│   │   └── temporal.go        # opt-in
│   ├── temporal/
│   │   ├── worker.go
│   │   ├── activities.go      # dynamic-activity dispatcher
│   │   └── workflows.go
│   ├── factory_client/
│   │   └── client.go          # gRPC client to agent-factory
│   ├── store/
│   │   ├── sqlite.go
│   │   ├── migrations/        # sqlc
│   │   └── queries/
│   ├── audit/
│   ├── llm/                   # orchestrator's own LLM client (multi-provider)
│   ├── git/                   # workspace git init / add / commit
│   ├── errors/
│   └── telemetry/
└── test/
    └── e2e/
```

---

## 3. Core domain model (SQLite schema highlights)

```
orgs(id, name, created_at)
users(id, org_id, email, password_hash|null, oidc_sub|null, role, created_at)
api_tokens(id, user_id, hash, scopes, expires_at)
projects(id, org_id, name, settings_json)
workspaces(id, project_id, path, created_at)
sessions(id, workspace_id, user_id, durable bool, status, doc_base_path,
         planner_model, executor_model, reviewer_model, review_threshold,
         created_at)
plans(id, session_id, prompt, status, approved_at, approved_by)
slices(id, plan_id, index, title, scope_summary, status, started_at, ended_at)
tasks(id, slice_id, kind, parallel_group, dependencies_json, status,
      assigned_agent_id, attempt, result_json)
artifacts(id, task_id, path, change_type, size, hash)
reviews(id, slice_id, score, threshold, verdict, notes, user_override_by)
commits(id, slice_id, git_sha, message, files_changed)
agent_skills(agent_kind, skill_id, scope)
audit(id, actor, action, target_type, target_id, payload_json, ts)
```

---

## 4. Workflow engine — the archon-inspired DAG

Borrowed concepts from Archon:

- **Nodes are typed:** `agentic` (LLM call), `deterministic` (git, fs, shell), `interactive` (user gate), `looping` (retry-until).
- **Edges encode dependencies**; the engine resolves topo-order and dispatches parallel branches concurrently.
- **Output → input binding** between nodes (typed contract).

What we add on top:

- **Slices** are top-level node groups; the planner produces them.
- **Each slice = sub-DAG** of `agentic-executor` nodes (parallel/sequential, marked by planner) followed by one `agentic-reviewer` node, followed by one `deterministic-commit` node (gated on score).
- **User-approval gate** between plan generation and execution.
- **Score gate** between review and commit — auto-commit if score ≥ threshold, else interactive gate (`accept-lower` / `re-execute`).

---

## 5. Slice progression state machine

```
DRAFT → AWAITING_APPROVAL → APPROVED →
  for each slice in order:
    SLICE_READY → SLICE_EXECUTING → SLICE_REVIEW →
      score ≥ threshold ? SLICE_COMMIT : SLICE_USER_DECISION
      SLICE_USER_DECISION → ACCEPTED → SLICE_COMMIT
                          → REJECTED → SLICE_EXECUTING (re-plan-of-slice)
    SLICE_COMMIT (git add+commit OR fs-only) → SLICE_DONE
→ COMPLETED
```

Persisted to SQLite every transition; UI subscribes via gRPC stream.

---

## 6. Task queue (dual backend)

**Interface** (Go):

```go
type Queue interface {
  Enqueue(ctx context.Context, task Task) (TaskID, error)
  Dequeue(ctx context.Context, capacity int) (<-chan Task, error)
  Complete(ctx, TaskID, Result) error
  Fail(ctx, TaskID, error, retry RetryPolicy) error
  Status(ctx, TaskID) (TaskStatus, error)
  Subscribe(ctx, sessionID) (<-chan TaskEvent, error)
}
```

**`inmem_sqlite` backend (default, zero-config):**

- In-memory ring + write-ahead to SQLite `task_log` table on every enqueue/complete/fail.
- On orchestrator restart: replay open tasks back into memory.
- Workers are goroutines pulling from a buffered channel; concurrency = sum of agent-factory `Capacity()` reports.
- Per-task timeout from session config.

**`temporal` backend (opt-in):**

- Each session becomes one Temporal workflow.
- Each agent call (planner, executor, reviewer) is a dynamic activity dispatched through a single `ExecuteAgent(ctx, agentKind, payload)` handler.
- Retry policy from session config (initial, backoff coeff, max interval, max attempts).
- Workflow signals carry user interactions (approve plan, accept lower score).

The orchestrator chooses backend based on `session.durable`. Logic above the `Queue` interface is identical.

---

## 7. Multi-tenancy + RBAC + Auth

- **JWT bearer** on every gRPC call (interceptor). HS256 dev secret; RS256 + JWKS for prod.
- **Optional OIDC** via Dex sidecar — `oidc.issuer_url` env enables it; UI redirects through standard OAuth code flow.
- **Roles:**
  - `Owner` — full org control, billing, delete.
  - `Admin` — manage projects/users/skills/models.
  - `Developer` — create sessions in assigned projects.
  - `Viewer` — read-only.
- **Tenant resolution** interceptor: extract `org_id` from JWT, pin all downstream queries.
- **Per-resource ACL** for sessions (owner + invited collaborators).
- **API tokens** (machine-to-machine) for the agent factory and external CI; scopes are project- and role-bounded.

---

## 8. Git integration

- `git init` workspace on first session if `--git` flag (default on).
- After slice commit gate: `git add <artifact paths>` + `git commit -m "<slice title>\n\n<scope summary>"`.
- Commits include a trailer `Pi-Archon-Slice: <slice-id>` for traceability.
- If workspace is NOT a git repo and user disabled git: fall back to filesystem-only mode; record artifacts in `commits` table with no SHA. UI clearly shows "non-git workspace — no commit recorded."

---

## 9. Config (env-only)

`services/orchestrator/.env.example`:

```
ORCH_GRPC_PORT=50051
ORCH_GRPCWEB_PORT=50052
ORCH_MCP_PORT=50053
ORCH_REST_PORT=50054
ORCH_TLS_CERT=
ORCH_TLS_KEY=
ORCH_MTLS_CA=
ORCH_DB_DSN=file:/var/lib/pi-archon/orch.db?_journal_mode=WAL
ORCH_AUTH_MODE=jwt|oidc
ORCH_JWT_SIGNING_KEY=
ORCH_JWT_ISSUER=pi-archon
ORCH_OIDC_ISSUER_URL=
ORCH_OIDC_CLIENT_ID=
ORCH_OIDC_CLIENT_SECRET=
ORCH_FACTORY_TARGETS=localhost:50061   # comma-separated for scale-out
ORCH_TEMPORAL_HOST=
ORCH_TEMPORAL_NAMESPACE=
ORCH_QUEUE_DEFAULT=inmem               # inmem|temporal
ORCH_TASK_DEFAULT_TIMEOUT_SEC=1800
ORCH_LLM_PROVIDER=anthropic            # orchestrator's own thinking LLM
ORCH_LLM_MODEL=claude-haiku-4-5-20251001
ORCH_LLM_API_KEY_REF=env:ANTHROPIC_API_KEY
ORCH_AUDIT_RETENTION_DAYS=365
OTEL_EXPORTER_OTLP_ENDPOINT=
```

---

## 10. Error handling — extra extra attention

| Failure | Strategy |
|---|---|
| Agent-factory unavailable | Circuit-breaker per factory target; fail-over across `ORCH_FACTORY_TARGETS`; surface `UNAVAILABLE` to UI; retry per session policy. |
| Pi process crash mid-stream | Factory restarts, returns `AGENT_RESTARTED`; orchestrator marks task `RETRY`; bounded retries per task policy. |
| LLM provider 429/5xx | Honor `Retry-After`, exponential backoff, fall back to alternate model if `fallback_models` set on session. |
| Sandbox violation | Hard-fail task with `SANDBOX_VIOLATION`; flag in review notes; never silently swallow. |
| User-approved low-score commit | Audit-logged with actor + reason. |
| Temporal worker disconnect (durable mode) | Workflow survives via Temporal; activities resume on reconnect — by design. |
| In-mem queue under restart | Replay from SQLite WAL on boot; open tasks resume; deduplication via `task.attempt`. |
| Concurrent slice edits by two users | Optimistic concurrency token on slices; conflict → 409 with UI merge prompt. |
| Workspace path collision | Reject `StartSession` if workspace already has an active session (override flag for admin). |
| Skill install failure (network) | Atomic — install into staging dir, move on success only; no partial state. |
| Disk full | Pre-flight check before slice execution; abort with actionable error. |
| Clock skew (Temporal / JWT) | Tolerate ±60s; documented. |

All errors carry a stable `code` enum, a `message`, and an optional `correlation_id` so the UI can deep-link to audit log entries. Full catalog in `06-error-catalog.md`.
