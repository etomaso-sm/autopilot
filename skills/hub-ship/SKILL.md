---
name: hub-ship
description: >
  Use when implementation is complete, a feature is ready to ship, or a merged
  hub/autopilot branch needs development deploy verification. Triggers on:
  hub-ship, hub-release, ship it, ready to merge, done implementing,
  deploy to development, verify development deploy, feature complete.
user_invocable: true
---

# Hub Ship — Delivery Gate

**Announce at start:** "Running /hub-ship — hub ship and deploy verification gate."

## Overview

Complete post-implementation ship pipeline for the Impactia team. This is now
the hub family's deploy verification gate: it can run the full interactive
ship process, or it can receive a structured handoff from
`hub-driven-autopilot` after a merge and verify development deploy evidence.

It still replaces `superpowers:finishing-a-development-branch` for manual work,
but deploy evidence is delegated to `/hub-check-deploy` and deployed frontend
verification is delegated to `/hub-e2e-frontend` using their hub contracts.

> **Note:** If you're using `superpowers:executing-plans` or `superpowers:subagent-driven-development`
> outside of `/dev-flow`, those skills will invoke `finishing-a-development-branch` instead.
> The team's preferred ship path is `/hub-ship` via `/dev-flow`.

## Hub Contract

Structured handoff:

```json
{
  "mode": "autopilot",
  "branch": "autopilot/2026-04-27-add-retry",
  "base_branch": "development",
  "environment": "development",
  "pr_url": "https://github.com/org/repo/pull/123",
  "changed_files": ["src/app/checkout/page.tsx"],
  "quality_gate": "pass",
  "review_verdict": "approve",
  "deploy_only": false,
  "require_deploy": true,
  "require_post_deploy_e2e": true
}
```

| Field | Default | Purpose |
|---|---|---|
| `mode` | `interactive` | `interactive` runs all phases; `autopilot` never prompts. |
| `branch` | current branch | Feature/autopilot branch being shipped. |
| `base_branch` | `development` | Branch that triggers the target deploy. |
| `environment` | `development` | Deploy target to verify. |
| `pr_url` | `null` | PR associated with this ship. |
| `changed_files` | git diff fallback | Used to decide post-deploy frontend E2E scope. |
| `quality_gate` | `null` | Hub quality result from upstream. |
| `review_verdict` | `null` | Fresh-agent review verdict from upstream. |
| `deploy_only` | `false` | When true, skip pre-merge ship phases and only verify deploy/post-deploy gates for an already-merged PR. |
| `require_deploy` | `true` | Whether deploy verification gates success. |
| `require_post_deploy_e2e` | `true` | Whether deployed frontend E2E gates success when frontend changed. |

Final machine-readable marker:

```text
HUB-SHIP-RESULT:
{
  "ship_status": "verified | failed | blocked | partial",
  "branch": "autopilot/...",
  "base_branch": "development",
  "environment": "development",
  "pr_url": "https://github.com/org/repo/pull/123",
  "deploy_status": "verified | failed | timeout | unconfigured | skipped",
  "post_deploy_e2e": "pass | fail | skipped | unconfigured",
  "artifact_paths": [],
  "warnings": []
}
```

`ship_status=verified` means development is deployed and all required
post-deploy checks passed. `ship_status=partial` means code was merged or PR
exists but deploy evidence is incomplete. `ship_status=blocked` means the skill
needs human input in interactive mode or cannot continue safely in autopilot.

## Pre-flight Check

Before starting, verify:

1. You are on a feature branch (not `main`, `master`, or `development`)
2. There are no uncommitted changes:
```bash
git status --porcelain
```
If dirty, ask user: "There are uncommitted changes. Should I commit them before continuing?"

In `mode=autopilot`, do not ask. If the tree is dirty, return
`ship_status=blocked` with a warning unless the dirty files are only deploy
evidence artifacts produced by this run.

If `deploy_only=true`, or if already on `base_branch` after an upstream merge,
skip Phases 1-9 and start at Phase 10. In `deploy_only`, derive
`HAS_PLAYWRIGHT` from `playwright.config.*` and use input `changed_files`
instead of branch diff for post-deploy frontend E2E scope.

---

## Phase 1: Analyze Changes

Determine what changed in this feature branch:

