# TODO — review-walkthrough skill

## Orchestrator mode

review-walkthrough becomes a single entry point that can orchestrate the full review pipeline.

### Invocation

```
/review-walkthrough <cible>                                        # défaut: critical-code-reviewer
/review-walkthrough <cible> --reviewer skill-adversary             # autre reviewer
/review-walkthrough <cible> --reviewer full-review                 # full-review gère ses propres agents, l'orchestrateur attend le rapport
/review-walkthrough                                                # pas de cible: consomme un rapport existant (comportement actuel)
```

### Pipeline

When a target is provided and no review report exists in the conversation:
1. Detect deployment context (see context detection below)
2. If context injection applies, prepend calibration block to the reviewer prompt
3. Launch the reviewer skill on the target
4. Run the walkthrough on the resulting report

When a review report already exists (no target argument), skip steps 1-3 and proceed as today.

### Reviewer parameter

The `--reviewer` parameter accepts any review skill name. Default: `critical-code-reviewer`.
Each reviewer is launched via its skill invocation as a black box — the orchestrator doesn't interfere with its internals. For full-review, this means full-review still spawns its own parallel agents, deduplicates, and consolidates as usual. The orchestrator's only role is to (a) optionally prepend context calibration to the reviewer's prompt, then (b) wait for the final report and feed it into the walkthrough.

### Context detection

Automatically infer deployment context from path heuristics and memory:
- `~/.local/bin/`, `~/bin/`, `~/scripts/` → `personal`
- Project root with CI config, Dockerfile, or deploy manifests → `production`
- Internal tooling repos (detectable via memory or `.internal` marker) → `internal`
- Fallback: ask the user on first use, persist in memory for the project

Context levels:
- `personal` — single-user, not distributed, no untrusted input. Suppress: theoretical attack vectors requiring same-UID access, portability for non-target platforms, defensive patterns for impossible conditions
- `internal` — team use, trusted network. Suppress: portability beyond the team's stack, but keep security and correctness findings
- `production` — full adversarial review, no suppression

### Context injection scope

Context injection only applies to reviewers that evaluate **application code**:
- critical-code-reviewer — yes
- full-review — yes

It does **not** apply to reviewers that evaluate **skill/tool artefacts** (the personal/internal/production distinction is irrelevant):
- skill-adversary — no (reviews SKILL.md structure)
- mcp-adversary — no (reviews MCP server schemas)

When context injection does not apply, the reviewer is launched with its original prompt unmodified.

The context is prepended to the reviewer prompt as a calibration block, e.g.:
> "Context: this is a personal utility script on a single-user workstation. Calibrate severity accordingly — flag real bugs and data-loss risks, but do not flag theoretical attacks requiring same-UID access, portability issues, or defensive patterns for conditions that cannot occur in this context."

## Batch mode for obvious verdicts

The current walkthrough processes every finding sequentially with a user confirmation step.
On large reviews (30+ findings), this becomes tedious — especially when most findings are clear-cut.

**Idea:** after re-evaluation, group findings into three buckets:
- Clearly valid (apply fix without individual confirmation)
- Clearly false positive (reject in batch, show summary)
- Uncertain (walk through individually as today)

Present the batched verdicts as a summary table, let the user override any before proceeding.
The individual walkthrough only fires for the uncertain bucket.

Challenge: defining "clearly valid" vs "uncertain" without being overconfident.
Could use Ouroboros QA score as a confidence signal — high score = clearly false positive, low score = clearly valid, mid-range = uncertain.

Interaction with context detection: context-aware reviews should produce fewer false positives, making the batch mode less critical but still valuable for large codebases.

## Same-model bias in author's defense

The reviewer and the walkthrough are the same model in the same session.
The "author's defense" is simulated, not genuinely adversarial — the model can convince itself too easily in either direction.

**Idea:** use Ouroboros Advocate/Devil's Advocate more aggressively — not just on ambiguous findings, but on all Blocking/Required findings.
This adds a structured multi-perspective evaluation even when the walkthrough model feels confident.

Trade-off: doubles the Ouroboros calls, adds latency. Could be opt-in (`--thorough` flag).

## Observations from first orchestrator run (2026-03-31)

Target: `~/.local/bin/cleanup` (personal bash script). Reviewer: critical-code-reviewer (calibrated personal context).

- Context detection worked correctly (path heuristic → personal)
- Calibration block reduced noise: reviewer self-withdrew 2 of its own findings mid-report (points 2 and 6), correctly self-correcting on closer inspection
- Author's defense held on 1/4 findings (locale-dependent snap filter) → REJECTED, which is the right call for a personal script
- Ouroboros evaluate caught a real issue (pipefail + snap) that the reviewer missed — confirms the value of the final evaluation step
- Ouroboros QA auto never triggered (all verdicts were clear-cut) — need a genuinely ambiguous review to test it
- 5-finding review took ~10 min interactive — batch mode not needed at this scale, but would matter for full-review with 20+ findings
- The reviewer's self-withdrawn points (2 and 6) cluttered the report — consider asking reviewer skills to emit only confirmed findings, or filtering withdrawn findings before the walkthrough

