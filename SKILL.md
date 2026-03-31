---
name: review-walkthrough
disable-model-invocation: true
description: >
  Interactive, point-by-point walkthrough of a review report produced by any review skill (skill-adversary, critical-code-reviewer, or any other). Two modes: **orchestrator mode** (provide a target + optional `--reviewer` and `--adversarial` flags — detects deployment context, calibrates severity, launches the reviewer, then walks through its report) and **walkthrough-only mode** (processes an existing report from the conversation). Parses review findings and processes each one at a time: re-evaluates validity, proposes and applies fixes, checks impacted files for regressions, and waits for user approval before moving on.

  Usage: `/review-walkthrough [target] [--reviewer name] [--adversarial] [--batch|--no-batch]`

  Orchestrator mode: provide a target (file, directory, or glob). Default reviewer: `critical-code-reviewer`. Supported reviewers: `critical-code-reviewer`, `full-review`, `skill-adversary`, `mcp-adversary`.

  Walkthrough-only mode: invoke without a target when a review report already exists in the conversation.
---

# Review Walkthrough

You are conducting an interactive, point-by-point walkthrough of review findings. Your role is to help the user process each issue methodically — re-evaluating it with fresh eyes, fixing what needs fixing, and making sure fixes don't break anything — while keeping the user in control of the pace.

## Step 0: Orchestrate (only when a target is provided)

If the user provided a target (file, directory, or glob) to review, delegate to `agents/orchestrator.md`. Pass the full user request (target + any flags). The orchestrator handles argument parsing, deployment context detection, calibration injection, and reviewer launch.

When the orchestrator finishes, it emits an `--- ORCHESTRATOR COMPLETE ---` block with context values and ends with `--- PROCEED TO STEP 1 ---`. Parse that block for: deployment context (level + detection method), reviewer used, calibration status, `--adversarial` flag value, and `--batch`/`--no-batch` override.

**CRITICAL: Do not stop here.** The review report is now in the conversation. Immediately proceed to Step 1 — do not summarize the review, do not ask the user what to do next, do not treat the reviewer's output as the end of your task. Your task is the walkthrough, not the review. The review was just the input. Continue now.

If no target was provided (walkthrough-only mode), parse `--adversarial` and `--batch`/`--no-batch` from the user's invocation and skip directly to Step 1.

## Step 1: Extract the review points

Scan the current conversation for the most recent review report. Review reports come in many formats (numbered lists, severity tiers, markdown sections, bullet points). Identify each discrete finding regardless of format. If multiple review reports from different skills or different targets exist in the conversation, ask the user which one to process. If there are successive runs of the same skill on the same target, default to the most recent one. If the format is ambiguous or unstructured, present the extracted list of findings to the user for confirmation before processing. If the user corrects the list (adds, removes, or merges items), update it accordingly before proceeding.

If no review report is found in the conversation, tell the user and ask them to either run a review skill first or paste the review content directly.

If the review report uses severity tiers (e.g., Blocking/Required/Suggestion, Critical/Major/Minor, or similar), reorder the findings so that the highest-severity items are processed first. Within the same tier, preserve the original order. If no severity structure is present, process in the order they appear.

If no findings are found (the review reports zero issues), say so and offer to run a quick independent check on the files that were reviewed — a lightweight scan for anything the original reviewer might have missed. If the user declines, end the walkthrough.

If exactly one finding is found, process it directly without the "N points found" preamble — just go straight into the point.

For two or more findings, state the total number of points found, then start processing. Do not produce an upfront summary list of all findings — go straight to the first point.

### Transparency status

Before processing the first finding, report a brief capabilities status block so the user knows exactly what mechanisms are active for this walkthrough:

- **Deployment context** (only if Step 0 ran): report the detected context level and how it was determined. E.g., "Context: personal (detected from path ~/scripts/)." or "Context: production (CI config found)." If the context was asked to the user, say "Context: [level] (user-provided)."
- **Ouroboros**: report the result from the bridge's detection probe (available/not available, consensus enabled/fallback).
- **Author's defense**: "active on N/N findings" — count findings classified at Important severity or above (see Step 2b). If all findings qualify, say "active on all findings". If none, say "skipped — no Important+ findings".
- **Severity reordering**: "applied" (if reordering happened) or "original order preserved" (if no tiers detected).
- **Batch mode**: "active (N findings >= 15)" when Step 1b will run, "inactive (N findings < 15)" when it won't, or "forced via --batch" / "disabled via --no-batch" when overridden by the user.
- **Cross-model validation**: report the active level based on `--adversarial` flag and bridge detection results (L1 always on Important+; L2 on Blocking/Required with `--adversarial` or on L1 divergence — see `agents/ouroboros-bridge.md` for details).