```bash
# Priority: 1) CLAUDE.md config, 2) git default, 3) fallback
BASE_BRANCH=$(grep -i 'BASE_BRANCH:' CLAUDE.md 2>/dev/null | awk '{print $2}')
if [ -z "$BASE_BRANCH" ]; then
  BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||')
fi
BASE_BRANCH=${BASE_BRANCH:-development}
echo "Merge target: $BASE_BRANCH"
BASE=$(git merge-base HEAD "$BASE_BRANCH")
git diff "$BASE"...HEAD --name-only
```

This `$BASE_BRANCH` is used throughout all subsequent phases.

Categorize changed files into backend (`.py`) and frontend (`.ts`, `.tsx`).

Detect if the project has Playwright configured:

```bash
HAS_PLAYWRIGHT=false
test -f playwright.config.ts && HAS_PLAYWRIGHT=true
test -f playwright.config.js && HAS_PLAYWRIGHT=true
```

This flag controls Phase 3.5 (frontend E2E) and the Playwright runner in Phase 4.

---

## Phase 1.5: Spec Compliance Check

Verify that the implementation covers all requirements from the design doc and plan.

1. Find the design doc and plan:
```bash
DESIGN=$(ls -t docs/plans/*-design.md docs/superpowers/specs/*-design.md 2>/dev/null | head -1)
PLAN=$(ls -t docs/plans/*-plan.md docs/superpowers/plans/*.md 2>/dev/null | head -1)
```

2. If a design doc exists, read it and extract all requirements (look for bullet points,
   numbered lists, "must", "should", "will" statements, and acceptance criteria).

3. For each requirement, check if the changed files address it:
   - Read the relevant changed files
   - Determine if the requirement is implemented

4. Report:
```
Spec Compliance:
  Requirements found:    N
  Implemented:           M
  Missing:               K
  Missing items:
    - <requirement from spec> (expected in <file>)
```

- If all requirements covered → continue
- If missing requirements → warn the user:
  > "These requirements from the design are not covered by the implementation: [list].
  > Should I continue anyway, or do you want to implement them first?"

  If user says implement → go back to `/tdd` for the missing items.
  If user says continue → proceed (user takes responsibility).

- If no design doc or plan found → skip this phase with a note:
  > "No design doc found — skipping spec compliance check."

---

## Phase 2: Unit Test Gate

Verify that changed files have associated unit tests and that all pass.

### 2a: Check test coverage for changed files

For each changed source file, check that a corresponding test file exists:

```bash
# Backend: app/users/views.py → app/users/tests/test_views.py
# Frontend: src/components/Button/Button.tsx → src/components/Button/__tests__/Button.test.tsx
```

List any changed files that have **no associated test file**. Report them:
> "These changed files have no unit tests: [list]. Should I write tests before continuing?"

If the user says yes, invoke `/tdd` for those files before proceeding.

### 2b: Run unit tests

```bash
# Backend
pytest tests/ -v --tb=short 2>&1

# Frontend (if applicable)
npx vitest run 2>&1
```

**If any test fails**: fix the issue, commit the fix, and re-run (max 3 iterations).

**Do NOT proceed to Phase 3 with failing unit tests.**

---

## Phase 3: Write E2E Tests

For each changed area that lacks E2E coverage:

**Smoke tests** (CRUD pattern):
```python
def test_create_resource(client, cleanup):
    resp = client.create_resource({...})
    assert resp.status_code == 201, resp.text
    data = resp.json()
    cleanup.register("resource_type", data["id"])
    assert "id" in data
```

**Flow tests** (lifecycle pattern):
```python
def test_feature_lifecycle(client, cleanup):
    # Setup: create all prerequisite resources
    # Action: execute the feature's main flow
    # Assert: verify the expected end state
    # (cleanup handles teardown)
```

Verify tests are syntactically valid:
```bash
python -m py_compile path/to/test_file.py
```

Commit E2E tests:
```bash
git add e2e/
git commit -m "test(e2e): add E2E tests for <feature>"
```

---

## Phase 3.5: Frontend E2E (Playwright)

Only runs when `HAS_PLAYWRIGHT=true` (detected in Phase 1).

Invoke `/hub-e2e-frontend` skill. It will:
1. Analyze changed frontend files
2. Generate missing E2E tests (functional + visual snapshots)
3. Run all Playwright tests
4. Handle visual baseline updates (with user approval)

If `/hub-e2e-frontend` reports failures after 3 attempts: STOP and report to user.

**Do NOT proceed to Phase 4 with failing frontend E2E tests.**

---

## Phase 4: Run E2E Suite

