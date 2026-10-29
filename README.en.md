# negative-test-first

An agent skill for testing discipline in code whose job is to refuse:
access control, validation, audit, lockouts, rate limits — everything where
"the feature works" proves nothing and only "the bypass fails" does.

The skill grew out of practice: building a PostgreSQL access-control
extension where every protection mechanism must carry a negative test, and
every escaped defect becomes a named lesson that lives in code comments and
in the tests. Not textbook theory — a checklist of what actually broke and
how it got caught.

## The core idea

A requirement is covered only when a test shows that **bypassing it does not
work**. A suite that checks only happy paths proves nothing about
protection. For every claim like "only role X may write table T" there must
be a test that tried to violate it from role Y, from a role-less login, and
from no session at all — and failed.

## What's inside

Eight patterns and eight anti-patterns, briefly:

* every positive claim pairs with a negative test;
* after an expected refusal, assert the world did not change: nothing was
  created, granted, or counted;
* refusals are classified by **who** refused — your mechanism or the
  platform; "caught some exception" does not count;
* platform branching is honest: if another build's platform catches the
  bypass earlier, the test says so ("our guard is defense in depth") instead
  of reporting a green pass for nothing;
* refusal evidence is checked in two channels: what dies with the
  transaction and what survives it (the server log);
* negative tests rerun after a migration on live data;
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
for skills. It activates when the task involves writing tests for
permissions or validation code, reviewing negative-case coverage,
classifying refusals, or setting performance baselines.

## Why agents need this

Agents write positive tests well and negative tests poorly: they check that
code does what was asked and almost never that code refuses what was not.
On security-bearing code this bias is most expensive: a green suite looks
like "protection works" while not a single bypass was even attempted. The
skill forces the question "which test tried to violate this rule?" before a
feature is called done.

## License

MIT — see [LICENSE](LICENSE).
