# 06 — Error Catalog

Every error returned by any service follows the same envelope:

```
{
  code: <STABLE_ENUM>,
  message: <human-readable, actionable>,
  doc_url: <link to this catalog anchor>,
  correlation_id: <ULID for log/audit lookup>,
  retry_after: <seconds, optional>,
  details: { ... } // typed per code
}
```

**Rules:**
- Codes are stable across versions. Renames are breaking.
- Messages are written for the **recipient** (an agent or a human), not the developer who wrote the error site.
- Every error is also written to the `audit` table with the same correlation ID.
- gRPC, MCP, and REST share the catalog — the transport just carries it differently.

---

## Agent Factory (Project 1)

| Code | gRPC mapping | Meaning | Recovery |
|---|---|---|---|
| `POOL_EXHAUSTED` | `RESOURCE_EXHAUSTED` | `PI_FACTORY_POOL_MAX` reached, no idle slots. | Caller waits or back-pressures; factory exposes queue depth via `Capabilities()`. |
| `PI_PROCESS_FAILED_START` | `INTERNAL` | Pi binary returned non-zero on launch. | Check `PI_BIN`, factory logs. Crash backoff applies. |
| `PI_PROCESS_CRASHED` | `UNAVAILABLE` | Pi exited mid-stream after start. | Factory auto-restarts up to 5×; orchestrator retries task with `attempt += 1`. |
| `PI_RPC_TIMEOUT` | `DEADLINE_EXCEEDED` | A pi RPC call exceeded its env-configured timeout. | Caller chooses to abort or retry. |
| `PI_RPC_MALFORMED` | `INTERNAL` | Pi emitted an unparseable JSON frame. | Logged + skipped; if pattern persists, session aborted. |
| `WORKSPACE_NOT_MOUNTED` | `FAILED_PRECONDITION` | The provided workspace path is not reachable or not writable from the factory host. | Operator fixes the mount; factory will not start the agent until the path is healthy. |
| `WORKSPACE_OUTSIDE_ALLOWED_ROOTS` | `INVALID_ARGUMENT` | Workspace path is not inside `PI_FACTORY_WORKSPACE_ROOTS`. | Caller picks a valid path. |
| `SANDBOX_VIOLATION` | `FAILED_PRECONDITION` | The agent attempted a write/bash outside its workspace. | Task hard-fails. Audit captures the attempted path. Review score = 0 for the affected task. |
| `SKILL_SOURCE_INVALID` | `INVALID_ARGUMENT` | Unrecognized skill source (must be GITHUB/LOCAL/URL/NPM). | Fix the source. |
| `SKILL_MANIFEST_INVALID` | `INVALID_ARGUMENT` | `SKILL.md` frontmatter failed validation (name regex, description length, allowed-tools shape). | Fix the skill manifest. |
| `SKILL_FETCH_FAILED` | `UNAVAILABLE` | Network failure pulling a GitHub/URL/npm skill. | Auto-retry with backoff; final failure rolls back staging dir atomically. |
| `MODEL_PROVIDER_UNAVAILABLE` | `UNAVAILABLE` | LLM provider returned 5xx or DNS failure. | Honor `Retry-After`; fall back to session's `fallback_models` if set. |
| `MODEL_PROVIDER_RATE_LIMIT` | `RESOURCE_EXHAUSTED` | LLM provider returned 429. | Use returned `retry_after`; back off. |
| `MODEL_AUTH_INVALID` | `UNAUTHENTICATED` | API key rejected by provider. | Operator updates the secret reference; never returned to a non-admin caller. |
| `SECRET_RESOLVE_FAILED` | `INTERNAL` | Configured secret backend (env/keytar/vault) could not produce the value. | Operator fixes the secret config. |
| `AUTH_TOKEN_INVALID` | `UNAUTHENTICATED` | JWT or API token failed verification. | Caller refreshes credentials. |
| `AUTH_TOKEN_EXPIRED` | `UNAUTHENTICATED` | Token expired. | Caller refreshes. |
| `AUTH_TOKEN_SCOPE_INSUFFICIENT` | `PERMISSION_DENIED` | Token does not include the required scope. | Operator grants the scope or caller uses a different token. |

---

## Orchestrator (Project 2)

