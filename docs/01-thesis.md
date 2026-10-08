# 1. The Thesis

Bug bounty hunting is a decision-routing problem.

At every step there are more things to check than time allows: many endpoints, many possible issue categories, a handful of test accounts, and a lot of tool output that is mostly noise. The hard part is not knowing one clever trick. It is deciding what to check next, judging whether a result means anything, and knowing when to dig deeper versus move on. This project's working claim is that an AI agent can do that routing job well — not because it is clever, but because it is methodical, keeps itself accountable, and is honest about what it did not check.

## A CLI-first brain

The hunter is Claude Code, Anthropic's CLI agent. It runs in a terminal, executes shell commands, reads files, parses the JSON that tools return, and decides what to run next. There is no separate "AI backend" working behind the scenes. The conversation itself is the hunt record: every command, result, and decision lives in one transcript that can be reviewed afterward. Including the decisions that turned out to be wrong, which is most of them.

Around the agent sits a toolkit of 26 purpose-built command-line programs, plus a set of standard, widely used security utilities, all producing structured JSON. The agent's job is to combine them using judgment.

## Why this isn't an automated scanner

A typical automated scanner runs a fixed list of checks against a target and pattern-matches the responses. It doesn't know why it's running a given check, can't tell that a wall of identical error codes across a sweep is probably a network appliance blocking it rather than the application behaving a certain way, and can't decide that a minor issue is worth a follow-up because it might combine with something else.

This system instead reasons about the work:

- Which category of issue fits what earlier reconnaissance actually showed.
- Whether a result is a meaningful signal or just a side effect of the network path.
- Whether to continue down a lead or set it aside.
- What a skeptical reviewer would say in response to a claim, and whether the evidence holds up anyway.

The tools themselves are kept simple and deterministic on purpose. The reasoning about how to sequence and interpret them is where the actual value lives.

## The authorization model

Every target studied here is a live, public bug bounty program. The company that owns the target publishes a scope describing exactly what it has invited outside researchers to test, under its own disclosure terms. That published scope is the authorization for the work — nothing here operates on an assumption of permission.

This boundary is enforced by the tooling itself, not just by intent. Before any phase of work proceeds, a compliance check requires that the program's own rules have actually been read. Every session-aware tool call checks the target against that program's recorded scope before it sends a request. An additional safety check sits in front of anything higher-risk. Work outside a program's published scope, or against a target that has not published a program at all, is simply out of bounds by design.

## Prove it or stop

The operating rule is: no theoretical claims, and no inflated severity. Every claim needs evidence that was actually observed, not reasoned about in the abstract. Before writing anything up, the process deliberately drafts the strongest skeptical response first, then checks whether the evidence survives it. The bar for submitting anything is: would this be worth paying for? If not, it doesn't go out.

A corollary matters just as much as the rule: ending a session with "no issues found, thoroughly checked" is a legitimate and honest result. It is also, statistically, the most common one. A well-built target should often produce exactly that outcome, and a process that always reports something is a process that is padding its results.

## The real innovation: accountability

The most valuable part of this lab isn't its ability to turn up issues. It's the discipline around proving what was actually checked and what wasn't.

Interesting results are the visible part of this kind of work, but the harder part is being honest about coverage. "I checked that and it was fine" only means something if there's a clear record of what "that" covers. A familiar failure mode in this field is a category getting waved off with a plausible-sounding reason, the reason never being challenged, and the real issue later turning out to live in exactly that waved-off category. A clean-looking checklist is not the same as a thoroughly checked target.

So the framework leans heavily on bookkeeping that holds itself to a standard:

- **A coverage ledger** that breaks each target down into individual elements and categories, each of which must be given a recorded disposition.
- **Accountable skips.** Marking something as skipped is treated as a claim, not a free pass — it has to be backed by an actual probe and a recorded result, or by a specific, named reason such as "unreachable by the available tooling." A vague justification is rejected outright, and a record with too many unsupported skips is flagged rather than accepted.
- **Checkpoints** that must be satisfied before work can move to the next phase or before anything gets written up — covering authorization, coverage, completeness, and verification.
- **A feedback loop**, where friction and mistakes from one session get written down and folded back into the tooling, so the next session starts from a better baseline.

The following chapters describe each of these components in detail.
