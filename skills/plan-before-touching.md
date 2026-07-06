# Plan Before Touching Anything

Never open an editor on a feeling. Every non-trivial change in this repo starts as a written plan with tasks, file lists, and a machine-checkable exit gate. The plan is cheap; un-planning a half-done change across 22 routers is not.

## The procedure

1. **Locate yourself.** Read `CLAUDE.md`, then `.planning/STATE.md` (current position, key decisions, open TODOs), then the relevant phase's PLAN/SUMMARY files under `.planning/phases/`. Prior summaries tell you what already exists so you don't rebuild it — e.g., Plan 01-01's summary explicitly enumerates which subprocess launch sites were already patched "so Plan 02 doesn't re-patch."
2. **Read the code you'll change before deciding how to change it.** Find the actual line numbers. Plans here cite `dub_core.py:540`, `model_manager.py:480` — not "the dubbing code." If you can't name file:line, you haven't read enough.
3. **Write the plan as tasks, each with:**
   - files created / files modified (explicit paths),
   - the verification gate (a command whose output proves the task done),
   - assumptions, each marked with how it will be checked at execute time.
4. **Resolve open questions before executing, and write the resolution down.** STATE.md records resolutions like "localhost hardcodes: `frontend/src/utils/media.js:20` confirmed as only site" — confirmed by grep, not assumed.
5. **Sequence by dependency, not preference.** Phase 2 (SubprocessBackend) had to precede Phases 3 and 4 because three new engines plug into that primitive. Ask: does anything I'm building become the substrate for something else? Build the substrate first.
6. **After execution, write the summary** (what landed, deviations, known stubs, self-check). The summary is the next person's step 1.

## Rules I was following

- **Assumptions are liabilities with names.** Every plan assumption gets verified at execute time. Plan 01-01's Assumption A1 ("cryptography arrives transitively") was checked with `uv pip list` as the first step of the task that depended on it — and it was false. Checking assumptions first means the failure costs minutes, not a rewrite.
- **A plan without an exit gate is a wish.** "Patch the token read sites" is a wish; "this grep returns zero matches" is a plan.
- **Plans state their boundary.** Say what you will NOT do, by file name, so parallel work doesn't collide.
- **Match plan weight to task weight.** A one-line fix gets a one-paragraph plan (repro, fix, test, gate). A cross-cutting change gets waves. Don't skip planning because the task is small; shrink the plan instead.

## Worked example (Phase 1 wave structure)

Phase 1 covered 17 requirements. Instead of one giant change, it was planned as three waves with explicit data dependencies:

- **Wave 1:** token resolver + encrypted settings store + log redactor + patch 5 read sites (pure backend).
- **Wave 2:** docs + Settings UI panel — *depends on Wave 1's* `GET /system/hf-token/state` endpoint, so it could not sensibly be written first.
- **Wave 3:** installer/bundler fixes (AppImage launcher, .deb ffprobe, apiBase.ts) — independent of both, batched last because it needs platform testing.

The wave boundaries were drawn along the dependency edge (UI consumes resolver endpoint), which meant Wave 1 could ship and be verified alone. When Wave 1 landed with 35 green tests, Wave 2 started against a real endpoint instead of a mock.

## Failure signs

- You are editing a file you have not read top-to-bottom (or at least the whole function and its callers).
- You cannot state the verification command for the task you're doing.
- Your plan's step 3 depends on an assumption you could check right now with one command, and you haven't.
- You're building a consumer before its producer exists.