If Ouroboros is available, add a brief glossary of the mechanisms that may fire during the walkthrough, so the user understands the transparency lines they will see later:

> **Mechanisms available for this walkthrough:**
> - *QA auto* — automated second opinion when the verdict on a finding is genuinely uncertain (via `ouroboros_qa`)
> - *Cross-model L1 (intra-family)* — independent re-evaluation by an Agent with an alternate Claude model (e.g. Sonnet if main is Opus); triggers on Important+ findings
> - *Cross-model L2 (cross-provider)* — independent verdict from a different provider via OpenRouter, using `ouroboros_evaluate` with `trigger_consensus: true`; triggers on Blocking/Required (`--adversarial`) or L1 divergence
> - *Lateral think* — creative unblocking when a point stays stuck after 2+ exchanges
> - *Evaluate* — final validation of all applied changes (triggers when ≥ 2 fixes)
> - *Drift check* — detects whether cumulative fixes shifted the code away from its original intent (triggers when ≥ 4 fixes)

This glossary appears only once, before the first finding. Keep it compact — one line per mechanism, no elaboration.

Keep the status block itself to 2-4 short lines. Example:
> Context: personal (detected from path ~/scripts/). Reviewer: critical-code-reviewer (calibrated). Ouroboros available (consensus: single-model fallback). Author's defense active on 4/6 findings. Severity reordering applied — 2 Blocking first. Batch mode: active (32 findings ≥ 15).

## Step 1b: Triage and batch processing

This step activates automatically when the review contains **15 or more findings**. Below 15, skip directly to Step 2. The user can force batch mode with `--batch` (active regardless of count) or suppress it with `--no-batch`.

When active, delegate to `agents/batch-triage.md`. Pass the full findings list, deployment context, and `--adversarial` flag. The batch triage agent handles rapid pre-verdict, classification (auto-fix/auto-reject/manual), user overrides, batch execution with verification, and post-fix hooks.

When the batch triage agent finishes, its output contains: the manual bucket (findings for Step 2), batch results (for the wrap-up table in Step 3), and batch stats (for the transparency status). Proceed to Step 2 with only the manual bucket.

## Step 2: Process each point (manual bucket)

For each point, follow this exact sequence:

### 2a. Context

Briefly paraphrase the original finding. Quote verbatim only when the exact wording matters. Identify the file(s) and line(s) involved. Read the relevant code so you have the current state in front of you. If a referenced file cannot be read (deleted, moved, or inaccessible), state this, mark the finding DEFERRED with "file not accessible" as reason, and move on. If the finding references no specific files (e.g., high-level architectural feedback), identify the most relevant module or files yourself and state the assumption to the user.

### 2b. Re-evaluate

Start from the code, not from the review report. Read the relevant source and form your own assessment before comparing with the reviewer's claim. This reduces confirmation bias — you are a second pair of eyes, not a rubber stamp of the first.

Assess the finding critically and honestly:
- Is the issue real, or is it a false positive?
- Is it relevant given the project's context and conventions?
- Is the severity appropriate?
- Is the suggested fix (if any) the right approach?
- If the finding flags a real issue but does not propose a concrete fix, formulate one yourself — turn "potential issue with X" into "do Y at line Z to fix X". If after evaluation the finding is purely informational (no code change warranted), it is not noise — assign it NOTED.

**Author's defense.** Applies to findings classified at **Important severity or above** — i.e., any tier whose name signals a required or blocking change (e.g. Important, Required, Blocking, Critical, Major, High). Skip the defense for tiers that signal optional, cosmetic, or informational intent (e.g. Minor, Suggestion, Nit, Info, Style). Match case-insensitively; when a tier name is ambiguous, err toward applying the defense. If the review report uses no severity tiers at all, apply the defense to every finding.

When the defense applies: before concluding, generate the strongest counter-argument the code author could make to dismiss the finding. Then evaluate that counter-argument honestly. If the defense holds, downgrade or reject the finding. If it doesn't, the finding is reinforced. Present both the defense and your verdict to the user — this prevents rubber-stamping confident-sounding reviewers.

**Mechanism transparency.** For each finding, state which mechanisms were applied and which were skipped, with the reason. Use a compact inline format after the assessment, before the status label. Examples:
- "Author's defense: applied — defense does not hold."
- "Author's defense: skipped (finding classified Minor)."
- "QA auto: triggered (uncertain verdict) — score 0.72, finding confirmed."
- "QA auto: skipped (clear verdict)."
- "Cross-model L1: Agent (sonnet) agrees — finding confirmed."
- "Cross-model L1: Agent (sonnet) disagrees → escalating to L2."
- "Cross-model L2: score 0.38 (model: anthropic/claude-sonnet-4 via OpenRouter) — finding confirmed."
- "Cross-model: L1 only (not Blocking/Required, no divergence)."
- "Cross-model: skipped (finding classified Minor)."

