# Acme Examples — Good and Bad Proposals

Generic illustrations for an imaginary **Acme** project. Not copy-paste
boilerplate — they show evidence shape and brevity.

## Good

**Correctness fix** — Invoice totals double-count tax in `billing/total.py`
when `include_tax` is set; test `test_total_include_tax` expects a single add.
Fix the branch; keep the public API.

**Useful test** — `Scheduler.next_slot` has no coverage for empty calendars;
callers already handle `None`. Add one focused test proving the existing
contract.

**Dead code / small refactor** — Private helper `_legacy_format` has no
callers (repo-wide search). Delete it; no behavior change.

## Bad

**Weak evidence** — “Clean up naming in utils” with no bug, no caller pain, no
failing test. Cosmetic churn fails the value floor.

**Scope creep** — “Fix null check in parser” plus reformatting three unrelated
packages and a new config framework. Not one coherent improvement.
