# ADR-0004 — Filesystem sandbox lives in the factory, not in pi

**Status:** Accepted
**Date:** 2026-05-17

## Context

Pi (the Earendil-Works agent runtime) has **no built-in workspace sandbox** — its tools can read and write anywhere the process can. For our product, every agent must be strictly confined to its assigned workspace directory: artifacts inside, no escape outside.

Options:
1. Fork pi to add a sandbox.
2. Run pi inside an OS-level container/jail per agent.
3. Implement the sandbox as a pi **extension** that intercepts tool calls.

## Decision

Ship a pi extension named `workspace-guard`, owned by the agent factory and injected into every pi process at `StartAgent` time. It hooks `onToolCall` and rejects `write` / `edit` / `bash` calls whose resolved absolute path falls outside the agent's `WORKSPACE_ROOT`. The same extension records every accepted file mutation into `${WORKSPACE_ROOT}/.pi-archon/artifacts/<agent-id>.jsonl` for later artifact listing.

The workspace root is enforced as one of `PI_FACTORY_WORKSPACE_ROOTS` — agents cannot point at arbitrary paths via prompt injection.

## Consequences

**Positive**
- Pi stays vendor-clean — easy to upgrade, easy to swap.
- Sandbox logic is centralized and auditable in our codebase.
- Artifact tracking is a free side-effect.
- Symlink-escape attacks are caught at the resolved-path layer.

**Negative**
- A malicious skill could conceivably bypass the extension by spawning a child process that itself writes. Detected because `bash` tool calls are also gated, but determined attackers could chain escapes.
- This is process-level isolation, not kernel-level. Compromised pi → compromised factory user.

**Mitigations**
- v2: run each pi process inside a rootless container or `bwrap` jail. Captured as future work.
- The factory runs as a non-root, non-privileged user; even a sandbox escape stays inside the factory's permissions.
- Strict skill installer validation (no path traversal in `scripts/`, no postinstall hooks).