Run all tests (not just the new ones):
```bash
# Backend E2E
pytest e2e/ -v --tb=long 2>&1

# Frontend E2E — skip if Phase 3.5 already ran /hub-e2e-frontend successfully.
# Only run here if HAS_PLAYWRIGHT=true AND Phase 3.5 was skipped or not invoked.
if [ "$HAS_PLAYWRIGHT" = true ]; then
  npx playwright test --reporter=dot 2>&1
fi
```

**Fix loop (max 3 iterations):**

If tests fail:
1. Analyze each failure — determine if it's a **feature bug** or a **test bug**
2. Fix the issue
3. Commit the fix: `git add <files> && git commit -m "fix: <description>"`
4. Re-run the full suite
5. Repeat up to 3 times

If still failing after 3 iterations, **STOP** and report:
> "After 3 attempts, these tests are still failing: [list]. I need your input to continue."

**Do NOT skip this phase.** All tests must pass before proceeding.

---

## Phase 5: Simplify

Invoke the `simplify` skill to run 3 parallel review agents:

1. **Code Reuse Review** — find existing utilities that replace new code
2. **Code Quality Review** — detect hacky patterns, redundant state, copy-paste
3. **Efficiency Review** — unnecessary work, missed concurrency, hot-path bloat

Fix any issues found by the agents.

---

## Phase 6: Re-run Tests

Verify that simplify didn't break anything:
```bash
# Unit tests
pytest tests/ -v --tb=short 2>&1
npx vitest run 2>&1

# E2E tests
pytest e2e/ -v --tb=long 2>&1

# Frontend E2E (only if HAS_PLAYWRIGHT=true)
if [ "$HAS_PLAYWRIGHT" = true ]; then
  npx playwright test --reporter=dot 2>&1
fi
```

If tests fail, fix and re-run. Commit fixes.

---

## Phase 7: Code Review

Invoke `/code-review` skill which:
1. Runs `superpowers:requesting-code-review`
2. Checks SOLID/DRY compliance against `~/.ai-skills/method/PRINCIPLES.md`
3. Runs stack-specific checks (Django, React)

Additionally, verify the 10 principles from PRINCIPLES.md are not violated by the changes:

```bash
# Read principles
cat ~/.ai-skills/method/PRINCIPLES.md
```

For each of the 10 principles, check if any changed file violates it.
Flag violations with the principle number: `P1: Tests Before Code`, `P7: Security by Default`, etc.

Handle findings:
- **Critical/Important**: Fix immediately → go back to Phase 6
- **Minor**: Note in commit message but continue
- **Principle violations**: Fix before proceeding — these are non-negotiable

---

## Phase 8: Security Review

Invoke `/security-review` skill which launches 4 parallel agents:
1. OWASP Top 10 scan
2. Auth & data exposure check
3. Dependency & config audit
4. AI security scan

**If any agent reports FAIL**: fix the findings, re-run security review.
**Do NOT proceed to merge with any security failure.**

---

## Phase 8.5: Migration Safety Check (Django only)

Only runs if `.py` migration files are in the changed files list from Phase 1.

```bash
MIGRATIONS=$(git diff "$BASE"...HEAD --name-only | grep '/migrations/' | grep -v '__pycache__')
```

If migrations found, check each one for safety:

1. **Destructive operations**: Look for `RemoveField`, `DeleteModel`, `AlterField` that
   narrows a column (e.g., reducing `max_length`, changing nullable to non-nullable).
   ```bash
   grep -n 'RemoveField\|DeleteModel\|AlterField' $MIGRATION_FILE
   ```

2. **Reversibility**: Check that `RunSQL` operations have `reverse_sql`:
   ```bash
   grep -A2 'RunSQL' $MIGRATION_FILE | grep -c 'reverse_sql'
   ```

3. **Data migrations**: Check for `RunPython` with `reverse_code`:
   ```bash
   grep -A2 'RunPython' $MIGRATION_FILE | grep -c 'reverse_code'
   ```

4. **Large table operations**: Flag `AddIndex` or `AlterField` on models that are
   likely large (check if model name contains "User", "Order", "Transaction", "Log",
   or similar high-volume patterns).

Report:
```
Migration Safety:
  Migrations found:    N
  Destructive ops:     M (list)
  Non-reversible:      K (list)
  Large table risk:    L (list)
```

- If destructive or non-reversible → warn:
  > "These migrations have safety concerns: [list]. Review carefully before deploying.
  > Consider adding `reverse_sql`/`reverse_code` for reversibility."
  This is non-blocking but the warning must be shown.
- If clean → continue silently.
- If no migrations → skip this phase.

---

## Phase 9: Merge to Base Branch