This takes one line per mechanism — do not let it bloat the output.

State your assessment clearly. If the point is invalid or not worth fixing, say so with a short explanation. The user decides whether to skip it or act on it anyway. If the user chooses to act on a point you assessed as invalid, apply the fix without further pushback — your role is advisory.

### 2c. Fix (if warranted)

Apply the minimal, targeted correction. Rules:
- Only touch code directly related to this point.
- No opportunistic refactoring of surrounding code.
- No inline comments added to the code.
- If the correct fix requires changes beyond the scope of this single point (e.g., structural refactoring), flag it to the user instead of applying an incomplete fix. Let them decide whether to broaden the scope or skip. If they approve broadening, propose a short plan of the changes involved and get confirmation before applying. Then resume the normal walkthrough flow.

### 2d. Verify impacted files

After each fix, re-read the files you modified and files one level away. For code, "one level away" means files that import the changed module or call the changed function directly. For non-code files (SKILL.md, configs, docs), it means files in the same directory that reference or depend on the changed file. Check for:
- Broken references or imports
- Type mismatches or signature changes that affect callers
- Inconsistencies introduced between related files (e.g., a SKILL.md body that now contradicts an agent file)
- Tests that need updating

If the fix modified a dependency manifest (`pyproject.toml`, `package.json`, `Cargo.toml`, `DESCRIPTION`, `renv.lock`), run the appropriate lock command (see post-fix hooks table in `agents/batch-triage.md`) before continuing verification.

Do NOT expand this into a full project review. Stay scoped to the blast radius of your change.

If a regression is detected:
1. **Revert** all changes made for this point (across all files touched) immediately — do not leave broken code in place while discussing.
2. **Explain** the conflict clearly: what the fix changed, what broke, and why.
3. **Propose options**: (a) a different approach to fix the original finding without the regression, (b) skip the point and mark it DEFERRED with the regression as justification, or (c) accept the trade-off if the regression is minor relative to the fix. Let the user choose.

Also check whether the fix makes any of the remaining review points obsolete, already resolved, or partially addressed. If so, flag them to the user — fully resolved points will be skipped when reached, partially addressed ones will note what remains.

**Verification transparency.** Always report what was checked, explicitly listing each file read and its relationship to the change. Use a compact format:
> Verification: `collector.py` (modified), `pipeline.py` (imports collector), `test_collector.py` (tests collector) — no regression.

or if no dependents exist:
> Verification: `SKILL.md` (modified), no dependent files detected.

If the fix was skipped (REJECTED/NOTED/DEFERRED with no code change), state explicitly: "No change applied — verification not needed." Do not silently skip this step.

### 2e. Wait for user approval

Before asking for approval, explicitly assign a status to this finding:

- **ACCEPTED** — the finding is valid and the fix was applied (or the code was already correct and no fix was needed, but the point is acknowledged)
- **REJECTED** — the finding is invalid, irrelevant, or not worth fixing. State why in one line.
- **DEFERRED** — the finding is valid but the fix is out of scope, too risky right now, or requires broader changes. State what would need to happen.
- **NOTED** — the finding is informational or a matter of taste. Acknowledged, no action taken.

The user has the final say — if they disagree with your proposed status, update it without pushback.

After presenting your assessment, status, and any changes, **always** stop and ask the user before moving to the next point — regardless of the status assigned (including DEFERRED). Use a brief prompt like:

- "Next point?"
- Or if a fix was applied: "Fix applied — ACCEPTED. OK to move on?"
- Or if deferred: "DEFERRED — [reason]. Move to the next point?"

Never auto-advance. Never ask for additional context instead of offering to move on — if context is missing, that is itself a reason to DEFER and move forward. The user might want to discuss, adjust, or revert before proceeding.

The user may also deviate from the linear order: jump to a specific point, revisit a previous one, or abandon the walkthrough. Follow their lead — if they abandon, skip to the wrap-up summary with what was completed so far. When revisiting a previously fixed point, re-read the current file state first. If subsequent fixes modified the same areas, flag the interaction to the user before re-applying changes.

## Step 3: Wrap up

After the last point (or if the user abandons mid-walkthrough), give a brief summary table:

| # | Finding | Status | Mode |
|---|---------|--------|------|
| 1 | (short description) | ACCEPTED / REJECTED / DEFERRED / NOTED | batch / manual |
| ... | ... | ... | ... |

The Mode column appears only when batch mode was active. It indicates whether the finding was processed in batch (auto-fix or auto-reject) or through the individual walkthrough.

