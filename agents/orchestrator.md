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
   - Project root contains `.internal` marker or a project memory tags it as internal → `internal`

2. **CI classification** (if `.github/workflows/`, `.gitlab-ci.yml`, or `Jenkinsfile` exists):

   Split CI signals into two categories:

   - **CI deploy** (triggers `production`): Dockerfile, `docker-compose.yml`, `k8s/`, `deploy/`, `terraform/`, `helm/`, `.elasticbeanstalk/`, `appspec.yml`, `fly.toml`, `render.yaml`, or workflow files whose name contains `deploy`, `release`, or `cd` (case-insensitive).
   - **CI quality-only** (triggers `internal`): everything else — test, check, lint, coverage, docs workflows (e.g. `R-CMD-check.yaml`, `pytest.yml`, `pkgdown.yaml`, `test-coverage.yaml`, `eslint.yml`). These indicate project hygiene, not deployment.

   If **any** CI deploy signal is found → `production`. If CI exists but **only** quality signals → `internal`.

3. **Memory check**: look for `project_deployment_context.md` memory file. If found, use it.

4. **Fallback**: ask the user — "Is this personal tooling, internal team tooling, or production code?" Persist the answer in a `project_deployment_context.md` memory file.

## Load target project memories

The review-walkthrough skill runs from its own working directory, not from the target project. This means Claude Code's automatic project memory loading does **not** include the target's memories. You must load them explicitly.

1. Resolve the target's absolute path (e.g., `/home/julien/Documents/pro/r_pkg/pkg_edstr`).
2. Derive the Claude Code project memory directory: `~/.claude/projects/<encoded-path>/memory/`, where `<encoded-path>` is the absolute path with `/` replaced by `-` and leading `-` preserved (e.g., `/home/julien/Documents/pro/r_pkg/pkg_edstr` → `-home-julien-Documents-pro-r-pkg-pkg-edstr`).
3. Check if `feedback_review_severity.md` exists in that directory. If it does, read it — this contains reviewer calibration rules from prior sessions (dismissed false positives, R idioms not to flag, etc.).
4. Check if `MEMORY.md` exists in that directory. If it does, scan it for other feedback-type memories that might be relevant to the review (e.g., `feedback_code_text_english.md`). Read any that seem review-relevant.
5. Collect all loaded memory content into a `[prior calibration]` block.

If no target project memories exist, skip this step — the reviewer runs without prior calibration context.

## Inject calibration and launch

Context injection applies **only** to code reviewers (`critical-code-reviewer`, `full-review`). Skill/tool reviewers (`skill-adversary`, `mcp-adversary`) get no calibration — launch with the original prompt.

When injection applies:
- **personal**: read `templates/calibration-personal.md` and prepend it to the reviewer prompt
- **internal**: read `templates/calibration-internal.md` and prepend it
- **production**: no calibration block — full adversarial severity

**Prior calibration** (from target project memories) is injected for **all** code reviewers, regardless of deployment context. It supplements the context-based calibration, not replaces it. Append it after the context calibration block (or as the only calibration if context is production).

**Launch the reviewer as a foreground Agent** (not a Skill). This is critical — running the reviewer in the same context window as the walkthrough exhausts the context budget and causes the walkthrough to silently abort. The Agent isolates the reviewer's work (file reads, sub-agents, bash commands) and returns only the final report.

Build the Agent prompt as follows:

```
[context calibration block, if applicable]

[prior calibration block, if found — prefix with "Prior review calibration for this project (from previous sessions):"]

You are running the <reviewer> skill. Review: <target>

IMPORTANT — output format: return ONLY the structured findings report. Do not include your intermediate reasoning, file contents you read, or tool call results. Each finding must include: severity tier, file(s) and line(s), description, and suggested fix. Keep each finding to one short paragraph. Maximum 15 findings — if more exist, keep the 15 highest-severity ones and note how many were omitted. Do NOT report findings that contradict the prior calibration rules above — those patterns have been explicitly validated by the project author.
```

Launch with `Agent(prompt, description="review <target>")` in **foreground** mode — the walkthrough cannot proceed without the report.

For skill/tool reviewers (no calibration):
```
You are running the <reviewer> skill. Review: <target>

[same output format instructions as above]
```

If the reviewer agent fails or returns an empty result, tell the user and offer to retry or fall back to walkthrough-only mode.

## Output

After the reviewer Agent returns its report, emit the following structured block **exactly** (the parent skill parses it), followed by the Agent's report verbatim:

```
--- ORCHESTRATOR COMPLETE ---
context: <level> (<detection method>)
reviewer: <reviewer name>
calibrated: <yes|no>
adversarial: <true|false>
batch: <--batch|--no-batch|none>
--- REVIEW REPORT ---
<paste the Agent's returned report here, unmodified>
--- PROCEED TO STEP 1 ---
```

Do not stop after this block. Do not summarize the review. Do not ask the user what to do next. The parent skill takes over immediately.
