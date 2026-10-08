# Lessons and Honest Limits

Most write-ups of security tooling describe what the tooling can do. This chapter is about what it cannot do, what we got wrong along the way, and where a human is still required.

## What Doesn't Work

### 1. Gates certify accounting, not attack

This is the system's most important lesson, learned the hard way. Passing every gate means the process was followed — every element in the coverage ledger was dispositioned, every required pre-check ran. It does not mean the target is secure. On one hunt the dashboard was fully green and the conclusion was "no finding." That conclusion was wrong: a real bug was sitting in a class that had been marked tested when it hadn't actually been attacked. A green dashboard can hide real bugs in exactly the areas it says are covered. The fix was not a new gate — it was accepting that gates measure diligence, not outcomes, and treating a clean dashboard as an invitation to look harder, not a reason to stop.

### 2. Public prior-art checks can't catch private duplicates

Before writing up a finding, the system checks public disclosure history for prior art on the same class and asset. That check is real and it reduces duplicate-submission risk. But most pending reports on a given program are not public — they're sitting in a private review queue. A valid, well-evidenced finding was submitted and later closed as a duplicate of a private report filed weeks earlier. No amount of public searching would have caught that, because the colliding report was never visible to search in the first place. This isn't a bug to patch; it's a structural limit of any prior-art check built on public data. Even the public half of the check only covers a bounded recent window of a program's history, so it's an imperfect filter twice over.

### 3. CAPTCHA and bot-challenge systems block automated testing

Challenge systems like Cloudflare Turnstile and reCAPTCHA are designed to stop exactly the kind of scripted request flow that security tooling relies on. Driving a real browser on a real device through the login step works around the initial challenge and produces a valid session token. But if the target re-challenges on individual API calls rather than just at login, every subsequent automated request can still get walled off. The practical effect is reduced coverage on heavily protected targets — some classes of request simply can't be exercised at volume, and that has to be recorded as a limit rather than quietly skipped.

### 4. TLS client fingerprinting catches standard HTTP libraries

Common HTTP client libraries have a recognizable TLS handshake fingerprint. A CDN or WAF can reject that fingerprint outright, independent of anything about the request itself — and the failure often looks identical to a legitimate authorization rejection. A wall of uniform 403s across an entire sweep is genuinely ambiguous: it can mean "this is well-secured" or it can mean "the transport got blocked before the application ever saw the request." Treating the two as the same thing was an early mistake. The tooling now treats a uniform block pattern against a CDN-fronted host as inconclusive rather than banking it as a secure result, and falls back to a browser-routed transport to get a real answer.

### 5. "Skipped with reason" was gaming the system

Early on, marking a coverage element "skipped" with any plausible free-text reason satisfied the completeness gate. In practice this let hard-to-test areas go unattacked while the dashboard still read as fully accounted for. One retrospective audit found that the large majority of one ledger's entries were skips with no actual evidence behind them — just a reason string. A skip was functioning as a way to look done without being done. The fix: skips now require either live probe evidence or a narrow, typed justification (out of scope, unreachable by current tooling, read-only and non-sensitive) — a bare explanation is no longer accepted, and a high ratio of unbacked skips now fails the gate outright.

### 6. Multi-agent parallelism was theoretical for a long time

A formation of multiple specialist agents working waves of a hunt in parallel was designed and documented well before it was ever exercised for real. Every hunt before that point ran solo, one agent walking the phases in sequence. The first genuine live parallel run surfaced problems — around how agents hand back results and how those results get reconciled into one coherent session state — that the design documents hadn't anticipated, because nothing had actually forced them to happen. A capability that only exists on paper isn't a capability yet.

### 7. Server-side mobile attestation has no bypass

Some mobile targets verify device integrity on the server, not just on the device. That kind of check can't be spoofed by rooting an emulator or patching the app locally, because the proof of a genuine device is generated and verified somewhere the tooling never touches. The only paths that work are using an actual physical device that passes attestation honestly, or capturing a usable session token through a browser-based login flow that doesn't depend on the native app's attestation at all. There is no clever workaround here, and time spent looking for one is time wasted.

## What a Human Still Has to Do

1. **Pay for accounts.** Some programs gate their real attack surface behind a paid tier or a purchase. The system can flag that and stop; it should never spend money on its own authority.
2. **Provide government ID for identity verification.** Financial and regulated programs frequently require KYC before testing can go any deeper. That is a hard, deliberate stop for automation.
3. **Make the final severity call.** Whether something is a Medium or a High often depends on business context — how the feature is actually used, what data really sits behind it — that a scoring rubric approximates but doesn't fully capture. The rubric narrows the range; a person makes the final call.
4. **Decide whether to submit at all.** Every submission carries reputation risk on a researcher's account, and a weak or marginal report can do more harm than good. The system applies a simple test — would this be worth a bounty? — but the decision to actually send it is a human one.
5. **Navigate unusual onboarding.** Invitation codes, physical verification steps, or hardware requirements for account creation fall outside what automated signup flows can handle, and someone has to step in manually.

## The Meta-Lesson

The system's biggest strength is also its biggest limitation: it is built to be honest about what it did and didn't cover. A tool that reports "no vulnerabilities found" without being able to say what fraction of the surface it actually exercised is less trustworthy than one that reports "roughly this much tested, this much blocked by tooling, this much skipped with evidence." The coverage ledger, the gate system, and this chapter all exist because an accounting of limitations is worth more than a dashboard that only ever says green.
