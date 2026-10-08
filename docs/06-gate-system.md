# 6. The 9-Gate System

## Why gates exist

The first version of this process was a checklist: read the program rules, map the surface, get an authenticated session, test, validate, report. Checklists have an obvious failure mode. Under time pressure, a step gets skipped, and whoever skipped it has a ready-made justification for why that particular step didn't matter this time. That's true whether the one doing the skipping is a person or an agent — a plausible-sounding rationalization is cheap to generate and easy to believe in the moment.

A checklist item is self-graded. It's satisfied the moment someone decides it's satisfied. A gate is not. A gate is a predicate over the actual state of the session — a file that has to exist, a field that has to contain a specific value, a count that has to clear a threshold — checked by code that doesn't care how convincing the reasoning for skipping it was. The gate doesn't ask "did you mean to check this?" It asks "does the evidence this specific check requires actually exist on disk?" If not, the command that would advance the hunt exits with a non-zero status and a JSON error describing exactly what's missing.

This is the difference between a policy and an enforcement mechanism. The program rules say testing shouldn't start before the scope is understood; the `compliance` gate makes starting testing before that point a command that fails. Every later gate in the system exists because an earlier hunt found a specific way a checklist-style "step done" claim turned out to be false, and the fix was never "remember to be more careful" — it was "write a check that would have caught it."

## The nine gates, in sequence

Each gate's `blocks` field names the phases or gates that cannot proceed until it passes — so the table below isn't just nine independent boxes, it's a dependency graph the tooling walks in order.

| # | Gate | What it checks | What it blocks |
|---|------|-----------------|-----------------|
| 1 | `compliance` | The program's own rules have actually been ingested — not just a "scope" listing, but the policy page carrying the real narrative — and a session has been scaffolded against it | Recon, all testing |
| 2 | `recon` | The attack surface has been mapped: tech fingerprinted, APIs discovered, and if the program has in-scope mobile apps, every declared package has a pull artifact (or an explicit "no app found" record) | Authenticated session work, all testing |
| 3 | `auth_session` | An authenticated session actually exists — either a verified captured token or a bearer token stored through the session's auth store that is structurally plausible as a real token | All testing |
| 4 | `validation` | A finding has been reproduced, scored against a standard severity calculator, and the escalation/devil's-advocate methodology steps were actually run and logged | Exploitation proof |
| 5 | `exploitation_proof` | The finding demonstrates that an attacker gained something concrete, and a written devil's-advocate artifact — the skeptical dismissal, answered — exists for it | Coverage review |
| 6 | `coverage` | Every high-priority element in the attack-surface inventory has a disposition: tested, or skipped with a real reason, not silently absent | Completeness check |
| 7 | `completeness` | Every enumerated element in the full inventory has been dispositioned, and a written "what didn't we try" pass has been recorded | Creative pass |
| 8 | `creative_pass` | At least three novel, stack-specific attack vectors beyond the fixed, mechanical sweep have been attempted, each with a recorded probe result | Reporting |
| 9 | `hacktivity_check` | A public prior-art search has been run and a verdict recorded, reducing (not eliminating) duplicate-submission risk | Report submission |

A hunt can't jump ahead. Testing a target before `compliance` passes isn't a matter of discipline — the session tooling checks the gate and refuses. Writing up a finding before `coverage`, `completeness`, and `creative_pass` all clear doesn't produce a report; it produces an error naming which gate is still open.

## How the count grew

The system didn't start at nine. It grew by exactly one gate each time a hunt exposed a specific gap a smaller gate set didn't catch:

- **Coverage and completeness** came from a hunt where every existing gate was green and the session was confidently logged as "zero findings, thoroughly secure" — and a real bug was sitting the entire time inside a category that had been marked `skipped` with a plausible-sounding reason nobody had actually probed. The fix wasn't "look harder." It was a gate that makes an unbacked skip impossible to pass through silently: a skip now has to carry either a live probe result and evidence reference, or a specific named reason from a fixed set (unreachable by available tooling, genuinely out of scope, read-only public data). A bare "didn't seem worth it" is rejected outright, and a ledger where too high a fraction of skips lack real backing fails the gate on that basis alone.
- **The creative pass** came from recognizing that a fixed, mechanical sweep — run the same category checklist against every endpoint — has a ceiling. It will reliably find the bugs that fit categories someone already thought to write a check for. It will never find the bug that's shaped like something nobody anticipated. So this gate requires actually attempting a minimum number of vectors that go beyond the standard sweep, each with a real attempted-and-recorded outcome, not just a described idea that was never tried.
- **The prior-art check** came from a submission that was closed as a duplicate of a private report filed weeks earlier — invisible to any check that only looks at public disclosure. The gate that resulted doesn't pretend to solve that; its own description says plainly that it reduces but does not eliminate duplicate risk, because a private, undisclosed report in a program's queue is structurally invisible to any public search. What it does is force the public check that's actually possible to happen every time, with a recorded verdict, instead of being skipped under the obviously tempting excuse that it probably wouldn't have caught the real risk anyway.

Each addition follows the same pattern: a specific, concrete failure happened, and the response was a programmatic check narrow enough to have caught exactly that failure — not a vaguer instruction to be more thorough next time.

## Gates certify accounting, not attack

This is the single sentence worth taking away from the whole system: a gate passing means the *process* was followed — the right artifact exists, the right count was logged, the right verdict was recorded. It does not mean the target is secure, and it does not mean the right bug was found.

The clearest demonstration of this is the exact hunt that produced gates 6 and 7. Every gate that existed at the time was green. The session's own summary called the target "zero-finding, thoroughly secure." And a real, valid finding was sitting the entire time inside a disposition that the gates of that era accepted at face value: "skipped, not worth testing." The gates were satisfied. The target was not secure. The two facts were simultaneously true, and the gap between them is exactly what a checklist mentality misses — a checklist that says "disposition every category" is satisfied by any disposition, including a wrong one.

The fix that followed is structural, not aspirational: a skip is no longer a free sentence. It's a claim that itself requires evidence — a probe that was actually run, or a reason drawn from a fixed, narrow vocabulary that a human reviewing the ledger can audit. The gate still only certifies that the accounting was done honestly. It still can't certify that the right attack was tried. No gate in this system claims otherwise, and that's intentional: treating gate-passing as proof of security would recreate the exact failure the gates exist to prevent, one level up.

## Gate operations

Gates are a small, direct CLI surface, not a hidden state machine:

- **`gate check <name>`** — verify whether a gate is currently passed. Read-only; exits non-zero if not.
- **`gate pass <name>`** — attempt to pass a gate. This is where the evidence checks live: the command inspects the session's actual files and state before recording a pass, and refuses with a specific error naming what's missing if the evidence isn't there. A gate also refuses to pass if a prerequisite gate that blocks it hasn't passed yet.
- **`gate waive <name> --reason "..."`** — mark a gate passed *without* its evidence check, for situations where the honest answer really is "this cannot be satisfied in this hunt" — a paywalled feature on a no-spend engagement, for instance. A waiver always requires a stated reason; there's no silent variant.
- **`gate status`** — show every gate's current state across the session.

The distinction that matters is between `pass` and `waive`. `gate check` treats a waived gate as satisfied — later phases proceed exactly as if the evidence had been earned — but `gate status` renders a waiver distinctly from a real pass, so a waiver is never mistaken for actual proof after the fact. The audit trail survives even when the gate itself has been bypassed: anyone reviewing the session later can see not just that a gate was open, but whether it was earned or explicitly waived, by whom, and why.

That visibility is the point. A system that let gates be silently satisfied would be exactly as trustworthy as the checklist it replaced.
