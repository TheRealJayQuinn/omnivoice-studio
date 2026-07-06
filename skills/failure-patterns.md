# Failure Patterns That Cost Real Time in This Project

Each of these burned hours at least once. Check the list before starting and when something behaves impossibly.

## 1. The fix-regression cycle (the central one)

A fix for issue #N breaks something that worked, because the touched path is shared with installed engines, on-disk model state, or one platform's quirks. Defense: re-run the pre-existing gates (smoke tests, fixture DB load) after every change, not just your new tests. If a report says "this used to work in the previous version" within days of your merge, assume your change until proven otherwise, and revert just the offending commit — never the whole batch.

## 2. Python import-state pollution in tests (bit us, will bite you)

`sys.modules.pop("core.config")` is NOT enough to reset config between tests: the parent `core` package object keeps an attribute pointing at the cached submodule, so `from core.config import DB_PATH` still resolves to the stale value. First test passes, second test silently uses the wrong DB path. **Rule:** purge every key where `mod == "core" or mod.startswith("core.")` (same for `services`, `api`). The canonical fixture is in `tests/backend/services/test_settings_store.py` — copy it, don't reinvent it.

## 3. Tooling writes landing outside the worktree

When working in a git worktree, a relative path like `pyproject.toml` can resolve against the wrong checkout. This happened during Plan 01-01: an edit landed in the main repo instead of the worktree and was only caught by `git status`. **Rule:** in any worktree session, use absolute paths for every file operation, and run `git status` in BOTH checkouts before declaring done.

## 4. Alembic env.py clobbering the caller's database URL

`backend/migrations/env.py` used to unconditionally set `sqlalchemy.url` to the production `DB_PATH` — which meant a test passing a fixture DB would silently run migrations against the developer's real `omnivoice_data/`. Now guarded with `if not config.get_main_option("sqlalchemy.url")`. **Rule:** any code path that resolves "the" data directory must be overridable by tests, and the test must verify it's hitting the override (assert on the tmp path, not just on success).

## 5. Bare `pytest` walking into `research/`

`research/` holds ~1.2 GB of vendored upstream projects whose `test_*.py` files call `sys.exit` at module import — collecting them INTERNALERRORs the whole run. `pyproject.toml` pins `testpaths = ["tests"]` and `norecursedirs`. **Rule:** run tests via `uv run pytest tests/...`; if you add a vendored tree, add it to `norecursedirs` in the same commit.

## 6. Cross-platform test coverage theater

`cargo check` on three OSes catches type errors only; the Python backend's runtime, the venv bootstrap, and engine imports run in CI on Linux only. Green CI does not mean "works on Windows." **Rule:** see `skills/cross-platform-parity.md` — platform claims require platform reasoning, stated explicitly in the commit.

## 7. `is_available()` probes hiding real errors

Engine availability probes return False on ANY import failure, so a dependency conflict shows up as "engine not available" instead of an error. If an engine mysteriously disappears from the picker after a dependency change, the cause is almost never the engine's own code — diff `uv.lock` and try the import by hand: `uv run python -c "import <engine module>"`.

## 8. The blanket workaround tax

`WEBKIT_DISABLE_COMPOSITING_MODE=1` fixes the AppImage white screen on NVIDIA but forces software rendering on everyone if applied unconditionally. **Rule:** platform workarounds are applied on detection of the triggering condition (e.g., NVIDIA driver present), surfaced in diagnostics UI, and left with a tracking note for removal when upstream fixes ship. Never set-and-forget a perf-degrading env var.

## 9. Mid-job auth/state changes

Long pipelines (50-segment dubs) hold state for minutes. A token that validated at job start can 401 mid-job; a resolver that only reads once will fail the whole batch. The token resolver has an `on_401` invalidation callback for exactly this. **Rule:** anything resolved at job start that can expire needs a mid-job re-resolve path, and a test simulating the mid-job failure.

## 10. Assumption rot between plan time and execute time

Plan 01-01 assumed `cryptography` was already installed transitively; at execute time it wasn't. The plan survived because the assumption was written down with a check command and a fallback clause. **Rule:** every "X should already be true" in a plan becomes step 1 of the task that needs it: verify X, with a command.

## Worked example (pattern 2 in the wild)

During Plan 01-01, the settings-store test passed alone but failed when run after the resolver test in the same session — `DB_PATH` pointed at the first test's tmp dir. Diagnosis took the shape: impossible behavior → suspect shared state → inspect `sys.modules` and find `core` still holding the old `core.config`. Fix: widen the purge to the whole package family, document it in the fixture docstring, and reuse that fixture in the other two test files rather than copying the broken half-purge. The docstring is why the next person doesn't lose the same hour.
