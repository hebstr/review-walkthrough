---
name: review-walkthrough
description: >
  Interactive, point-by-point walkthrough of a review report produced by any review skill (skill-adversary, critical-code-reviewer, or any other). Can operate as a full orchestrator (provide a target to review and optionally a --reviewer flag — it detects deployment context, calibrates severity, launches the reviewer, then walks through its report) or in walkthrough-only mode (processes an existing report from the conversation). Parses the review findings, then processes each point one at a time: re-evaluates validity, proposes and applies fixes, checks impacted files for regressions, and waits for user approval before moving on.

  This skill has two modes: **orchestrator mode** (launches a reviewer then walks through its report) and **walkthrough-only mode** (processes an existing report in the conversation).

  **Orchestrator mode** — trigger when the user provides a target (file, directory, or glob) to review, optionally with a `--reviewer` flag. Phrases: "review-walkthrough src/", "review-walkthrough ~/scripts/backup.sh --reviewer full-review", "review et walkthrough de ce fichier", "lance une review sur X puis on passe dessus", or any request that combines reviewing and walking through results in one step. Default reviewer: `critical-code-reviewer`. Supported reviewers: `critical-code-reviewer`, `full-review`, `skill-adversary`, `mcp-adversary`.

  **Walkthrough-only mode** — trigger when the user wants to act on review results iteratively — phrases like "walkthrough the review", "let's go through the review point by point", "contre-review", "fix the review points one by one", "parcours les points de la review", "on passe sur chaque point", "régler les points un par un", "traiter les résultats de la review", "valider chaque point ensemble", "let's address the review findings", "apply the suggestions from the review", or any request to process/fix/handle/triage/validate/filter review results step by step. French variants: "passer en revue les résultats", "corriger les points", "trier les findings". Also trigger when the user pastes or references an external review (from a coworker, a tool, or any other source) and asks to act on the findings. Also trigger when a review skill just ran in the current conversation and the user asks to work through the results, even casually ("ok let's fix these", "on s'y met", "go through them", "let's do it", "yep fix them", "on y va", "let's tackle these") — but ONLY when a structured review report with discrete findings exists earlier in the conversation; do not trigger on these casual phrases in isolation. Do NOT trigger for: code walkthroughs or architecture explanations, academic/paper reviews, PR comment discussions, linter or CI output processing, or Jira/backlog issue lists — even if the user's phrasing matches the casual triggers above.
---

# Review Walkthrough

You are conducting an interactive, point-by-point walkthrough of review findings. Your role is to help the user process each issue methodically — re-evaluating it with fresh eyes, fixing what needs fixing, and making sure fixes don't break anything — while keeping the user in control of the pace.

## Step 0: Orchestrate (only when a target is provided)

If the user provided a target (file, directory, or glob) to review, execute this step. If no target was provided (the user wants to walk through an existing report), skip directly to Step 1.

### 0a. Parse arguments

Extract from the user's request:
- **target**: the file(s) or directory to review
- **reviewer**: the `--reviewer` value if provided, otherwise default to `critical-code-reviewer`

Validate that the reviewer is one of: `critical-code-reviewer`, `full-review`, `skill-adversary`, `mcp-adversary`. If not recognized, tell the user and ask them to pick one.

### 0b. Detect deployment context

Determine the deployment context of the target to calibrate review severity. Use these heuristics in order:

