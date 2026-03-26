---
name: review-walkthrough
description: >
  Interactive, point-by-point walkthrough of a review report produced by any review skill (skill-adversary, critical-code-reviewer, or any other). Parses the review findings from the current conversation, then processes each point one at a time: re-evaluates validity, proposes and applies fixes, checks impacted files for regressions, and waits for user approval before moving on.

  Use this skill whenever the user wants to act on review results iteratively — phrases like "walkthrough the review", "let's go through the review point by point", "contre-review", "fix the review points one by one", "parcours les points de la review", "on passe sur chaque point", "régler les points un par un", "traiter les résultats de la review", "valider chaque point ensemble", "let's address the review findings", or any request to process/fix/handle review results step by step. Also trigger when a review skill just ran in the current conversation and the user asks to work through the results, even casually ("ok let's fix these", "on s'y met", "go through them", "let's do it", "yep fix them", "on y va", "let's tackle these"). These casual triggers require a review report to be present earlier in the conversation — do not trigger on them in isolation. Do NOT trigger for: code walkthroughs or architecture explanations, academic/paper reviews, PR comment discussions, linter or CI output processing, or Jira/backlog issue lists.
---

# Review Walkthrough

You are conducting an interactive, point-by-point walkthrough of review findings. Your role is to help the user process each issue methodically — re-evaluating it with fresh eyes, fixing what needs fixing, and making sure fixes don't break anything — while keeping the user in control of the pace.

## Step 1: Extract the review points

Scan the current conversation for the most recent review report. Review reports come in many formats (numbered lists, severity tiers, markdown sections, bullet points). Identify each discrete finding regardless of format. If multiple review reports from different skills or different targets exist in the conversation, ask the user which one to process. If there are successive runs of the same skill on the same target, default to the most recent one. If the format is ambiguous or unstructured, present the extracted list of findings to the user for confirmation before processing. If the user corrects the list (adds, removes, or merges items), update it accordingly before proceeding.

If no review report is found in the conversation, tell the user and ask them to either run a review skill first or paste the review content directly.

If the review report uses severity tiers (e.g., Blocking/Required/Suggestion, Critical/Major/Minor, or similar), reorder the findings so that the highest-severity items are processed first. Within the same tier, preserve the original order. If no severity structure is present, process in the order they appear.

If no findings are found (the review reports zero issues), say so and offer to run a quick independent check on the files that were reviewed — a lightweight scan for anything the original reviewer might have missed. If the user declines, end the walkthrough.

If exactly one finding is found, process it directly without the "N points found" preamble — just go straight into the point.

For two or more findings, state the total number of points found, then start processing. Do not produce an upfront summary list of all findings — go straight to the first point.

### Transparency status

Before processing the first finding, report a brief capabilities status block so the user knows exactly what mechanisms are active for this walkthrough:

- **Ouroboros**: "available" or "not available" (based on whether `ouroboros_qa` etc. are in the tool list). If available, also report whether multi-model consensus is enabled (test by checking if `OPENROUTER_API_KEY` is set in the environment — if not, note "consensus: single-model fallback").
- **Author's defense**: "active on N/N findings" — count how many findings will trigger the defense based on severity tiers (all findings minus those explicitly in the low-severity exclusion list). If all findings trigger it, say "active on all findings". If none (all are low-severity), say "skipped — all findings are low-severity".
- **Severity reordering**: "applied" (if reordering happened) or "original order preserved" (if no tiers detected).

Keep this to 2-3 short lines. Example:
> Ouroboros available (consensus: multi-model via OpenRouter). Author's defense active on 4/6 findings. Severity reordering applied — 2 Blocking first.

## Step 2: Process each point

For each point, follow this exact sequence:

### 2a. Context

Briefly paraphrase the original finding. Quote verbatim only when the exact wording matters. Identify the file(s) and line(s) involved. Read the relevant code so you have the current state in front of you.

### 2b. Re-evaluate

Start from the code, not from the review report. Read the relevant source and form your own assessment before comparing with the reviewer's claim. This reduces confirmation bias — you are a second pair of eyes, not a rubber stamp of the first.

Assess the finding critically and honestly:
- Is the issue real, or is it a false positive?
- Is it relevant given the project's context and conventions?
- Is the severity appropriate?
- Is the suggested fix (if any) the right approach?
- If the finding does not propose a concrete fix, formulate one yourself. A finding without an actionable remedy is noise — turn "potential issue with X" into "do Y at line Z to fix X".

**Author's defense.** For every finding **unless** it is explicitly classified as one of the following low-severity tiers by the review report: Suggestion, Minor, Low, Info, Cosmetic, Nit, Note, Nitpick, Style, or Informational (match case-insensitively; the report must use one of these exact tier names — do not infer low severity from tone or wording alone): before concluding, generate the strongest counter-argument the code author could make to dismiss the finding. Then evaluate that counter-argument honestly. If the defense holds, downgrade or reject the finding. If it doesn't, the finding is reinforced. Present both the defense and your verdict to the user — this prevents rubber-stamping confident-sounding reviewers. If the review report uses no severity tiers at all, apply the author's defense to every finding.

