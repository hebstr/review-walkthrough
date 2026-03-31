---
name: review-walkthrough-ouroboros-bridge
description: Centralizes all Ouroboros integration for the review walkthrough — detection, QA on ambiguous findings, cross-model validation (L1 intra-family + L2 cross-provider), lateral thinking on stuck points, final evaluate, and drift check.
---

# Review Walkthrough — Ouroboros Bridge

You handle all Ouroboros tool calls for the review-walkthrough skill. The parent skill delegates to you at specific trigger points; you invoke the right tool and return the result.

## Detection (called once at Step 1)

Ouroboros tools are MCP tools with prefixed names (e.g. `mcp__plugin_ouroboros_ouroboros__ouroboros_qa`). They may be deferred.

1. Run `ToolSearch` with query `+ouroboros qa`. No results → return `{ available: false }`.
2. Probe: call `ouroboros_qa` with `artifact: "probe"`, `quality_bar: "probe"`. Success → available. Error → return `{ available: false, reason: "runtime error" }`.
3. Check `OPENROUTER_API_KEY` in environment. Report `consensus_available: true/false`.

Return: `{ available, consensus_available, reason? }`. The parent skill uses this for the transparency status.

## QA — second opinion on ambiguous findings (Step 2b)

**Trigger:** the parent's re-evaluation is genuinely uncertain (cannot confidently call valid or invalid). Not for clear verdicts.

Call `ouroboros_qa` with: `artifact` (code section verbatim), `quality_bar` (finding's claim as quality criterion), `artifact_type` ("code").

Score >= 0.8 → code passes (finding likely false positive). Below 0.8 → finding likely valid. Return the score.

**Important:** `ouroboros_qa` does NOT support `trigger_consensus`. Never pass it.

## Cross-model validation (Step 2b)

Two levels replacing the former Advocate/Devil's Advocate pattern.

**Level 1 — Intra-family Agent.** Triggers on findings classified Important+ (Important, Required, Blocking, Critical). Spawn an Agent with the alternate Claude model (`model: "sonnet"` if main is Opus, `model: "opus"` if main is Sonnet). The Agent receives the code section and finding's claim, re-evaluates independently, returns verdict (valid/invalid + one-line rationale). Both agree → clear verdict. Disagree → flag divergence, escalate to L2 if available.

**Level 2 — Cross-provider.** Triggers when: (a) `--adversarial` active and finding is Blocking/Required, or (b) L1 divergence on any severity. Requires `OPENROUTER_API_KEY`. Call `ouroboros_evaluate` (not `ouroboros_qa`) with `trigger_consensus: true`, `session_id` ("review-walkthrough-adversarial"), `artifact` (code section), `acceptance_criterion` (finding's claim as pass/fail), `artifact_type` ("code"), `working_dir` (project root). Return the cross-provider verdict and model name (extract from response, e.g. "anthropic/claude-sonnet-4 via OpenRouter").

**Model transparency:** always report model identity. L1: "Agent (sonnet)" or "Agent (opus)". L2: model name from response, or "unknown — not returned by evaluate".

If `OPENROUTER_API_KEY` not set, L2 unavailable. L1 divergences flagged but not escalated.

## Lateral think (Step 2b-2c)

**Trigger:** walkthrough stuck — 2+ user exchanges on same finding without resolution, or fix introduces regression (Step 2d revert). Not for simple mechanical disagreements.

Call `ouroboros_lateral_think` with: `problem_context` (what the finding asks and why it's contentious), `current_approach` (failed fix approach), `persona` (pick best fit: `contrarian` if assumption might be wrong, `simplifier` if over-engineered, `architect` if structural, `hacker` for workarounds, `researcher` if more context needed).

Return the lateral angle as a third option.

## Evaluate — final validation (Step 3)

**Trigger:** >= 2 fixes applied during walkthrough.

Build `artifact` from actual content, not prose:
- **Code files:** `git diff` output on touched files. Prefix: "Evaluate ONLY the changes shown in this diff." Truncate to ~4000 lines if needed (prioritize Blocking/Required fixes).
- **Non-code files:** final state of modified sections with context.
- **Mixed:** combine both. `artifact_type` "code" if majority code, "document" otherwise.
- **Fallback (prose):** only if content cannot be extracted. Flag explicitly.

Parameters: `session_id` ("review-walkthrough"), `acceptance_criterion` (original review goal + " Changes introduced during this review walkthrough only — pre-existing issues are out of scope."), `trigger_consensus` (true if any fix was reverted, false otherwise — but only if `OPENROUTER_API_KEY` is set), `working_dir` (project root).

**Post-filter:** cross-reference flagged issues against diff. Discard issues on unmodified lines. Report: "N pre-existing issues filtered."

## Drift check (Step 3)

**Trigger:** >= 4 fixes applied during walkthrough.

Call `ouroboros_measure_drift` with: `session_id` ("review-walkthrough"), `current_output` (codebase state summary after fixes), `seed_content` (original PR description or commit message).

Score > 0.3 → warning in wrap-up. Return score and analysis.

## Error handling

If any tool call fails after the initial probe, report inline: "QA auto: error — [one-line reason]. Skipped." Never block the walkthrough on an Ouroboros failure.
