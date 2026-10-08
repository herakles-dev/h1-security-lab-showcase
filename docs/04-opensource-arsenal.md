# Chapter 4: The Open-Source Arsenal

None of the raw capability in this lab is proprietary. Every scanner, crawler, and
fuzzer underneath it is a tool anyone can `go install`, `pip install`, or `git clone`
today — much of it maintained by the ProjectDiscovery team, OWASP, or a handful of
well-known independent security researchers. Counting templates and wordlists, the
underlying toolchain spans 37 distinct open-source tools, 9,600+ vulnerability
templates, and 6,000+ wordlist files. None of that is the hard part. A tool you can install is a tool. A tool an AI can read is an instrument.

The hard part — and the actual engineering contribution of this project — is the
layer wrapped around that toolchain: a set of CLI adapters that convert raw,
inconsistent tool output into structured JSON an AI agent can reason about, filter
every result against a program's published scope *before* it's returned, and chain
tools together so the output of one becomes the input of the next. This chapter
walks through what's installed, organized by function, and then explains why the
wrapper layer — not the tools themselves — is where the value actually lives.

## Reconnaissance

Reconnaissance is the first phase of any hunt: mapping what actually exists before
testing any of it. This lab layers ten tools across four sub-tasks.

**Subdomain enumeration** — `subfinder` does fast passive discovery by querying
public sources (certificate transparency logs, DNS aggregators, threat-intel feeds)
without ever touching the target directly. `amass`, OWASP's own attack-surface
mapper, goes further with active enumeration when passive discovery isn't enough.
`assetfinder` rounds out the pass with a lighter-weight, source-diverse subdomain
sweep.

**DNS resolution** — `dnsx` takes the raw subdomain lists those three tools produce
and resolves them at speed, filtering out wildcard DNS noise and pulling full record
sets (A, CNAME, TXT) so later stages aren't wasting requests on dead hosts.

**HTTP probing** — `httpx` and `httprobe` take a resolved host list and determine
which ones are actually serving HTTP, extracting page titles, status codes, and
technology fingerprints along the way. This is the step that turns "500 possible
subdomains" into "40 live web services worth looking at."

**Web crawling and URL harvesting** — `katana` and `hakrawler` crawl live sites to
enumerate endpoints, including JavaScript-rendered routes that a naive crawler would
miss. `gau` and `waybackurls` take a different approach entirely: they pull
historically-known URLs for a domain from the Wayback Machine, Common Crawl, and
URL-scan archives — surfacing old API routes, forgotten admin panels, and parameters
that may still be live but were never linked from the current site.

## Scanning

Once a target's surface is mapped, scanning tools check it for known issues.
`nuclei` is the workhorse here — a template-based scanner currently loaded with
9,660 community-maintained vulnerability templates covering CVEs, misconfigurations,
exposed panels, and default credentials. `nikto` runs a complementary, older-school
sweep focused on web-server misconfigurations and outdated software fingerprints.
`nmap` and `masscan` handle the network layer: service discovery and port scanning,
with `masscan` built for speed when a scope includes a large IP range.

## Fuzzing

Fuzzing tools find things that aren't linked anywhere. `ffuf` is a fast, flexible
fuzzer used for directory brute-forcing, virtual-host discovery, and parameter
fuzzing — anywhere a wordlist needs to be thrown at a target position. `feroxbuster`
and `gobuster` cover similar directory/file brute-force ground with different
performance characteristics, useful when one tool's request pattern gets rate-limited
and another's doesn't. `arjun` is narrower and more surgical: it discovers hidden
HTTP parameters (GET and POST) that aren't visible in any documented API, which is
often where business-logic bugs hide.

## Exploitation

