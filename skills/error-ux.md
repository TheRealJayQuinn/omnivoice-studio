# Error UX — Every Failure Tells the User What To Do

The core value of this project is "a first-run that actually works — and when something goes wrong, the error or docs tell them exactly what to do." Half the closed issues here were not bugs in features; they were bugs in how failures were reported. Treat error paths as product surface, not plumbing.

## The procedure

1. **Never swallow, never generically wrap.** If a pipeline stage fails, the real exception class and message must reach the user (redacted per `skills/secrets-and-redaction.md`), not "An error occurred" and not a silently-dropped stream. A user who sees the real cause files a good issue; a user who sees a hang files "app frozen" and asks Discord.
2. **Classify known failures.** Recurring diagnosable conditions get a stable error class (e.g., the macOS quarantine probe emits `GATEKEEPER_QUARANTINE`; diarization has its own error class in `tests/test_diarization_error_class.py`). Classes are what the frontend can switch on.
3. **Map class → remediation.** `backend/core/error_docs_map.py` (with a TS half on the frontend) maps error classes to docs deeplinks. A new known failure class isn't done until it has a map entry and the docs anchor exists. The React ErrorBoundary renders the deeplink.
4. **Distinguish "you need to act" from "we degraded gracefully."** If a fallback engaged (CPU instead of GPU, silence-gap instead of diarization), tell the user what quality they got and why — a silent fallback reads as "the feature is bad."
5. **Blame accurately.** Before writing an error message that instructs the user ("set your HF token"), verify the code actually checked all the ways the user could have done it. Issue #35's worst part wasn't the broken read — it was the message telling users to set a token they had already set.
6. **Long jobs report progress with substance:** current stage name, MB/sec or ETA for downloads — never a bare percentage. A stalled "65%" is indistinguishable from a hang.

## Rules I was following

- **Errors are part of the contract, so they get tests.** Failure-path tests (`tests/test_dub_error_transparency.py`, `test_failure_helper.py`, `test_diarization_error_class.py`) assert that specific failures surface with specific classes/messages. When you fix an error-reporting bug, the test asserts on the *message the user sees*, not just the exception type.
- **Every documented workaround is surfaced in-app, in the same wording as the docs.** The Gatekeeper `xattr -cr` guidance appears in the error UI with the pre-populated verified path — if docs and error UI phrase it differently, users can't tell which is the real command (and mismatched phrasing is a phishing opening).
- **Client disconnects are not errors.** The global handler in `backend/main.py` distinguishes them (499) from real failures (500 + crash log). Don't pollute crash logs with users closing tabs.
- **The remediation must be actionable on the user's platform.** "Not available — install with: <command>" beats "Not Available". Per-OS remediation goes through the docs deeplink, not a one-platform command in the message.

## Worked example (issue #255 — dub jobs "hanging", fixed in commit 2aa6e35)

Users reported dubs stalling forever. The actual failures were real and varied — ASR model-load errors, missing cuDNN — but the SSE stream handling dropped the exception instead of emitting it, so the frontend just stopped receiving events: a hang with no evidence. The fix's insight is that **the reporting bug hid several distinct real bugs**: once failures surfaced with their real messages, the follow-ups became individually diagnosable and fixable — commit 63a0d00 (PyTorch-Whisper fallback works without cuDNN 8) exists because a user could finally see and report the true cuDNN error. Order of operations when a subsystem "hangs": fix the error transparency FIRST, then fix whatever the now-visible errors turn out to be. Fixing invisible bugs is guesswork; making them visible converts them into normal work.

## Failure signs

- `except Exception: pass` or a caught exception that only reaches `logger.debug`.
- A user-facing message that names no next step and links no doc.
- A new known-failure condition without an entry in `error_docs_map.py`.
- An error message instructing the user to do something the code didn't actually verify was missing.
