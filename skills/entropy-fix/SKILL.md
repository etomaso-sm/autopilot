---
name: entropy-fix
description: >
  Execute correction plans from entropy-scan findings. Takes QUALITY_SCORE.md,
  establishes test baseline, applies fixes with full-suite gating, and re-scans.
  Triggers on: entropy-fix, fix quality, fix entropy, apply corrections,
  fix tech debt, correct findings.
user_invocable: true
---

<!-- SYNC: entropy-fix depends on entropy-scan dimension names and priority order.
     Update priority list if entropy-scan agents change.
     Also syncs with entropy-loop for strategy levels and autonomous mode.
     Also used by entropy-bugfix for single-cycle simple corrections after
     elevated or sensitive bug fixes. -->

# Entropy Fix

**Announce at start:** "Running /entropy-fix — applying corrections from quality scan."

## Overview

Takes the output of `/entropy-scan` (specifically `docs/QUALITY_SCORE.md` and
`docs/quality-score.json`) and systematically fixes the findings. Every fix is
gated by the full test suite. This is the execution counterpart to
`/entropy-scan` (diagnostic).

### Partial Fix

Supports fixing specific domains: `/entropy-fix billing frontend/dashboard`

When domains are specified:
- Only process findings for those domains
- Still run the full test suite as gate (partial suites miss cross-domain regressions)
- Re-scan only the fixed domains at the end

### Options

- `--skip-rescan`: Skip Phase 4 re-scan. Use when the caller will run its own
  scan (e.g., entropy-loop manages scan cycles independently).
- `--strategy=<simple|aggressive|rewrite>`: Set the strategy level for the
  correction plan. Defaults to `simple`. See Strategy Levels below.
- `--autonomous`: Suppress "3 reverts → STOP" behavior. Instead of stopping to
  ask the user, log the issue and continue to next domain. Use when invoked by
  entropy-loop or other autonomous orchestrators.

## Strategy Levels

When invoked with a strategy level, entropy-fix adjusts the aggressiveness of
its correction plan:

| Strategy | What's allowed | What's NOT allowed |
|----------|---------------|-------------------|
| **simple** (default) | Atomic fixes: replace hardcoded values with tokens, add aria attributes, add input validation, add characterization tests, lint fixes, add logging, remove unused imports | Extract class, split files, apply patterns, change architecture |
| **aggressive** | Everything in simple PLUS: extract class/service, split components >200 lines, apply design patterns (Strategy, Observer, Factory), dependency injection, i18n extraction, state management refactors | Rewrite modules, change public interfaces, remove/merge domains |
| **rewrite** | Everything in aggressive PLUS: partial module rewrites preserving public interfaces, redesign internal module structure, rewrite with TDD from scratch | Change public API contracts, merge domains, delete domains |

The strategy constrains Phase 1 planning. When writing the correction plan, only
include tasks that match the current strategy level.

## Pre-flight

Read `docs/QUALITY_SCORE.md` and `docs/quality-score.json`. If neither exists:
> "No quality score found. Run `/entropy-scan` first to generate findings."
> Stop.

If they exist, parse Detailed Findings and Top 5 Refactoring Priorities.
Use the JSON for grade comparisons; use the MD for detailed findings.

### Detect project test runners

Before planning, detect what's available:

```bash
# Backend runners
grep -q '\[tool.pytest' pyproject.toml 2>/dev/null && echo "pytest"
grep -q 'pytest' requirements*.txt 2>/dev/null && echo "pytest"
[ -f "pytest.ini" ] && echo "pytest"
[ -f "manage.py" ] && echo "django-test"

# Frontend runners
grep -q '"vitest"' package.json 2>/dev/null && echo "vitest"
grep -q '"jest"' package.json 2>/dev/null && echo "jest"
grep -q '"mocha"' package.json 2>/dev/null && echo "mocha"

# E2E runners
grep -q '"playwright"' package.json 2>/dev/null && echo "playwright"
grep -q '"cypress"' package.json 2>/dev/null && echo "cypress"

# Linters (used as gate for frontend quality fixes)
grep -q '"eslint"' package.json 2>/dev/null && echo "eslint"
grep -q '"stylelint"' package.json 2>/dev/null && echo "stylelint"
ls .eslintrc* eslint.config.* 2>/dev/null && echo "eslint-config"

# Type checking
grep -q '"typescript"' package.json 2>/dev/null && echo "tsc"
```

Store detected runners. Use them instead of hardcoded commands throughout.

## Phase 1: Write the correction plan