This is where candidates get tested, not just found. `dalfox` and `xsstrike` both
hunt cross-site scripting, with `dalfox` doing DOM-aware parameter analysis and
`xsstrike` adding WAF-bypass fuzzing and context-aware payload generation. `sqlmap`
and `ghauri` both automate SQL-injection detection and exploitation, `ghauri` tuned
for speed on time-based blind cases where `sqlmap` runs slow. `jwt_tool` decodes and
attacks JSON Web Tokens — checking algorithm confusion, weak signing keys, and `kid`
header injection the moment a token shows up anywhere in a capture. `nomore403`
automates the long tail of 403-bypass techniques (header spoofing, path tricks,
verb tampering) so a blocked endpoint gets a real second look before anyone gives
up on it.

## JavaScript Analysis

Modern web apps ship most of their real attack surface inside bundled JavaScript,
not HTML. `jsluice` and `LinkFinder` both extract endpoints, URLs, and routes out of
`.js` files — the API calls a single-page app makes that never appear in any
documentation. `SecretFinder` runs the same files through a second pass looking for
hardcoded API keys, tokens, and credentials accidentally shipped to the browser —
a surprisingly common and surprisingly high-value finding class.

## Secret Detection

`trufflehog` and `gitleaks` both scan for leaked credentials — API keys, passwords,
private keys, tokens — but across different surfaces. `trufflehog` can crawl entire
git histories (not just the current checkout), object storage, and filesystems,
verifying live credentials where possible rather than just pattern-matching.
`gitleaks` focuses tightly on git repositories with a fast, regex-driven ruleset
tuned for CI pipelines.

## Mobile

Mobile testing uses a different toolkit entirely, all wrapped behind a single
`h1-mobile` interface described in Chapter 3. `apktool` decompiles Android APKs back
into readable resources and near-source code for static analysis. `frida` provides
runtime instrumentation — hooking live method calls on a running app to observe or
modify behavior without rebuilding it. `mitmproxy` intercepts and inspects the
app's network traffic, which is how a mobile app's API calls get captured for the
same scope-filtered analysis the web tools run.

## Browser Automation

`nodriver` drives an undetected, real Chrome browser instance — not a headless
automation flag that a bot-detection script can fingerprint — paired with `Xvfb` to
give it a genuine virtual display to render into rather than a stripped-down
headless mode. Together they back the lab's `./b` command: a browser every hunt
session gets by default, built specifically to survive the anti-automation defenses
that block naive Selenium or Puppeteer scripts outright.

## Wordlists and Patterns

Underneath the fuzzing and scanning tools sits `SecLists` — 6,026 files of
curated wordlists for subdomains, directories, parameters, and injection payloads,
one of the most widely used resources in the security community. Alongside it, 14
`gf` (grep-fu) patterns provide fast regex-based triage: piping a list of crawled
URLs through the `xss` or `sqli` pattern instantly narrows thousands of URLs down to
the handful worth manually testing.

## Why the Wrapper Layer Is the Point

Every tool in this chapter is free, documented, and already installed on tens of
thousands of security researchers' machines. Having `nuclei` or `ffuf` on a box is
not a differentiator — it's table stakes. What actually matters is what happens
around them, and that's the part described in Chapter 3's `h1-*` CLI layer:

- **Structured output.** Raw tool output is inconsistent — some tools emit JSON,
  some emit line-delimited text, some emit colored terminal output meant for a
  human's eyes. The wrapper layer normalizes all of it into a consistent JSON
  contract an AI agent can parse and reason over without ad-hoc regex scraping.
- **Scope enforcement before execution.** Every wrapped call checks the target
  against the program's published scope *before* a single request fires — not
  after, as a post-hoc filter on results. A tool that can't tell an in-scope asset
  from an out-of-scope one is a liability, not a capability.
- **Composability.** The real power shows up in pipelines: subdomain output feeds
  directly into HTTP probing, which feeds into crawling, which feeds into secret
  scanning, each stage's output becoming the next stage's input without a human
  manually reformatting files in between.
- **Error handling and rate limiting.** Production-grade wrapping means a scan that
  hits a WAF, a timeout, or a malformed response degrades gracefully and reports
  *why*, instead of silently returning empty results that look like a clean scan.

The arsenal is the raw material. The orchestration layer is what turns 37
individually unremarkable tools into a system that can actually reason about a
target end to end.