```bash
git fetch origin "$BASE_BRANCH"
git checkout "$BASE_BRANCH"
git pull origin "$BASE_BRANCH"
git merge --no-ff <feature-branch> -m "merge: <feature-branch> into $BASE_BRANCH"
```

If there are merge conflicts:
1. Report conflicts to the user
2. Resolve them (ask for guidance if ambiguous)
3. Commit the resolution

```bash
git push origin "$BASE_BRANCH"
git checkout <feature-branch>
```

---

## Phase 10: Verify Deploy

Canonical hub path: invoke `/hub-check-deploy` with structured handoff and parse
`HUB-CHECK-DEPLOY-RESULT`. The commands below remain as fallback implementation
detail when a caller cannot invoke another skill.

```json
{
  "branch": "<BASE_BRANCH>",
  "environment": "<environment or BASE_BRANCH>",
  "mode": "<interactive | autopilot>",
  "require_ci": true,
  "require_health": true,
  "timeout_seconds": 900
}
```

Gate:
- `deploy_status=verified` → proceed to Phase 10.5 and Phase 11.
- `deploy_status=failed | timeout | unconfigured` → go to Phase 12 in
  interactive mode. In `mode=autopilot`, stop with `ship_status=partial` and
  surface the deploy result for the caller.

Detect CI provider:

```bash
# GitHub Actions (default)
if [ -d .github/workflows ]; then
  CI_PROVIDER="github-actions"
# GitLab CI
elif [ -f .gitlab-ci.yml ]; then
  CI_PROVIDER="gitlab"
# CircleCI
elif [ -d .circleci ]; then
  CI_PROVIDER="circleci"
else
  CI_PROVIDER="unknown"
fi
```

- **github-actions**: Use `gh run list` / `gh run watch` (existing commands below)
- **gitlab**: Use `glab ci list` / `glab ci view` (if `glab` is installed)
- **circleci**: Use `circleci` CLI or check the CircleCI dashboard
- **unknown**: Ask the user how to verify the deploy:
  > "No CI configuration detected. How should I verify the deploy? Provide a command or URL to check."

Poll for the GitHub Actions run (no hardcoded sleep):

```bash
echo "Waiting for CI run to appear..."
for i in $(seq 1 12); do
  RUN=$(gh run list --branch "$BASE_BRANCH" --limit 1 --json databaseId,status,conclusion 2>/dev/null)
  RUN_STATUS=$(echo "$RUN" | jq -r '.[0].status' 2>/dev/null)
  if [ "$RUN_STATUS" != "null" ] && [ -n "$RUN_STATUS" ]; then
    echo "Found run: $RUN"
    break
  fi
  sleep 5
done
```

If no run appears after 60 seconds, warn:
> "No CI run detected. Check if the workflow is configured for the `$BASE_BRANCH` branch."

Watch the workflow:
```bash
gh run watch <run_id> --exit-status
```

- **Exit code 0 (success)**: Proceed to health check.
- **Exit code 1 (failure)**: Proceed to Phase 12 (Fix Deploy Errors).

### Health Check

Search for health check URL in project's CLAUDE.md:
```bash
grep -i 'HEALTH_CHECK_URL:' CLAUDE.md 2>/dev/null
```

If found, verify the deployed app responds:
```bash
curl -s -o /dev/null -w "%{http_code}" --max-time 10 <url>
```

- **200**: Deploy fully verified. Proceed to Phase 11.
- **Other**: Report status code. Ask user if they want to retry or proceed.
- **Not configured**: Skip health check, rely on GitHub Actions result. Proceed to Phase 11.

---

## Phase 10.5: Generate Changelog Entry

Skip this phase when `deploy_only=true`; the PR has already been merged and
this skill is only collecting deploy evidence.

Summarize what this feature delivers in human-readable format.

1. Read the design doc (if exists) for context
2. Analyze all commits on this branch:
```bash
git log --oneline "$BASE"...HEAD
```

3. Generate a changelog entry:
```markdown
## [Feature Name] — YYYY-MM-DD

### Added
- <new capabilities, endpoints, components>

### Changed
- <modifications to existing behavior>

### Fixed
- <bug fixes included in this feature>

### Technical
- <migrations, dependency changes, config changes>
```

4. Append to `CHANGELOG.md` (create if it doesn't exist). Insert at the top,
   after the `# Changelog` header (or create the header if the file is new).

5. Commit:
```bash
git add CHANGELOG.md
git commit -m "docs: changelog entry for <feature>"
```

If the project doesn't use changelogs (no `CHANGELOG.md` and `CLAUDE.md` says
`CHANGELOG: false`), skip this phase.

