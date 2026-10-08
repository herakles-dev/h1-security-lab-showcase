# Chapter 9: Hek — Genuine-Device Mobile Integration

## Why a Real Phone

Most security researchers test mobile apps on emulators: boot an AVD, root it, drop in Frida, intercept traffic with mitmproxy, done. It works great — until the app you're testing notices.

Modern mobile apps increasingly ask the platform a question before they'll talk to their own backend: *is this a real device?* Several independent checks feed that answer:

- **Play Integrity / device attestation** — a cryptographic statement from Google that the device is a genuine, unmodified build with a locked bootloader. Emulators and rooted phones fail this by construction.
- **TLS certificate pinning** — the app refuses to trust any certificate except the one it ships with, which breaks every classic MITM proxy that terminates TLS to inspect traffic.
- **JA3/JA4 TLS fingerprinting** — the shape of the TLS handshake itself can out a non-standard HTTP client even before certificate validation happens.
- **Root/emulator detection** — direct checks for `su` binaries, build fingerprints, and QEMU artifacts.

Stack all four together and a rooted emulator — the default tool in most mobile pentesting guides — fails every one of them simultaneously. The app simply refuses to function, and there's nothing left to intercept.

The fix isn't a cleverer bypass. It's removing the premise: test on a device that actually passes attestation. A real Pixel phone running a hardened, unmodified OS satisfies Play Integrity's strictest checks while still giving a researcher the control an emulator would have provided — because the control comes from *driving* the device over ADB, not from rooting it.

## Hek: The Device

The lab's mobile testing platform is a physical Pixel 6a running GrapheneOS, nicknamed "Hek." It is not rooted. It is not an emulator. It is, as far as any attestation API is concerned, an ordinary consumer phone — which is exactly the point.

Hek connects over wireless ADB, tunneled through the lab's own WireGuard VPN rather than exposed on a local network. That means the orchestrator (running on a separate machine) can drive the device — tap screens, type text, read notifications, install apps, pull APKs — as if it were plugged in locally, from anywhere on the VPN.

One operational detail matters enough to call out: GrapheneOS's wireless-debugging TLS pairing handshake doesn't work with the stock Android `adb` binary shipped on most Linux distros (version ~29). It silently lacks the `pair` subcommand entirely. The fix is using a current `adb-new` binary from a modern platform-tools release — once paired, the device otherwise behaves like any `adb` target. The device's wireless-debugging port also drifts across reboots and toggles, so the tooling re-resolves it from live `adb devices` output rather than trusting a cached value.

Everything downstream — app installs, UI automation, screenshot capture, static analysis, log reading — is driven through one CLI tool that treats Hek (and a secondary rooted emulator, described below) as interchangeable `--target` backends behind a shared command surface: pull an APK, run static analysis, intercept traffic, seed captured credentials into the authenticated-testing layer.

## The 3-Tier Integrity Classification

Before touching a target app at all, static analysis of its APK answers one question: *which interception strategy will actually work against this app?* The answer sorts into three tiers:

- **None** — no attestation calls, no certificate pinning in the decompiled code. The emulator's MITM lane (Frida + mitmproxy) works cleanly and is the fastest path.
- **Medium** — certificate pinning present, but no device-attestation calls. A rooted emulator with Frida-based SSL-unpinning scripts can still strip the pinning check at runtime and get a clear-text view of traffic.
- **Hard** — both attestation and pinning are present. Neither a rooted emulator nor a proxy on the genuine device will work: the attestation check fails on a rooted/emulated device, and the pin blocks any proxy regardless of device. The only remaining option is **web capture** on the genuine device, described next.

This classification isn't advisory — it's enforced in code, fail-closed. Pointing the tooling at a `hard`-tier app with `--target emulator` is refused outright rather than attempted and silently failing. That single guardrail prevents a wasted interception session against an app that was never going to let an emulator see its traffic in the first place.

## Web Capture (The Hard-Tier Solution)

A `hard`-tier app can't be intercepted — but it almost always shares a backend with a web login, and the web surface usually isn't gated by the same mobile-specific attestation. That asymmetry is the opening.

The technique, proven live against a social-commerce target with full certificate pinning:

1. Open a real Chrome tab on Hek itself, driven remotely over the Chrome DevTools Protocol (CDP) tunneled through ADB.
2. Navigate to the target's web login page — the same login a human would use in a desktop browser.
3. Drive the login (or registration) flow through that real browser tab: it passes Cloudflare Turnstile challenges and any device checks naturally, because it *is* a real browser on a real, unmodified device.
4. Read the resulting authentication token or cookie directly off the live tab's network traffic.
5. **Verify** the captured credential is actually load-bearing — issue a real authenticated request with it and confirm a non-trivial response — before trusting it for anything downstream.
6. Feed the verified token into the lab's authenticated-testing tools (IDOR/BOLA sweeps, GraphQL probing, broken-access-control testing), which can now route requests through that same live browser tab when the target's transport defenses would otherwise block a plain HTTP client.

No proxy. No certificate stripped. No root. The credential comes from a login flow the app's own backend considers completely legitimate, because it was. The most sophisticated bypass in the toolkit is logging in.

## DEX Route Mining

Static analysis doesn't stop at classifying integrity tiers. The same pass decompiles the APK's bytecode and mines it for API route strings and endpoint patterns, producing a structured catalog of the app's backend surface — paths, likely parameters, service groupings — without ever sending a single network request.

That catalog becomes direct input to the lab's automated broken-access-control sweep: a systematic pass that issues one representative request per discovered endpoint and flags any response that doesn't match an expected authorization boundary. The practical effect is that a researcher gets the app's *entire* API surface mapped before deciding where to spend manual testing time — turning what used to be "guess which endpoints exist by watching traffic" into "read the full list straight out of the compiled app."

## OTP & Verification

Account registration and login flows on mobile apps are gated by verification codes almost universally — SMS, push notifications, or email. A unified verification-code reader checks three sources in parallel: SMS and push notifications surfaced on the device itself, a device-linked mail account, and a separate IMAP mailbox used as the lab's standing test identity — returning whichever code arrived most recently.

This closes the last manual step in account creation. The orchestrating AI can register a fresh test account, trigger a verification code, read it off whichever channel the app actually used, and complete the flow — all without a human relaying a code from their own phone.

## Aurora Store

Apps land on Hek through Aurora Store, an anonymous, open-source client for the Play ecosystem. It installs real, unmodified Play builds — identical to what a consumer would get — without any Google account tied to the device and without Play Protect's device-management interference. A single automated flow deep-links into Aurora, taps install, confirms GrapheneOS's install-confirmation dialog, and polls until the package is present: fresh apps onto a genuine device, start to finish, with no manual phone-handling step.

## The Emulator Lane

Not every target needs the genuine-device treatment. For apps with no attestation at all — the `none` and `medium` tiers — a second, local device fills in: a KVM-accelerated rooted Android emulator running a recent API level, paired with Frida for runtime instrumentation and mitmproxy as the intercepting proxy. It runs alongside Hek as an independent device, giving the lab two simultaneous, isolated sessions — useful for anything that needs two distinct accounts or two distinct cookie jars in flight at once, without contending for the one physical phone.

Together, the two lanes cover the full spectrum: the emulator handles everything that doesn't fight back, and Hek handles everything that does.
