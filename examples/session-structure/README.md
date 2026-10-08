<!-- This is a fabricated example for illustration purposes. -->
<!-- No real targets, vulnerabilities, or program data is included. -->

# Session Structure

Every hunt against an authorized HackerOne program gets its own self-contained
session directory. Below is an annotated tree of what a session looks like
once a hunt is underway, using a fictional "example-program" target.

```
sessions/example-program/
├── spec.md                    # Hunt specification: target, scope summary, constraints, identity
├── gates.md                   # Human-readable gate definitions (9 gates + pass criteria)
├── scope.json                 # Program scope: assets, eligibility, severity caps (source of truth)
├── CLAUDE.md                  # Session-specific instructions layered on top of the repo-level CLAUDE.md
├── RUNBOOK.md                 # Phase-by-phase runbook for this specific hunt
├── b                          # Symlink to the anti-detection browser (tools/browser)
├── h1-*                       # Symlinks to all 26 h1-* CLI tools, so every tool invocation
│                               #   is scoped to this session directory by default
│
├── scratchpad/                # Working files: one-off scripts, scratch data, exploratory notes
│
├── state/                     # Machine-readable session state — the gates read/write here
│   ├── gates.json              # Gate status: passed/failed/waived, timestamps, blocking relationships
│   ├── coverage.json           # Element-granular coverage ledger (asset × attack-class × status)
│   ├── auth.json                # Captured/seeded auth tokens for authenticated testing
│   ├── completeness.json       # "What didn't we try" adversarial-completeness accounting
│   ├── creative_pass.json      # Creative-pass vectors attempted + probe results
│   ├── hacktivity_check.json   # Public prior-art search results + verdict
│   ├── methodology_log.json    # Decision-tree + think-like-dev usage log (devil's advocate trail)
│   ├── reproduction_log.json   # 3x-repro / 2x-invalidation attempts per finding
│   ├── RESUME.md               # Auto-generated resumable context for picking the hunt back up
│   └── agent-reports/          # Structured reports returned by formation agents (parallel mode)
│
└── artifacts/
    ├── evidence/               # Screenshots, raw HTTP responses, proof-of-concept captures
    ├── findings/               # One markdown doc per finding (FINDING-001.md, FINDING-002.md, ...)
    ├── intel/                  # Program intelligence gathered specifically for this hunt
    ├── mobile/                 # APK pulls, static-analysis output, mined route catalogs
    ├── recon/                  # Subdomain lists, tech fingerprints, endpoint inventories
    ├── reports/                # Draft HackerOne submission reports, ready to paste in
    ├── tools/                  # Raw stdout/stderr logs from every tool invocation
    └── validated/              # Evidence packages for findings that cleared the validation gate
```

## Why this shape

- **Everything scoped to one directory.** Symlinked tools mean every command run
  inside a session is automatically bound to that program's `scope.json` — there's
  no way to accidentally fire a request at an out-of-scope host from here.
- **State is append-only and machine-checked.** The gate files aren't just notes —
  `h1-scope gate check <name>` reads them and blocks phase progression until the
  criteria are actually met.
- **Findings and coverage are cross-checked.** A `finding` cell in `coverage.json`
  without a matching doc in `artifacts/findings/` (or vice versa) fails as an
  orphan — the accounting has to match the attack.
