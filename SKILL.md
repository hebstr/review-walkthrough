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

**Author's defense.** For findings classified as Blocking or Required (or equivalent high-severity): before concluding, generate the strongest counter-argument the code author could make to dismiss the finding. Then evaluate that counter-argument honestly. If the defense holds, downgrade or reject the finding. If it doesn't, the finding is reinforced. Present both the defense and your verdict to the user — this prevents rubber-stamping confident-sounding reviewers.

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

Report what you checked and whether anything else needs attention.

### 2e. Wait for user approval

Before asking for approval, explicitly assign a status to this finding:

- **ACCEPTED** — the finding is valid and the fix was applied (or the code was already correct and no fix was needed, but the point is acknowledged)
- **REJECTED** — the finding is invalid, irrelevant, or not worth fixing. State why in one line.
- **DEFERRED** — the finding is valid but the fix is out of scope, too risky right now, or requires broader changes. State what would need to happen.
- **NOTED** — the finding is informational or a matter of taste. Acknowledged, no action taken.

The user has the final say — if they disagree with your proposed status, update it without pushback.

After presenting your assessment, status, and any changes, stop and ask the user before moving to the next point. Use a brief prompt like:

- "Next point?" / "On passe au suivant ?"
- Or if a fix was applied: "Fix applied — ACCEPTED. OK to move on?" / "Correction faite — ACCEPTED. On continue ?"

Never auto-advance. The user might want to discuss, adjust, or revert before proceeding.

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

Keep it short — the user was there for the whole walkthrough.

## Behavioral notes

- Be concise. No filler, no restating what the user already knows. Step 2b can produce substantial analysis for high-severity findings (re-evaluation + author's defense + verdict) — keep each section to 2-3 sentences max. The user needs your conclusion, not your reasoning process.
- When a finding is clearly wrong, say so directly — don't hedge excessively.
- When a finding is valid, fix it without editorializing.
- If unsure whether a point is valid, say so and let the user decide.
- Adapt to the user's language: if they speak French, respond in French; if English, respond in English.

## Ouroboros integration (optional)

When the Ouroboros plugin is available (check: tools like `ouroboros_evaluate`, `ouroboros_qa`, `ouroboros_lateral_think` appear in the available tool list), offer the following as opt-in options at specific moments. Never invoke them automatically — always propose and let the user decide. If these tools are not present, skip this entire section silently.

### During re-evaluation (Step 2b)

**`ouroboros_qa` — second opinion on ambiguous findings.** When your re-evaluation is uncertain (you cannot confidently call a finding valid or invalid), offer to run a QA verdict for a structured second opinion. Do not offer this for findings that are clearly valid or clearly false — only for genuinely ambiguous cases.

How to invoke:
```
ouroboros_qa(
  artifact: "<the code section under review, verbatim>",
  quality_bar: "<the finding's claim, phrased as a quality criterion — e.g. 'SQL queries must use parameterized inputs, not f-string interpolation'>",
  artifact_type: "code"
)
```
A score >= 0.8 means the code passes the quality bar (finding is likely a false positive). Below 0.8, the finding is likely valid. Present the score and the tool's suggestions to the user alongside your own assessment.

**Advocate / Devil's Advocate pattern.** For Blocking or Required findings where the author's defense (see 2b) produces a compelling counter-argument, offer to escalate to a structured adversarial evaluation. Invoke `ouroboros_qa` twice with opposing quality bars on the same artifact:
- Advocate (for the finding): `quality_bar: "The retry logic must use exponential backoff with jitter to avoid thundering herd — fixed delay is a production risk"`
- Devil's Advocate (against): `quality_bar: "Fixed 100ms retry is acceptable here because the upstream service has a 5s timeout and max 3 retries caps total wait at 300ms"`

Both calls use the same `artifact` (the code under review). Present both scores and verdicts to the user and let them judge. This replaces gut-feel with structured deliberation on the hardest calls.

### When stuck

**`ouroboros_lateral_think` — lateral thinking.** If the walkthrough gets stuck on a point — two or more exchanges without resolution, the user and you disagree on the right approach, or the fix has non-obvious trade-offs — offer to invoke a lateral thinking persona for a fresh angle. This is specifically for design disagreements, not for mechanical fixes.

How to invoke:
```
ouroboros_lateral_think(
  problem_context: "<what the finding asks to fix and why it's contentious>",
  current_approach: "<the fix approach currently being debated>",
  persona: "contrarian"  // or "simplifier", "architect", "hacker", "researcher"
)
```
Pick the persona that fits the impasse: `contrarian` if the assumption itself might be wrong, `simplifier` if the fix is over-engineered, `architect` if the root cause is structural, `hacker` for unconventional workarounds, `researcher` when more context is needed. If unsure, let the user choose.

### During wrap-up (Step 3)

**`ouroboros_evaluate` — final validation.** For walkthroughs on production or critical code (regulatory, clinical, shared pipelines), offer to run a structured evaluation on the modified files as a final check. Do not offer this for exploratory code or minor fixes.

How to invoke:
```
ouroboros_evaluate(
  session_id: "review-walkthrough",  // arbitrary string — use a descriptive label, no active Ouroboros session required
  artifact: "<summary of all changes applied during the walkthrough>",
  artifact_type: "code",
  acceptance_criterion: "<the original review's overall goal, e.g. 'all blocking findings resolved without regressions'>",
  trigger_consensus: true,  // set to true for critical code, false otherwise
  working_dir: "<path to the project root>"  // so mechanical checks (lint, tests) run in the right directory
)
```
This runs mechanical checks (stage 1), semantic alignment (stage 2), and optionally multi-model consensus (stage 3). Report the results to the user as part of the wrap-up.

**`ouroboros_measure_drift` — drift check.** If more than 5 fixes were applied during the walkthrough, offer to measure whether the cumulative changes have drifted from the original intent.

How to invoke:
```
ouroboros_measure_drift(
  session_id: "review-walkthrough",  // same label as ouroboros_evaluate, no active session required
  current_output: "<summary of the codebase state after all fixes>",
  seed_content: "<the original PR description or commit message that motivated the review>"
)
```
A drift score > 0.3 warrants discussion with the user — the fixes may have collectively shifted the code's purpose. Present the score and the tool's analysis.
