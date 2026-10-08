<!-- This is a fabricated example for illustration purposes. -->
<!-- No real targets, vulnerabilities, or program data is included. -->

# Hunt: example-program

## Intent
Security assessment of the example-program HackerOne bug bounty program.

## Target
- Program: example-program
- Platform: HackerOne
- URL: https://hackerone.com/example-program
- Hunt Success Score (HSS): 71.2 (confidence 1.0)
- Bounty range: $100 - $15,000
- Launch: fabricated date

## Scope Summary
- `api.example.com` — primary API host, eligible, max severity: critical
- `app.example.com` — web application, eligible, max severity: high
- `com.example.app` — Android/iOS mobile app (Google Play / App Store), eligible, max severity: critical
- `staging.example.com` — explicitly NOT eligible (out of scope)

## Constraints
- **Free tier only.** No payment flows, no card-present testing, no real-money spend
  without explicit operator authorization.
- **No destructive testing.** No data deletion, no mass account creation, no load/DoS.
- Rate limits respected per the program's published policy.

## Identity (authoritative)
- Primary test email: `researcher@example.com` (fabricated)
- Plus-alias for additional accounts: `researcher+example+{tag}@wearehackerone.com`
- Unauth research header: `X-HackerOne-Research: researcher-example`
- OTP channel order: SMS-on-device → email (IONOS IMAP) → Gmail-on-device

Never guess the email or OTP path — read the session's identity config. Never use a
personal or ad-hoc address for any program signup.

## Hunt Type
Solo formation — single-agent walk through recon → test → validate → report.
(Parallel formation available via `h1-scope formation plan` for larger scopes.)

## Priority Attack Classes
1. IDOR / BOLA — cross-user object access, the highest-signal class for this target
   shape (multi-tenant REST + GraphQL API, sequential and UUID object ids observed)
2. Auth bypass — session/token handling across the web app and mobile client
3. GraphQL — introspection, batched queries, field-level authorization

## Bounty Table (fabricated)
| Severity | Range |
|----------|-------|
| Critical | $5,000 - $15,000 |
| High | $1,500 - $5,000 |
| Medium | $500 - $1,500 |
| Low | $100 - $500 |
