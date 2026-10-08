# The Coverage Ledger

## The Problem

A typical penetration test report says "we tested for IDOR" and moves on. It doesn't say which endpoints were hit, which parameters were manipulated, what identifier format they used, or what came back. The claim is unfalsifiable by construction — there's no artifact a skeptical reader could check against. If the tester missed a class of endpoint entirely, nothing in the report would reveal that gap, because the report only describes what *was* found, never the shape of what was attempted.

That asymmetry is dangerous specifically because it's invisible. A report with zero findings can mean "we tested exhaustively and it's clean" or "we tested the three endpoints we had time for" — and from the outside, those two reports look identical. H1 Security Lab's answer is to make "we tested X" a checkable claim instead of a sentence: every enumerated endpoint, route, and parameter becomes a row in a ledger, and that ledger has to reconcile with both the gate system and the findings directory before a report can go out.

## Element-Granular Tracking

The ledger lives at `state/coverage.json` inside each hunt session, built and maintained through `h1-scope coverage`. The unit of accounting isn't "the API" or "the auth flow" — it's the individual element: one endpoint, one route, one GraphQL operation, one parameter. Each element carries:

- **Asset** — which host or service it belongs to (`api.example.com`, not just "the backend")
- **Class** — which vulnerability type it's being tested against (`idor`, `xss`, `cors`, `ssrf`, `mass_assignment`, and the rest of the taxonomy)
- **Status** — `tested`, `finding`, `skipped`, `blocked`, or `n/a`
- **Tool** — which `h1-*` tool produced the disposition
- **Evidence** — what was actually observed (a status code, a response body reference, a captured request)

A coarser asset×class matrix sits above the element layer for reporting (`h1-scope coverage report` prints it as a tested%/skipped-with-reason summary), but the enforcement — the part that actually blocks a hunt from shipping a sloppy report — operates at the element level, because that's the granularity where a real gap hides.

## Skip Accountability

This is the part of the system that exists because of a real failure, not a hypothetical one. Early in the ledger's life, marking a cell `skipped` with a free-text reason — "couldn't test, out of scope" — was sufficient to close it out. On a later hunt, a real bug sat inside exactly one of those loosely-dismissed classes, and it went unreported because the skip looked no different from a hundred legitimate skips around it. The lesson distilled into one line that now shows up across the lab's methodology docs: *gates certify accounting, not attack.* A ledger full of green checkmarks proves the paperwork is in order; it doesn't prove anyone actually pulled the trigger.

The fix was to make a skip cost something. `h1-scope coverage set-element` now rejects a bare `--reason` outright. To mark something skipped, you need one of two things:

- `--probe <http_status_code> --evidence <reference>` — proof you actually sent a request and got a result, even if the result was benign (a 404, a 403, a clean 200 with nothing sensitive in it)
- A typed `--skip-kind`: `unreachable-by-tooling`, `out-of-scope`, or `read-only-public` — a structured admission that this is a *category* of non-test, not a disguised "didn't get to it"

And the reckoning doesn't stop at the individual cell. If more than 60% of a session's elements end up skipped without that backing, the `coverage` gate fails outright, flagged `high_skip_ratio_unbacked` — the system refuses to let a hunt call itself covered when most of its "coverage" is actually absence of coverage with a note attached. (There's a deliberate escape valve — `--ack-skip-ratio` — for the rare case where that ratio is genuinely justified, but it's recorded to `gates.json` and never silent.)

## Inventory Building

Coverage isn't declared up front — it grows as recon finds more surface. `h1-scope coverage build-inventory` takes a catalog (a mined route list from static analysis, a GraphQL operation set pulled from introspection or webpack chunks, a crawled endpoint list) and turns each entry into a dispositionable ledger element with a specified class. `add-inventory` extends an existing ledger the same way, so a session that starts with fifty known endpoints and later mines two hundred more from a mobile app's DEX routes doesn't need a new ledger — it absorbs the new elements into the one it already has.

## ID Format Tracking

Not every element carries equal suspicion by default, and the ledger encodes that. `h1-scope coverage set-idformat <asset> --id-type int_sequential|uuid|opaque` records how an asset mints its object identifiers — and setting `int_sequential` automatically spawns a HIGH-priority cross-user `authz_idor` element for that asset. Sequential integer IDs are the single strongest signal that an authorization check is skippable by just incrementing a number, so the ledger treats that fact as self-evidently worth a dedicated test, rather than waiting for a human to notice the pattern and remember to add it.

## Finding Cross-Check

The ledger and the findings directory are required to agree with each other, and that agreement is checked mechanically, not by reviewer attention. A cell marked `status: finding` with no corresponding document in `artifacts/findings/` fails as an `orphan_finding_cell` — the ledger is claiming a bug exists that was never actually written up with evidence. The reverse direction is checked too: a findings document with no matching ledger cell gets flagged, because it means a real result exists somewhere that the accounting never recorded. Either direction of mismatch means the two systems that are supposed to be describing the same hunt have drifted apart, and the `exploitation_proof` gate treats that drift as a blocker.

## Real Numbers

On the DoorDash hunt, a 193-operation GraphQL catalog mined from webpack chunks (introspection was disabled) turned into 195 dispositioned elements — two more than the raw operation count, from parameter-level splits — closed out with the coverage and completeness gates both green and zero findings. That's what honest, exhaustive, boring security testing looks like in ledger form: not a triumphant bug, just a fully accounted "we checked, and it's clean."

The WHOOP hunt is the counter-example that justified building this system in the first place. Before skip-ratio enforcement existed, that session's ledger carried 87% unbacked skips — cells marked done with nothing behind the mark — and still read as a complete hunt. One of those skipped classes was where the real, reportable bug turned out to live. That's the data behind "gates certify accounting, not attack": a ledger can be 100% filled in and still be lying about what was actually tested, unless something forces every skip to show its work.