**Mechanism transparency.** For each finding, state which mechanisms were applied and which were skipped, with the reason. Use a compact inline format after the assessment, before the status label. Examples:
- "Author's defense: applied — defense does not hold." / "Défense de l'auteur : appliquée — la défense ne tient pas."
- "Author's defense: skipped (finding classified Minor)." / "Défense de l'auteur : non appliquée (finding classé Minor)."
- "QA automatique : lancé (verdict incertain) — score 0.72, confirme le finding."
- "QA automatique : non lancé (verdict clair)."

This takes one line per mechanism — do not let it bloat the output.

State your assessment clearly. If the point is invalid or not worth fixing, say so with a short explanation. The user decides whether to skip it or act on it anyway. If the user chooses to act on a point you assessed as invalid, apply the fix without further pushback — your role is advisory.

### 2c. Fix (if warranted)

Apply the minimal, targeted correction. Rules:
- Only touch code directly related to this point.
- No opportunistic refactoring of surrounding code.
- No inline comments added to the code.
- Mirror the user's language (French or English) in any communication.
- If the correct fix requires changes beyond the scope of this single point (e.g., structural refactoring), flag it to the user instead of applying an incomplete fix. Let them decide whether to broaden the scope or skip. If they approve broadening, propose a short plan of the changes involved and get confirmation before applying. Then resume the normal walkthrough flow.

### 2d. Verify impacted files

After each fix, re-read the files you modified and files one level away. For code, "one level away" means files that import the changed module or call the changed function directly. For non-code files (SKILL.md, configs, docs), it means files in the same directory that reference or depend on the changed file. Check for:
- Broken references or imports
- Type mismatches or signature changes that affect callers
- Inconsistencies introduced between related files (e.g., a SKILL.md body that now contradicts an agent file)
- Tests that need updating

Do NOT expand this into a full project review. Stay scoped to the blast radius of your change.

If a regression is detected:
1. **Revert** all changes made for this point (across all files touched) immediately — do not leave broken code in place while discussing.
2. **Explain** the conflict clearly: what the fix changed, what broke, and why.
3. **Propose options**: (a) a different approach to fix the original finding without the regression, (b) skip the point and mark it DEFERRED with the regression as justification, or (c) accept the trade-off if the regression is minor relative to the fix. Let the user choose.

Also check whether the fix makes any of the remaining review points obsolete, already resolved, or partially addressed. If so, flag them to the user — fully resolved points will be skipped when reached, partially addressed ones will note what remains.

**Verification transparency.** Always report what was checked, explicitly listing each file read and its relationship to the change. Use a compact format:
> Vérification : `collector.py` (modifié), `pipeline.py` (importe collector), `test_collector.py` (teste collector) → OK, pas de régression.

or if no dependents exist:
> Vérification : `SKILL.md` (modifié), aucun fichier dépendant détecté.

If the fix was skipped (REJECTED/NOTED/DEFERRED with no code change), state explicitly: "Pas de modification → vérification non nécessaire." Do not silently skip this step.

### 2e. Wait for user approval

Before asking for approval, explicitly assign a status to this finding:

- **ACCEPTED** — the finding is valid and the fix was applied (or the code was already correct and no fix was needed, but the point is acknowledged)
- **REJECTED** — the finding is invalid, irrelevant, or not worth fixing. State why in one line.
- **DEFERRED** — the finding is valid but the fix is out of scope, too risky right now, or requires broader changes. State what would need to happen.
- **NOTED** — the finding is informational or a matter of taste. Acknowledged, no action taken.

The user has the final say — if they disagree with your proposed status, update it without pushback.

After presenting your assessment, status, and any changes, **always** stop and ask the user before moving to the next point — regardless of the status assigned (including DEFERRED). Use a brief prompt like:

- "Next point?" / "On passe au suivant ?"
- Or if a fix was applied: "Fix applied — ACCEPTED. OK to move on?" / "Correction faite — ACCEPTED. On continue ?"
- Or if deferred: "DEFERRED — [reason]. Move to the next point?" / "DEFERRED — [raison]. On passe au suivant ?"

Never auto-advance. Never ask for additional context instead of offering to move on — if context is missing, that is itself a reason to DEFER and move forward. The user might want to discuss, adjust, or revert before proceeding.

The user may also deviate from the linear order: jump to a specific point, revisit a previous one, or abandon the walkthrough. Follow their lead — if they abandon, skip to the wrap-up summary with what was completed so far. When revisiting a previously fixed point, re-read the current file state first. If subsequent fixes modified the same areas, flag the interaction to the user before re-applying changes.

## Step 3: Wrap up

After the last point (or if the user abandons mid-walkthrough), give a brief summary table:

| # | Finding | Status |
|---|---------|--------|
| 1 | (short description) | ACCEPTED / REJECTED / DEFERRED / NOTED |
| ... | ... | ... |

