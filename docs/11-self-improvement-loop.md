# The Self-Improvement Loop

Most security tooling is static: someone writes a scanner, ships it, and the scanner tests the
same things the same way until a human decides to rewrite it. This system does something
different. Every hunt leaves behind a trail of friction — a tool that choked on an edge case, a
gate that passed when it shouldn't have, a class of bug the methodology didn't know to look for —
and that trail gets mechanically converted into code changes and tests before the next hunt
starts. The system that hunts today is not the system that hunted six months ago, and the
difference isn't a redesign. It's the accumulated residue of every prior hunt, absorbed one
dated note at a time.

## The Cycle

**Hunt.** A real session against a real HackerOne program's published scope produces more than
findings (or a clean "no findings" verdict). It produces friction: a CLI flag that behaved
unexpectedly, a scope rule that got silently dropped, a gate that felt too easy to satisfy. That
friction gets captured immediately, while it's fresh, as a dated `orchestrator-notes/` entry —
one file per observation, named for the hunt and the problem (`2026-08-10-ripio-mobile-ui-tap-
coordinate-scaling-mismatch.md`, `2026-08-01-system-no-adversarial-creativity-phase.md`).

**System Review.** Periodically — four full cycles have run so far — every open note gets read in
one sweep, patterns get pulled out across hunts, and fixes get prioritized by severity and
frequency rather than by whichever bug is loudest that day.

**Improvement Spec.** The review's output isn't a to-do list, it's a structured specification:
what's changing, why, and how to verify the change actually closed the gap. Twenty-nine of these
have been authored, each one traceable back to the notes that motivated it.

**Agent Build.** Specialist agents implement the spec, write tests against the verification
criteria the spec defined, and submit the result for review — not a human skimming a diff, but an
adversarial pass checking the build against the spec's own acceptance bar.

**Merge.** Changes land, the full test suite runs, and the next hunt inherits the fix. The loop
closes and starts again on the next target.

## Concrete Examples

Four changes trace this arc end to end, from a single observed failure to a permanent rule
encoded in code and gates:

- **A CORS rejection taught an evidence standard.** An early CORS finding against Netflix's
  public bug bounty program — public knowledge, Netflix runs one — came back rejected as Not
  Applicable: permissive CORS alone, without proof of sensitive data exposure, cookie-based auth,
  and a working exfiltration proof-of-concept, isn't an impact claim. That post-mortem became a
  standing rule: every CORS finding now needs all three proofs before it's written up at all.

- **A severity miscall produced an evidence gate.** A finding reported at a higher severity than
  the evidence actually supported led to the session template's eight-point evidence standard and
  a dedicated exploitation-proof gate that now blocks any report lacking a working reproduction.

- **A "0-finding, all-gates-green" session that wasn't actually secure.** A hunt closed out
  clean — every gate passed — and still missed a real bug sitting inside a class the ledger had
  marked "skipped" rather than "tested." The lesson got encoded, not just remembered: coverage
  gates now require element-level accountability, and a skip has to carry evidence or a typed
  reason, or the gate fails it outright.

- **A validated finding closed as a duplicate of a private report.** A submission that passed
  every gate still lost to prior art the system had no way to see — a private submission on the
  same program. That gap produced a ninth gate, a mandatory public hacktivity check before any
  report goes out, plus documentation that's honest about what the check can't catch: private
  queues stay invisible until disclosure.

## The Numbers

- **52** orchestrator-notes filed across every hunt run so far
- **4** full system-review cycles completed
- **29** improvement specs authored from those reviews
- Test suite grown from roughly **40** tests at the project's start to **861**
- Each review cycle has added somewhere between **40 and 130** new tests

## Why It Works

The loop works because it's mechanical, not aspirational. Notes get filed during the hunt, while
the friction is still fresh enough to describe precisely — not reconstructed from memory weeks
later. Reviews are periodic and exhaustive, not triggered only when something breaks loudly
enough to notice. Specs carry their own verification criteria, so "fixed" has a concrete
definition instead of a feeling. And every fix ships with tests, which means the lesson is now
enforced by the test suite, not stored in someone's recollection of what went wrong last time.

## The Meta-Point

This is the part of the system that matters most. Any static methodology degrades as targets
evolve — new frameworks, new auth providers, new ways of hiding the same old bug class. A fixed
playbook from a year ago is worse today than it was when it was written, simply because it never
learned anything in between. This system doesn't have that problem, not because someone sat down
and redesigned it, but because twenty-some hunts' worth of lessons have been mechanically
absorbed into its code and its tests, one dated note at a time. Gut instinct with a bus factor of one is not a methodology — it's a countdown.