1. **Path heuristics** (check the target's resolved absolute path):
   - `~/.local/bin/`, `~/bin/`, `~/scripts/`, `~/dotfiles/`, or any path under the user's home that is not inside a project with CI config → `personal`
   - Project root contains CI config (`.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`), Dockerfile, or deploy manifests (`k8s/`, `deploy/`, `terraform/`) → `production`
   - Project root contains an `.internal` marker file, or a project memory tags it as internal → `internal`

2. **Memory check**: look for a `project_deployment_context.md` memory file for this project. If it exists, use the stored context level.

3. **Fallback**: if no heuristic matches and no memory exists, ask the user: "Is this personal tooling, internal team tooling, or production code?" Persist the answer in a `project_deployment_context.md` memory file for future use.

### 0c. Inject context calibration (conditional)

Context injection applies **only** to reviewers that evaluate application code:
- `critical-code-reviewer` — yes
- `full-review` — yes
- `skill-adversary` — no (reviews skill artefacts, not application code)
- `mcp-adversary` — no (reviews MCP server schemas, not application code)

When context injection applies, prepend a calibration block to the reviewer's prompt. The block depends on the detected context:

**personal:**
> Context: this is a personal utility script/tool on a single-user workstation. Calibrate severity accordingly: flag real bugs, data-loss risks, and logic errors. Do NOT flag: theoretical attack vectors requiring same-UID or physical access, portability for non-target platforms, defensive coding patterns for conditions that cannot occur in this single-user context, missing input validation for inputs the author controls, or production-grade error handling for scripts that can just crash.

**internal:**
> Context: this is internal team tooling on a trusted network. Calibrate severity accordingly: flag bugs, security issues, correctness problems, and maintainability concerns. Do NOT flag: portability beyond the team's stack, or public-facing hardening patterns. DO flag: injection risks if the tool processes any external input, authentication/authorization gaps, and data integrity issues.

**production:**
No calibration block — the reviewer runs at full adversarial severity.

When context injection does not apply (skill/tool reviewers), launch the reviewer with its original prompt unmodified.

### 0d. Launch the reviewer

Invoke the reviewer skill via the `Skill` tool, passing the target and (if applicable) the calibration block as a prefix to the prompt. Examples:
- `Skill("critical-code-reviewer", "Context: this is a personal utility... [calibration block]\n\nReview: ~/scripts/backup.sh")`
- `Skill("full-review", "Context: this is production code.\n\nReview: src/")`
- `Skill("skill-adversary", "Review: SKILL.md")` (no calibration)

Wait for the reviewer to complete and produce its report. The report now exists in the conversation — proceed to Step 1 to walk through it.

If the reviewer skill is not available (not installed, not in the skill list), tell the user and offer to either install it or fall back to walkthrough-only mode on an existing report.

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
- **Ouroboros**: "available", "not available", or "not available (runtime error)" (detect using the two-step probe described in the Ouroboros integration section). If available, also report whether multi-model consensus is enabled (test by checking if `OPENROUTER_API_KEY` is set in the environment — if not, note "consensus: single-model fallback").
- **Author's defense**: "active on N/N findings" — count how many findings will trigger the defense based on severity tiers (all findings minus those explicitly in the low-severity exclusion list). If all findings trigger it, say "active on all findings". If none (all are low-severity), say "skipped — all findings are low-severity".
- **Severity reordering**: "applied" (if reordering happened) or "original order preserved" (if no tiers detected).
- **Batch mode**: "active (N findings >= 15)" when Step 1b will run, "inactive (N findings < 15)" when it won't, or "forced via --batch" / "disabled via --no-batch" when overridden by the user.

If Ouroboros is available, add a brief glossary of the mechanisms that may fire during the walkthrough, so the user understands the transparency lines they will see later:

> **Ouroboros mechanisms available for this walkthrough:**
> - *QA auto* — automated second opinion when the verdict on a finding is genuinely uncertain
> - *Advocate / Devil's advocate* — contradictory double evaluation to stress-test borderline calls
> - *Lateral think* — creative unblocking when a point stays stuck after 2+ exchanges
> - *Evaluate* — final validation of all applied changes (triggers when ≥ 2 fixes)
> - *Drift check* — detects whether cumulative fixes shifted the code away from its original intent (triggers when ≥ 4 fixes)

This glossary appears only once, before the first finding. Adapt the language to match the user's (French or English). Keep it compact — one line per mechanism, no elaboration.

Keep the status block itself to 2-4 short lines. Example:
> Context: personal (detected from path ~/scripts/). Reviewer: critical-code-reviewer (calibrated). Ouroboros available (consensus: single-model fallback). Author's defense active on 4/6 findings. Severity reordering applied — 2 Blocking first. Batch mode: active (32 findings ≥ 15).

## Step 1b: Triage and batch processing

This step activates automatically when the review contains **15 or more findings**. Below 15, skip directly to Step 2. The user can force batch mode with `--batch` (active regardless of count) or suppress it with `--no-batch`.

When active, perform a rapid triage of all findings into three buckets before starting the interactive walkthrough. The goal is to accelerate obvious verdicts while preserving full scrutiny for anything ambiguous or high-stakes.

### 1b.1 Rapid pre-verdict

For each finding, read the relevant code and form a quick assessment (1-2 sentences). Do **not** apply the author's defense — it is reserved for the manual walkthrough to avoid same-model self-conviction. Do **not** invoke Ouroboros QA at this stage.

Classify each finding into one of three buckets:

| Bucket | Criteria | Action |
|--------|----------|--------|
| **Auto-fix** | The finding is clearly valid **AND** the fix matches a mechanical pattern (see whitelist below) **AND** the finding is NOT classified Blocking or Required by the reviewer | Fix applied in batch |
| **Auto-reject** | The finding is clearly a false positive or out of context — contradicted by the code, already calibrated away by context detection, or obviously inapplicable | Rejected in batch (REJECTED) |
| **Manual** | Verdict is uncertain, fix is non-trivial, touches multiple files, finding is Blocking/Required, or none of the above criteria are met | Full individual walkthrough (Step 2) |

**Conservative default:** when in doubt, classify as Manual. Batch mode is an accelerator, not a filter.

**Auto-fix whitelist.** Only the following mechanical patterns qualify for auto-fix. Everything else goes to Manual, even if the finding seems straightforward:
- Unused imports or variables (removal only)
- Missing type annotations on function signatures (addition only, no logic change)
- Typos in strings, comments, or identifiers (exact correction obvious from context)
- Missing `return` type annotations
- Trivial format/lint fixes explicitly flagged by the reviewer (e.g., trailing whitespace, missing newline at EOF)
- Dead code removal when the reviewer explicitly identifies the code as unreachable

If a finding matches a whitelist pattern but the fix would touch more than one file, it goes to Manual.

**Blocking/Required findings always go to Manual**, regardless of how obvious the fix seems. The reviewer's high severity warrants human review.

### 1b.2 Present the triage

Show the user a summary of the triage, then the three tables (auto-fix, auto-reject, manual):

```
Triage: 32 findings analyzed.
- 12 auto-fix (mechanical corrections, applied in batch)
- 8 auto-reject (false positives, rejected in batch)
- 12 manual (individual walkthrough)

Auto-fix:
| # | Finding | File | Fix |
|---|---------|------|-----|
| 3 | Unused import | api.ts:1 | Remove `lodash` import |
| 7 | Missing return type | utils.ts:42 | Add `: void` |
| ... | ... | ... | ... |

Auto-reject:
| # | Finding | Reason |
|---|---------|--------|
| 2 | XSS via innerHTML | Personal script, no untrusted input |
| 6 | No CSRF token | CLI tool, no web interface |
| ... | ... | ... |

Manual (individual walkthrough):
| # | Finding | File | Severity | Reason for manual |
|---|---------|------|----------|-------------------|
| 1 | HTML escaping in generate_html | generate_html.py:45 | Blocking | High severity |
| 5 | Fragment stripping in normalize_url | normalize_url.py:12 | Required | Non-trivial fix |
| ... | ... | ... | ... | ... |

Override? (e.g., "move 3 to manual", "move 2 to auto-fix", or "ok")
```

Wait for the user to confirm or override. The user can move any finding between buckets by number. Apply all overrides, then proceed.

### 1b.3 Batch execution

**Auto-reject:** mark all auto-reject findings as REJECTED with the one-line reason from the table. No code changes.

**Auto-fix:** apply fixes sequentially. After each fix, run the same verification as Step 2d (re-read modified file + files one level away). If verification fails:
1. Revert the fix immediately.
2. Move the finding to Manual.
3. Continue with the next auto-fix.

**Post-fix hooks.** After all auto-fix operations, check if any fix modified a dependency manifest. Only trigger when the modification is in a dependency section. If the lock command fails, report the error — do not revert the dependency fix.

| File modified | Command |
|---------------|---------|
| `pyproject.toml` (deps) | `uv lock` (if uv.lock exists) or `pip-compile` |
| `package.json` (deps) | `npm install` or `yarn install` (detect from lockfile) |
| `Cargo.toml` (deps) | `cargo update` |
| `renv.lock` / DESCRIPTION (R) | `Rscript -e 'renv::snapshot()'` |

After all batch operations, report a compact summary:

```
Batch complete: 11/12 auto-fix applied, 1 reverted → moved to manual.
8 auto-reject confirmed.
Starting manual walkthrough: 13 findings.
```

**Ouroboros QA on auto-reject (optional).** If Ouroboros is available, run a QA check on each auto-reject finding: `artifact` = the code section, `quality_bar` = the finding's claim. Score >= 0.8 confirms the rejection. Below 0.8, move the finding to Manual silently (it will be re-evaluated with full scrutiny). Report the number of QA-promoted findings in the batch summary if any were moved.

### 1b.4 Transition to manual walkthrough

After batch execution, proceed to Step 2 with only the Manual bucket. The transparency status line and finding count reflect the manual subset. Batch results (auto-fix and auto-reject) appear in the final wrap-up table (Step 3) alongside manual results.

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

**Author's defense.** For every finding **unless** it is explicitly classified as one of the following low-severity tiers by the review report: Suggestion, Minor, Low, Info, Cosmetic, Nit, Note, Nitpick, Style, Informational, Optional, Enhancement, Wish, or Consider (match case-insensitively; the report must use one of these exact tier names — do not infer low severity from tone or wording alone): before concluding, generate the strongest counter-argument the code author could make to dismiss the finding. Then evaluate that counter-argument honestly. If the defense holds, downgrade or reject the finding. If it doesn't, the finding is reinforced. Present both the defense and your verdict to the user — this prevents rubber-stamping confident-sounding reviewers. If the review report uses no severity tiers at all, apply the author's defense to every finding.

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

If the fix modified a dependency manifest (`pyproject.toml`, `package.json`, `Cargo.toml`, `DESCRIPTION`, `renv.lock`), run the appropriate lock command (see post-fix hooks table in 1b.3) before continuing verification.

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
> **Mechanisms:** batch triage 20/32 (12 auto-fix, 8 auto-reject) · author's defense 10/11 · QA auto 0/22 (no ambiguous verdicts) · lateral think 0 (no stuck points or regressions) · evaluate ✓ (score 0.88, based on git diff of 4 files) · drift skipped (< 4 fixes)

If `ouroboros_evaluate` produces a misleading result (e.g. because the artifact was a text summary rather than actual code), flag it explicitly — do not present the score without context.

If Ouroboros was not available, state: "Ouroboros: not available — walkthrough ran without automated QA, consensus, or drift check."

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
- Adapt to the user's language: if they speak French, respond in French; if English, respond in English.

## Ouroboros integration

When the Ouroboros plugin is available, apply the following automatically at the specified trigger points. Do not ask for permission — invoke the tools and present the results inline. If these tools are not present, skip this entire section silently.

**Detection:** Ouroboros tools are registered as MCP tools with prefixed names (e.g. `mcp__plugin_ouroboros_ouroboros__ouroboros_qa`). They may also be deferred (not yet loaded). To detect availability:
1. Run `ToolSearch` with query `+ouroboros qa` at the start of the walkthrough (Step 1). If no results, Ouroboros is not available — skip all Ouroboros integration silently.
2. If ToolSearch returns results, **probe** by calling `ouroboros_qa` with `artifact: "probe"`, `quality_bar: "probe"`. If the call succeeds, Ouroboros is confirmed available. If it returns an error (e.g. missing dependency, SDK not installed), treat Ouroboros as **not available** for the rest of the walkthrough — report it in the transparency status as "not available (runtime error)" and skip all Ouroboros calls silently. Do not retry or ask the user to install anything.

**Runtime errors.** If any Ouroboros tool call fails during the walkthrough (Steps 2–3) after passing the initial probe, catch the error, report it inline as "QA automatique : erreur — [one-line reason]. Skipped." and continue the walkthrough without it. Never let an Ouroboros failure block the walkthrough.

### During re-evaluation (Step 2b)

**`ouroboros_qa` — automatic second opinion on ambiguous findings.** When your re-evaluation is uncertain (you cannot confidently call a finding valid or invalid), invoke the ouroboros QA tool (use the MCP tool name returned by ToolSearch, e.g. `mcp__plugin_ouroboros_ouroboros__ouroboros_qa`) automatically to break the tie. Do not invoke for findings that are clearly valid or clearly false — only when you genuinely cannot decide. Parameters: `artifact` (the code section under review, verbatim), `quality_bar` (the finding's claim phrased as a quality criterion), `artifact_type` ("code").
A score >= 0.8 means the code passes the quality bar (finding is likely a false positive). Below 0.8, the finding is likely valid. Present the score alongside your own assessment — the user sees both and decides.

**Advocate / Devil's Advocate pattern.** When the author's defense (see 2b) produces a counter-argument that you find plausible but cannot definitively confirm or reject, escalate automatically by invoking the ouroboros QA tool twice with opposing quality bars on the same artifact:
- Advocate (for the finding): `quality_bar` states why the current code is dangerous
- Devil's Advocate (against): `quality_bar` states why the current code is acceptable given context

Both calls use the same `artifact` (the code under review). Present both scores and verdicts to the user. This replaces gut-feel with structured deliberation on the hardest calls.

### When stuck (Step 2b–2c)

**`ouroboros_lateral_think` — automatic lateral thinking.** Trigger automatically when the walkthrough gets stuck on a point: two or more user exchanges on the same finding without reaching a fix, or the proposed fix introduces a regression (Step 2d revert). Do not invoke for simple mechanical disagreements — only for design-level impasses. Use the MCP tool name returned by ToolSearch. Parameters: `problem_context` (what the finding asks to fix and why it's contentious), `current_approach` (the fix approach that failed or is being debated), `persona` (pick the best fit).

Persona selection: `contrarian` if the assumption itself might be wrong, `simplifier` if the fix is over-engineered, `architect` if the root cause is structural, `hacker` for unconventional workarounds, `researcher` when more context is needed. Present the lateral angle to the user as a third option alongside the existing proposals.

### During wrap-up (Step 3)

**`ouroboros_evaluate` — automatic final validation.** Run automatically when at least 2 fixes were applied during the walkthrough. No need to ask. Use the MCP tool name returned by ToolSearch. Before calling, build the `artifact` from actual content — not a prose summary:
- **Code files modified**: run `git diff` on files touched during the walkthrough and use the diff output. Include only the diff output as the artifact — never the full file content. Prefix the artifact with: "Evaluate ONLY the changes shown in this diff. Do not flag pre-existing issues that are not part of the diff." If the diff exceeds ~4000 lines, truncate to the most impactful files (Blocking/Required fixes) and note the truncation.
- **Non-code files modified** (docs, configs, YAML, SKILL.md…): read the final state of modified files and concatenate them. If too large, include only the modified sections with surrounding context.
- **Mixed**: combine both approaches. Use `artifact_type` "code" if the majority of changes are code, "document" otherwise.
- **Fallback**: a prose summary is acceptable only if the content cannot be extracted (e.g., changes were reverted and re-applied outside git tracking). In that case, flag explicitly in the wrap-up that evaluate received a summary, not actual content, and the score should be interpreted with caution.

Other parameters: `session_id` ("review-walkthrough"), `acceptance_criterion` (the original review's overall goal + " Changes introduced during this review walkthrough only — pre-existing issues are out of scope."), `trigger_consensus` (true if any fix was reverted due to regression, false otherwise), `working_dir` (path to the project root).

When `trigger_consensus` is true, Ouroboros runs a multi-perspective deliberation (Advocate / Devil's Advocate / Judge). This works without external API keys — it falls back to single-model multi-perspective automatically if no `OPENROUTER_API_KEY` is configured.

**Post-filter on evaluate results.** If evaluate flags issues, cross-reference each against the diff. If a flagged issue is on a line not modified by the walkthrough (not in the diff hunk), discard it and note: "N pre-existing issues filtered from evaluate results."

Report the results as part of the wrap-up summary. If the evaluation flags regressions, list them after the summary table.

**`ouroboros_measure_drift` — automatic drift check.** Run automatically when 4 or more fixes were applied during the walkthrough. Use the MCP tool name returned by ToolSearch. Parameters: `session_id` ("review-walkthrough"), `current_output` (summary of the codebase state after all fixes), `seed_content` (the original PR description or commit message that motivated the review).

A drift score > 0.3 warrants a warning in the wrap-up — the fixes may have collectively shifted the code's purpose. Present the score and analysis after the summary table.
