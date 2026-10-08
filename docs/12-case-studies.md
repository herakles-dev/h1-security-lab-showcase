# Case Studies

Architecture diagrams describe how the lab is supposed to work. Hunts show how it actually behaves. This chapter walks through four of them: one where the system's own conclusion was wrong, one where it proved a negative, one where it adapted to a technical obstacle, and one where it hit a wall it couldn't get past and said so honestly.

Every target below runs a public HackerOne bug bounty program, and all testing stayed inside the scope that program published. Findings are described by vulnerability class only — no endpoints, parameters, report IDs, or response data appear here.

---

## Case Study 1: The Humbling (WHOOP)

WHOOP makes fitness wearables and runs a public bounty program.

**Setup.** This was the most complete hunt the lab had run at the time: the genuine-device mobile lane, two free test accounts, full reconnaissance, and a coverage ledger tracking every vulnerability class against every asset.

**The false all-clear.** The system completed its mechanical sweep and reported zero findings, every gate green, target secure. The human operator didn't accept that at face value and pushed back, asking the system to re-examine the classes it had marked as skipped rather than tested.

**The bug.** One access-control check was missing on a resource where a closely related endpoint correctly enforced it — the kind of inconsistency that only shows up when you actually compare sibling routes rather than trusting that a class was "covered." It had been sitting inside a class the ledger recorded as skipped, with a plausible-sounding written justification and no actual probe behind it.

**Lesson one: gates certify accounting, not attack.** A gate can confirm that a step was logged. It cannot confirm the step was done honestly. All-green does not mean secure; it means the paperwork was filled in. This became the lab's central doctrine and drove a rewrite of how skips work:

- A "skipped" disposition now requires either a real probe result with evidence, or a specific, limited reason code (unreachable by tooling, out of scope, read-only public). A free-text excuse alone is rejected.
- If more than 60% of a ledger's skips lack that backing, the coverage gate blocks outright — replaying WHOOP's original ledger against this rule, over 85% unbacked, failed immediately.
- Coverage and completeness gates now require every tracked element to carry a real disposition, and a cell that turns out to hide a genuine bug has to be recorded as a finding, cross-checked against the write-up itself.

**Lesson two: the duplicate.** The validated finding was submitted and closed as a duplicate of a report filed into the program's private queue two weeks earlier. No amount of public searching could have surfaced that, because it was never public.

That gap became a new gate: before any report is generated, the system searches public disclosure history and records an explicit verdict — including an honest "acknowledged, can't rule out a private duplicate" outcome when that's the true state, rather than quietly treating a clean public search as proof of originality.

**System changes:** backed-skip enforcement, the unbacked-skip threshold block, the coverage/completeness gates, the prior-art check gate, and the "gates certify accounting, not attack" doctrine now written into the lab's operating rules.

---

## Case Study 2: The Clean Kill (DoorDash)

DoorDash is a food delivery platform with a public bounty program.

**Setup and pivot.** The initial plan targeted cross-role access issues — can a consumer reach merchant or courier systems. Scope review killed that lead outright: every merchant and courier host was explicitly out of scope, leaving only the consumer-facing web property and two consumer apps. The hunt pivoted to consumer-to-consumer attack surface instead: object access, payments, promotions, and account security between ordinary users.

**Working around hidden introspection.** The API's schema introspection was disabled, so there was no direct listing of available operations. The system instead recovered the operation surface by analyzing what the client application itself was built to call, producing a catalog of 193 distinct operations to test.

**Full accountability.** Every one of those operations, plus additional surface found along the way, became a tracked, dispositioned element — 195 in total, each tested with evidence or marked with a specific reason it couldn't be reached. Coverage and completeness gates went green with zero findings.

**The outcome.** No bugs. For a mature, well-hardened target, that's the correct result, and it has real value: the lab can state precisely what was tested and how, rather than gesturing at "we looked around." This was the first fully accounted clean kill in the lab's history. One lead — a possible issue behind a real-money purchase flow — went unresolved because exercising it required actual spend the operator declined to authorize; it's recorded as an open lead, not a pass.

**Reusable technique:** recovering an operation catalog from a client application's own code, rather than from a disabled introspection endpoint, is now a standard recon step for any single-page application that hides its schema.

---

## Case Study 3: The Proof of Concept (Whatnot)

Whatnot is a live-auction social commerce platform with a public bounty program.

**The challenge.** The mobile app pins its TLS certificates, so a standard intercepting proxy can't read its traffic. Getting an authenticated API client working needed a different path to a valid token.

**The adaptation.** The same service exposes a web login that doesn't go through the pinned mobile client at all. Using a real browser on a genuine test device — not an emulator, which this program's anti-fraud checks reject — the system completed that web login end to end, including reading the one-time verification code delivered to the test device, and captured a working session token directly from the browser's own traffic. This was the lab's first live proof that a pinned mobile target could still be reached this way, and it productized into a repeatable capability (`h1-mobile web-capture`) with its own gate confirming the resulting session actually works before anything else proceeds.

**The sweep.** With valid tokens for two separate accounts, the system ran object-level authorization testing against the GraphQL API. Every object lookup stayed scoped to its own caller, identifiers were non-guessable, and introspection was disabled — nothing crossed the account boundary.

**The outcome.** All secure. One lead remained open: a possible timing issue in the live-auction purchase flow that would require real money and live participation in an auction to exercise, which the lab left undemonstrated rather than spend into.

**System changes:** the capture technique became a reusable tool with a dedicated readiness gate, and a browser-routed request mode was added for other testing commands so they can operate against targets that block scripted HTTP clients outright.

---

## Case Study 4: The Honest Block (Ripio)

Ripio is a cryptocurrency exchange with a public bounty program.

**The first wall.** Registration of a second test account repeatedly failed on the target's own email-verification step — multiple attempts, multiple resends, same failure every time. The cause sat entirely on the target's side. That mattered because two-account comparative testing is the lab's strongest technique for finding cross-user access issues, and with only one working account it simply couldn't run.

**The second wall.** The features that move real money required full identity verification with government-issued ID. The operator declined to supply that documentation, so those features stayed untested.

**Honest documentation.** Both obstacles were recorded as blocked, not skipped and not tested. The distinction matters: a skip claims "this didn't apply here," while a block states plainly "this could not be reached, and here is why." Everything else reachable pre-verification — account-security flows, redirect handling, cross-origin behavior, injection points, and the identity-provider configuration — was tested and closed out secure. Six of eight gates passed; the two gates that depend on having an actual finding to validate stayed open, because there was nothing to validate.

**The lesson.** External blockers are a legitimate hunt outcome, not a failure to paper over. The system's obligation is to record them accurately rather than quietly mark them tested — exactly the discipline that WHOOP's original all-clear had failed to apply.

---

## What the Four Hunts Share

| Hunt | Result | What it taught the system |
|------|--------|---------------------------|
| WHOOP | Finding, later ruled a duplicate | Green gates mean process followed, not target secure; private prior art is a real blind spot |
| DoorDash | Zero findings, fully accounted | A clean result can be proven with evidence, not just asserted |
| Whatnot | Zero findings, new capability | A blocked path (pinning) can be routed around through a legitimate alternate flow |
| Ripio | Zero findings, two honest blockers | "Blocked" and "skipped" have to stay distinct states, or the ledger lies |

The common thread is that the lab's biggest improvements never came from a successful sweep. They came from a human questioning a clean result, a target that held up under full scrutiny, an obstacle that forced a new technique, and a wall the system was willing to describe instead of quietly stepping around. Each of those became a permanent change to how the system works, not just a note about one hunt.