Follow with:
- Count by status (e.g., "4 accepted, 1 rejected, 2 deferred")
- If batch mode was active: breakdown by mode (e.g., "batch: 11 auto-fix, 8 auto-reject, 1 reverted to manual · manual: 12 walked through")
- List of DEFERRED items with their one-line justification — these are the user's follow-up backlog

After the status counts, add a **Mechanisms used** block summarizing what fired during the walkthrough and — critically — **why each non-fired mechanism was not triggered**. For each mechanism, report: count of invocations, and if zero, the reason in parentheses. Example:
> **Mechanisms:** batch triage 20/32 (12 auto-fix, 8 auto-reject) · author's defense 10/11 · QA auto 0/22 (no ambiguous verdicts) · cross-model L1 6/8 Important+ (Agent sonnet, 1 divergence → escalated to L2) · cross-model L2 3/4 Blocking/Required (model: anthropic/claude-sonnet-4 via OpenRouter) · lateral think 0 (no stuck points or regressions) · evaluate ✓ (score 0.88, based on git diff of 4 files) · drift skipped (< 4 fixes)

The bridge returns pre-formatted mechanism summaries (cross-model status, evaluate results, drift score). Include them verbatim. If Ouroboros was not available, state: "Ouroboros: not available — walkthrough ran without automated QA, consensus, or drift check."

Keep it to 2-3 lines max — the user was there for the whole walkthrough.

## Step 4: Persist

After the wrap-up summary, automatically perform these two persistence actions. Do not ask the user — just do them and report what was written.

### 4a. Update DEFERRED.md

If any findings have status DEFERRED, append them to `DEFERRED.md` at the project root. Create the file if it does not exist, using this format:

```markdown
# Deferred

Findings DEFERRED lors des code reviews. À revisiter périodiquement.

| Date | Finding | Fichier | Raison du report | Échéance |
|------|---------|---------|-----------------|----------|
```

For each DEFERRED finding, add one row with: today's date, a concise description of the finding, the file(s) involved, the reason for deferral, and `—` for échéance unless the user specified a deadline.

If `DEFERRED.md` already exists, append rows to the existing table — do not overwrite.

### 4b. Update memory with review calibration

If any findings were REJECTED, check whether the project already has a `feedback_review_severity.md` memory file. If it exists, update it with any new calibration rules derived from the rejected findings. If it does not exist, create it.

The memory should capture the general calibration pattern (e.g., "this is a personal package, do not suggest X-type defensive patterns") rather than listing each individual rejected finding. Only add rules that are likely to recur in future reviews — skip one-off rejections that are too specific to generalize.

Do not create duplicate rules — if a rejection is already covered by an existing rule in the memory, skip it.

After both actions, briefly report what was persisted (e.g., "2 items added to DEFERRED.md, memory updated with 1 new calibration rule" or "Nothing to persist — no DEFERRED or REJECTED findings").

## Behavioral notes

- Be concise. No filler, no restating what the user already knows. Step 2b can produce substantial analysis for high-severity findings (re-evaluation + author's defense + verdict) — keep each sub-section (re-evaluation, defense, verdict) to 2-3 sentences max; mechanism transparency lines do not count toward this limit. The user needs your conclusion, not your reasoning process.
- When a finding is clearly wrong, say so directly — don't hedge excessively.
- When a finding is valid, fix it without editorializing.
- If unsure whether a point is valid, say so and let the user decide.
- **Language rule:** mirror the user's language in all output (detect from their messages). All examples in this file are in English — translate them to match the user's language at runtime. This is the single source of truth for language behavior; no other section overrides it.

## Ouroboros integration

All Ouroboros tool calls (detection, QA, cross-model validation, lateral think, evaluate, drift check) are handled by `agents/ouroboros-bridge.md`. Delegate to it at the trigger points listed below. If Ouroboros is not available, skip silently.

**Trigger points:**
- **Step 1 (detection):** call the bridge to probe availability. Use the result for the transparency status.
- **Step 2b (QA):** when your re-evaluation is genuinely uncertain, delegate to the bridge for a QA second opinion.
- **Step 2b (cross-model L1/L2):** on Important+ findings, delegate to the bridge for cross-model validation. It handles Agent spawning (L1) and `ouroboros_evaluate` with consensus (L2).
- **Step 2b-2c (lateral think):** when stuck (2+ exchanges or regression revert), delegate to the bridge.
- **Step 3 (evaluate):** when >= 2 fixes applied, delegate to the bridge for final validation. It builds the artifact from git diff.
- **Step 3 (drift):** when >= 4 fixes applied, delegate to the bridge for drift check.

Present all Ouroboros results inline as described in the mechanism transparency format (Step 2b). Runtime errors are caught by the bridge — never let an Ouroboros failure block the walkthrough.
