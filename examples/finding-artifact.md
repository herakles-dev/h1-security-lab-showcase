<!-- This is a fabricated example for illustration purposes. -->
<!-- No real targets, vulnerabilities, or program data is included. -->

# FINDING-001: IDOR in User Profile API Allows Reading Other Users' Data

**Program**: example-program
**Asset**: `api.example.com`
**Severity**: Medium (CVSS 3.1: 6.5)
**CWE**: CWE-639 — Authorization Bypass Through User-Controlled Key
**Status**: Validated — all gates passed, ready for submission

## Summary

The `GET /api/v1/users/{id}/profile` endpoint returns full profile data (email,
phone number, date of birth, home address) for any sequential integer user ID,
regardless of whether the requesting account has any relationship to that user.
Authorization checks the caller's session token for *authentication* only — it
never checks whether the caller is *authorized* to view the specific `{id}`
requested.

## Steps to Reproduce

1. Authenticate as **User A** (`researcher@example.com`) and
   capture the bearer token:

   ```bash
   curl -s -X POST https://api.example.com/api/v1/auth/login \
     -H "Content-Type: application/json" \
     -d '{"email":"researcher@example.com","password":"[redacted]"}' \
     | jq -r '.access_token'
   ```

2. Fetch User A's own profile to confirm the endpoint shape and that `id`
   is a plain sequential integer (User A's id observed: `48213`):

   ```bash
   curl -s https://api.example.com/api/v1/users/48213/profile \
     -H "Authorization: Bearer <USER_A_TOKEN>" | jq .
   ```

3. Replay the same request substituting an adjacent, unrelated account's id
   (`48214` — belongs to **User B**, created independently, no relationship
   to User A):

   ```bash
   curl -s https://api.example.com/api/v1/users/48214/profile \
     -H "Authorization: Bearer <USER_A_TOKEN>" | jq .
   ```

   **Result**: HTTP 200, full profile JSON for User B returned to User A's
   token — email, phone, DOB, and home address all present.

## Impact

An attacker with any valid account can enumerate `id` sequentially and harvest
PII (email, phone, date of birth, home address) for the entire user base.
Reproduced 3x against 3 independent target ids; 2 invalidation attempts against
a deactivated test account and a soft-deleted account both correctly returned
403/404, confirming the authorization check exists and fires in those cases —
it is specifically missing for active-account cross-user reads.

## Evidence

- `artifacts/evidence/finding-001-idor-profile.json` — full request/response pairs for both accounts
- `artifacts/evidence/finding-001-screenshot-user-a.png` — User A's own profile view
- `artifacts/evidence/finding-001-screenshot-user-b-leaked.png` — User B's data rendered from User A's session
- `artifacts/evidence/finding-001-invalidation-deactivated.json` — control case, correctly blocked

## Devil's Advocate

**Developer's likely dismissal**: "This endpoint is only used internally by the
mobile app's own profile screen — no client ever requests an arbitrary id, so
there's no real-world exposure."

**Rebuttal**: The endpoint is reachable directly over the public internet with
no additional gating beyond a valid bearer token — client-side usage patterns
are not an access control. The `id` parameter is a plain, enumerable integer
with no rate limiting observed on this route after 50 sequential requests,
making bulk harvesting trivial. "The official client doesn't do this" is not a
security boundary; the API itself is the boundary, and it has none here.

## CVSS Breakdown

| Metric | Value |
|---|---|
| Attack Vector | Network |
| Attack Complexity | Low |
| Privileges Required | Low (any valid account) |
| User Interaction | None |
| Scope | Unchanged |
| Confidentiality | High |
| Integrity | None |
| Availability | None |
| **Score** | **6.5 (Medium)** |