## Observations from second orchestrator run — full-review batch mode (2026-03-31)

Target: `~/.claude/skills/bookmarks-manager` (personal Python skill). Reviewer: full-review (4 agents, calibrated personal context). Batch mode forced via `--batch`.

### What worked

- Full pipeline end-to-end: orchestrator → full-review (4 agents) → consolidated report → batch triage → manual walkthrough → evaluate → persist. No manual intervention needed between phases.
- Context detection correct (path heuristic → personal). Calibration block injected into all 4 agent prompts.
- 3-way convergence on HTML escaping bug (agents A, B, C independently flagged it) — strong signal, correctly promoted to Blocking.
- Batch triage correctly identified 2 mechanical auto-fix candidates (unused deps, sys.path hack). Both applied cleanly, 49 tests pass.
- Author's defense applied 5 times on Required+ findings. Held once (partially, on fragment stripping), rejected 4 times — appropriate ratio for real bugs.
- Finding #1 fix (HTML escaping) correctly predicted that finding #4 (misleading test) would break — flagged the interaction proactively.
- Fix for finding #3 (normalize_url) partially addressed finding #6 (schemeless URLs) — cross-finding awareness worked.
- Ouroboros QA auto never triggered (all verdicts clear-cut). Still untested on genuinely ambiguous findings.

### Issues to fix

1. **Agent B (Explore) scope violations** — Agent B (architecture) strayed into Agent A's scope (HTML escaping = correctness), Agent C's scope (test validity, docstrings). 3 of its 5 findings were out-of-scope duplicates. The `Explore` agent type does not respect scope boundaries as well as `general-purpose`. Consider switching Agent B to `general-purpose` or adding stronger scope enforcement in the prompt.

2. **Batch triage: manual findings not in table format** — User explicitly requested that manual findings be presented in a table (same format as auto-fix table) during triage step 1b.2. Currently only auto-fix and auto-reject get tables. Already saved as feedback memory.

3. **Ouroboros evaluate false positive on lint** — `ouroboros_evaluate` ran `ruff` and flagged 2 pre-existing lint issues (`import sys`, f-string without placeholder) as failures. These were not introduced by the review. Evaluate should either diff against the baseline or only lint modified files. Workaround: the checkup phase caught and fixed these, but evaluate's REJECTED verdict was misleading.

4. **No `uv lock` after dependency removal** — Batch auto-fix removed deps from pyproject.toml but did not regenerate `uv.lock`. Caught in the final checkup. The batch execution step (1b.3) should run `uv lock` when pyproject.toml dependencies change.

5. **Linter reformatting during edits** — A formatter hook reformatted files on save (expanding single-line dicts to multi-line), which changed line numbers between the review report and the actual code. This didn't break anything but made line references in the report stale. Not actionable from the skill side, but worth noting.

6. **12 findings below batch threshold (15) but --batch forced** — The `--batch` flag correctly overrode the threshold, but the triage only found 2 auto-fix candidates out of 12 (no auto-reject). Batch mode's ROI is low at this scale. The 15-finding threshold seems right; `--batch` override is still useful for testing.

7. **Same-model bias still present** — Author's defense arguments were plausible but never truly surprising. The model didn't generate a defense it hadn't already considered. Ouroboros Advocate/Devil's Advocate was never triggered (no ambiguous verdicts). The bias mitigation mechanisms exist but haven't been stress-tested.

### Severity calibration learned

- Personal skill scripts: skip docstring findings (API contract is in SKILL.md, not docstrings)
- Persisted in `feedback_review_severity.md` for future reviews

## Priority order

1. ~~**Orchestrator mode** — highest impact, reduces false positives at source, single entry point~~ DONE (2026-03-31)
2. ~~**Batch mode** — quality-of-life for large reviews~~ DONE in SKILL.md (2026-03-31), tested on 12-finding review. Works but ROI is low below 15 findings. Needs testing on 30+ finding review.
3. ~~**Agent B scope enforcement** — strengthen scope prompt in full-review Agent B definition + add scope filter in Phase 2 consolidation~~ DONE (2026-03-31). Kept `Explore` type, added explicit scope enforcement instruction and post-dedup scope filter.
4. ~~**Evaluate baseline diffing** — diff-only artifact, scoped acceptance_criterion, post-filter on evaluate results~~ DONE (2026-03-31). Evaluate now receives only diff output with "do not flag pre-existing issues" prefix, acceptance_criterion scoped to walkthrough changes, and post-filter discards findings on unmodified lines.
5. ~~**Batch post-fix hooks** — auto-run `uv lock` / `npm install` / etc. when batch fixes modify dependency files~~ DONE (2026-03-31). Post-fix hooks table in 1b.3 + lock command mention in Step 2d for manual fixes.
6. **Same-model bias** — hardest to solve well, opt-in `--thorough` flag is fine for now
