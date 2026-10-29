# negative-test-first

An agent skill for the general testing discipline of any product with
claims that must hold: business rules ("a withdrawal cannot exceed the
balance"), data invariants ("balance equals the sum of movements"),
persistence ("data survives a restart"), concurrency ("no double
spending"), API contracts, access control, and audit — everything where
"the feature works" proves nothing and only "the violation fails" does.

The skill grew out of practice: building a PostgreSQL access-control
extension where every protection mechanism must carry a negative test, and
every escaped defect becomes a named lesson that lives in code comments and
in the tests. Not textbook theory — a checklist of what actually broke and
how it got caught.

## The core idea

A requirement is covered only when a test shows that **violating it does not
work**. A suite that checks only happy paths proves nothing about the rule.
For every claim — "you cannot withdraw more than the balance", "only role X
may write table T", "data survives a restart" — there must be a test that
tried to violate it: at the boundary, one past the boundary, in parallel,
through an alternative path — and failed.

The usual order is reversed: positives first, negatives "if time permits".
Here the negative is written first; the positive is the cheap half. Security
is not a separate topic but the sharpest case of the same discipline: a
security claim with no negative test is a claim nobody has checked even once.

## What's inside

Three layers: a universal "claim → required negative" table (seven kinds of
claims: feature conditions, business rules, data invariants, persistence,
API contracts, concurrency, security), nine patterns and eight
anti-patterns. The patterns, briefly:

* every requirement pairs with a negative test — the claim list becomes the
  coverage index: each claim names the test that violates it;
* input classes are enumerated and tested individually, not by example:
  empty, one, many, at the limit, past the limit — off-by-one errors live
  exactly at the boundary;
* after an expected failure, assert the world did not change: nothing was
  created, charged, or counted — a half-applied error is a bug in a test's
  clothes;
* failures are classified by **who** failed — your code, the platform, the
  driver, a typo; "caught some exception" does not count;
* platform branching is honest: if another build's platform catches the
  violation earlier, the test says so ("our check is defense in depth")
  instead of reporting a green pass for nothing;
* failure evidence is checked in two channels: what dies with the
  transaction and what survives it (the server log);
* negative tests rerun after a migration on live data — an upgrade that
  silently relaxes a rule is worse than a bug: a regression with a green
  release note;
* performance thresholds come from an A/B baseline; absolute limits are
  valid only under proven-quiet conditions;
* counts are exact ("57 OK, 0 FAIL", failures by name), a release waits for
  a green run on every platform, and an escaped defect becomes a named
  lesson.

## Install

The skill is agent-neutral — a `SKILL.md` with principles and patterns.

```bash
git clone https://github.com/Bra1nZ/negative-test-first
cp -r negative-test-first ~/.config/opencode/skills/negative-test-first
```

For agents that read skills elsewhere, put `SKILL.md` where the agent looks
for skills. It activates when the task involves designing a test plan for a
feature or a product, writing tests for business rules or invariants,
reviewing negative-case coverage, classifying failures, or setting
performance baselines.

## Why agents need this

Agents write positive tests well and negative tests poorly: they check that
code does what was asked and almost never that code refuses what was not.
The more expensive a violated rule is, the more this bias costs: a green
suite looks like "the rule works" while not a single attempt to violate it
was ever made. The skill forces the question "which test tried to violate
this rule?" before a feature is called done.

## License

MIT — see [LICENSE](LICENSE).
