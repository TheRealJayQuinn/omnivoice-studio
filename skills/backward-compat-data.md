# Backward-Compatible User Data

`omnivoice_data/` (SQLite DB, voice references, projects, prefs) belongs to the user, predates your change, and must keep working with zero manual migration. Treat every schema or layout change as an upgrade path you must prove, in both directions of install history.

## The procedure

1. **Two install histories must converge on identical state:**
   - *Fresh install:* `init_db()` creates current schema from `_BASE_SCHEMA` in `backend/core/db.py`.
   - *Upgrade:* an existing older DB gets there via alembic migrations in `backend/migrations/versions/`.
   Any schema change edits BOTH: add to `_BASE_SCHEMA` (as `CREATE ... IF NOT EXISTS`) AND write a migration.
2. **Make the migration idempotent.** It will run against fresh DBs where `_BASE_SCHEMA` already created the table. Check `sqlite_master` before `create_table`. Neither path may error on the other's output.
3. **Test against the real fixture, not a synthetic DB.** `tests/fixtures/omnivoice_data/` holds a realistic prior-version user state (DB + voices). The migration test loads it, runs `alembic upgrade head`, and asserts (a) new schema present, (b) **existing rows still readable** — the second assertion is the one that catches destructive migrations.
4. **Point migrations at the test DB, never the real one.** `backend/migrations/env.py` only falls back to the production `DB_PATH` when the caller hasn't set `sqlalchemy.url` (see `skills/failure-patterns.md` #4). If you write a migration test, pass the fixture URL explicitly and assert you're on the tmp path.
5. **File-layout changes get the same treatment:** old layout must load, or be migrated automatically on startup with a log line. "User must move their folder" is not an option.
6. **Engine on-disk state is user data too.** Installed engine weights/venvs must survive your change without reinstall. If an engine's expected path or format changes, code reads the old location first.

## Rules I was following

- **Downgrade paths matter for a beta.** Users follow `main` for previews and may step back. Migrations get a working `downgrade()`, and the tests exercise upgrade AND downgrade on the fixture.
- **`IF NOT EXISTS` is the convergence trick, not laziness.** Dual-path schema (base schema + idempotent migration) is deliberate: it means no code ever needs to ask "which path created this DB?"
- **Never regenerate the fixture to make a test pass.** The fixture represents shipped reality. If your migration can't handle it, your migration is wrong.
- **WAL-mode SQLite, no ORM.** Raw `sqlite3` via the context-managed `db_conn()` helper. Don't introduce SQLAlchemy models for app queries; alembic is used for migrations only.

## Worked example (the `settings` table, migration 0001, Plan 01-01)

The encrypted settings store needed a new `settings` table. What landed:

- `_BASE_SCHEMA` in `backend/core/db.py` gained `CREATE TABLE IF NOT EXISTS settings (...)` — fresh installs get it instantly.
- `backend/migrations/versions/0001_phase1_settings_table.py` creates the same table, but first checks `sqlite_master` so it's a no-op on DBs where the base schema already ran.
- `init_db()` runs alembic upgrade as part of startup, so prior-version users converge on the same schema on next launch with no manual step.
- Test `test_alembic_upgrade_on_v027_db_preserves_existing_tables` runs the migration against the prior-version fixture DB and asserts the pre-existing tables and rows survive. Writing that test is also what exposed the env.py URL-clobbering bug — the test would otherwise have migrated the developer's real data.

Fresh install, upgrade, and downgrade all converge; nobody is asked to delete a folder.

## Failure signs

- A migration with `create_table` and no existence check.
- A schema change in `_BASE_SCHEMA` with no accompanying migration file (or vice versa).
- A test that builds its own empty DB instead of loading `tests/fixtures/omnivoice_data/`.
- Any instruction to users containing "delete your omnivoice_data folder."
