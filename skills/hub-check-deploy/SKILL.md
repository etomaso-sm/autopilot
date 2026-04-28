---
name: hub-check-deploy
description: >
  Use when CI/CD deploy status or deployed health must be verified after a
  merge, hub run, development deploy, or "is it deployed?" question.
  Triggers on: hub-check-deploy, verify deploy, deploy status, did it
  deploy, check ci, check pipeline, check actions, development deploy.
user_invocable: true
---

# Hub Check Deploy — Deploy Gate

**Announce at start:** "Running /hub-check-deploy — hub deploy verification gate."

## Overview

Verifies that a branch or environment was deployed by checking the latest CI/CD
run and, when configured, the deployed HTTP health endpoint. This skill is now
part of the hub family: it has a structured input contract, a non-interactive
autopilot mode, an evidence artifact, and a machine-readable result block that
`hub-ship`, `hub-driven-autopilot`, and `hub-linear-autopilot` can parse.

It does not merge, redeploy, or fix failures. It only reports deploy evidence.
Callers decide whether to retry, fix, or leave a ticket open.

## Invocation

Interactive:
```text
/hub-check-deploy
/hub-check-deploy development
```

Structured handoff:
```json
{
  "branch": "development",
  "environment": "development",
  "health_url": "https://dev.example.com/health",
  "timeout_seconds": 900,
  "mode": "autopilot",
  "require_ci": true,
  "require_health": true
}
```

## Input Schema

| Field | Default | Purpose |
|---|---|---|
| `branch` | prompt in interactive, else `development` | Git branch whose workflow run should be checked. |
| `environment` | same as `branch` | Logical target: `development`, `staging`, `production`, or custom. |
| `health_url` | discovered | URL to request after CI succeeds. |
| `timeout_seconds` | `900` | Maximum time to wait for queued/in-progress CI. |
| `mode` | `interactive` | `interactive` may prompt; `autopilot` never prompts. |
| `require_ci` | `true` in autopilot, `false` interactive | Whether missing CI makes deploy unverified. |
| `require_health` | `true` in autopilot, `false` interactive | Whether missing health URL makes deploy unverified. |

Structured input can be passed as JSON or as a freeform branch name. If a
freeform argument is present and is not JSON, treat it as `branch`.

## Status Contract

The top-level `deploy_status` is:

| Status | Meaning |
|---|---|
| `verified` | Required CI passed and required health check passed or was not required. |
| `failed` | CI failed, health check failed, or a required signal was absent. |
| `timeout` | CI did not complete before `timeout_seconds`. |
| `unconfigured` | Required deploy evidence could not be checked because configuration is missing. |

`ci_status` is one of `success | failure | timeout | absent | in_progress | queued | unknown`.
`health_status` is one of `passed | failed | skipped | unconfigured`.

## Phase 1: Parse Input

1. If input is JSON, parse fields using the schema above.
2. If input is a non-empty freeform string, set `branch` to that string.
3. If no branch was supplied:
   - `mode=interactive`: ask which branch to verify, suggesting `development` and `main`.
   - `mode=autopilot`: use `development`.
4. Normalize `environment` to the supplied value or `branch`.
5. In `mode=autopilot`, do not ask questions. Missing configuration becomes
   `unconfigured` or `failed` according to `require_*`.

## Phase 2: Resolve Health URL

Resolution order:

1. `input.health_url`
2. Environment-specific agent doc line:
   ```bash
   grep -i "HEALTH_CHECK_URL_${ENVIRONMENT}:" CLAUDE.md AGENTS.md 2>/dev/null
   ```
3. Generic agent doc line:
   ```bash
   grep -i "HEALTH_CHECK_URL:" CLAUDE.md AGENTS.md 2>/dev/null
   ```
4. Environment variables:
   - `HEALTH_CHECK_URL_${ENVIRONMENT_UPPER}`
   - `HEALTH_CHECK_URL`

If no URL is found:
- `mode=interactive`: ask for a URL or allow skip.
- `mode=autopilot` and `require_health=true`: set `health_status=unconfigured`
  and final `deploy_status=unconfigured` unless CI later fails first.
- `mode=autopilot` and `require_health=false`: set `health_status=skipped`.

## Phase 3: CI/CD Check