| Code | gRPC mapping | Meaning | Recovery |
|---|---|---|---|
| `SESSION_NOT_FOUND` | `NOT_FOUND` | `session_id` does not exist or belongs to another tenant. | Caller verifies the ID. |
| `WORKSPACE_IN_USE` | `FAILED_PRECONDITION` | Another active session has the workspace mounted. | Wait for the other session to close, or `--force` (admin only). |
| `PLAN_NOT_APPROVED` | `FAILED_PRECONDITION` | Caller tried to start execution before the plan was approved. | Approve the plan first. |
| `PLAN_LOCKED_CONCURRENT_EDIT` | `ABORTED` | Optimistic concurrency token mismatch — another user edited the plan. | UI prompts the user with a merge view. |
| `SLICE_OUT_OF_ORDER` | `FAILED_PRECONDITION` | Caller tried to skip ahead. | Run slices in order. |
| `SLICE_USER_DECISION_REQUIRED` | `FAILED_PRECONDITION` | Reviewer score below threshold; waiting on user. | UI/agent calls `DecideSlice(accept|reject)`. |
| `REVIEW_THRESHOLD_INVALID` | `INVALID_ARGUMENT` | Threshold outside `[0,10]`. | Pick a valid value. |
| `TASK_TIMEOUT` | `DEADLINE_EXCEEDED` | A task exceeded `ORCH_TASK_DEFAULT_TIMEOUT_SEC` or session override. | Bounded retry per session policy; otherwise marks slice failed. |
| `TASK_RETRIES_EXHAUSTED` | `FAILED_PRECONDITION` | Task hit max attempts. | Surfaces to user with full attempt log. |
| `FACTORY_UNAVAILABLE` | `UNAVAILABLE` | All targets in `ORCH_FACTORY_TARGETS` are down. | Circuit-breaker holds new requests until health returns; existing tasks fail per policy. |
| `FACTORY_PROTOCOL_MISMATCH` | `FAILED_PRECONDITION` | Factory's reported protocol version is outside the orchestrator's supported range. | Operator aligns versions. |
| `QUEUE_BACKEND_UNHEALTHY` | `UNAVAILABLE` | SQLite WAL write failed, or Temporal client cannot reach server. | Operator inspects; in-mem backend retries once before failing. |
| `OPERATION_NOT_FOUND` | `NOT_FOUND` | `GetOperation(id)` for a correlation ID that doesn't exist or has been GC'd. | Operations live for `ORCH_AUDIT_RETENTION_DAYS`; check ID. |
| `GIT_NOT_A_REPO` | `FAILED_PRECONDITION` | Commit-gate enabled but workspace has no `.git`. | Either enable git-init at session start or accept filesystem-only mode. |
| `GIT_COMMIT_FAILED` | `INTERNAL` | `git add` or `git commit` exited non-zero. | Surfaces stderr to user; common causes: no changes, hook failure. |
| `DISK_FULL_PREFLIGHT` | `FAILED_PRECONDITION` | Pre-execution disk-space check below threshold. | Free space and retry. |
| `IDEMPOTENCY_KEY_REPLAY` | `OK` (returns cached response) | Same mutating call seen twice with the same key. | Working as intended — caller's retry is safe. |
| `IDEMPOTENCY_KEY_CONFLICT` | `ABORTED` | Same key, different payload. | Use a fresh key for a different operation. |
| `AUTH_REQUIRED` | `UNAUTHENTICATED` | No bearer token presented. | Sign in. |
| `AUTH_INVALID` | `UNAUTHENTICATED` | Token failed signature or issuer check. | Sign in again. |
| `AUTH_FORBIDDEN_ROLE` | `PERMISSION_DENIED` | Caller's role is insufficient (e.g., Viewer trying to mutate). | Request elevation. |
| `AUTH_FORBIDDEN_RESOURCE` | `PERMISSION_DENIED` | Caller's role is sufficient but ACL does not include this resource. | Owner adds caller to the session/project. |
| `TENANCY_MISMATCH` | `PERMISSION_DENIED` | Resource belongs to a different org than the caller's token. | Use the right token. |
| `OIDC_HANDSHAKE_FAILED` | `UNAUTHENTICATED` | OIDC code exchange failed (bad state, expired code, issuer down). | Restart sign-in. |
| `RATE_LIMITED` | `RESOURCE_EXHAUSTED` | Tenant-level rate limit hit. | Respect `retry_after`. |

---

## Cross-cutting

| Code | gRPC mapping | Meaning | Recovery |
|---|---|---|---|
| `TLS_HANDSHAKE_FAILED` | `UNAVAILABLE` | Client cert untrusted, hostname mismatch, or expired. | Operator fixes certs. UI shows "trust this server?" prompt. |
| `CLOCK_SKEW` | `INVALID_ARGUMENT` | Token or workflow timestamp outside ±60s tolerance. | Caller syncs clock. |
| `INTERNAL_PANIC` | `INTERNAL` | Recovered panic in handler. | Correlation ID points to full stack in logs. Surfaces only as "Something went wrong on our end" to the user; agent clients get the code. |

---

## Versioning

This catalog is part of the public API contract. Adding codes is non-breaking. Removing or renaming codes requires a major version bump on the affected service.
