# Target Selection — Hunt Success Score

Before a single request gets sent, the system has to answer a quieter question: *which program should we even be testing?* HackerOne lists hundreds of public bug bounty programs. Picking the wrong one wastes days against a hardened target; picking the right one turns a few hours of mobile automation into a real finding. That decision is itself a modeling problem, and it gets the same treatment as everything downstream of it — a scored, explainable, config-driven pipeline rather than a gut call.

## The Problem with Prestige

The first version of this scoring model ranked programs by generic profitability: bounty ceiling, program reputation, report volume. It put Netflix at #1. On paper that looks right — huge payouts, a famous name, lots of reports. In practice it's a terrible pick. Netflix runs a $6.4M-plus bounty program, has attracted ten years of researchers, and is about as hardened as a consumer-facing target gets. Hunting Netflix with a general-purpose toolkit is like entering a marathon you have no chance of placing in — the competition has already run every obvious attack a thousand times. Netflix has paid out over six million dollars in bounties. None of it to us.

The scoring was optimizing for *prestige*, not for *where bugs are actually findable*. Those are different axes, and conflating them meant the system kept recommending the targets everyone else was already beating on.

## Hunt Success Score (HSS)

The replacement model asks a narrower, more useful question: expected value for *our specific capabilities*. Not "which program pays the most" but "given what we can do that most hunters can't, where is our edge highest?"

That reframing changes the whole shape of the ranking. A smaller, less famous program with a thinner researcher pool but a mobile app that requires a genuine attested device to test can outscore Netflix by a wide margin — not because the bugs are bigger, but because almost nobody else can reach them.

## Five Signal Groups

HSS combines five weighted signal groups into a single score per program:

| Group | Weight | Question it answers |
|---|---|---|
| `capability_fit` | 30% | Does this target's stack match one of our four strength lanes? |
| `moat` | 22% | How hard is it for *other* researchers to even test this? |
| `findability` | 22% | How likely are bugs to actually exist here? |
| `payout_ev` | 18% | Expected payout, weighted by severity distribution |
| `reachability` | 8% | Can we actually get in and start testing? |

`capability_fit` is scored against four defined lanes, each carrying its own weight and its own set of associated CWEs so the model can recognize the signal even before a bug is found: genuine-device mobile testing (certificate pinning, attestation, hardcoded secrets), two-account IDOR/BOLA differential testing (access control, authorization, object references), CDP-based web-token capture (session and auth weaknesses behind bot protection), and GraphQL/Cognito/BFF testing (access control and authorization in GraphQL-shaped backends).

`moat` scores the opposite side of the same coin: identity-verification requirements, KYC gates, and genuine-device-required mobile apps all raise the bar for the average researcher. If a program's policy text mentions "identity verification required" or "proof of identity," most hunters without a verified, equipped setup simply can't compete there — and that's exactly the kind of friction that should work in our favor, not against it.

`findability` looks at program age, how fresh the in-scope assets are, and report-volume trends — a sign that bugs are still being found, not that the surface is picked clean. `payout_ev` weighs expected payout by severity distribution rather than just citing the program's advertised maximum. `reachability` is the practical gate underneath all of it: free signup, no paid-account wall, an API surface that's actually testable without a six-week KYC process.

## Edge-Maximizing

The underlying thesis is simple: hunt where your specific capabilities create a structural advantage. A genuine Pixel 6a running GrapheneOS over wireless ADB can install and exercise mobile apps that most researchers can't — because those apps check device attestation, and a browser-based or rooted-emulator setup fails that check outright. That's not a generic strength; it's a moat that happens to run in our favor instead of against us. The same logic applies to two-account differential testing: it requires two real, independent accounts and the discipline to diff live responses between them, which is more setup than most casual testers bother with.

HSS is built to notice exactly that kind of asymmetry and rank it above a bigger, more famous, more thoroughly-tested target.

## The Hunter Capability Profile

All of this tuning lives in one JSON config file, not scattered through code: a capability profile that describes what the hunter can actually do — the four lanes and their weights, the moat categories and their weights, policy-friction penalties (programs that ban automated tooling or impose strict rate limits get scored down), and a newness curve that rewards the freshest in-scope asset aggressively rather than averaging across a whole scope table.

The ranking engine scores every program against this profile. Change the profile — say, drop a lane's weight because a tool broke, or add a new lane because a new capability came online — and the rankings shift accordingly, with zero code changes. The output is never "the best program." It's "the best program for this profile." That distinction matters: the moment the hunter's actual capabilities change, the profile should change with them, and the rankings follow automatically.

## CLI Usage

The ranking surfaces through a single small CLI:

```
h1-rank list --top 10                  # top 10 programs by HSS
h1-rank explain <program>              # full signal-group breakdown for one program
h1-rank list --lane mobile --fresh     # filter to the mobile-capability lane, freshest assets first
h1-rank refresh                        # recompute rankings from current program data
```

`explain` is the important one for trust: it doesn't just hand back a number, it shows which signal groups drove it — how much came from capability fit, how much from moat, where the policy-friction penalties landed — so a ranking can be audited and argued with, not just taken on faith.
