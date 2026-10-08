# 3. The CLI Toolkit

If the agent is the brain, these 26 programs are the hands. Each one is a small, single-purpose Python CLI living under `bin/h1-*`. None of them think — they validate scope, send requests, parse structured data, and print JSON. All reasoning about what to run, in what order, and what a result actually means happens one layer up, in the agent's own judgment.

## Design philosophy

Three rules hold across all 26 tools, without exception:

**JSON in, JSON out.** Every tool prints a JSON object to stdout on success and a JSON object to stderr on failure. There is no tool that returns prose, a bare exit code, or a human-formatted table as its primary output (a few, like `h1-scope coverage report`, also offer a human-readable view, but the machine-readable path is always there). This matters because the agent is the consumer: it needs to parse results reliably across hundreds of tool calls in a session, not scrape text. Prose is for humans. An agent that has to regex its own tool output is debugging two problems at once.

**Scope-filtered by construction.** Any tool that can send a network request to a target — `h1-cors`, `h1-graphql`, `h1-test`, `h1-smuggle`, `h1-gobuster`, `h1-httpx`, `h1-scan`, `h1-jwt`, `h1-surface`, `h1-mobile`, and others — accepts a `--session-dir` flag. When it's passed, the tool loads that session's `scope.json` and checks the target against it *before* the first byte goes out. If the target isn't in scope, the tool exits with an `OUT_OF_SCOPE` JSON error and code 2 — it never fires the request to find out. Run one of these tools without `--session-dir`, and it falls back to "standalone mode" with no scope check, which is useful for ad-hoc testing against something you've already confirmed by hand, but every tool call inside an actual hunt session passes the flag. If the scope-checking module itself fails to import, the tool fails closed (blocks) rather than silently skipping the check.

**Composable, not monolithic.** Each tool does one job — decode a JWT, probe CORS, mine a mitmproxy capture — and hands its JSON output to the next tool or to the agent's own reasoning. The payoff is a pipeline: a mobile app's static analysis output becomes a route inventory, the route inventory becomes a broken-access sweep, the sweep's candidates become individually validated IDOR tests. No single tool tries to be the whole pipeline.

## The toolkit at a glance

| Tool | Purpose |
|------|---------|
| `h1-api` | Read-only HackerOne hacker-API client — the single chokepoint for `api.hackerone.com`. Authoritative source for program scope, eligibility, policy, and accepted weaknesses (not market stats like report counts — that's still the scraper's job) |
| `h1-auth` | Centralized auth-token store, keyed per session, so every other tool can share one set of credentials |
| `h1-browser` | CLI front-end for the anti-detection browser automation server |
| `h1-cors` | Tests CORS misconfiguration — origin reflection, null-origin handling, subdomain bypass, credential inclusion — and renders an exploitability verdict based on the target's actual auth model |
| `h1-email` | Autonomous email verification-code reader over IMAP, no OAuth, no browser |
| `h1-evidence` | Collects and packages evidence for a finding, and formats it against HackerOne's real submission form fields |
| `h1-gobuster` | Directory and DNS brute-force, wrapped to emit structured JSON instead of gobuster's text output |
| `h1-graphql` | GraphQL-specific testing: introspection, query-depth abuse, batched queries, field-level authorization diffing across two tokens, alias abuse |
| `h1-httpx` | HTTP probing wrapper (tech detection, status, title) that accepts piped host lists and always emits JSON |
| `h1-jwt` | JWT analysis and attack testing — algorithm confusion, `kid`/`jku` header abuse, cracking |
| `h1-methodology` | Athenaeum-backed methodology engine: tech-specific attack playbooks, exploit-chain paths, decision trees, and devil's-advocate reasoning templates |
| `h1-mobile` | Mobile app testing across a genuine physical device and a rooted emulator — install, pull, static analysis, dynamic instrumentation, token capture |
| `h1-nuclei-gen` | Generates targeted nuclei YAML templates from an observed tech fingerprint |
| `h1-otp` | Unified verification-code reader across SMS, on-device Gmail notifications, and IONOS email — returns whichever channel's code is freshest |
| `h1-proxy` | Wraps a local Caido instance's GraphQL API for programmatic traffic history and export |
| `h1-rank` | Hunt Success Score (HSS) target ranking — expected value for this toolkit's specific capabilities, not generic program profitability |
| `h1-recon` | Reconnaissance: subdomain enumeration, scope-diffing against a previous run, tech fingerprinting, JS analysis, program-intel scraping |
| `h1-safety` | Pre-execution risk gate — checks a proposed action's risk level and scope before anything runs |
| `h1-scan` | Runs standard scanners (nuclei, basic XSS probes, port scans, tech detection) and normalizes their output to JSON |
| `h1-scope` | The core engine — scope validation, the gate system, the coverage ledger, formation planning, and the creative-pass log. Described in detail below |
| `h1-siwe` | Sign-In With Ethereum (EIP-4361) signer — drives a wallet-auth challenge/response round trip for web3 targets |
| `h1-smuggle` | HTTP request-smuggling testing (CL.TE/TE.CL/TE.TE over HTTP/1.1, plus HTTP/2 downgrade smuggling) |
| `h1-surface` | Turns a captured session (`flows.mitm`) into a scope-filtered, deduped endpoint inventory with IDOR/BOLA candidate tags, then drives validated tests over the differential candidates |
| `h1-test` | The vulnerability test runner — IDOR, auth bypass, race conditions, XSS, and multi-step business-logic workflows |
| `h1-validate` | Validates a finding against compliance rules and generates the "devil's advocate" developer objections a real triager would raise |
| `h1-verify` | Screenshot-based verification through a vision model, for pipelines running without an interactive agent in the loop |

