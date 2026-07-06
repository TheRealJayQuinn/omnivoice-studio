# Skill Library Index — OmniVoice Studio

Fourteen disciplines, distilled from this project's plans, summaries, commits, and scars. Ranked by **quality bought per token spent reading**: how much a mechanically-followed read of the skill improves the median change, divided by its length. Read tier 1 before every task; pull the rest in when their trigger fires.

## Tier 1 — read before every task (highest leverage, always applicable)

| Rank | Skill | Why it ranks here |
|------|-------|-------------------|
| 1 | [verify-your-own-work.md](verify-your-own-work.md) | The single defense against this project's central failure mode (the fix-regression cycle). A grep gate + re-run of pre-existing gates converts "I believe it works" into "it works." Cheapest tokens, biggest quality delta. |
| 2 | [scope-and-constraints.md](scope-and-constraints.md) | Wrong-scope work is 100% wasted regardless of quality. Five written constraints (engine compat, platform parity, data compat, local-first, single version line) decide most changes before any code is read. |
| 3 | [plan-before-touching.md](plan-before-touching.md) | Forces file:line reading, assumption checks, and an exit gate before edits. Prevents the expensive class of error (building the wrong thing) rather than the cheap class (building it slightly wrong). |
| 4 | [failure-patterns.md](failure-patterns.md) | Ten pre-paid lessons (import-state pollution, env.py URL clobbering, worktree path mixups, coverage theater...). Each entry read once saves the hour it originally cost. Skim the headers; deep-read on a match. |

## Tier 2 — read when the trigger fires (high leverage, conditional)

| Rank | Skill | Trigger |
|------|-------|---------|
| 5 | [debugging-method.md](debugging-method.md) | Any bug report or failing test. The "search for prior art in this codebase first" step alone routinely turns a day of invention into 6 lines of reuse (issue #270). |
| 6 | [cross-platform-parity.md](cross-platform-parity.md) | Diff touches paths, subprocess, GPU, packaging, launchers, or anything user-visible. CI cannot check this; only the discipline can. |
| 7 | [backward-compat-data.md](backward-compat-data.md) | Diff touches `db.py`, migrations, `omnivoice_data/` layout, or engine on-disk state. Violations are unrecoverable for users, so the per-token value is extreme when triggered. |
| 8 | [secrets-and-redaction.md](secrets-and-redaction.md) | Diff touches tokens, logging, subprocess env, or anything leaving the machine. One leak permanently breaks the local-first promise. |
| 9 | [error-ux.md](error-ux.md) | Adding/handling any failure path. Encodes the project's core value; the #255 lesson (fix transparency first, then the newly-visible bugs) reorders whole debugging efforts. |
| 10 | [engine-work.md](engine-work.md) | Adding or touching any TTS/ASR engine. Highest blast radius code in the repo; the lockfile-diff and license-first steps prevent the two worst outcomes. |
| 11 | [spike-before-integrate.md](spike-before-integrate.md) | Any dependency on an unverified third-party artifact. A few hours of evidence-gathering versus days of wrong-shaped integration (SPIKE-01's llama.cpp discovery). |

## Tier 3 — reference while writing (steady, mechanical value)

| Rank | Skill | Trigger |
|------|-------|---------|
| 12 | [conventions-checklist.md](conventions-checklist.md) | Writing any code. Mostly prevents review round-trips rather than defects — real but smaller value per token. The layering and i18n rules are the exceptions: those are CI-enforced and architectural. |
| 13 | [commit-hygiene.md](commit-hygiene.md) | Committing. Pays out later (future debugging via history) rather than now, which is why it ranks low per token today — and why it's still mandatory. |
| 14 | [docs-that-dont-rot.md](docs-that-dont-rot.md) | Touching `docs/install/`, install scripts, or error remediation text. Narrowest trigger of the set; when it fires, the validate-marker mechanics are non-obvious and worth the read. |

## How to use this library

1. Before starting: skim tier 1 (four files, ~5 minutes).
2. Match your diff against the tier 2/3 triggers; read every skill that fires — most real tasks fire two or three.
3. When something behaves impossibly, go straight to [failure-patterns.md](failure-patterns.md).
4. When you learn a new lesson the hard way, add it: extend failure-patterns.md or the relevant skill's rules, **with the worked example**, in the same PR as the fix. A skill library that doesn't grow with the project rots like the docs it warns about.

Each skill has the same shape: procedure (numbered, mechanical), the judgment rules written out explicitly, one worked example from THIS repository, and failure signs to self-check against. If you follow only the numbered steps and the failure signs, you get 90% of the value.