Invoke `superpowers:writing-plans` to create a detailed implementation plan based
on the findings. The plan should:

1. **Classify each finding by fix type** (determines verification strategy):

   | Fix Type | Examples | Verification |
   |----------|----------|-------------|
   | **Structural refactor** | Extract class, split component, apply pattern, remove duplication | Characterization test → refactor → full suite |
   | **Security fix** | Add auth guard, input validation, remove hardcoded secret | Characterization test → fix → full suite |
   | **Frontend markup** | a11y attributes, semantic HTML, `<div>` → `<button>` | Snapshot/render test OR lint check → fix → full suite |
   | **Frontend styling** | Replace hardcoded values with tokens, responsive fixes | Visual snapshot OR lint check → fix → full suite |
   | **Configuration** | i18n extraction, lazy loading setup, bundle optimization | Build check → fix → full suite |
   | **Documentation** | Update stale docs, fix inaccurate references, update counts/lists | Read doc → compare with code → update doc → build check |
   | **Repo hygiene** | Delete orphaned plans/specs, move files from deprecated locations, clean state files | Verify file is unused → delete/move → full suite |
   | **CI/CD** | Add missing stage from Impactia 5-stage pipeline (PR gate / build / deploy / post-deploy smoke / rollback), pin image tag to SHA instead of `:latest`, move hardcoded secret to `secrets.*`, document required secrets in `docs/ENV.md`, point E2E baseURL at deployed URL instead of localhost | Lint workflow YAML → commit on feature branch → verify CI run passes → verify deploy + smoke succeeded in dev env |

2. **Filter by strategy level**: Only include tasks allowed by the current strategy
   (see Strategy Levels table). Skip findings that require a higher strategy.

3. **Order by priority** (process in this order):
   - **Critical security** — Security dimension findings rated D/F (always first)
   - **CI/CD critical** — Agent 13 findings that are security-equivalent (hardcoded secrets in workflows, broken pipeline with no gates). Processed right after Security.
   - **Principles (SOLID)** — architectural risk
   - **DRY** — maintenance risk
   - **Testability** — enables future fixes
   - **Design Patterns** — structural improvements
   - **Frontend Quality** — design system, a11y, performance, state, i18n
   - **Observability** — logging, tracing, metrics
   - **Documentation accuracy** — stale references, wrong counts, outdated lists (Agent 4 findings)
   - **Repo Hygiene** — orphaned files, deprecated locations, dead docs (Agent 12 findings)
   - **CI/CD (non-critical)** — Agent 13 findings that aren't security-equivalent: undocumented secrets, stale workflow steps, missing env mapping, missing rollback docs
   - **Git Health** — usually process changes, not code fixes

   **Agent 4 vs Agent 12 mapping**: Agent 4 findings (doc *content* is wrong) → Documentation
   fix type. Agent 12 findings (files are *orphaned/misplaced*) → Repo hygiene fix type.
   If a doc is both inaccurate AND orphaned, treat as Repo hygiene (delete > fix).

4. Order by impact within each priority: fixes that improve the most domains first
5. Each task should be atomic: one refactoring at a time
6. Include the fix type classification for each task
7. Save to `docs/superpowers/plans/YYYY-MM-DD-entropy-corrections.md`
   - If file already exists (repeated invocations), append cycle number:
     `YYYY-MM-DD-entropy-corrections-cycle-2.md`

## Phase 2: Establish test baseline

Before touching any code, run the **full** test suite and record the baseline.

Use the detected runners from Pre-flight:

```bash
# Run each detected runner and capture results
# Example for pytest:
pytest tests/ -v --tb=short 2>&1 | tee /tmp/entropy-baseline-backend.txt
BACKEND_PASSED=$(grep -c "PASSED" /tmp/entropy-baseline-backend.txt)
BACKEND_FAILED=$(grep -c "FAILED" /tmp/entropy-baseline-backend.txt)

# Example for vitest/jest:
npx vitest run 2>&1 | tee /tmp/entropy-baseline-frontend.txt
# OR
npx jest --ci 2>&1 | tee /tmp/entropy-baseline-frontend.txt

# Linter baseline (for frontend quality fixes):
npx eslint src/ 2>&1 | tee /tmp/entropy-baseline-lint.txt
LINT_ERRORS=$(grep -c "error" /tmp/entropy-baseline-lint.txt || echo 0)

# Type check baseline:
npx tsc --noEmit 2>&1 | tee /tmp/entropy-baseline-types.txt
TYPE_ERRORS=$(grep -c "error TS" /tmp/entropy-baseline-types.txt || echo 0)
```

