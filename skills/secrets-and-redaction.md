# Secrets and Redaction

This app holds exactly one credential class (HF tokens) and makes one promise (local-first). A single token or home-path leak into a log, screenshot, or public GitHub issue breaks the promise permanently. Every rule below exists because the leak vectors are real: HF tokens historically appear in 401 error URLs, tracebacks capture locals, and users paste logs into Discord.

## The procedure

1. **Never read a secret ad hoc.** All HF token access goes through `backend/services/token_resolver.py` (cascade: App settings → env var → HF CLI file). Adding `os.environ.get("HF_TOKEN")` anywhere else is a regression — there is a grep gate against it, and `tests/backend/test_engine_spawn_token.py` carries a source-level guard.
2. **Storage is encrypted at rest.** App-entered tokens live in the SQLite `settings` table via `backend/services/settings_store.py` (Fernet, per-install scrypt-derived key in `_secret_key.py`). Never write a secret to prefs.json, logs, or any plaintext file.
3. **Redact at the logging chokepoint, not at call sites.** `backend/core/logging_filter.py:HFTokenRedactor` is installed on the root logger and every handler at startup; it masks both `record.msg` and `record.args`. If you add a new handler, install the filter on it too.
4. **Subprocesses get secrets via env injection at spawn, scoped to need.** Canonical pattern: `sonitranslate.py`'s `start()` injects `HF_TOKEN` (and the child's expected alias) into the child env. Launchers with no HF need (ffmpeg, `open`/`explorer`/`xdg-open`) get nothing. Never inject into a spawn "just in case."
5. **Anything leaving the machine is default-deny.** The bug reporter builds its payload from an explicit allow-list of fields (os, version, gpu, engine id, error class, first line of error). Unknown fields are dropped at the boundary, not "redacted." Home paths are rewritten (`/Users/<name>/` → `~/`). The user sees the literal payload before it's sent.
6. **Endpoints that touch secrets are loopback-gated.** Token CRUD routes use `Depends(require_loopback)`; a LAN client can use the app but cannot read or set credentials.
7. **Docs never show a realistic secret.** Placeholder tokens are obviously fake but shape-correct: `hf_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`.

## Rules I was following

- **Precision in redaction regexes is a usability decision.** The redactor matches `hf_[A-Za-z0-9]{30,}` — 30+ chars — deliberately, so short literals like `hf_hub` and `hf_token` in debug messages stay readable while real tokens (36–40 chars) are masked. An over-broad regex would have made logs useless and trained people to bypass the filter.
- **Validate tokens proactively, cache the result, invalidate on 401.** The resolver runs a whoami check with a 300s cache and an `on_401` callback so a token that expires mid-dub-job falls through to the next source instead of failing the batch. Never whoami on every call (50-segment dub = 50 round trips).
- **"We can scrub it on the server" is never acceptable** — there is no server. Redaction happens before serialization, before render, before network.
- **A secrets change ships with leak tests.** Plan 01-01's tests include a plaintext-leakage check on the settings DB file and redactor tests for msg/args/multi-token/short-token cases. If your change touches a secret path and adds no such test, it's not done.

## Worked example (the read-side fix for issue #35)

The original bug: `dub_core.py:540` read `os.environ.get("HF_TOKEN")` directly, so a token saved any other way was invisible to dubbing — and the error message blamed the user. The fix wasn't "patch line 540"; it was: build the resolver chokepoint, patch all 5 read sites onto it (`dub_core.py`, `system.py`, `model_manager.py`, `sonitranslate.py` ×2), install the redactor so the token can't leak into logs while flowing through more code, gate the new CRUD endpoints to loopback, and land the grep gate so a 6th bare read can't appear. Secret handling is only safe when there is exactly one door.

## Failure signs

- `HF_TOKEN` appearing in a grep of your diff outside `token_resolver.py` / its tests.
- A new `logging.FileHandler` or stream handler without the redactor filter.
- A payload builder using `to_dict()` / `model_dump()` on an object not authored for export.
- An f-string log line containing a variable that could ever hold a token or home path.
