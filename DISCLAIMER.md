# Disclaimer

This repository documents the methodology, architecture, and tooling behind an
authorized bug bounty research practice. It is a showcase of process and
engineering, not a product and not a how-to guide for attacking systems
without permission.

## Authorization Framework

Every target referenced in this repository's methodology is a live
HackerOne-hosted bug bounty program. In each case, the vendor itself
publishes a scope table that publicly invites outside researchers to test
exactly those listed assets, under a coordinated-disclosure agreement and
safe-harbor policy that the vendor wrote and maintains.

This matters because it changes what the testing described here actually is:
authorized third-party security testing performed at the vendor's own
invitation. The authorization is the program's published scope — not an
assumption, a verbal agreement, or something inferred from context. If a
system isn't named in a program's current scope table, it isn't in scope,
full stop, regardless of how interesting it looks or how it's reached.

## Mechanical Enforcement

Authorization here isn't just a policy written down and hoped for — it's
enforced in the tooling itself, before any request leaves the machine:

- **Compliance gate** — every hunt session is blocked from proceeding past
  its first phase until the program's own rules of engagement have actually
  been read and acknowledged.
- **Scope checks on every tool call** — each tool in the toolkit checks the
  target it's about to touch against that program's recorded scope
  (`scope.json`) before it fires a single request. A target outside scope
  doesn't get tested; the tool call doesn't go out.
- **Pre-execution risk gate** — a dedicated safety check sits in front of
  execution generally, independent of the scope check, to catch
  higher-risk actions before they run.
- **Fail-closed, not fail-open** — if scope can't be confirmed, or program
  rules haven't been read, the default behavior is to stop, not to proceed
  optimistically.

The intent is that scope violations are something the tooling structurally
can't do, rather than something a careful operator has to remember not to
do.

## What This Repository Contains

- Methodology documentation describing how authorized testing is planned,
  executed, and validated (recon, testing, proof-of-exploitation standards,
  reporting discipline).
- System architecture and tooling design — how the orchestration, gating,
  and automation pieces fit together.
- Sanitized, generic examples used to illustrate a technique or workflow.
- Honest case studies, including write-ups of findings that didn't pan out,
  were rejected, or turned out to be duplicates — the practice is "prove it
  or kill it," and that includes documenting the kills.

## What This Repository Does NOT Contain

- Exploits, payloads, or attack code usable against a live, specific target.
- Secrets, credentials, API keys, tokens, or session material of any kind.
- Target-specific intelligence (real endpoints, internal infrastructure
  details, or scope data tied to a named, still-active program).
- Private or undisclosed vulnerability report content, including anything
  still under a program's coordinated-disclosure embargo.

If any of the above were ever found in this repository, it would be a
packaging mistake, not an intended disclosure — treat it as such and flag it.

## Responsible Use

The methodology, architecture patterns, and tooling concepts described here
are meant to be read, learned from, and adapted — but only applied against
systems you are actually authorized to test: a program you've enrolled in
and whose current published scope covers the asset, or a target where you
hold explicit written permission from the system's owner.

Accessing, probing, or testing a computer system without that authorization
is illegal in most jurisdictions, regardless of intent, and regardless of
whether a vulnerability is ultimately found. Scope tables change; programs
get suspended or shut down; "it used to be in scope" is not authorization.
Verify current scope and current program status directly before testing
anything.

## HackerOne Safe Harbor

HackerOne-hosted programs commonly include a safe-harbor commitment: when a
researcher's testing stays within the program's published scope and follows
its stated rules of engagement, the vendor commits not to pursue legal
action over that testing. That protection is conditional on the program's
own terms — it covers good-faith testing done *within* the invitation, not
testing generally, and it does not extend to any asset, technique, or
behavior the program has explicitly excluded. This repository does not
reproduce any specific program's safe-harbor or legal text; read the actual
policy published by the specific program before relying on it.

## No Warranty

This repository is a methodology and engineering showcase, not a product,
service, or guarantee of outcomes. Nothing here promises that following this
approach will find a vulnerability, that a finding will be accepted, or that
a bounty will be paid. Programs, scopes, policies, and payout structures are
set unilaterally by each vendor and change over time; this repository is not
responsible for keeping pace with those changes or for any outcome of
applying this methodology elsewhere.

Everything in this repository is provided as-is, for educational and
reference purposes, with no warranty of any kind, express or implied.
