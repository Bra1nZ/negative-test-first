---
name: negative-test-first
description: >
  A general testing discipline for any product whose claims have to hold:
  a requirement is covered only when a test shows that violating it fails.
  Use when designing a test plan for a feature or a whole product; writing
  tests for business rules, data invariants, persistence, concurrency,
  payments, validation, auth, or permissions; reviewing whether a test
  suite actually proves anything; deciding whether a feature is done;
  classifying test failures and refusals; or setting performance baselines
  for critical paths.
---

# Negative Test First

Find out whether the rule actually holds, then prove nothing slipped past it.
This is an adversarial testing discipline: for every claim a product makes,
somebody — a test — must have tried to break it.

## The core rule

A requirement is covered only when a test shows that **violating it does not
work**. A test that only asserts the happy path proves nothing about the
rule. For every claim, ask: "which test attempts the violation?" If the
answer is none, the claim is uncovered — no matter how much green the suite
shows.

This inverts the usual question. Not "does my feature work?" but "what could
break it, and did anything actually try?" Write the negative first; the
positive is the cheap half.

The rule is universal. Security is where it is most expensive to skip, but
it applies to every kind of claim a product makes — business rules, data
invariants, state, contracts, and behavior under bad conditions.

## What counts as a violation

| Kind of claim | Required negative |
| --- | --- |
| Feature works under condition C | The same feature outside C: empty input, zero, one, many, maximum, wrong type, wrong order, expired, missing |
| Business rule ("a withdrawal cannot exceed the balance") | Each way to break it: exactly at the boundary, one past it, via a different entry point, in a concurrent transaction |
| Data invariant ("counter never decreases", "balance equals the sum of movements") | The operations that could break it: rollback mid-way, crash between steps, retry, replay of an old message |
| State survives ("data is not lost on restart", "the setting sticks") | Kill or restart at the worst moment: right after the write is acknowledged, before the flush, during migration |
| Contract ("the API returns error E for bad input") | Each rejected input class individually; "invalid" is not one bucket; and no partial write before the error |
| Concurrency ("no double spending", "one instance runs at a time") | Parallel attempts racing for the same resource, repeat delivery, two writers on one row |
| Security ("only role X can write table T") | Role Y, a role-less session, and X's neighbors cannot read or write T; the bypass through every alternative path is rejected |

A negative that is never attempted is a hope, not coverage.

## Patterns

### 1. Every claim pairs with its negative

Take the list of claims (README, regulator requirements, invariants, API
docs). For each one, write down the test that tries to violate it before
writing the test that confirms it. A claim without a paired negative goes
onto the backlog of uncovered requirements — visible, counted, scheduled.

### 2. Boundaries and classes, not examples

"Invalid input rejected" proves nothing. Enumerate the classes — empty,
one, many, at the limit, one past the limit, wrong type, wrong order,
duplicates, expired — and test each class individually. Off-by-one lives
exactly at the boundary; a random sample of bad values will miss it.

### 3. After a refusal, verify the world did not change

An error that leaves half-applied state is a bug wearing a test's clothes.
After every expected failure, assert the side effects:

- the object was not created (`SELECT count(*) ... WHERE name IN (...) = 0`);
- the money was not moved, the counter did not move, the file did not appear;
- the row count is unchanged, the old value is still in place.

The assertion of absence is part of the negative test, not optional cleanup.

### 4. Classify who failed

When an operation is expected to fail, the failure detail tells you which
layer did it: your code, the platform, the driver, or a typo. Capture both
facts:

```python
def expect_fail(cur, sql, errcode=None, marker=None):
    """Returns (refused: bool, detail: str). Direction matters."""
    try:
        cur.execute(sql)
    except Exception as e:
        detail = str(e)
        assert errcode is None or errcode in detail, f"wrong layer: {detail}"
        assert marker is None or marker in detail, f"not our mechanism: {detail}"
        return True, detail
    return False, ""
```

Never write a negative test that passes on any exception. "Something
failed" and "our rule refused with reason R" are different claims. If the
platform refuses before your code runs, report that honestly — your check
is then defense in depth, and the test says so in its name, not in a
comment.

