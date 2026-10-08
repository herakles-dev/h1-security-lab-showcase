# Chapter 13: The Test Suite

## Scale

As of this writing, the suite holds **909 tests across 73 files** (72 test
modules plus a shared `conftest.py`) under `tests/`. It started at roughly 40
tests. The growth curve tracks the project's own history: this is a
spec-driven system (see [Chapter 11](11-self-improvement-loop.md)), and every
improvement spec that touched a gate, a CLI tool, or the coverage ledger
shipped its own verification tests alongside the fix. The suite is a byproduct
of the improvement loop, not a separate QA initiative bolted on afterward.

Two directories carry almost all of it: `tests/cli/` (69 files — one per
`h1-*` subcommand or gate behavior) and `tests/integration/` (pipeline and
agent-contract tests). A couple of root-level files (`test_debug_mode.py`,
`test_new_modules.py`) predate that split.

## What's Tested

**Gate enforcement.** Files like `test_h1_scope_compliance_gate.py`,
`test_h1_scope_coverage_gate` (via `test_h1_scope_completeness_gate.py`), and
`test_h1_evidence_proof_gate_s4.py` exist to answer one question: does the
gate actually block when its criteria aren't met? These tests don't just
check the happy path — they reconstruct a specific real failure first, write
a test that reproduces it, then verify the fix. `test_h1_scope_compliance_gate.py`
is a clean example: it documents a live incident where the compliance gate
passed despite an incomplete intel scrape (missing the "policy" tab), then
asserts that `gate pass compliance` now hard-blocks with a
`blocked_by: incomplete_intel_scrape` error until the scrape is complete, and
that `h1-scope lint` retroactively flags any session that slipped through
before the fix existed.

**Scope validation.** Tests exercise whether `h1-*` tools refuse to fire
against a target not present in a session's `scope.json`, and whether the
scope-derived eligibility fields (`eligible_for_submission`, `max_severity`)
propagate correctly from the authoritative H1 API response through to the
tools that consume them.

**Coverage ledger.** `test_h1_scope_coverage.py` and its siblings
(`test_h1_scope_element_gate.py`, `test_h1_scope_set_element.py`,
`test_h1_scope_completeness_gate.py`) verify the accountability mechanics
described in [Chapter 7](07-coverage-ledger.md): that a `skipped` disposition
without a live probe or a typed skip-kind is rejected, that an unbacked-skip
ratio above 60% blocks the `coverage` gate, and that setting an ID format to
`int_sequential` auto-spawns a HIGH-priority cross-user `authz_idor` element.

**Evidence standards.** `test_h1_evidence_proof_gate_s4.py`,
`test_h1_evidence_mobile_gate.py`, and `test_h1_evidence_hacktivity_gate_block.py`
hold the line on the proof bar — a finding without an attested
`exploitation_proof` gate pass (or explicit `--proof-complete`) cannot be
reported, regardless of how the underlying classifier labels the finding
type.

**CLI tool tests.** The bulk of `tests/cli/` is one file per tool
(`test_h1_api.py`, `test_h1_mobile_static.py`, `test_h1_surface.py`,
`test_rank_programs.py`, and so on), checking that each wrapper emits valid
JSON, handles its documented error cases (missing credentials, malformed
upstream response, target unreachable), and fails closed rather than silently
degrading.

**Integration tests.** `tests/integration/test_pipelines.py` and
`test_agents.py` check that tool chains compose — that one tool's JSON output
is shaped the way the next tool in the chain expects to consume it (recon
output into surface mapping, surface output into IDOR testing).

**Formation tests.** `test_formation_plan.py` verifies that
`h1-scope formation plan` resolves the wave structure in
`configs/formations/security-hunt.json` against a session's actual scope —
firing conditional agents (mobile, GraphQL) only when the relevant asset type
is present, and never silently dropping an unconditional tester.

## What's NOT Tested

In the spirit of "prove it or kill it," the honest gaps:

- **No live-target integration tests.** Every test runs against mocked or
  fixture-backed state — synthetic `scope.json` files, stubbed subprocess
  calls, hermetic module loads. Nothing in the suite makes a real request to
  a real HackerOne program.
- **No end-to-end hunt simulation.** There is no test that scaffolds a
  session, runs it through all phases, and asserts a report comes out the
  other end. Each gate and tool is tested in isolation.
- **Formation parallelism is tested structurally, not live.** The formation
  tests check that wave plans and artifact-handoff schemas are well-formed —
  not that agents spawned in parallel actually complete their work and hand
  off correctly under real concurrency.
- **No performance or load testing.** None of the 26 `h1-*` tools are
  benchmarked for latency or resource usage.

## Why Tests Matter Here

In a system that improves itself through a recurring review-to-spec cycle
(Chapter 11), tests are the immune system, not decoration. Every spec that
lands a fix also lands the test that proves the fix holds — and, more
importantly, that it keeps holding the next time something else changes.
Without that, the feedback loop inverts: instead of each cycle compounding on
the last, it would eventually re-break something a prior cycle already
closed. The 29 specs that have shipped against this codebase did not
regress each other's fixes. The 909 tests are the reason that's a
verifiable claim and not an assertion.
