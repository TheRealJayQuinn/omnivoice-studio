# Spike Before Integrate

When a task depends on a third-party artifact you haven't personally verified (a model, a runtime, a library, a mirror URL), you run a spike first: a time-boxed investigation whose only output is a GO/NO-GO decision document. Code written before the spike is gambling with the integration's whole budget.

## When this applies

- Adding any model or engine from HF/GitHub.
- Depending on a claimed capability ("supports MPS", "loads in llama.cpp", "constructor takes X") you've only read about.
- Adopting a mirror, CDN, or download endpoint.
- Any task where RESEARCH notes say confidence is MEDIUM or lower.

## The procedure

1. **Write the questions first, as a table.** Each question must be answerable with evidence (an API response, a file listing, a license text), not vibes. The recurring five: Is this artifact what its name claims? Is the license shippable? What runtime does it actually need? What are the real sizes/footprints? Does it fit an integration primitive we already have?
2. **Answer from primary sources.** The HF model API (`https://huggingface.co/api/models/<repo>` — gives `siblings`, tags, `gguf.architecture`, `lastModified`), the actual LICENSE file, the actual README build scripts. A blog post or the model card prose alone is not evidence.
3. **Record verdicts per question** — YES / NO / CONDITIONAL, each with the evidence link. CONDITIONAL items get explicit acceptance criteria and a named downgrade path ("if Metal build fails, macOS keeps in-process default; update this ADR's Status line").
4. **Pin SHAs at spike time** and write them into both the decision doc AND a machine-read file next to the code, so the decision and the implementation can't drift.
5. **Write the decision doc** in `.planning/decisions/` with: Context, Decision (GO/NO-GO + integration shape in two sentences), spike verification table, Consequences (positive AND negative), Mitigations per risk, Sources with verification dates.
6. **A NO-GO is a successful spike.** The affected requirements move to out-of-scope with a link to the decision doc. Do not soften a NO-GO into "maybe later with workarounds" — that re-opens the gamble.

## Rules I was following

- **Verify the artifact's identity, not just its quality.** Popular names get squatted and overloaded. The `base_model` tag chain in the HF API is the provenance check for models; for repos, cross-check the org against the official project links.
- **Re-verify before executing if time passed.** SPIKE-01 was researched 2026-05-18 and re-verified 2026-05-20 with fresh SHAs before Wave 1 started. Upstream repos move; a spike older than the code it gates is stale.
- **Date-stamp every source.** "Verified 2026-05-20" turns a dead link two months later from a mystery into a known re-verification task.
- **The spike's output shapes the plan, not vice versa.** Don't write the integration plan first and spike to confirm it — the SPIKE-01 discovery that the quants need a custom runtime (not llama.cpp) changed the entire integration shape from "add a llama.cpp wrapper" to "bundle per-platform binaries built from a pinned SHA."

## Worked example (SPIKE-01, `.planning/decisions/SPIKE-01-gguf.md`)

Question: adopt `Serveurperso/OmniVoice-GGUF` as the hardware-adaptive default cloning engine? The spike table answered six questions with evidence: identity confirmed via `base_model:quantized:k2-fsa/OmniVoice` tags in the API response; licenses read (Apache-2.0 + MIT); runtime probed via `gguf.architecture = "omnivoice-lm"` (custom, NOT llama.cpp-loadable — the single most plan-changing fact); all 8 quant files and sizes confirmed from `siblings`; platform build scripts enumerated from the repo (Metal missing → CONDITIONAL with Wave 1 acceptance criteria); CLI shape confirmed to fit the existing `SubprocessBackend` primitive. Verdict GO, with pinned SHAs mirrored into `quant_map.json`, a fallback guarantee (in-process backend remains default on any GGUF failure), and every negative consequence listed with a mitigation. Status started as "Proposed — flips to Accepted in Task 3", i.e., even a GO stays provisional until the build actually smokes.

## Failure signs

- Integration code exists and no decision doc does.
- A capability claim in your plan sourced from a README sentence you didn't test.
- "Latest" or unpinned references surviving past the spike.
- A CONDITIONAL verdict with no acceptance criteria or downgrade path attached.
