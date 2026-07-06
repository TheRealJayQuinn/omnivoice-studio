# Conventions Checklist

Mechanical rules. Match the codebase, don't improve it in passing. Full derivations live in `.planning/codebase/CONVENTIONS.md` and `ARCHITECTURE.md`; this is the working subset that gets violated most.

## Layering (backend)

- Routers (`backend/api/routers/`) are thin: validate, delegate to services, shape the response. Services (`backend/services/`) own ML/pipeline logic. Core (`backend/core/`) has no ML deps.
- **Never import `api.*` from `services.*` or `core.*`** — routers already import downward; going upward creates cycles. Services communicate upward via return values or `core.event_bus.emit(...)`.
- **Never run PyTorch/heavy work inline in an `async def` route.** Wrap it: `await loop.run_in_executor(_gpu_pool, fn, *args)` (see `backend/api/routers/generation.py`). The `_gpu_pool` is single-worker on purpose — it serializes GPU access; don't widen it.
- Never write to the project root at runtime; all user data goes under `backend/core/config.py:get_app_data_dir()`.

## Frontend

- **One Zustand store.** New state = new slice file in `frontend/src/store/`, spread into `frontend/src/store/index.ts`. Never `create()` a sibling store — it fragments persisted state across localStorage keys.
- Backend URL comes from the central api-base module / `frontend/src/api/client.ts`. Never hardcode `localhost:3900` in a component (this exact hardcode broke Docker LAN mode, issue #80).
- Errors: `apiFetch` throws `ApiError`; user-visible errors via `react-hot-toast`.

## i18n (hard rule, CI-enforced)

- No hardcoded CJK user-facing text outside `frontend/src/i18n/`. UI strings go through `t('...')` keys in `locales/*.json`; native language names live in `i18n/index.ts` `LANGUAGES`.
- Functional CJK (regexes, engine vocab, localized error matching, fixtures) is allowed only via the allowlist in `tests/test_no_hardcoded_cjk.py::_ALLOWED_FILES` — extend it with a one-line justification, in the same commit.
- Adding any English UI string? Add its key to every `locales/*.json`, not just `en.json` — the i18n coverage gap (issues #230/#232) came from partial additions.

## Python style

- `from __future__ import annotations` at the top of every new typed file.
- Logging: module-level `logger = logging.getLogger("omnivoice.<domain>")`; lazy `%s` formatting (`logger.warning("job %s failed: %s", job_id, e)`), never f-strings in log calls (f-strings evaluate even when the level is off, and they bypass redaction-friendly arg handling).
- Optional heavy deps are imported lazily inside functions with `try/except ImportError` (+ `# noqa: F401` where needed) so `is_available()` stays cheap and a missing engine doesn't break import of the whole module.
- Exceptions: services raise `ValueError`/`RuntimeError`; routers convert to `HTTPException(status_code=..., detail=...)` with `from e` chaining. Broad `except Exception` only in entry-point/recovery paths, tagged `# noqa: BLE001`.
- Section dividers: `# ── Section Name ─────────`. Pinned-dependency rationale is documented inline in `pyproject.toml` — if you pin, say why on the same line.
- Naming: `snake_case` functions, `_`-prefixed private helpers and module state, `SCREAMING_SNAKE_CASE` module constants, `PascalCase` classes.

## Versioning discipline

- Everything ships on the single open version line. Never label work with a future version, suggest an RC, or defer "to the next release" — unless the user explicitly asks. This applies to code comments, PR text, and issue replies.

## Worked example (functional CJK done right)

The dubbing pipeline needs Chinese punctuation regexes and CosyVoice speaker identifiers — legitimate functional CJK. Instead of an exemption comment at each site (unenforceable) or moving the strings to i18n (wrong — they aren't UI text), the rule is enforced by `tests/test_no_hardcoded_cjk.py`: it scans the whole repo for CJK codepoints and fails CI unless the file is in `_ALLOWED_FILES`, where each entry carries a one-line justification. When you add a text-processing regex with CJK, the mechanical procedure is: write the code → run that test → watch it fail → add the file + justification to `_ALLOWED_FILES` → test green. The allowlist turns a style judgment into a diff a reviewer can see.

## Failure signs

- An import of `api.` inside `backend/services/` or `backend/core/`.
- `torch.` anything directly inside a route handler.
- A new `create(` store call in the frontend outside `store/index.ts` composition.
- An f-string inside a `logger.*()` call.
- A user-visible string in JSX that isn't a `t('...')` call.