### 5. Honest branching beats false green

Supported platforms differ. When the same negative can be caught by the
platform on one build and by your code on another, branch explicitly:

```python
if "APP_ERROR" in detail:
    report("our rule rejected the operation", True)
elif "platform denial" in detail:
    report("platform blocks the operation (our check is defense in depth)", True)
else:
    report("our rule rejected the operation", False, detail)
```

Both branches are passes, but each names what actually protected. A test
suite that cannot tell you who refused will hide the day the platform
stops doing it.

### 6. Two evidence channels

Records written inside a failing transaction die with the rollback. If
reliable evidence of a failure matters, keep a second channel that survives
it (server log, syslog, stderr marker) and test both: the in-transaction
record when the caller catches the failure, and the survive-channel when it
does not.

### 7. Negative tests survive migrations

Live data, then upgrade, then run the negatives again. An upgrade that
silently widens permissions or relaxes a business rule is worse than a bug —
it is a regression with a green release note. A migration block rerunning
the negative checks on migrated data is part of coverage, not a nicety.

### 8. Performance claims need a baseline, not a threshold

Absolute latency limits are valid only in proven-quiet conditions (load
average below your agreed bound, no neighbor builds). Prefer:

- A/B alternating runs (old vs new, repeated, interleaved) to attribute a
  regression to code rather than to the machine;
- medians of several server-side measurements, not single client-side shots;
- thresholds derived from the A/B baseline with headroom, and stated
  conditions of validity ("F1 is valid only at load < 3") next to the
  threshold in the test itself.

A suite that fails under load on every machine is worse than no suite:
it trains everyone to ignore red.

### 9. Count, name, gate

- Report exact counts: "57 OK, 0 FAIL", and list the failures by name.
- Gate releases: a release commit waits until the full matrix is green on
  every supported platform. If you must move before that, record exactly
  what passed where — partial truth, not implied completeness.
- When a defect escapes into the wild, distill it into a named lesson
  ("lesson I-12: in-transaction log rows die with the rollback") and put
  that name in the code comment and the test that now covers it. Lessons
  compound; folklore evaporates.

## The sharp end: security claims

For security-bearing code the discipline stops being optional. A security
claim with no negative test is not "partially covered" — it is a claim
nobody has checked even once. The positive test ("admin can grant") is
almost worthless: the question that matters is whether anyone *else* can.

- Every permission claim pairs with its outsider tests: neighboring roles,
  role-less sessions, expired sessions, alternative entry points.
- Guards are tested through every path that reaches the protected resource,
  not just the front door (direct calls, prepared statements, nested calls,
  batch operations).
- Audit trails are tested adversarially: can the actor modify or delete
  their own trail? Writes go through exactly one channel?
- Lockouts and rate limits are tested with the unlock path: the unlock
  itself must be permission-gated.
- A machine-readable traceability map (requirement → mechanism → test)
  when the code answers to auditors; every row needs a living test.

## Anti-patterns

1. Coverage theater: 100% line coverage of a validation function, zero
   tests of the inputs that must be rejected.
2. `assert raises Exception` without layer, error code, or reason.
3. Treating a platform's refusal as proof your mechanism works.
4. Asserting the failure but not the absence of side effects.
5. One green run standing in for a matrix of platforms and upgrade paths.
6. Absolute performance thresholds measured on a loaded machine, then
   "fixed" by retrying until green.
7. A documented claim (README, contract, compliance map) with no test in
   the traceability map.
8. Writing the positive test first and shipping before the negative exists.

## Suite shape that works

- One suite per claim family, each check named for the claim it kills
  (`F8a: withdrawal over balance under concurrent retry rejected, no
  partial write`).
- Fresh-install block, migration block, and a known-secrets convention for
  fixtures (never real credentials in the suite).
- Suits that branch honestly on platform semantics instead of skipping.
- The claim list is the index: for every claim, the name of its negative
  test. If the index has a hole, the coverage has a hole.
