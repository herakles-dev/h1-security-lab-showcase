<!-- This is a fabricated example for illustration purposes. -->
<!-- No real targets, vulnerabilities, or program data is included. -->

# Gates: example-program Hunt

Nine sequential gates. Each one is machine-checked against `state/gates.json` —
`h1-scope gate check <name>` reads the live state, not this document, to decide
whether a phase may proceed. A gate can be `passed`, left open, or explicitly
`waived` with a logged reason; it can never be silently skipped.

## Gate 0: Compliance (BLOCKING)
**Criteria**:
- [ ] Program rules read and understood
- [ ] Scope boundaries documented
- [ ] Test accounts created
- [ ] Out-of-scope assets identified

**Unlocks**: recon, testing phases

## Gate 1: Recon Complete
**Criteria**:
- [ ] Subdomain enumeration complete
- [ ] Technology stack identified
- [ ] API endpoints discovered
- [ ] Attack surface mapped (mobile routes mined, if in scope)

**Unlocks**: auth_session, testing phases

## Gate 2: Auth Session
**Criteria**:
- [ ] Authenticated session obtained — a verified capture (`capture.json`) OR a
      bearer token seeded via `seed-auth`/`web-capture`
- [ ] Second account obtained where differential (two-account) testing applies

**Unlocks**: testing phases

## Gate 3: Validation
**Criteria**:
- [ ] Each finding reproduced 3x (success)
- [ ] Each finding has 2x invalidation attempts (must FAIL — proves the vuln is real)
- [ ] CVSS scored — every metric backed by observed evidence, not assumptions

**Unlocks**: exploitation_proof

## Gate 4: Exploitation Proof (BLOCKING)
**Criteria**:
- [ ] Attacker demonstrably gained something concrete (data, access, funds-equivalent)
- [ ] Devil's-advocate pass completed — developer's likely dismissal written and rebutted

**Unlocks**: reporting

## Gate 5: Coverage
**Criteria**:
- [ ] Every high-priority asset × attack-class cell is `tested` or explicitly
      `skipped`/`blocked` with a backed reason (probe code + evidence, or a typed
      skip-kind) — no silent under-coverage

**Unlocks**: reporting

## Gate 6: Completeness
**Criteria**:
- [ ] Every enumerated coverage element dispositioned
- [ ] A written "what didn't we try" pass reviewed and accepted

**Unlocks**: reporting

## Gate 7: Creative Pass
**Criteria**:
- [ ] At least 3 adversarial-creativity vectors logged, each with a probe result
      (`tested` / `attempted-blocked` / `infeasible`) — a described-but-untried
      vector does not count

**Unlocks**: reporting

## Gate 8: Hacktivity Check
**Criteria**:
- [ ] Public prior-art search run for every reportable finding
- [ ] Verdict recorded: `no_prior_art` / `prior_art_public_variant` /
      `prior_art_public_same` / `ack_dup_risk_private_queue`

**Unlocks**: `h1-evidence report` (final submission generation)
