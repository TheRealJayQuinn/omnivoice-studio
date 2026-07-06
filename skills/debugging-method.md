# Debugging Method

Debugging here is a search problem, not a creativity problem. The bug you're looking at has almost always been solved once already — in this codebase, in a pinned dependency's changelog, or in an upstream issue. Your job is to find the prior art before writing new code.

## The procedure

1. **Get the exact error string.** Not a paraphrase. From the user report, `backend.log`, or a local repro. The exact string is your search key.
2. **Reproduce, or explain precisely why you can't.** If the bug is platform- or GPU-specific and you can't repro, you must instead build the causal chain by reading code (step 4) and state in the fix what you could and couldn't verify.
3. **Search this codebase for the error class first.** `grep -rn "<distinctive fragment>" backend/ tests/`. The many TTS engines, ASR, and diarization share failure modes (model loading, pickling, HF auth, CUDA/MPS fallbacks) — a sibling subsystem has usually already handled yours.
4. **Build the causal chain before touching code.** Write it as: trigger → mechanism → symptom. If you can't fill in "mechanism," keep reading; a fix aimed at the symptom will regress something.
5. **Prefer reusing an existing fix over writing a parallel one.** One chokepoint, two callers — never two copies of the same workaround that will drift.
6. **Preserve the existing graceful fallback.** Most ML paths here have one (silence-gap heuristic when diarization fails, in-process engine when subprocess fails, CPU when GPU OOMs). Your fix must not remove it — the fallback is what keeps a partial failure from becoming a dead app.
7. **Add the regression test keyed to the original repro,** then run the full ladder in `skills/verify-your-own-work.md`.
8. **Write the causal chain into the commit message.** See `skills/commit-hygiene.md`.

## Rules I was following

- **A version bump in a dependency is a suspect by default.** When something that "used to work" breaks, `git log` on `uv.lock` and the dep's changelog before reading your own code. Pinned-dependency rationale lives inline in `pyproject.toml` — read it.
- **"Even when X is correct" reports are gold.** If the user did everything right (license accepted, token set) and it still fails, the bug is in code, not in docs — stop looking for a user error.
- **Fix the class, not the instance.** Issue #35 was one bad `os.environ.get("HF_TOKEN")` call; the fix patched all 5 sites and added a grep gate so the class can't recur.
- **Errors must fail loud, not vanish.** Issue #255: real ASR/model-load failures were being dropped from the stream, so users saw a hang instead of a cause. If your debugging reveals a swallowed exception, surfacing it IS part of the fix.

## Worked example (issue #270 — diarization broken on torch>=2.6, fixed in commit f7d34a1)

- **Exact error:** `Weights only load failed ... Unsupported global: GLOBAL torch.torch_version.TorchVersion` when loading `pyannote/speaker-diarization-3.1`. Reported on v0.3.4, RTX 4070 Ti, HF license accepted — so not a user error.
- **Causal chain:** PyTorch 2.6 flipped `torch.load`'s default to `weights_only=True` (trigger) → its secure unpickler rejects the pyannote checkpoint's metadata globals (mechanism) → diarization load raises even with valid auth (symptom).
- **Prior art search:** grepping for the pickling error class found `WhisperXBackend._allow_vad_pickle_globals()` in `backend/services/asr_backend.py` — the VAD load had already hit the identical torch 2.6 change and already allowlists `TorchVersion`, omegaconf nodes, pyannote metadata, numpy.
- **The fix:** `get_diarization_pipeline` in `backend/services/model_manager.py` calls that same idempotent allowlist before the pyannote load, wrapped in try/except with a `logger.debug` so a missing allowlist degrades instead of crashing. The silence-gap fallback stayed intact. Test: `tests/test_diarization_weights_only.py` asserts the allowlist runs before load. Total new logic: ~6 lines, because the search found the real fix already written.

## Failure signs

- You're writing a fix and haven't grepped the codebase for the error string.
- Your explanation of the bug contains "somehow" or "for some reason."
- Your fix deletes or bypasses a fallback path you don't fully understand.
- The fix works but you can't say why the bug appeared NOW (what changed?).
