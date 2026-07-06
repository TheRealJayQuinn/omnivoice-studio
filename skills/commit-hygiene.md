# Commit Hygiene — The History Is a Debugging Tool

Commit messages here are written for the person who, six months from now, finds the commit via `git log -S` while debugging a regression. That person needs the causal chain, not a summary of the diff (they can read the diff). Most of this project's hardest bugs were solved by finding a prior commit whose message explained a mechanism.

## The template (from this repo's actual practice)

```
<type>(<area>): <symptom-level summary> (closes #N) (#PR)

<Paragraph 1 — the trigger and mechanism: what changed in the world,
 why it breaks, the EXACT error string in quotes.>

<Paragraph 2 — why THIS fix: what prior art it reuses, what it
 deliberately does not do, what fallback is preserved.>

Tests: <test file(s) and what they assert>
Cross-platform: <verified / reasoned, and on what>
```

Types in use: `fix`, `feat`, `test`, `chore(release)`, `docs`. Area is the subsystem (`dub`, `asr`, `diarization`, `bootstrap`, `i18n`, `ci`).

## The rules

1. **Quote the exact error string.** `git log --grep "Unsupported global"` must find your commit when the bug recurs in a new place.
2. **State the mechanism, not just the action.** "PyTorch 2.6 flipped torch.load's default to weights_only=True and its secure unpickler rejects the checkpoint's metadata globals" — that sentence lets a future reader decide if their new bug is the same class without reading any code.
3. **Include the reporter's context** when a user report drove the fix ("reported on v0.3.4, RTX 4070 Ti, license accepted") — it defines the repro environment forever.
4. **Name the tests and the platform claim.** "Tests: tests/test_diarization_weights_only.py (allowlist runs before load)" and an honest cross-platform line ("affects all platforms" vs "Linux-verified").
5. **One task, one commit.** Plan tasks land as separate commits — Plan 01-01's summary records its 3 tasks landing as 3 commits — so a regression bisects to a task, not to a week of work. (Note: worktree commit SHAs recorded in `.planning/` summaries may not resolve in this repo's history after squash-merge; the discipline is per-task commits, not those literal SHAs.) Never mix a mechanical change (rename, format) with a behavioral one.
6. **Reference the issue AND let the fix stand alone.** `(closes #270)` links the discussion, but the message itself must contain enough that the fix is understandable if GitHub vanished.
7. **Deviations from plan are visible in commits too** — if a commit does something the plan didn't call for (adding a dep, fixing infra), the message says so, matching the summary's Deviations section.

## Worked example (commit f7d34a1, dissected)

```
fix(diarization): register torch safe-globals before pyannote load (#270) (#271)
```
Then the body, in order: (1) the exact failing call and quoted error — `Pipeline.from_pretrained("pyannote/speaker-diarization-3.1")` fails with "Weights only load failed ... Unsupported global: GLOBAL torch.torch_version.TorchVersion"; (2) the mechanism — PyTorch 2.6 flipped the `weights_only` default; (3) the disambiguator — "even when the license IS accepted", killing the obvious wrong diagnosis a future reader would reach for; (4) the reporter's environment; (5) the prior art reused — the WhisperX VAD path "already solved this via `_allow_vad_pickle_globals()`... `get_diarization_pipeline` just never called it", telling the reader this is a reuse, not new mechanism, and where the canonical allowlist lives; (6) what's preserved — "Graceful fallback (silence-gap heuristic) is preserved"; (7) tests named; (8) the platform claim with its reasoning. Every sentence answers a question a debugger will actually ask.

## Failure signs

- A message that describes the diff ("add try/except around load") instead of the cause.
- No error string anywhere in the body of a `fix` commit.
- A commit touching two unrelated concerns because "they were both small."
- "Fix bug" / "address review comments" as the entire message.