---

## Phase 11: Post-Deploy E2E Gate (MANDATORY — NO EXCEPTIONS)

This phase runs AFTER deploy is verified (Phase 10). It validates the **deployed
environment** works end-to-end, not just the CI pipeline.

### Why this exists

A green CI pipeline does NOT guarantee the deployed app works. Environment
variables, database migrations, external services, and infrastructure
differences can all cause failures that only appear in the deployed environment.

### Steps

1. **Run backend E2E against deployed environment:**
```bash
E2E_BASE_URL=$(grep -i 'HEALTH_CHECK_URL:' CLAUDE.md 2>/dev/null | awk '{print $2}')
pytest e2e/ -v --tb=long 2>&1
```

2. **Run frontend E2E against deployed environment** using the hub contract
   if `HAS_PLAYWRIGHT=true` or frontend files changed:

```json
{
  "description": "<feature title>",
  "changed_files": ["<files from Phase 1 or upstream handoff>"],
  "affected_urls": ["<inferred routes>"],
  "mode": "post_deploy",
  "target_env": "<environment>",
  "autopilot": true
}
```

Invoke `/hub-e2e-frontend` and parse `HUB-E2E-FRONTEND-RESULT`.

Gate:
- `result=pass` → continue.
- `result=skipped` → continue only if no frontend files changed.
- `result=fail | unconfigured` → in interactive mode, enter the Phase 12
  fix/redeploy loop. In `mode=autopilot`, stop with `ship_status=partial`.

3. **Create post-deploy report artifact:**
```bash
cat > e2e-post-deploy-report.md << 'REPORT'
# Post-Deploy E2E Report

## Backend E2E
- Tests run: <N>
- Passed: <N>
- Failed: <N>
- Failures: <list or "none">

## Frontend E2E
- Tests run: <N> | skipped (no Playwright)
- Passed: <N>
- Failed: <N>
- Failures: <list or "none">
- Visual regressions: <N> | N/A

## Environment
- URL: <deployed URL>
- Timestamp: <ISO 8601>
REPORT
```

Also write a structured artifact:

```text
docs/hub-ship-runs/YYYY-MM-DDTHH-MM-SSZ-<environment>.json
```

Include parsed input, deploy result from `/hub-check-deploy`, backend E2E summary,
frontend `HUB-E2E-FRONTEND-RESULT`, artifact paths, final `ship_status`, warnings,
and timestamp.

4. **Commit the report:**
```bash
# We're on the base branch after Phase 9's merge
git add e2e-post-deploy-report.md
git commit -m "test: post-deploy E2E report for <feature>"
git push origin "$BASE_BRANCH"
```

### Gate rules

- ⛔ You CANNOT proceed to the Final Report without this file existing
- ⛔ You CANNOT report "deploy successful" without post-deploy E2E evidence
- ⛔ If any post-deploy E2E test fails: fix → re-deploy → re-run (max 3 attempts)
- ⛔ After 3 failures: STOP and report to user
- ⛔ In autopilot mode, do not ask. Return `ship_status=partial` with the
  failing artifact path so `hub-linear-autopilot` can keep the ticket open.

### Rationalization table

| Excuse | Reality |
|--------|---------|
| "CI already ran E2E tests" | CI tests run against test infra, not the deployed environment |
| "Health check passed, so it works" | A 200 on `/health` doesn't prove features work |
| "Same code, same results" | Different env = different behavior. That's the whole point |
| "Post-deploy E2E is overkill" | It catches the bugs that matter most: production regressions |
| "I'll save time by skipping" | You'll waste 10x more time debugging a broken deploy |
| "The user didn't ask for this" | The pipeline requires it. It's not optional |

---

## Phase 12: Fix Deploy Errors (max 3 attempts)

Read the failed logs:
```bash
gh run view <run_id> --log-failed 2>&1 | tail -100
```

Common deploy errors:

| Error pattern | Fix approach |
|---|---|
| `ModuleNotFoundError` | Fix the import |
| `django.db.utils.ProgrammingError` | Run makemigrations |
| `SyntaxError` | Fix the syntax |
| `docker build` failure | Fix Dockerfile or requirements |

Fix, commit, merge to development, and re-push:
```bash
git add <files>
git commit -m "fix: <deploy fix description>"
git checkout "$BASE_BRANCH" && git pull origin "$BASE_BRANCH"
git merge --no-ff <feature-branch> -m "fix: merge deploy fix"
git push origin "$BASE_BRANCH"
git checkout <feature-branch>
```

