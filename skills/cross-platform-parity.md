# Cross-Platform Parity

Hard rule from CLAUDE.md: a feature that ships in default mode must behave identically on macOS, Windows, and Linux. Platform-specific *implementation* is fine; divergent *default behavior* is a P0. And CI will not save you: the Python backend's runtime tests run on Linux only, so macOS/Windows correctness is established by reasoning and stated explicitly.

## The procedure (for every change that could touch a platform seam)

1. **Identify the seams your diff crosses.** The recurring ones in this codebase:
   - paths (`/` vs `\`, `Path` vs string concat, symlinks — broken on Windows, caused the Triton issues),
   - subprocess spawning (`open` / `explorer` / `xdg-open` are per-OS; shell quoting differs),
   - GPU stack (CUDA / MPS / ROCm / CPU — every ML feature needs the "none of the above" answer),
   - shell rc files and env persistence (`~/.zshrc` macOS default vs `~/.bashrc` vs PowerShell `[Environment]::SetEnvironmentVariable`),
   - packaging (DMG + Gatekeeper quarantine, AppImage + webkit2gtk versions, .deb postinst, NSIS),
   - "Linux" is not one platform: Fedora/Ubuntu/Debian differ in webkit versions and package managers.
2. **For each seam, answer per-OS in writing:** what happens on macOS (Apple Silicon AND Intel), Windows x64, Linux (AppImage AND deb)? If the answer differs, either fix it or move the feature behind explicit opt-in (Settings toggle, env var, CLI flag). There is no third option.
3. **Trust code over environment.** Anything user-visible that reports platform facts must read them from the backend process, not the client.
4. **State your platform verification honestly in the commit:** "Cross-platform (the torch 2.6 weights_only change affects all platforms)" or "Linux-verified; macOS path reasoned from X, not run." Honesty here is what lets the next debugger trust the history.
5. **If you touch `frontend/src-tauri/` (Rust) or launcher scripts,** remember `cargo check` in CI catches type errors only — it never runs the binary. Runtime bugs (missing system libs, loader paths, ffmpeg invocation) pass CI on all three OSes while being broken on two.

## Rules I was following

- **Default-deny on platform divergence.** When you can't make a default work on one platform, demote it to opt-in rather than shipping it broken there. The macOS-only global shortcut is opt-in for exactly this reason.
- **Detect, don't blanket-apply, workarounds.** The AppImage white-screen fix (`WEBKIT_DISABLE_COMPOSITING_MODE=1`) is overwhelmingly an NVIDIA-proprietary-driver issue; applying it unconditionally forces software rendering on AMD/Intel users forever. The launcher detects NVIDIA first.
- **Platform facts come from the layer that owns them.** GPU from torch probes in the backend; OS paths from `backend/core/config.py:get_app_data_dir()`; never re-derive in the frontend.
- **Even CI scripts have platform seams.** Commit 35b62ad exists because a release-workflow checksum step used bash features newer than macOS's bundled bash 3.2. When writing CI shell, target bash 3.2 or declare `shell: bash` with a version check.

## Worked example (commit e740786 — About panel showed the wrong CPU arch, issue #262)

Symptom: a user running the backend in Docker on an x86 server, viewed from an ARM MacBook's browser, saw "Apple Silicon" in the About panel. Cause: the frontend rendered `navigator.platform` — the *client browser's* platform — instead of asking the backend. On desktop (webview and backend on the same machine) the two coincide, so the bug was invisible in default testing; the web/Docker mode exposed the seam. Fix: About panel reads arch from the backend's `/system/info` (the process that actually runs the models), with the browser value used only as a fallback label for the client row. The general rule extracted: *whenever desktop mode makes two machines coincide, ask which machine each displayed fact belongs to, because server mode will split them.*

## Failure signs

- A path built with string `+` or f-string instead of `pathlib`.
- `navigator.*` or any client-side probe used for a fact about the backend host.
- A workaround env var set unconditionally in a launcher.
- The commit message contains no statement about which platforms were verified vs reasoned.
