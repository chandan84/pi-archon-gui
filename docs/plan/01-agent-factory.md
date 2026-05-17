# 01 — Agent Factory (TypeScript)

**Path:** `services/agent-factory/`
**Runtime:** Node.js 22 LTS, TypeScript 5.6, esbuild bundle, optional Bun for dev speed.
**Deployment:** remote network service. Single static container image; horizontally scalable behind a gRPC load balancer.
**API surface:** `@grpc/grpc-js` + `ts-proto` for gRPC; `@modelcontextprotocol/sdk` for the MCP server (mounted on the same process, separate port); `grpc-gateway`-equivalent (`grpc-gateway-ts` or hand-written Fastify shim) for REST.

---

## 1. Responsibilities — single concern: "give me a configured pi agent"

1. Manage a **pool of pi processes** (start/stop/terminate, health-check, autoscale within configured bounds).
2. Expose **gRPC + MCP + REST** for: lifecycle, prompting, skills mgmt, model selection, artifact listing, abort/steer/follow-up.
3. Enforce a **filesystem sandbox**: every file write tool call is intercepted via a pi **extension** that rejects paths outside the assigned workspace dir.
4. **Skill installer**: install skills from GitHub repos, local folders, npm tarballs, or HTTP URLs into per-agent or global skill directories.
5. **Model registry**: maintain `models.json`, surface available models, switch model per session via `set_model`.
6. **Auth/credentials**: load API keys from env, OS keychain (via `keytar`), or HashiCorp Vault when configured — never shipped in code.
7. **Caller authentication**: verify JWT (orchestrator-issued) or scoped API token (external agent) on every gRPC/MCP/REST request.
8. Persist agent-level config in `${PI_FACTORY_HOME}/agents/<agent-id>/`.

Because the workspace dir is on a **shared filesystem** (NFS / object-store gateway / per-deployment volume), the factory enforces that the dir is reachable and writable at `StartAgent` time; "workspace not mounted" is a typed pre-flight error.

---

## 2. Folder layout

```
services/agent-factory/
├── .env.example              # ALL config: ports, paths, secrets refs, pool sizes
├── package.json
├── tsconfig.json
├── proto/
│   └── agent_factory.proto   # gRPC service definitions (shared via buf)
├── src/
│   ├── server.ts             # gRPC server entrypoint
│   ├── config.ts             # zod-validated env → typed config
│   ├── pool/
│   │   ├── PiProcessPool.ts  # spawn/recycle/health
│   │   ├── AgentSlot.ts      # one pi process = one slot
│   │   └── policies.ts       # min/max, idle TTL, restart on crash
│   ├── rpc/
│   │   ├── PiRpcClient.ts    # JSON-RPC over stdin/stdout to pi
│   │   └── methods.ts        # typed wrappers for prompt, set_model, etc.
│   ├── skills/
│   │   ├── SkillInstaller.ts # github | local | tarball | url
│   │   ├── SkillStore.ts     # filesystem layout under ~/.pi/agent/skills
│   │   └── manifest.ts       # validate SKILL.md frontmatter
│   ├── sandbox/
│   │   └── workspace-guard.ts # pi extension; rejects writes outside workspace
│   ├── models/
│   │   ├── ModelRegistry.ts  # load/persist models.json
│   │   └── providers.ts      # supported provider catalog
│   ├── credentials/
│   │   └── secrets.ts        # env / keytar / vault
│   ├── grpc/
│   │   ├── AgentService.ts   # implementations of generated stubs
│   │   ├── SkillService.ts
│   │   └── ModelService.ts
│   ├── mcp/
│   │   ├── server.ts         # MCP server, tools auto-derived from gRPC
│   │   ├── tool-bridge.ts    # gRPC method → MCP tool adapter
│   │   └── descriptions.md   # per-tool descriptions for LLM readers
│   ├── rest/
│   │   └── gateway.ts        # Fastify-based grpc-gateway shim
│   ├── auth/
│   │   ├── jwt.ts            # verify orchestrator-issued JWTs
│   │   └── api-token.ts      # scoped tokens for external agents
│   ├── telemetry/
│   │   └── otel.ts           # OpenTelemetry traces + metrics
│   └── errors/
│       └── types.ts          # typed gRPC error codes
├── extensions/
│   └── workspace-guard/      # pi extension shipped with factory
│       └── index.ts
└── test/
    └── e2e/                  # see AGENTS.md
```

---

## 3. gRPC surface (excerpt — full proto at `proto/agent_factory.proto`)

