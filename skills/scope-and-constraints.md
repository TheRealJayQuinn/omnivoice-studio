# Scope and Constraints — Deciding What Is In

Before touching anything, decide what the change is allowed to be. In this project scope is not a feeling; it is a short list of written constraints, and almost every bad PR here violated one of them.

## The procedure

1. **Re-read the Constraints section of `CLAUDE.md`** (and `.planning/PROJECT.md` if it exists). Do this per task, not per week. The constraints that bind most work:
   - Existing installed engines must keep working without reinstall (on-disk model state is sacred).
   - Cross-platform parity: default features behave identically on macOS/Windows/Linux; platform-only behavior must be behind explicit opt-in. A default that fails on one platform is a P0.
   - `omnivoice_data/` keeps working without manual migration; schema changes go through alembic with a tested upgrade path.
   - Local-first: no required cloud calls, accounts, or API keys; bug reporting is opt-in and GitHub-Issues-only.
   - Everything ships on the single open version line. Never propose deferring to a future version, never suggest RCs or release ceremony. Zero unprompted version chatter.
2. **Map the request to a requirement or issue number.** If it doesn't map, it's a scope addition — name it as one explicitly (the Phase 1 plans record "#76 .deb ffprobe" and "#80 Docker LAN" as *accepted scope additions*, not silent creep).
3. **Write the out-of-scope list next to the in-scope list.** Plans in `.planning/phases/` state what a plan does NOT own (e.g., Plan 01-01's summary: "This plan does **NOT** create `backend/core/links.py`. That file is owned by Plan 01-02."). Boundary statements prevent two changes from colliding.
4. **When you discover mid-task work that isn't yours,** don't do it silently and don't drop it silently. Record it (a deferred-items note, a TODO in the summary, or a follow-up issue) and continue.

## Rules I was following

- **A documented workaround can count as "closed" — but only by explicit decision.** Issues #54 (macOS Gatekeeper) and #56 (AppImage white screen) were decided to count as closed if documented + surfaced in error UI, because the real fixes (signing cert, upstream Tauri bug) are out of this project's control. That decision was written into STATE.md Key Decisions BEFORE the work, not invented at close time. Never grant yourself this exemption ad hoc.
- **Scope additions are cheap to accept and expensive to smuggle.** Accepting #76/#80 into Phase 1 cost one line in the plan. Smuggling them in would have made the phase's exit gate unauditable.
- **"Empty the inbox" is a Goodhart trap.** Closing an issue without a shipped fix, a repro test, or an explicit won't-do rationale is metric gaming. A close needs one of those three.
- **Decline kindly and in writing.** Out-of-scope community requests get a labeled decline with a reason, not silence.

## Worked example (cryptography dependency, Plan 01-01)

Mid-execution, the plan's assumption "cryptography arrives transitively via pyannote/huggingface_hub" proved false (`uv pip list | grep cryptography` was empty). Options: (a) silently add the dep, (b) stop and ask, (c) add it AND record the deviation. The plan itself had a fallback clause for exactly this, so the executor took (c): added `cryptography>=41` to `pyproject.toml`, ran `uv lock --upgrade-package cryptography && uv sync`, and wrote it up under "Deviations from Plan → Auto-fixed Issues" in the summary with the trigger, the evidence, and the commit. The deviation record is what keeps a scope change from becoming scope creep: anyone auditing later sees exactly when and why the boundary moved.

## Failure signs

- You are adding a feature while fixing a bug ("while I'm here...").
- Your change mentions a future version, an RC, or "defer to next release."
- You closed an issue and cannot point to a shipped commit, a repro test, or a written won't-do.
- A default-mode feature works on your platform and you haven't checked the other two.