Record:
> "Test baseline: X passed, Y failed. Lint errors: N. Type errors: M.
> Any correction that increases failures will be reverted."

**This baseline is the safety net. Do NOT proceed without it.**

## Phase 3: Execute the plan (Full Suite Gate)

Invoke `superpowers:subagent-driven-development` (if subagents available) or
`superpowers:executing-plans` to execute the correction plan.

**Each correction follows this cycle based on its fix type:**

### Structural Refactors & Security Fixes

1. **Characterization test** — write a test that captures the exact current behavior
   of the code being refactored (inputs → outputs). This test must pass BEFORE
   the refactoring begins.

2. **Refactor** — apply the fix (extract class, remove duplication, apply pattern,
   add auth guard, etc.)

3. **Run full test suite** — not just the new test, ALL tests (use detected runners).

4. **Compare against baseline** (see Gate Rules below).

### Frontend Markup & Styling Fixes

1. **Snapshot or render test** — if the project has snapshot tests, update them.
   If not, verify the component renders without errors:
   ```bash
   # If snapshots exist, they'll catch unintended changes
   npx vitest run --reporter=verbose 2>&1
   # OR verify render
   npx vitest run src/features/<domain>/ 2>&1
   ```
   If no test infrastructure exists for the component, write a minimal render test:
   ```tsx
   // Verify component renders without throwing
   test('<Component> renders', () => {
     expect(() => render(<Component {...minimalProps} />)).not.toThrow()
   })
   ```

2. **Apply fix** — replace hardcoded values with tokens, add `aria-*` attributes,
   change `<div onClick>` to `<button>`, etc.

3. **Run full test suite + lint** — tests AND linter (if available):
   ```bash
   npx vitest run 2>&1
   npx eslint src/ 2>&1  # if eslint detected
   ```

4. **Compare against baseline** (see Gate Rules below).

### Configuration Fixes (i18n, lazy loading, bundle)

1. **Build check** — verify the project builds before the change:
   ```bash
   npm run build 2>&1 | tee /tmp/entropy-build-before.txt
   ```

2. **Apply fix** — extract strings to i18n, add `React.lazy()`, replace
   `import _ from 'lodash'` with `import groupBy from 'lodash/groupBy'`, etc.

3. **Build check + full test suite** — the build must still succeed and tests must pass.

4. **Compare against baseline** (see Gate Rules below).

### Documentation Fixes (stale docs, inaccurate references)

1. **Read the doc** — identify the stale claim (wrong count, dead reference, missing
   feature in list, outdated endpoint).

2. **Verify against code** — confirm what the correct value should be:
   ```bash
   # Example: doc says "10 agents" — count actual agents
   grep -c "### Agent" skills/entropy-scan/SKILL.md
   # Example: doc references app/billing/processor.py — check if it exists
   ls app/billing/processor.py 2>/dev/null || echo "MISSING"
   ```

3. **Update the doc** — fix the inaccuracy. Only change what's wrong, preserve
   the rest of the document. No rewrites.

4. **Build check** — if the project has a doc build (mkdocs, docusaurus, etc.),
   verify it still builds. Otherwise, run full test suite (doc changes can break
   tests that assert on doc content or generated docs).

5. **Compare against baseline** (see Gate Rules below).

### Repo Hygiene Fixes (orphaned files, deprecated locations)

1. **Verify the file is truly orphaned** — before deleting anything:
   ```bash
   # Check if any file in the repo references this doc
   basename="$(basename <file>)"
   grep -rl "$basename" --include="*.md" --include="*.py" --include="*.ts" \
     --include="*.json" . 2>/dev/null | grep -v "$file"
   # Check git log — was it recently modified?
   git log -1 --format="%ar" -- <file>
   ```
   If referenced or modified in the last 7 days, **skip** — it may still be active.

2. **Apply action**:
   - **Delete orphaned files**: `git rm <file>`
   - **Move from deprecated location**: `git mv docs/plans/<file> docs/superpowers/plans/<file>`
   - **Clean completed state files**: `git rm docs/entropy-loop-state.json`

3. **Run full test suite** — file deletions can break imports, references, and test fixtures.

4. **Compare against baseline** (see Gate Rules below).

**Safety rule**: Never delete files that are less than 7 days old or that have
references from other files. When in doubt, skip.

### CI/CD Fixes (Impactia 5-stage pipeline)

Target stages: (1) PR gate, (2) Build, (3) Deploy, (4) Post-deploy smoke,
(5) Rollback. See entropy-scan Agent 13 for full shape.