## The hub: `h1-scope`

Most of these tools are a few hundred lines. `h1-scope` is roughly 4,400 — by a wide margin the largest program in the toolkit, because it's where every piece of session bookkeeping converges:

- **Scope validation** (`check`, `list`, `validate-file`) — the actual in/out-of-scope decision every other tool defers to.
- **The gate system** (`gate check|pass|waive`) — compliance, coverage, completeness, and the rest of the checkpoints a session must clear before advancing. A gate can be explicitly waived with a logged reason, but never silently skipped.
- **The coverage ledger** (`coverage init|set|add-asset|report|gaps|add-inventory|set-element|set-elements|build-inventory|set-idformat`) — the element-by-element accounting of what's been tested, what found something, and what was skipped and why. `set-element --status skipped` refuses a bare `--reason`; it demands either a live probe result or a named, specific skip kind.
- **Formation planning** (`formation show|plan`) — resolves which multi-agent formation (solo vs. parallel) applies to a session and which conditional agents fire based on what's actually in scope.
- **Agent-report handling** (`agent-report validate|aggregate`) — validates the JSON a spawned agent hands back and rolls multiple agents' reports into one view.
- **Completeness** (`completeness accept|suggest`) — the check that every coverage cell actually got a disposition, not just the easy ones.
- **The creative-pass log** (`creative-pass record|report`) — the adversarial, off-the-checklist testing pass gets its own accountable record, separate from the mechanical class sweep.
- **Session status** (`status`) and the **prior-art check** (`hacktivity-check`) that has to run and pass before any finding gets written up.

One tool, one file, because all of it is really one concern: can this session prove, on demand, exactly what it did and didn't check. Every other tool does the hunting. This one keeps the receipts.

## A composability example

Here's the chain the toolkit is actually built around, using sanitized placeholders (`example-program`, `api.example.com`) in place of a real target:

**1. Mine a mobile app's routes.**

```bash
h1-mobile static com.example.app --session-dir sessions/example-program
# -> emits routes.json: every API route the app's compiled code references,
#    plus an integrity classification and auth-provider fingerprint
```

**2. Turn the route inventory into a broken-access sweep.**

```bash
h1-surface sweep --inventory routes.json --session-dir sessions/example-program
# -> one representative request per route; any status code outside
#    {401,403,404,405} on an unauthenticated call is flagged as a
#    missing_auth candidate, and the coverage ledger is auto-updated
```

**3. Validate — or kill — each candidate individually.**

```bash
h1-test idor https://api.example.com/v1/orders/{id} \
  --id-param id --tokens $TOKEN_A,$TOKEN_B \
  --session-dir sessions/example-program
# -> fires the actual request as both accounts; status is strictly one of
#    "vulnerable", "secure", or "inconclusive" — a candidate never
#    self-promotes to a finding without this step
```

A `candidate` from `h1-surface` is explicitly never treated as a finding. It's a flagged location worth looking at — the status field says so in the JSON itself. Only `h1-test` (or the equivalent manual check) converts a candidate into a disposition the coverage ledger will accept.

The same pattern repeats elsewhere in the toolkit: `h1-recon` output feeds `h1-nuclei-gen`; `h1-graphql introspect` output feeds the rest of the `h1-graphql` subcommands; two accounts' captured traffic feeds `h1-surface diff`, whose candidates feed `h1-test idor` again. No tool tries to own the whole chain — each one is a link, and the agent decides which link comes next based on what the previous one actually returned.

## Why 26 separate binaries instead of one

Splitting the toolkit this finely has a cost — more files, more argument parsers, more places scope enforcement has to be copy-pasted in. The payoff is that each tool can be read, tested, and reasoned about in isolation: `h1-cors` can't accidentally touch the coverage ledger, `h1-jwt` can't accidentally fire a request outside scope, and `h1-scope` is the one place where session state actually changes. When something misbehaves, the blast radius is one file, not the whole system — which is the same bet the rest of this project makes about keeping the deterministic parts simple, so the one actually-hard part, deciding what to do with the results, has a stable, inspectable foundation to run on.
