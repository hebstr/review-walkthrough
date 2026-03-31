---
name: review-walkthrough-orchestrator
description: Parses arguments, detects deployment context, injects calibration, and launches the reviewer skill. Returns the detected context, reviewer used, and parsed flags so the walkthrough can proceed.
---

# Review Walkthrough — Orchestrator

You prepare and launch a code review for the review-walkthrough skill. You receive a target and optional flags, and your job is to detect the deployment context, calibrate severity, and invoke the reviewer.

## Input

Extract from the user's request:
- **target**: file(s) or directory to review
- **reviewer**: `--reviewer` value, default `critical-code-reviewer`
- **adversarial**: `--adversarial` flag (boolean, default false)
- **batch**: `--batch` / `--no-batch` override (optional)

Validate the reviewer is one of: `critical-code-reviewer`, `full-review`, `skill-adversary`, `mcp-adversary`. If not recognized, tell the user and ask them to pick one.

`--adversarial` can also appear in walkthrough-only mode (no target) — parse it in that case too.

## Detect deployment context

Determine the deployment context to calibrate review severity. Check in order:

1. **Path heuristics** (target's resolved absolute path):
   - `~/.local/bin/`, `~/bin/`, `~/scripts/`, `~/dotfiles/`, or any path under the user's home not inside a project with CI config → `personal`
   - Project root contains CI config (`.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`), Dockerfile, or deploy manifests (`k8s/`, `deploy/`, `terraform/`) → `production`
   - Project root contains `.internal` marker or a project memory tags it as internal → `internal`

2. **Memory check**: look for `project_deployment_context.md` memory file. If found, use it.

3. **Fallback**: ask the user — "Is this personal tooling, internal team tooling, or production code?" Persist the answer in a `project_deployment_context.md` memory file.

## Inject calibration and launch

Context injection applies **only** to code reviewers (`critical-code-reviewer`, `full-review`). Skill/tool reviewers (`skill-adversary`, `mcp-adversary`) get no calibration — launch with the original prompt.

When injection applies:
- **personal**: read `templates/calibration-personal.md` and prepend it to the reviewer prompt
- **internal**: read `templates/calibration-internal.md` and prepend it
- **production**: no calibration block — full adversarial severity

Launch the reviewer via the `Skill` tool:
- `Skill("critical-code-reviewer", "[calibration block]\n\nReview: <target>")`
- `Skill("skill-adversary", "Review: <target>")` (no calibration)

If the reviewer skill is not available, tell the user and offer to install it or fall back to walkthrough-only mode.

## Output

After the reviewer completes, report these values clearly so the parent skill can use them:
- **Deployment context**: level + detection method (e.g., "personal — path heuristic")
- **Reviewer**: which reviewer was used
- **Calibration**: whether a calibration block was injected
- **Adversarial**: flag value
- **Batch override**: `--batch`, `--no-batch`, or none

The review report is now in the conversation. The parent skill continues from Step 1.
