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

State the total number of points found, then start processing them in the order they appear in the report. Do not produce an upfront summary list of all findings — go straight to the first point.

## Step 2: Process each point

For each point, follow this exact sequence:

### 2a. Context

Briefly paraphrase the original finding. Quote verbatim only when the exact wording matters. Identify the file(s) and line(s) involved. Read the relevant code so you have the current state in front of you.

### 2b. Re-evaluate

Assess the finding critically and honestly:
- Is the issue real, or is it a false positive?
- Is it relevant given the project's context and conventions?
- Is the severity appropriate?
- Is the suggested fix (if any) the right approach?

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

If a regression is detected, revert all changes made for this point (across all files touched), explain the conflict to the user, and let them decide how to proceed (different approach, skip the point, or accept the trade-off).

Also check whether the fix makes any of the remaining review points obsolete, already resolved, or partially addressed. If so, flag them to the user — fully resolved points will be skipped when reached, partially addressed ones will note what remains.

Report what you checked and whether anything else needs attention.

### 2e. Wait for user approval

After presenting your assessment and any changes, stop and ask the user before moving to the next point. Use a brief prompt like:

- "Next point?" / "On passe au suivant ?"
- Or if a fix was applied: "Fix applied. OK to move on?" / "Correction faite. On continue ?"

Never auto-advance. The user might want to discuss, adjust, or revert before proceeding.

The user may also deviate from the linear order: jump to a specific point, revisit a previous one, or abandon the walkthrough. Follow their lead — if they abandon, skip to the wrap-up summary with what was completed so far. When revisiting a previously fixed point, re-read the current file state first. If subsequent fixes modified the same areas, flag the interaction to the user before re-applying changes.

## Step 3: Wrap up

After the last point, give a brief summary:
- How many points were addressed, skipped, or flagged as invalid
- Any remaining items that need follow-up

Keep it short — the user was there for the whole walkthrough.

## Behavioral notes

- Be concise. No filler, no restating what the user already knows.
- When a finding is clearly wrong, say so directly — don't hedge excessively.
- When a finding is valid, fix it without editorializing.
- If unsure whether a point is valid, say so and let the user decide.
- Adapt to the user's language: if they speak French, respond in French; if English, respond in English.
