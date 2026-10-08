# Chapter 2: System Architecture

The H1 Security Lab is not one program. It is nine layers, each answering one question, stacked so the answer to one layer becomes the input to the next. No layer calls out to a remote microservice and no layer hides behind an opaque "AI does something here" box — every handoff between layers is a file on disk: a JSON blob, a markdown spec, a scope file. That's a deliberate choice. If Claude Code (or a human reviewing the session afterward) can't `cat` the handoff, it isn't trusted as ground truth.

The diagram at [`../diagrams/architecture-overview.mmd`](../diagrams/architecture-overview.mmd) lays all nine out top to bottom, with one feedback edge running back up from the bottom to the middle. This chapter walks each layer in that order, then explains the edges.

## Layer 1: Intel — "What should we hunt?"

Before any tool touches a target, the system has to pick one. With hundreds of live HackerOne programs at any given time, the naive move is to hunt the biggest name — but prestige and payout don't correlate with *findable* bugs, and a 10-year-old program with a seven-figure bounty history has usually already been picked clean by thousands of other researchers.

`h1-rank` computes a Hunt Success Score (HSS) per program instead: an expected-value estimate weighted toward *this operator's* actual capabilities — genuine-device mobile testing, two-account differential IDOR, web-capture against pinned apps, GraphQL/Cognito fluency — with a bonus for freshness (newly-launched programs have less accumulated researcher attention) and a discount for competition, scaled by how much of that competition the operator's specific toolset can bypass that a generic scanner can't. A program ranking highly on HSS but low on confidence gets flagged "verify first," not auto-selected — the score is a prioritization signal, not a submission guarantee.

Once a target is chosen, `h1-api` becomes the source of truth for *what's actually in scope*. It queries HackerOne's own hacker-facing API directly rather than scraping the public program page, which means scope, eligibility, and max-severity-per-asset come back authoritative and current rather than inferred from HTML that may be stale or ambiguous. That data lands in `intel/<program>/` as `scope.json` (the machine-readable scope boundary every other layer checks against) and `complete_intelligence.json` (the fuller program profile — asset list, policy text, weakness taxonomy). This layer's output is the first gate every later layer has to respect: nothing downstream is allowed to test an asset that isn't in this file.

## Layer 2: Session Lifecycle — "Where does hunt state live?"

Every hunt gets its own self-contained directory: `sessions/<program>/`. Scaffolding it is a single command (`/h1 hunt <program>`) that creates a consistent skeleton — `spec.md` (what this hunt is testing and why), `gates.md` (the live status of all nine enforcement gates for this run), a `state/` directory (machine-readable JSON — gate status, coverage ledger, resumability snapshot), and `artifacts/` (captures, screenshots, generated reports).

This isolation matters for two reasons. First, a hunt can be paused and resumed days later without losing context — `state/RESUME.md` is regenerated on demand and gives a cold-start summary of exactly where the hunt left off. Second, it means a hunt against Program A can never accidentally bleed into Program B's scope: every tool invocation that touches a session directory re-reads that session's own `scope.json` before firing a single request. The session boundary is also the scope boundary.

## Layer 3: CLI Toolkit — "How do we interact with targets?"

This is the layer that does the actual talking to targets, and it's deliberately boring in design even though the testing logic underneath is not: 26 custom Python tools, each named `h1-<something>` (`h1-surface`, `h1-graphql`, `h1-jwt`, `h1-mobile`, and so on), totaling roughly 24,000 lines. Every one of them takes structured input and emits structured JSON on stdout — no tool prints a prose summary that an AI then has to re-parse into meaning. That uniformity is what makes the tools *composable*: the endpoint inventory `h1-surface map` produces becomes the input to `h1-surface sweep`; the token `h1-mobile web-capture` extracts seeds `h1-auth`, which every later authenticated tool call reads from.

Each tool also independently re-validates scope before it fires. That's intentional redundancy, not an efficiency gap — the session-lifecycle layer checks scope once at setup, but a tool invoked directly, days into a hunt, with a hand-typed target argument, checks again at the point of the actual request. Two independent checkpoints catch two different classes of mistake.

## Layer 4: Agent Formation — "Who does the work?"

For small or focused hunts, the orchestrator — Claude Code itself — walks the phases directly: recon, then test, then validate, then report, one tool call at a time. For larger hunts with wide attack surfaces, the system instead resolves a *formation*: 13 specialist agents organized into four sequential waves matching that same recon → test → validate → report arc.

It's worth being precise about what these agents actually are, because "13 agents" sounds like a fleet of microservices and it isn't one. Each agent is a Claude Code session — a model invocation with a specialized system prompt (`h1-recon-agent`, `h1-auth-agent`, `h1-mobile-agent`, and so on, defined as markdown files under `.claude/agents/`) spawned by Claude Code's own Agent tool. There's no HTTP boundary, no separate deployment, no agent-to-agent network protocol. Coordination happens the same way every other layer coordinates: through files. An agent reads a session's state, does its narrow job, writes its findings back into that state, and the orchestrator reads the result when the agent reports back. The "formation" is a convention for which agents run in which wave and what each one is scoped to touch — not infrastructure.

