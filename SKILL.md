---
name: negative-test-first
description: >
  Testing discipline for security-critical and behavior-critical code: a claim
  is covered only when the bypass is shown to fail. Use when writing tests for
  auth, RBAC, permissions, validation, or audit code; reviewing whether a test
  suite actually proves anything; deciding whether a security feature is done;
  classifying test failures and refusals; or measuring performance baselines
  for enforcement paths.
---

# Negative Test First

Find out whether the refusal actually works, then prove nothing leaked when
it did. This is a source-first, adversarial testing discipline for code whose
whole point is to refuse.

## The core rule

A requirement is covered only when a test shows that **bypassing it does not
work**. A test that only asserts the happy path proves nothing about
protection. For every security or invariant claim, ask: "which test attempts
the violation?" If the answer is none, the claim is uncovered — no matter how
much green the suite shows.

This inverts the usual question. Not "does my feature work?" but "can
anything that should be refused get through?" Write the negative first; the
positive is the cheap half.

## Patterns

### 1. Every claim pairs with its negative

| Positive claim | Required negative |
| --- | --- |
| Role X can write table T | Role Y, role-less login, and X's neighbors cannot read or write T |
| Guard rejects operation Z | Z attempted through every alternative path (different entry points, prepared statements, nested calls) is also rejected |
| Audit log records changes | Nobody can modify or delete old records; writes go through exactly one channel |
| Lockout after N failures | Lockout clears only by the intended unlock; unlock itself is permission-gated |
| Input validation rejects bad input | Each rejected class is enumerated and individually tested; "invalid" is not one bucket |

### 2. After a refusal, verify the world did not change

A refusal that leaves half-applied state is a bug wearing a test's clothes.
After every expected rejection, assert the side effects:

- the object was not created (`SELECT count(*) ... WHERE name IN (...) = 0`);
- the grant did not happen (`pg_has_role(...)` is false);
- the counter did not move, the file did not appear, the row count is unchanged.

The assertion of absence is part of the negative test, not optional cleanup.

### 3. Classify who refused

When an operation is expected to fail, the failure detail tells you which
layer did it: your mechanism, the platform, the driver, or a typo. Capture
both facts:

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
failed" and "our guard refused with reason R" are different claims. If the
platform refuses before your mechanism runs, report that honestly — the
mechanism is then defense in depth, and the test says so in its name, not
in a comment.

### 4. Honest branching beats false green

Supported platforms differ. When the same negative can be caught by the
platform on one build and by your code on another, branch explicitly:

```python
if "RBAC_ERROR" in detail:
    report("guard rejected the grant", True)
elif "admin option" in detail:
    report("platform blocks the grant (guard is defense in depth)", True)
else:
    report("guard rejected the grant", False, detail)
```

Both branches are passes, but each names what actually protected. A test
suite that cannot tell you who refused will hide the day the platform
stops doing it.

### 5. Two evidence channels

Records written inside a failing transaction die with the rollback. If
reliable evidence of a refusal matters, keep a second channel that survives
it (server log, syslog, stderr marker) and test both: the in-transaction
record when the caller catches the refusal, and the survive-channel when it
does not. On PostgreSQL this is the difference between a table row and
`RAISE LOG` visible via `client_min_messages = log`.

### 6. Negative tests survive migrations

Live data, then upgrade, then run the negatives again. A migration that
silently widens permissions is worse than a bug — it is a regression with a
green release note. An M-series (migration) block rerunning the negative
checks on migrated data is part of coverage, not a nicety.

### 7. Performance claims need a baseline, not a threshold

Absolute latency limits are valid only in proven-quiet conditions
(load average below your agreed bound, no neighbor builds). Prefer:

- A/B alternating runs (old vs new, repeated, interleaved) to attribute a
  regression to code rather than to the machine;
- medians of several server-side measurements, not single client-side shots;
- thresholds derived from the A/B baseline with headroom, and stated
  conditions of validity ("F1 is valid only at load < 3") next to the
  threshold in the test itself.

A suite that fails under load on every machine is worse than no suite:
it trains everyone to ignore red.

### 8. Count, name, gate

- Report exact counts: "57 OK, 0 FAIL", and list the failures by name.
- Gate releases: a release commit waits until the full matrix is green on
  every supported platform. If you must move before that, record exactly
  what passed where — partial truth, not implied completeness.
- When a defect escapes into the wild, distill it into a named lesson
  ("lesson I-12: in-transaction log rows die with the rollback") and put
  that name in the code comment and the test that now covers it. Lessons
  compound; folklore evaporates.

## Anti-patterns

1. Coverage theater: 100% line coverage of a guard function, zero tests of
   the paths that must be refused.
2. `assert raises Exception` without layer, error code, or reason.
3. Treating a platform's refusal as proof your mechanism works.
4. Asserting the refusal but not the absence of side effects.
5. One green run standing in for a matrix of platforms and migration paths.
6. Absolute performance thresholds measured on a loaded machine, then
   "fixed" by retrying until green.
7. A security claim in documentation with no test in the traceability map.
8. Writing the positive test first and shipping before the negative exists.

## Suite shape that works

- One suite per claim family, each check named for the claim it kills
  (`F8a: GRANT rbac_admin under rbac_admin rejected, no grant happened`).
- Fresh-install block, migration block, and a known-secrets convention for
  fixtures (never real credentials in the suite).
- A machine-readable traceability map (regulator requirement → mechanism →
  test) when the code answers to auditors; every row needs a living test.
- Suits that branch honestly on platform semantics instead of skipping.