Go back to Phase 10. After 3 failed attempts, **STOP**:
> "After 3 attempts, deploy is still failing. Last error: [summary]. I need your input."

---

## Final Report

After all phases complete (including Phase 11):

```
RESULT: hub-ship

Feature:           <branch name>
Ship status:       verified | failed | blocked | partial
Environment:       development | staging | production
Unit tests:        X passed, Y files without tests
E2E backend:       X passed, Y written
E2E frontend:      X passed, Y generated, Z baselines updated | skipped (no Playwright)
Simplify:          N issues fixed
Code review:       passed / N findings fixed
Security review:   passed / N findings fixed
Deploy:            verified | failed | timeout | unconfigured (run #<run_id>)
Health check:      <url> -> <status_code> | skipped | not configured
Post-deploy E2E:   X backend passed, Y frontend passed | <failures>
Run URL:           <gh run URL>
Artifacts:         docs/deploy-checks/...; docs/hub-ship-runs/...
```

⚠️ **The Final Report is INCOMPLETE without the `Post-deploy E2E` line.**
If that line is missing, the report has not been generated correctly.

Then emit this machine-readable block exactly:

```text
HUB-SHIP-RESULT:
{
  "ship_status": "verified | failed | blocked | partial",
  "branch": "<feature branch>",
  "base_branch": "<base branch>",
  "environment": "development",
  "pr_url": "<url | null>",
  "deploy_status": "verified | failed | timeout | unconfigured | skipped",
  "deploy_result": {},
  "backend_e2e": {
    "result": "pass | fail | skipped",
    "passed": 0,
    "failed": 0
  },
  "post_deploy_e2e": "pass | fail | skipped | unconfigured",
  "frontend_e2e_result": {},
  "artifact_paths": [],
  "warnings": []
}
```

Fields that do not apply must be `null`, not omitted. Callers parse the
`HUB-SHIP-RESULT:` marker.

### Branch Cleanup

After the final report, offer to clean up:

> "Feature successfully deployed and verified. Clean up the feature branch?
> - `git branch -d <feature-branch>` (local)
> - `git push origin --delete <feature-branch>` (remote)
> (y/N)"

Only delete if:
1. The branch is fully merged into `$BASE_BRANCH`
2. The user confirms

If the user declines, that's fine — the branch stays. Never delete without confirmation.

---

## Common Mistakes

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Skipping Phase 2 (unit tests) because E2E "covers it" | E2E tests are slow and flaky; unit tests catch logic bugs fast | Always run unit tests first |
| Running only new tests in Phase 4 | Regressions in existing features go undetected | Run the FULL suite |
| Auto-accepting visual diffs in Phase 3.5 | Visual regressions ship to production | Always ask the user |
| Skipping Phase 10.5 because "CI passed" | Deployed app may not work despite green CI | Post-deploy E2E is mandatory |
| Merging without security review (Phase 8) | Vulnerabilities ship to production | Never skip security |
| Not returning to feature branch after merge | Next commands run on wrong branch | Always `git checkout <feature>` |

## Important Rules

- **Evidence before claims**: Never say "tests pass" without showing actual output
- **Full suite always**: Run ALL tests, not just the ones you changed
- **Commit between phases**: Each phase ends with a clean git state
- **No shortcuts**: Do not skip phases even if the change seems small
- **Feature branch preserved**: Always return to the feature branch after pushing
- **Deploy is the goal**: The skill doesn't end until deploy AND post-deploy E2E pass
- **Hub gate contract**: In autopilot mode, success means
  `HUB-SHIP-RESULT.ship_status == "verified"`. Anything else keeps the
  owning ticket open.
- **Delegate deploy evidence**: Use `/hub-check-deploy` for CI/CD + health evidence
  and `/hub-e2e-frontend` with `mode=post_deploy` for deployed frontend checks.
- **Autopilot never prompts**: missing config, failed deploy, or failed E2E must
  become structured `blocked`, `partial`, or `failed` results.

---

## Team Notification

After a successful deploy (all phases passed), suggest notifying the team:

> "Deploy successful. Want to notify the team?
> - Slack: I can post a summary to a channel (requires Slack MCP)
> - PR comment: I can comment on the associated PR
> - Skip: No notification needed"

If the user wants Slack notification, use the Slack MCP tools (if available) to
post a summary of the Final Report to the appropriate channel.

If no notification tools are available, suggest the user copy the Final Report
manually.