```proto
service AgentFactory {
  rpc StartAgent(StartAgentRequest) returns (Agent);          // returns agent_id
  rpc StopAgent(AgentId) returns (Empty);
  rpc TerminateAll(Empty) returns (Empty);
  rpc ListAgents(Empty) returns (stream Agent);

  rpc Prompt(PromptRequest) returns (stream PromptEvent);     // streams tokens + tool calls
  rpc Steer(SteerRequest) returns (Empty);
  rpc FollowUp(FollowUpRequest) returns (Empty);
  rpc Abort(AgentId) returns (Empty);

  rpc SetModel(SetModelRequest) returns (Empty);
  rpc ListModels(Empty) returns (ModelList);

  rpc ListArtifacts(AgentId) returns (ArtifactList);
  rpc Health(Empty) returns (HealthReport);
  rpc Capabilities(Empty) returns (CapabilitiesReport);       // for agent introspection
}

service Skills {
  rpc Install(InstallSkillRequest) returns (Skill);   // source: GITHUB|LOCAL|URL|NPM
  rpc Uninstall(SkillId) returns (Empty);
  rpc List(ListSkillsRequest) returns (SkillList);    // scope: GLOBAL|AGENT
  rpc AttachToAgent(AttachSkillRequest) returns (Empty);
}
```

---

## 4. Pool lifecycle & sandboxing details

- **Pool sizing:** `min`, `max`, `idle_ttl_seconds` from env. The orchestrator queries `Capacity()` before priming the queue.
- **Restart policy:** on pi process crash → exponential backoff (1s, 2s, 4s, max 30s, give up after 5 attempts).
- **Sandbox:** the shipped `workspace-guard` pi extension hooks `onToolCall` and rejects `write`/`edit`/`bash` tool calls whose resolved absolute path falls outside `${WORKSPACE_ROOT}`. The factory injects `WORKSPACE_ROOT` per agent at `StartAgent`. This makes pi safe-by-construction for our use case (pi has **no** built-in sandbox).
- **Artifact tracking:** the same extension records every file mutation into `${WORKSPACE_ROOT}/.pi-archon/artifacts/<agent-id>.jsonl`. `ListArtifacts` reads this.

---

## 5. Skills installer details

- **GitHub:** `Install({source: GITHUB, ref: "owner/repo@tag", subpath?})` → shallow-clone into a temp dir → validate `SKILL.md` → move to skill store.
- **Local folder:** symlink (configurable) or copy.
- **URL/tarball:** download + extract + validate.
- **npm package:** `npm pack <name>` then extract.
- **Per-agent vs global:** scope flag chooses `.pi/skills/<agent>/` (agent-local) vs `~/.pi/agent/skills/` (global). The orchestrator can request "attach skill X to planner only" — that maps to per-agent scope.
- **Validation:** name regex, description ≤1024 chars, no path traversal in `scripts/`.

---

## 6. Config (env-only)

`services/agent-factory/.env.example`:

```
PI_FACTORY_GRPC_PORT=50061
PI_FACTORY_MCP_PORT=50062
PI_FACTORY_REST_PORT=50063
PI_FACTORY_TLS_CERT=
PI_FACTORY_TLS_KEY=
PI_FACTORY_MTLS_CA=                    # if mTLS to orchestrator
PI_FACTORY_HOME=/var/lib/pi-archon/factory
PI_FACTORY_WORKSPACE_ROOTS=/mnt/workspaces  # allowed parent dirs for sandbox
PI_FACTORY_POOL_MIN=2
PI_FACTORY_POOL_MAX=16
PI_FACTORY_POOL_IDLE_TTL_SEC=300
PI_FACTORY_TEMP_DIR=/var/lib/pi-archon/factory/tmp
PI_BIN=/usr/local/bin/pi
PI_MODELS_FILE=/var/lib/pi-archon/factory/models.json
SECRETS_BACKEND=env|keytar|vault
VAULT_ADDR=
VAULT_TOKEN=
OTEL_EXPORTER_OTLP_ENDPOINT=
AUTH_ORCHESTRATOR_PUBKEY=/etc/pi-archon/orchestrator.pub  # verify orchestrator JWTs
AUTH_API_TOKEN_HMAC_KEY=               # for external-agent scoped tokens
```

---

## 7. Error handling

- All gRPC methods return typed codes (`INVALID_ARGUMENT`, `RESOURCE_EXHAUSTED` when pool full, `FAILED_PRECONDITION` for sandbox violations, `UNAVAILABLE` for pi crashes mid-stream).
- Pi RPC stream parser tolerates malformed JSON lines (logs + skips, does not crash session).
- Every async path has an upper-bound timeout from env (no infinite waits).

See `06-error-catalog.md` for the full code list and recovery semantics.