## Layer 5: Gate System — "When can we proceed?"

Nine sequential gates run across a hunt's lifecycle, from `compliance` (have the program's own rules actually been read before any request fires) through `hacktivity_check` (has a public prior-art search been run before a finding is reported). Each gate has a machine-checkable pass condition recorded in `gates.md` and `state/` — not a checklist item a hunt can silently skip past, but a hard block: later commands refuse to run, or refuse to advance the phase, until the gate's condition is actually satisfied and recorded.

This is the layer that turns "prove it or kill it" from a motto into an enforced constraint. A gate can be explicitly waived with a logged reason (`h1-scope gate waive <gate> --reason ...`) when a condition genuinely can't be met — but a waive is itself a recorded, auditable decision, not a silent bypass. The difference between a gated hunt and an honor-system hunt is exactly the difference between a CI pipeline that blocks a merge and a style guide that asks nicely.

## Layer 6: Coverage Ledger — "Did we actually test it?"

A gate tells you a *process* step happened. The coverage ledger tells you whether the *actual attack surface* got touched. Every endpoint, route, and parameter mined out of recon gets registered as an individually-dispositioned element — `tested`, `finding`, `skipped`, `blocked`, or `n/a` — and a `skipped` disposition isn't free: it requires either a live probe result and evidence reference, or a typed reason (`unreachable-by-tooling`, `out-of-scope`, `read-only-public`). A bare "didn't get to it" is rejected outright, and a session where skips without backing evidence cross a threshold fails its own completeness gate.

This layer exists because of a real failure mode the system hit early: a hunt can clear every gate, run every tool, and still miss the one endpoint where the actual bug was sitting, simply because nobody accounted for it. The ledger is the honest answer to "are we actually done," independent of how much activity happened.

## Layer 7: Feedback Loop — "How does the system improve?"

Every layer above this one is itself the product of the system testing *itself* against real hunts and failing in informative ways. When a hunt surfaces a bug in the tooling, a missing CLI flag, or a gap in a decision rule, that gets written up as a dated orchestrator-note. Periodically, those notes get reviewed together — not one at a time — so a reviewer can spot the pattern across several notes rather than firefighting each one in isolation. That review becomes an improvement spec, a spec gets built by specialist agents, tests get added to lock the fix in, and the result merges back to the main branch.

The numbers are the proof this loop actually turns: dozens of orchestrator-notes, several full review cycles, nearly thirty improvement specs, and a test suite that grew from roughly 40 tests to over 860 across that history — not written speculatively up front, but accreted one real failure at a time.

## Layer 8: Open-Source Arsenal — "What tools do the actual probing?"

Underneath the custom `h1-*` wrappers sits a conventional security-tooling stack: around 37 established open-source tools (`subfinder`, `nuclei`, `ffuf`, `frida`, `mitmproxy`, `jsluice`, and others), backed by roughly 9,660 Nuclei templates and about 6,000 SecLists wordlists. None of this is reinvented — the `h1-*` layer's job is narrowly to make these tools' output *agent-consumable*: normalizing inconsistent output formats into the same uniform JSON contract every other `h1-*` tool honors, and wiring scope-filtering in front of tools that otherwise have no concept of a HackerOne program boundary.

## Layer 9: Mobile Integration — "How do we test mobile?"

Mobile apps need a different approach than a browser-driven web target, because modern apps increasingly detect emulators and refuse to run, or pin their TLS certificates so a standard proxy-in-the-middle capture never sees plaintext traffic. The mobile lane runs against a genuine physical device — a Pixel 6a running GrapheneOS, reachable over wireless ADB — specifically so Play Integrity and similar attestation checks see a real, unrooted phone.

Captured apps get classified along two axes: integrity (does the app check device attestation at all) and pinning (does it pin certificates against a standard MITM proxy). That classification routes each app down one of three capture paths — a rooted-emulator MITM lane for apps with no attestation, a direct Frida/mitmproxy capture on the genuine device for attested-but-unpinned apps, or a Chrome-over-CDP web-login capture for apps that are both attested and pinned, where the token is read directly off a real authenticated web session rather than intercepted off the wire at all.

## How the Layers Connect

Read the diagram's arrows literally: Intel feeds Session Lifecycle, which scaffolds the directory the CLI Toolkit operates inside, which Agent Formation drives at scale, gated at every phase transition by the Gate System, with every tested surface landing in the Coverage Ledger. The Feedback Loop is the one layer that doesn't just hand off forward — its dashed arrow loops back up into the CLI Toolkit layer, because that's literally where fixes land: a new flag, a new subcommand, a bug fix in an existing tool. The Open-Source Arsenal and Mobile Integration layers both sit underneath the Toolkit layer as capability providers, and Mobile Integration's output — a captured token, a mined route inventory — flows forward into the Coverage Ledger exactly like any other tested surface.

No layer is magic, and no layer is optional. A hunt that skips the Gate layer isn't a faster hunt — in this system, it's just an unrun one, because nothing past that gate will execute. That's the whole point of building the enforcement into the tools instead of into a document someone has to remember to follow.
