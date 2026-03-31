---
name: review-walkthrough-batch-triage
description: Triages review findings into auto-fix, auto-reject, and manual buckets. Applies batch fixes with verification, runs post-fix hooks, and returns the manual bucket for interactive walkthrough.
---

# Review Walkthrough — Batch Triage

You accelerate large reviews by triaging findings into three buckets and batch-processing the obvious ones. You receive the extracted findings list and the deployment context, and return the manual bucket for the interactive walkthrough plus batch results for the wrap-up.

## Input

From the parent skill:
- **findings**: the full list of extracted findings (from Step 1), each with: number, description, file(s), severity tier (if any)
- **deployment context**: personal / internal / production (from orchestrator or user)
- **adversarial**: whether `--adversarial` is active

## 1. Rapid pre-verdict

For each finding, read the relevant code and form a quick assessment (1-2 sentences). Do **not** apply the author's defense — it is reserved for the manual walkthrough. Do **not** invoke Ouroboros QA.

Classify each finding into one bucket:

| Bucket | Criteria | Action |
|--------|----------|--------|
| **Auto-fix** | Clearly valid AND matches a mechanical pattern (whitelist below) AND NOT classified Blocking or Required | Fix applied in batch |
| **Auto-reject** | Clearly a false positive or out of context — contradicted by code, calibrated away by context detection, or obviously inapplicable | Rejected in batch (REJECTED) |
| **Manual** | Verdict uncertain, fix non-trivial, touches multiple files, Blocking/Required, or none of the above | Full individual walkthrough |

**Conservative default:** when in doubt, classify as Manual. Batch mode is an accelerator, not a filter.

**Blocking/Required findings always go to Manual**, regardless of how obvious the fix seems.

### Auto-fix whitelist

Only these mechanical patterns qualify. Everything else goes to Manual:
- Unused imports or variables (removal only)
- Missing type annotations on function signatures (addition only, no logic change)
- Typos in strings, comments, or identifiers (exact correction obvious from context)
- Missing `return` type annotations
- Trivial format/lint fixes explicitly flagged by the reviewer (trailing whitespace, missing newline at EOF)
- Dead code removal when the reviewer explicitly identifies the code as unreachable

If a finding matches a whitelist pattern but the fix would touch more than one file, it goes to Manual.

## 2. Present the triage

Show the user: a one-line summary (`Triage: N findings — X auto-fix, Y auto-reject, Z manual`), then three tables (auto-fix: `# | Finding | File | Fix`; auto-reject: `# | Finding | Reason`; manual: `# | Finding | File | Severity | Reason for manual`). End with `Override? (e.g., "move 3 to manual", "move 2 to auto-fix", or "ok")`. Wait for confirmation or overrides, apply them, then proceed.

## 3. Batch execution

**Auto-reject:** mark all as REJECTED with the one-line reason from the table. No code changes.

**Auto-fix:** apply fixes sequentially. After each fix, re-read the modified file + files one level away (same verification as Step 2d in the parent skill). If verification fails:
1. Revert the fix immediately.
2. Move the finding to Manual.
3. Continue with the next auto-fix.

### Post-fix hooks

After all auto-fix operations, check if any fix modified a dependency manifest (in a dependency section only). If the lock command fails, report the error — do not revert the dependency fix.

| File modified | Command |
|---------------|---------|
| `pyproject.toml` (deps) | `uv lock` (if uv.lock exists) or `pip-compile` |
| `package.json` (deps) | `npm install` or `yarn install` (detect from lockfile) |
| `Cargo.toml` (deps) | `cargo update` |
| `renv.lock` / DESCRIPTION (R) | `Rscript -e 'renv::snapshot()'` |

### Ouroboros QA on auto-reject (optional)

If Ouroboros is available, run a QA check on each auto-reject finding: `artifact` = the code section, `quality_bar` = the finding's claim. Score >= 0.8 confirms the rejection. Below 0.8, move the finding to Manual silently. Report the number of QA-promoted findings in the batch summary if any were moved.

## 4. Report and return

After all batch operations, report:

```
Batch complete: X/Y auto-fix applied, Z reverted → moved to manual.
W auto-reject confirmed.
Starting manual walkthrough: N findings.
```

## Output

Return to the parent skill:
- **manual bucket**: the list of findings for interactive walkthrough (Step 2)
- **batch results**: auto-fix and auto-reject outcomes for the wrap-up table (Step 3)
- **batch stats**: counts for the transparency status (auto-fix applied, reverted, auto-reject confirmed, QA-promoted)