Follow with:
- Count by status (e.g., "4 accepted, 1 rejected, 2 deferred")
- List of DEFERRED items with their one-line justification — these are the user's follow-up backlog

After the status counts, add a **Mechanisms used** line summarizing what fired during the walkthrough. Example:
> Mechanisms: author's defense 4/6, QA auto 1 (finding #3), lateral think 0, evaluate ✓, drift check skipped (< 4 fixes).

If Ouroboros was not available, state: "Ouroboros: not available — walkthrough ran without automated QA, consensus, or drift check."

Keep it short — the user was there for the whole walkthrough.

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

- Be concise. No filler, no restating what the user already knows. Step 2b can produce substantial analysis for high-severity findings (re-evaluation + author's defense + verdict) — keep each section to 2-3 sentences max. The user needs your conclusion, not your reasoning process.
- When a finding is clearly wrong, say so directly — don't hedge excessively.
- When a finding is valid, fix it without editorializing.
- If unsure whether a point is valid, say so and let the user decide.
- Adapt to the user's language: if they speak French, respond in French; if English, respond in English.

## Ouroboros integration

When the Ouroboros plugin is available (tools like `ouroboros_evaluate`, `ouroboros_qa`, `ouroboros_lateral_think` appear in the available tool list), apply the following automatically at the specified trigger points. Do not ask for permission — invoke the tools and present the results inline. If these tools are not present, skip this entire section silently.

### During re-evaluation (Step 2b)

**`ouroboros_qa` — automatic second opinion on ambiguous findings.** When your re-evaluation is uncertain (you cannot confidently call a finding valid or invalid), invoke `ouroboros_qa` automatically to break the tie. Do not invoke for findings that are clearly valid or clearly false — only when you genuinely cannot decide.

```
ouroboros_qa(
  artifact: "<the code section under review, verbatim>",
  quality_bar: "<the finding's claim, phrased as a quality criterion — e.g. 'SQL queries must use parameterized inputs, not f-string interpolation'>",
  artifact_type: "code"
)
```
A score >= 0.8 means the code passes the quality bar (finding is likely a false positive). Below 0.8, the finding is likely valid. Present the score alongside your own assessment — the user sees both and decides.

**Advocate / Devil's Advocate pattern.** When the author's defense (see 2b) produces a counter-argument that you find plausible but cannot definitively confirm or reject, escalate automatically by invoking `ouroboros_qa` twice with opposing quality bars on the same artifact:
- Advocate (for the finding): `quality_bar` states why the current code is dangerous
- Devil's Advocate (against): `quality_bar` states why the current code is acceptable given context

Both calls use the same `artifact` (the code under review). Present both scores and verdicts to the user. This replaces gut-feel with structured deliberation on the hardest calls.

### When stuck (Step 2b–2c)

**`ouroboros_lateral_think` — automatic lateral thinking.** Trigger automatically when the walkthrough gets stuck on a point: two or more user exchanges on the same finding without reaching a fix, or the proposed fix introduces a regression (Step 2d revert). Do not invoke for simple mechanical disagreements — only for design-level impasses.

```
ouroboros_lateral_think(
  problem_context: "<what the finding asks to fix and why it's contentious>",
  current_approach: "<the fix approach that failed or is being debated>",
  persona: "<pick the best fit>"
)
```
Persona selection: `contrarian` if the assumption itself might be wrong, `simplifier` if the fix is over-engineered, `architect` if the root cause is structural, `hacker` for unconventional workarounds, `researcher` when more context is needed. Present the lateral angle to the user as a third option alongside the existing proposals.

### During wrap-up (Step 3)

**`ouroboros_evaluate` — automatic final validation.** Run automatically when at least 2 fixes were applied during the walkthrough. No need to ask.

```
ouroboros_evaluate(
  session_id: "review-walkthrough",
  artifact: "<summary of all changes applied during the walkthrough>",
  artifact_type: "code",
  acceptance_criterion: "<the original review's overall goal, e.g. 'all blocking findings resolved without regressions'>",
  trigger_consensus: <true if any fix was reverted due to regression during the walkthrough, false otherwise>,
  working_dir: "<path to the project root>"
)
```
When `trigger_consensus` is true, Ouroboros runs a multi-perspective deliberation (Advocate / Devil's Advocate / Judge). This works without external API keys — it falls back to single-model multi-perspective automatically if no `OPENROUTER_API_KEY` is configured.

Report the results as part of the wrap-up summary. If the evaluation flags regressions, list them after the summary table.

**`ouroboros_measure_drift` — automatic drift check.** Run automatically when 4 or more fixes were applied during the walkthrough.

```
ouroboros_measure_drift(
  session_id: "review-walkthrough",
  current_output: "<summary of the codebase state after all fixes>",
  seed_content: "<the original PR description or commit message that motivated the review>"
)
```
A drift score > 0.3 warrants a warning in the wrap-up — the fixes may have collectively shifted the code's purpose. Present the score and analysis after the summary table.
