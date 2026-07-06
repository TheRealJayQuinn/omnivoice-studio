# Working On TTS/ASR Engines

Engines are the highest-blast-radius code in the repo: seven TTS backends plus ASR share one venv, one HF cache, and one registry, and users have already-installed on-disk engine state that must survive your change. Adding or touching an engine follows a fixed shape.

## The procedure (adding an engine)

1. **License and identity first, before any code.** Read the model card and LICENSE on HF. Confirm the artifact is what its name claims (see the SPIKE-01 example below — "OmniVoice" is an overloaded name). If the license is restrictive, it surfaces in the engine's `display_name` and it ships opt-in, not bundled.
2. **Pin everything.** Model repo by HF revision SHA, runtime repos by commit SHA — never `main`. Pins live next to the code (e.g., `backend/engines/omnivoice_gguf/quant_map.json` carries both SHAs in its `_meta` block) so the code and the decision doc can't drift. There's a helper precedent: `scripts/resolve_supertonic3_sha.py`.
3. **Isolate dependencies.** New engine deps go in `[project.optional-dependencies]` in `pyproject.toml`, never main deps. Then run `uv lock` and **diff the lockfile: if any existing engine's pin moved, stop** — that's the "adding engine 7 silently breaks engine 3" failure, and it violates the no-reinstall constraint.
4. **Subclass the existing primitives.** New engines are `TTSBackend` subclasses in `backend/services/tts_backend.py` (or a `SubprocessBackend` wrapper for out-of-process runtimes with a stdin/argv/output-file CLI). Third-party pluggability goes through `backend/services/plugin_sdk.py`. Do not invent a new integration shape.
5. **`is_available()` must be cheap and honest.** Disk/platform checks only — no heavy imports (boot time is already a complaint). On unsupported platforms return an actionable reason ("requires CUDA 12+ or Apple Silicon MPS"), not False-with-silence, and never crash on import.
6. **Model downloads are async with progress**, cached under the HF cache, size-warned before first use. Never download synchronously inside a generate call — the user reads that as a hang.
7. **Fallback chain intact.** Every new engine path falls back to the existing in-process `OmniVoiceBackend` on probe/download/load/generate failure. A new engine may fail to be better; it may not make anything worse.
8. **Smoke it per platform:** registry lists it, `is_available()` truthful on all three OSes (reasoned where not runnable), one real synthesis on hardware you have, and — critically — the OTHER engines still load (`from backend.services.tts_backend import _REGISTRY` and iterate).

## The procedure (touching an existing engine)

1. List every engine whose code path you share (registry, model_manager pools, HF cache, subprocess spawn).
2. After the change, load each of those engines, not just the one you fixed. The registry import-order shift that breaks a sibling engine's detection is a known real failure.
3. On-disk state compatibility: a user's already-downloaded weights must still be found. If you move a path, read the old one first.

## Rules I was following

- **Subprocess env injection follows the canonical site** (`sonitranslate.py:148`): inject only the vars the child needs (`HF_TOKEN` + the child's alias), sourced from the token resolver, never a blanket env copy.
- **Freeform paths/URLs are a supply-chain surface.** Quant/variant selection is a dropdown over a shipped map (`quant_map.json`), not a text field. Same rule as PyPI mirrors: allow-list, never freeform.
- **Availability probes swallow errors by design — so you must not rely on them while debugging.** `uv run python -c "import <module>"` tells the truth; `is_available()` tells the user-safe version.

## Worked example (SPIKE-01 — the GGUF engine, `.planning/decisions/SPIKE-01-gguf.md`)

Before a line of integration code, the spike answered, with evidence links: (1) identity — the HF model card's `base_model` tags prove `Serveurperso/OmniVoice-GGUF` really is a quantization of the shipped `k2-fsa/OmniVoice`, not an unrelated model with the same name; (2) license — Apache-2.0 model + MIT runtime, same chain as what already ships; (3) runtime — `gguf.architecture = "omnivoice-lm"` from the HF API proved it does NOT load in vanilla llama.cpp, so the integration needs the custom `omnivoice.cpp` binary — discovering that after writing a llama.cpp wrapper would have wasted the whole effort; (4) both repos pinned by SHA, mirrored into `quant_map.json` `_meta`; (5) macOS Metal flagged as CONDITIONAL (no published build script) with explicit downgrade criteria — if the Metal build fails, macOS keeps the in-process default and the ADR's Status line records it. The GO decision cost a few hours of reading and API calls; every one of those five answers, discovered late instead, would have cost days.

## Failure signs

- `uv lock` diff shows version movement outside your engine's direct deps.
- An engine dep in `[project.dependencies]` instead of an optional group.
- A model reference that says `main` or `latest` anywhere.
- Your PR touches `tts_backend.py` and you can't list which other engines you loaded afterward.