1. **Validate workflow YAML** before editing:
   ```bash
   # Preferred: actionlint
   actionlint .github/workflows/*.yml 2>&1
   # Fallback: YAML syntax check
   python -c "import yaml,sys; [yaml.safe_load(open(f)) for f in sys.argv[1:]]" \
     .github/workflows/*.yml
   ```

2. **Apply fix by stage**:
   - **Stage 1 missing (no PR gate)** → add `.github/workflows/ci.yml` with
     `on: pull_request` running lint + typecheck + unit tests. After merge,
     enable branch protection so the check is required.
   - **Stage 2 (`:latest` tag)** → change image tag to `${{ github.sha }}` in
     both the build step and the compose file pulled by the deploy step.
   - **Stage 3 (no env mapping)** → add `if: github.ref == 'refs/heads/main'`
     (prod) / `refs/heads/develop` (staging) guards, AND document the mapping
     in README or ARCHITECTURE.md.
   - **Stage 4 (no post-deploy smoke)** → add a job with `needs: deploy` that:
     - For Vercel: runs Playwright with `baseURL=${{ steps.deploy.outputs.url }}`
       (or the fixed prod URL from `secrets.DEPLOY_URL`).
     - For Hetzner: `curl -fsS https://<env-host>/health` + one Playwright
       flow against the deployed host.
     - Never uses `localhost` — that's a pre-deploy test, not a smoke.
   - **Stage 4 (smoke runs on localhost)** → change `baseURL` / target host
     to the deployed URL from env/secret. Fail the workflow on smoke failure.
   - **Stage 5 (no rollback)** → add a `Rollback` section to README or a
     `.github/workflows/rollback.yml` that re-runs deploy with a previous SHA
     input.
   - **Hardcoded secret** → replace with `${{ secrets.NAME }}`, add `NAME` to
     `docs/ENV.md` with purpose + owner, flag for rotation if already in git
     history.
   - **Undocumented secret** → add to `docs/ENV.md`, don't touch the workflow.
   - **Stale step** → verify the script/target exists in `package.json` /
     `pyproject.toml` / `Makefile`; remove or update.

3. **Verify on a feature branch** (never push workflow changes directly to
   `main`):
   ```bash
   git checkout -b chore/entropy-fix-cicd-<slug>
   git commit -m "ci: <what was fixed>"
   git push -u origin HEAD
   # Let the PR trigger the updated CI. Watch the run:
   gh run watch
   ```
   - PR gate run must be green.
   - If the fix touches Stages 2-4, merge to `develop` (staging) first and
     confirm the staging deploy + smoke succeeded before promoting to `main`.
   - Use the `entropy-check-deploy` skill to verify the deploy status if available.

4. **Run full test suite locally** — YAML edits can be paired with code/config
   changes (e.g., new test script in `package.json`).

5. **Compare against baseline** (see Gate Rules below).

**Safety rules**:
- Never `git push` a workflow change that would immediately run in production.
  Feature branch + PR only.
- Never skip Stage 4 smoke on the assumption "we'll add it later". Missing
  smoke is the #1 cause of silent prod breakage.
- If moving a secret out of hardcoded YAML, assume it has already leaked to
  git history — log it for rotation even if you're not rotating in this cycle.

### Gate Rules (all fix types)

After running verification:
- **Same or fewer failures/errors** → commit the fix:
  ```bash
  git add <specific files>
  git commit -m "refactor(<domain>): <what was fixed>"
  ```
- **More failures than baseline** → revert changes from this fix:
  ```bash
  git stash --include-untracked -m "entropy-fix: reverted <description>"
  git stash drop
  ```
  Log what broke and why, then move to the next fix. Do NOT retry
  the same approach — try a different strategy or skip.

Move to next correction — repeat the appropriate cycle.

**Hard rules:**
- NEVER skip the verification step. "It's just adding an aria-label" is not an excuse.
- NEVER commit with more failures than baseline. Zero tolerance.
- NEVER force a fix that breaks tests. Revert and move on.
- Prefer `git add <specific files>` over `git add -A` to avoid committing unrelated changes.
- If 3 consecutive fixes get reverted:
  - **Normal mode:** STOP and report to the user:
    > "3 corrections reverted in a row. The codebase may have tight coupling
    > that makes these refactorings risky. Need your input before continuing."
  - **Autonomous mode** (`--autonomous`): Log the issue, skip remaining fixes
    for this domain, and continue to the next domain. Do NOT stop.

