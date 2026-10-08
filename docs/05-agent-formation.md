# Chapter 5: Agent Formation

## How Agents Work

Every "agent" in this system is not a microservice, not an HTTP endpoint, and not a persistent daemon. It's a Claude Code session, spawned on demand, carrying a specialized system prompt.

Concretely: the orchestrator calls an `Agent` tool with a task description and a target agent type — say, `h1-auth-agent`. That spawns a fresh Claude session, loads the system prompt from `.claude/agents/h1-auth-agent.md`, hands it the task, and lets it run with its own full tool access (shell, file read/write, browser automation, the `h1-*` CLI wrappers). The spawned session works independently — it doesn't share live memory with its parent — and when it finishes, its final output is returned to the orchestrator as a result.

That's the entire mechanism. There's no message bus, no shared process pool, no container per agent. An "agent" is a role definition — a markdown file with YAML frontmatter describing its specialty, formation wave, and parallelizability — plus a model instance that briefly inhabits that role. Think of it less like microservices calling each other over a network, and more like a single investigator who can temporarily fork off a specialist clone of themselves, hand that clone a narrow brief, and read the clone's report when it comes back. The clone has no ongoing existence after it reports; the next time that specialty is needed, a brand-new instance is spawned from the same prompt file.

This has a consequence worth stating plainly: the agents are not actually running concurrently in the sense of separate always-on services. "Parallel" here means several of these spawned sessions are in flight at once, each independently calling tools against the same target, with the orchestrator waiting on all of them before moving to the next wave. It's parallelism at the level of reasoning sessions, not infrastructure.

## The Formation

The lab defines 13 specialist agents, each living in its own file under `.claude/agents/h1-*.md`. They're organized into an orchestration tier and four execution waves.

**Orchestration tier (always active):**

- **`h1-orchestrator`** — the hunt lifecycle coordinator. It decides formation (solo vs. parallel), sequences the waves, and aggregates results across the whole engagement.
- **`h1-skill`** — the phase-by-phase runbook executor. Where the orchestrator thinks at the level of "which wave runs next," `h1-skill` walks the methodology's numbered phases (recon → test → validate → report) and keeps the session's gate state honest as it goes.

**Wave 1 — Recon (sequential, builds the map):**

- **`h1-intel-agent`** — OSINT and program analysis: scope research, disclosed-vulnerability history on the target program, target prioritization. This runs first because everything downstream depends on knowing what's actually in scope and what's already been found and fixed.
- **`h1-recon-agent`** — subdomain enumeration, technology fingerprinting, API discovery, attack-surface mapping. Turns the intel agent's program-level picture into a concrete list of hosts, endpoints, and tech stacks to test.

These two run sequentially (or lightly parallel to each other) because Wave 2 can't meaningfully start without a target list and a tech fingerprint.

**Wave 2 — Testing (parallel, the main attack surface):**

This is where the formation actually fans out. Six specialists, each with a narrow, non-overlapping mandate:

- **`h1-hunter-agent`** — business logic and workflow-state abuse, auth flaws, XSS, SSRF. The generalist of the testing wave, covering the vulnerability classes that don't have their own dedicated specialist.
- **`h1-auth-agent`** — explicitly the *exclusive* owner of IDOR/BOLA, broken access control, and privilege escalation (horizontal and vertical). The agent definition states this ownership explicitly, to avoid two agents independently discovering (or disagreeing about) the same access-control bug through different lenses.
- **`h1-api-agent`** — REST API-specific testing: auth bypass, rate limiting, mass assignment, parameter tampering.
- **`h1-graphql-agent`** — conditional, spun up only if recon found a GraphQL endpoint. Covers introspection, query depth, field-level authorization, batched-query abuse, and GraphQL-specific DoS vectors.
- **`h1-race-agent`** — conditional, for TOCTOU bugs, concurrent-request exploitation, and double-spend-style race conditions. Only worth spawning when the target has a workflow with a money- or inventory-sensitive write path.
- **`h1-mobile-agent`** — conditional, engaged only when the program's scope includes mobile assets. Owns the genuine-device lane: installing the app, driving the device, and capturing authenticated traffic for the testing agents above to consume.

Because each of these agents owns a distinct vulnerability class (and `h1-auth-agent`'s mandate is explicitly carved out from the others), they can run concurrently against the same target without duplicating work or racing each other to the same finding.

**Wave 3 — Validation (sequential, the proof gate):**

- **`h1-validator-agent`** — takes every candidate finding Wave 2 produced and tries to kill it. Reproduces the steps, assesses real-world impact, verifies the vulnerability is actually exploitable (not just theoretically present), and prepares the evidence trail. This wave is deliberately sequential and singular — one agent, one pass, no parallel validators — because its entire job is to be the skeptical bottleneck that nothing passes through by accident. A finding that doesn't survive this wave never reaches Wave 4.

**Wave 4 — Reporting (sequential, the final output):**

- **`h1-reporter-agent`** — takes validated survivors and turns them into professional write-ups: impact articulation, evidence formatting, submission-ready structure. By the time a finding reaches this agent, it has already been proven, so this wave is about communication quality, not investigation.

## Artifact Handoffs

The waves aren't just a scheduling convenience — each one produces a concrete artifact that the next wave consumes as input, rather than agents re-deriving context from scratch or talking to each other directly.

Wave 1 produces recon reports and a target/endpoint list. Wave 2's specialists read that list as their test scope and produce a set of finding *candidates* — not findings, candidates, because nothing from Wave 2 is trusted as true until it survives Wave 3. Wave 3 takes each candidate and either kills it (with a documented reason) or promotes it to a validated finding with reproduction evidence attached. Wave 4 takes only the validated survivors and formats them into the final report.

This candidate → validated-finding distinction is load-bearing. The lab's own guiding principle — "prove it or kill it" — is enforced structurally by putting a dedicated, skeptical validation wave between the agents that generate hypotheses and the agent that writes them up. A testing agent being wrong or overeager doesn't propagate into a report; it just produces a candidate that Wave 3 discards.

## Solo vs. Parallel

Not every target gets the full four-wave treatment. A planning step — `h1-scope formation plan` — looks at the target's complexity (how many in-scope assets, how many distinct tech stacks, whether mobile is in scope) and decides between two modes:

- **Solo**: for a simple target — a handful of assets, one tech stack — the orchestrator walks all the phases itself, sequentially, without spawning the Wave 2 specialist roster. There's no benefit to six parallel specialists probing a single small REST API; one careful pass covers the ground.
- **Parallel**: for a complex target — multiple assets, multiple tech stacks, maybe a GraphQL API alongside a REST API alongside a mobile app — Wave 2 fans out into however many of the six testing agents are relevant, running concurrently against their respective slices of the attack surface.

This keeps the formation proportionate to the target instead of always paying the overhead of a 13-agent roster on something that doesn't need it.

## The First Live Run

For a long time, this multi-agent parallelism existed only as a design — the formation logic was built and the agent prompts were written, but no hunt had actually exercised Wave 2 with multiple agents running concurrently against a real target.

That changed on 2026-08-01, during a hunt against an example program. The formation planner selected parallel mode, and five agents ran across two waves, each producing a correctly-schemaed agent report that the orchestrator was able to aggregate without manual reconciliation. It was a small milestone by itself, but a meaningful one: it moved "parallel formation" from a theoretical capability in the agent definitions to a demonstrated behavior with real output artifacts. Every hunt before that date had run solo, with one session walking every phase in sequence — the design for concurrent specialist agents had been sitting unused until a target finally warranted it.