Detect GitHub Actions by `.github/workflows`. If no supported CI provider is
detected:
- `require_ci=true`: set `ci_status=absent`, final `deploy_status=failed`.
- `require_ci=false`: set `ci_status=absent` and continue to health.

For GitHub Actions, find the latest run for the branch:
```bash
gh run list --branch "$BRANCH" --limit 1 \
  --json databaseId,name,status,conclusion,headBranch,event,createdAt,url,headSha
```

Rules:
- No runs found → `ci_status=absent`; fail only when `require_ci=true`.
- `queued` or `in_progress` → watch bounded by `timeout_seconds`:
  ```bash
  timeout "${TIMEOUT_SECONDS}s" gh run watch "$RUN_ID" --exit-status
  ```
  Exit `0` → `ci_status=success`; exit `124` → `ci_status=timeout`;
  any other non-zero → `ci_status=failure`.
- Existing conclusion `success` → `ci_status=success`.
- Existing conclusion `failure`, `cancelled`, `timed_out`, or `action_required`
  → `ci_status=failure`; capture failed logs:
  ```bash
  gh run view "$RUN_ID" --log-failed 2>&1 | tail -80
  ```

Do not proceed to health when CI failed or timed out.

## Phase 4: Health Check

Run only when CI is `success` or CI was absent and `require_ci=false`.

```bash
HTTP_CODE=$(curl -s -o /tmp/hub-check-deploy-body.txt \
  -w "%{http_code}" --max-time 10 "$HEALTH_URL")
head -c 500 /tmp/hub-check-deploy-body.txt
```

Rules:
- `200` through `299` → `health_status=passed`.
- Any other status code or curl failure → `health_status=failed`.
- No URL and not required → `health_status=skipped`.
- No URL and required → `health_status=unconfigured`.

## Phase 5: Evidence Artifact

Write a JSON artifact for every run:

```text
docs/deploy-checks/YYYY-MM-DDTHH-MM-SSZ-<environment>-<branch>.json
```

Include:
- parsed input
- CI provider, run ID, run URL, commit SHA, conclusion
- health URL, status code, body excerpt
- `deploy_status`, `ci_status`, `health_status`
- warnings and timestamps

When invoked by `hub-ship` on the base branch, that caller commits the
artifact. When invoked standalone, leave it unstaged unless the user asks to
commit. When invoked by `hub-driven-autopilot`, the caller records the path
inside its archived state file and commits that state update.

## Phase 6: Final Report

Human-readable report:

```text
RESULT: hub-check-deploy

Branch:        <branch>
Environment:   <environment>
Deploy status: <verified | failed | timeout | unconfigured>
CI:            <success | failure | timeout | absent> (<workflow>, #<run_id>)
Run URL:       <url | n/a>
Health check:  <url> -> <status_code> | skipped | unconfigured
Artifact:      <path>
Warnings:      <count>
```

Then emit this machine-readable block exactly:

```text
HUB-CHECK-DEPLOY-RESULT:
{
  "deploy_status": "verified | failed | timeout | unconfigured",
  "branch": "development",
  "environment": "development",
  "ci_status": "success | failure | timeout | absent | in_progress | queued | unknown",
  "ci_provider": "github-actions | gitlab | circleci | unknown",
  "workflow": "CI/CD Pipeline",
  "run_id": "123456789",
  "run_url": "https://github.com/org/repo/actions/runs/123456789",
  "head_sha": "abc123",
  "health_status": "passed | failed | skipped | unconfigured",
  "health_url": "https://dev.example.com/health",
  "http_status": 200,
  "artifact_path": "docs/deploy-checks/...",
  "warnings": []
}
```

Fields that do not apply must be `null`, not omitted.

## Important Rules

- **Evidence before claims.** Never say deploy is verified without CI and/or
  health evidence in the final report.
- **Autopilot never prompts.** Missing branch, CI, or URL must become a
  structured status, not a question.
- **Required means required.** In autopilot mode, absent CI or absent health URL
  fails or returns `unconfigured`; it is not a pass.
- **No deploy mutations.** Do not merge, rerun workflows, redeploy, or edit code.
- **Bounded waits.** Always use `timeout_seconds` when watching CI.
- **Machine-readable block is stable.** Callers parse `HUB-CHECK-DEPLOY-RESULT:`.