## Phase 4: Re-scan

**Skip this phase if `--skip-rescan` was passed.**

After all corrections are applied, re-run `/entropy-scan` (or `/entropy-scan <domains>`
if partial fix) to verify improvements.

Present a before/after comparison:

```
Correction Results:
  Fixes applied:     N (structural: X, frontend: Y, config: Z)
  Fixes reverted:    M
  Tests before:      X passed, Y failed
  Tests after:       X' passed, Y' failed
  Lint errors:       N → N' (if applicable)
  Type errors:       N → N' (if applicable)
  Grade before:      C+
  Grade after:       B-
  Domains improved:  users (C→B), billing (D→C), frontend/dashboard (D→C)
```

If grades didn't improve as expected, investigate why and report to the user.

## Important Rules

- **Never break tests**: Every correction must pass the full suite against baseline. No exceptions.
- **Revert on failure**: If a fix causes more test failures than baseline, revert immediately. Don't debug — move on.
- **Safe revert**: Use `git stash --include-untracked` then `git stash drop` to cleanly remove both modified and new files without affecting other untracked files in the workspace.
- **Characterization tests first**: Before refactoring any code, capture its current behavior in a test. The test must pass before AND after the refactor.
- **Right verification for the fix type**: Structural refactors need characterization tests. Frontend markup needs render/snapshot tests. Config changes need build checks. All need full suite gate.
- **Security is always first**: D/F security findings are fixed before anything else, regardless of domain ordering.
- **Detect, don't assume**: Use the runners detected in Pre-flight. Don't hardcode `pytest` or `vitest` — the project may use `jest`, `mocha`, `django test`, or others.
- **Respect strategy level**: Never apply a fix that exceeds the current strategy level. Simple strategy = only atomic fixes. If a finding requires structural refactoring and strategy is simple, skip it.

## Rationalization Table

| Excuse | Reality |
|--------|---------|
| "It's just a rename, no need to run tests" | Renames break imports, serializers, and API contracts. Run the suite. |
| "The tests were already failing before" | That's what the baseline is for. You can't add MORE failures. |
| "I'll run tests after all fixes are done" | One broken fix contaminates all subsequent fixes. Test after EACH one. |
| "This refactoring is obviously correct" | Obvious refactorings break things. The test suite is the judge, not you. |
| "Writing a characterization test is overkill" | It takes 30 seconds and catches regressions that save hours. Write it. |
| "I'll skip the revert and fix the test instead" | Fixing the test means changing expected behavior. That's a feature change, not a refactor. Revert. |
| "Adding aria-label can't break anything" | It can break snapshot tests, E2E selectors, and test assertions. Run the suite. |
| "Design token replacement is just CSS" | CSS changes can break layout, responsive behavior, and visual regression tests. Verify. |
| "The linter doesn't matter, only tests" | Lint errors signal real issues (unused vars, type mismatches). They're part of the baseline. |
| "This domain has no tests, so nothing to gate" | Then write a render/smoke test first. No gate = no fix. |
| "This fix needs aggressive strategy but I'm on simple" | Then skip it. Strategy escalation is entropy-loop's job, not yours. |
| "Docs don't need tests, just update them" | Doc changes can break doc builds, test assertions, and generated content. Verify. |
| "This file looks orphaned, I'll just delete it" | Verify zero references first. Deleting a referenced file breaks things silently. |
| "Old plans are harmless, they're just history" | Orphaned plans pollute searches, confuse new contributors, and waste context. Clean up. |
| "CI works on my machine, no need to gate" | CI exists to catch what local runs miss. A pipeline without gates is decoration. Add them. |
| "Hardcoding the secret is faster" | Hardcoded secrets leak to git history forever. Always `secrets.*`. If it's already in history, rotate. |
| "Workflow YAML is not code, no need to verify" | A broken YAML blocks every future deploy. Validate before committing. |
| "Unit tests cover this, smoke is overkill" | Unit tests run on localhost; they've never caught a broken prod deploy. Smoke against the deployed URL is the only thing that does. |
| "E2E against localhost is the same thing" | It isn't. Localhost doesn't exercise the deployed infrastructure, env vars, SSL, CDN, or DNS. Stage 4 requires the deployed URL. |
| "`:latest` is fine for our size" | `:latest` makes rollback impossible and causes phantom deploys when the registry updates. SHA-pinned always. |
| "We deploy manually, we don't need rollback docs" | Manual deploys are exactly when rollback docs matter most. The person reverting at 2am is not you. |
