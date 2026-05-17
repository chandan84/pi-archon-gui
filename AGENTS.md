# AGENTS.md — Testing Charter

This file applies to all contributors (human and AI) working in this repo. It is intentionally short and opinionated. Follow it.

## The rule

**Manual end-to-end user testing is the source of truth.** Every milestone is gated on a real human running the system end-to-end against real backends and confirming the experience matches the spec in `docs/plan/`.

Automated tests exist to catch regressions between manual passes — not to replace them.

## What we DO write

- **Real E2E tests.** Playwright drives the Tauri UI against a real orchestrator, a real agent factory, a real pi binary, real SQLite, real LLM providers (or a record/replay layer at the HTTP boundary — never an in-code mock). Run nightly in CI.
- **Real integration tests.** The Go orchestrator's tests boot a real factory subprocess and a real SQLite file. The TS factory's tests spawn a real pi binary. No "fake factory," no "fake pi."
- **Proto contract tests.** `buf breaking` against the previous tag. Schema is a hard contract.
- **A small set of pure-function unit tests** for things that are genuinely pure — parsers, validators, formatters. If the function takes a database, a network client, or an LLM, it is not a unit test target.

## What we DO NOT write — anti-patterns

These are bugs disguised as tests. Reject them in code review.

- ❌ **Unit tests with a mocked LLM.** The LLM is the system under test. Mocking it tests the mock.
- ❌ **Mocked task queue, mocked factory, mocked filesystem, mocked git.** Same reason.
- ❌ **"Fake" services** that pretend to be pi, Temporal, OpenAI, or the orchestrator. Use the real thing or don't test it.
- ❌ **Snapshot tests of UI components.** They lock the markup, not the behavior, and break on every refactor.
- ❌ **Stubbing internal interfaces purely to inflate coverage.** Coverage is not a goal here.
- ❌ **Mocking the code under test.** If you find yourself mocking `OrchestratorService` while testing `OrchestratorService`, stop.
- ❌ **Monkey-patching modules at runtime** to swap dependencies — same category as mocking, plus harder to read.
- ❌ **`expect(true).toBe(true)`-style placeholder tests** waiting for an implementation.
- ❌ **Tests that pass when the network is unreachable but the code obviously can't work** — strong signal the test isn't testing anything.

## How to add coverage when the real path is expensive

If a real path takes minutes (long LLM call, slow Temporal warmup):

1. **Record once, replay many.** Capture real HTTP/gRPC traffic on the wire (VCR-style cassettes against the actual provider). Re-record when the contract changes. Tests still exercise our full code path.
2. **Cheap real fixtures.** Use the cheapest real model (Haiku, Gemini Flash, local Ollama) for tests that just need a real LLM-shaped response.
3. **Skip-by-default + nightly enablement.** Mark expensive tests with a tag; run them in nightly CI, not on every PR.

None of this requires mocking our own interfaces.

## Test naming

Tests describe user-observable behavior, not function names:

- ✅ `slice with score below threshold prompts the user before committing`
- ✅ `agent that writes outside the workspace dir fails with SANDBOX_VIOLATION and audits the attempt`
- ❌ `test_ReviewService_evaluateScore_returnsCorrectBoolean`

## Coverage

Coverage % is not tracked, reported, or used as a gate. A 100%-coverage suite of mocked unit tests is worse than no tests. Tests that exercise real behavior end-to-end are the asset.

## When in doubt

If you're not sure whether a test is worth writing, ask: "If this passes, do I trust the feature works for a real user?" If the answer is no, delete the test. Manual E2E is always available.
