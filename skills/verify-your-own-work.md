# Verify Your Own Work

The single highest-leverage discipline in this repo. A change is not done when the code compiles or when you believe it is correct. It is done when a machine-checkable gate says it is correct AND the pre-existing gates still pass.

## The verification ladder

Run every rung that applies, in this order. Stop and fix at the first failure.

1. **Grep gate** — if the change is "replace pattern X everywhere," prove it with a grep that must return zero matches. Write the grep BEFORE you start editing, so you can't unconsciously weaken it afterward.
2. **New targeted tests** — every behavioral change gets a test that would have failed before the change (RED before GREEN, even if you commit them together).
3. **The module's existing tests** — `uv run pytest tests/test_<area>*.py -x`.
4. **Full suite** — `uv run pytest tests/`. Never run bare `pytest` from an unusual cwd: pytest is configured with `testpaths = ["tests"]` because `research/` contains 1.2 GB of vendored projects whose test files call `sys.exit` at module import and will INTERNALERROR the run.
5. **Smoke tests** — `uv run pytest tests/smoke/` (boot smoke) and, before anything release-shaped, `bun run smoke-test:quick`. These catch "my unit tests pass but the app no longer boots."
6. **Regression fixture** — if you touched the DB schema, migrations, or `omnivoice_data/` layout, run the tests that load `tests/fixtures/omnivoice_data/` (a real v0.2.7-style user state: 1 voice, 1 DB, 1 prefs row). Backward compatibility is a hard constraint, not a nice-to-have.
7. **Cross-platform reasoning** — CI runs the Python backend on Linux only. See `skills/cross-platform-parity.md`; you must verify macOS/Windows behavior by reading, not by hoping.

## Rules I was following (write these down because they are invisible)

- **A verification gate is part of the plan, not an afterthought.** Every plan in `.planning/phases/*/` has an explicit exit gate (a grep, a test count, a file-exists check). If you can't state the gate in one command, you don't understand the change yet.
- **"Tests pass" is necessary, never sufficient.** Ask: which rung of the ladder would have caught the bug this change fixes? If none, add that rung (a new test) before closing.
- **End with a self-check list.** Every execution summary in this repo ends with a literal checklist of claims ("file X exists and contains `def resolve`", "grep gate exits 1", "35/35 green", "Phase 0 smoke 4/4 still green") that were re-verified at the end, not remembered from earlier.
- **Never assert something you didn't re-run.** If you ran the suite before your last edit, it doesn't count. Re-run.

## Worked example (Phase 1, Plan 01-01 — HF token read-side fix, issue #35)

The task: replace every bare `os.environ.get("HF_TOKEN")` with the new 3-source resolver. The plan pre-declared this gate:

```
grep -RnE "os\.(environ|getenv).*HF_TOKEN" backend/ --include='*.py' \
  | grep -v token_resolver.py | grep -v '^[[:space:]]*#'
```

Exit code 1 (zero matches) = done. The summary (`.planning/phases/01-.../01-01-SUMMARY.md`) records: 5 read sites patched, the grep gate output, 35 new tests green, AND "Phase 0 smoke tests: 4/4 still green (no regression)". The pre-existing gates being re-run is what made the claim "done" trustworthy — the fix-regression cycle (see `skills/failure-patterns.md`) is this project's central failure mode, and re-running old gates is the only mechanical defense.

## Failure signs

- You are about to write "should work" or "this fixes it" without a command output to point at.
- Your test only exercises the new code path, not the old one you might have broken.
- You changed `backend/core/db.py` or a migration and did not run anything against the v0.2.7 fixture DB.
