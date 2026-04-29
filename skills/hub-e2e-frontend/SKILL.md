---
name: hub-e2e-frontend
description: >
  Use when frontend code changed, Playwright or visual regression coverage is
  needed, or a deployed frontend must be verified after development, staging, or
  production deploy. Triggers on: hub-e2e-frontend, frontend tests, visual
  test, visual regression, playwright test, screenshot test, e2e frontend.
user_invocable: true
---

# Hub E2E Frontend — Frontend Gate

**Announce at start:** "Running /hub-e2e-frontend — frontend E2E tests with visual regression."

## Project Tracking & Governance

Source of truth for how Hub work is tracked lives in `_jockey/`:

- `_jockey/CONVENTIONS.md` — code, session, deploy, D1, verification rules. New conventions append under `## C## additions` sections.
- `_jockey/DECISIONS.md` — `DEC-C##-NN` decision log; cite when conventions reference a DEC.
- `_jockey/STATE.md` — current Control + active phase.
- Session prompts live in `_jockey/queue/` until fired, then move to `_jockey/archive/fired/`. Naming: `C[N]-S[#]v[ver]-[name].md`.
- New behavioral conventions or decisions that emerge from this skill's run must land in `_jockey/DECISIONS.md` (DEC entry) and a `## C## additions` block in `_jockey/CONVENTIONS.md`.

> **Coexists with skill-level tracking — neither invalidates the other.** This is program-level governance. The skill's own tracking artifacts (`docs/TICKETS.md`, `docs/QUALITY_SCORE.md`, `docs/hub-loop-state.json`, evidence files in `docs/`, etc.) remain the authoritative source for the skill's operational state. `_jockey/` is the program-level layer (Control, conventions, decisions). Both must coexist.

## Overview

Run Playwright E2E tests that cover functional flows and visual regression via
screenshot comparison. If tests don't exist for changed frontend files, generate
them in pre-merge mode. In post-deploy mode, verify the deployed environment and
never update baselines automatically.

Can be invoked standalone or by `/hub-ship` (Phase 3.5).
It is also the post-deploy frontend gate used by hub autopilot.

## Hub Contract

Structured handoff:

```json
{
  "description": "Add retry button to checkout",
  "changed_files": ["src/app/checkout/page.tsx"],
  "affected_urls": ["/checkout"],
  "mode": "pre_merge_local",
  "target_env": "local",
  "base_url": "http://localhost:3000",
  "autopilot": false
}
```

| Field | Default | Purpose |
|---|---|---|
| `description` | optional | Human-readable feature/bug context. |
| `changed_files` | git diff fallback | Frontend files to map to E2E coverage. |
| `affected_urls` | inferred | Routes to prioritize in generated tests. |
| `mode` | `pre_merge_local` | `pre_merge_local` generates/updates tests; `post_deploy` only verifies deployed behavior. |
| `target_env` | `local` | `local`, `development`, `staging`, `production`, or legacy `deploy`. |
| `base_url` | env/config | URL Playwright should hit for this run. |
| `autopilot` | `false` | When true, never prompt; return structured failure/unconfigured status. |

Result contract:

```text
HUB-E2E-FRONTEND-RESULT:
{
  "result": "pass | fail | skipped | unconfigured",
  "mode": "pre_merge_local | post_deploy",
  "target_env": "local | development | staging | production | deploy",
  "base_url": "https://dev.example.com",
  "changed_files": [],
  "tests": {"passed": 0, "failed": 0, "skipped": 0},
  "visual": {"new": 0, "updated": 0, "diffs": 0},
  "report_path": "playwright-report/index.html",
  "artifact_path": "docs/hub-e2e-frontend-runs/...",
  "warnings": []
}
```

Fields that do not apply must be `null`, not omitted.

---

## Phase 0: Credential Bootstrap

Ensures the skill has a base URL + test user credentials. On first interactive
run, prompts and persists to `.env.local`. In autopilot mode, never prompts:
missing required config returns `result=unconfigured`.

### 0a: Determine target env

When invoked with a structured handoff (see Phase 2a), read `mode` and
`target_env`:

- `mode="pre_merge_local"` + `target_env="local"` → required var: `VR_BASE_URL`
- `mode="post_deploy"` + `target_env="development"` → `VR_DEVELOPMENT_URL`, then `VR_DEPLOY_URL`
- `mode="post_deploy"` + `target_env="staging"` → `VR_STAGING_URL`, then `VR_DEPLOY_URL`
- `mode="post_deploy"` + `target_env="production"` → `VR_PRODUCTION_URL`, then `VR_DEPLOY_URL`
- Legacy `target_env="deploy"` → `VR_DEPLOY_URL`

If `base_url` is provided in the handoff, use it before env vars.

Credentials `VR_USER_EMAIL` and `VR_USER_PASSWORD` are required only when
Phase 3 determines the app has a login flow. Do not fail Phase 0 only because
credential vars are missing.

### 0b: Check .env.local

```bash
ENV_FILE=".env.local"
MISSING=()
AUTH_MISSING=()
for var in VR_USER_EMAIL VR_USER_PASSWORD; do
  if [ ! -f "$ENV_FILE" ] || ! grep -q "^${var}=" "$ENV_FILE"; then
    AUTH_MISSING+=("$var")
  fi
done

case "$TARGET_ENV" in
  local|"")
    URL_VAR="VR_BASE_URL"
    ;;
  development)
    URL_VAR="VR_DEVELOPMENT_URL"
    ;;
  staging)
    URL_VAR="VR_STAGING_URL"
    ;;
  production)
    URL_VAR="VR_PRODUCTION_URL"
    ;;
  deploy|*)
    URL_VAR="VR_DEPLOY_URL"
    ;;
esac

if [ -z "$BASE_URL" ]; then
  if [ ! -f "$ENV_FILE" ] || ! grep -q "^${URL_VAR}=" "$ENV_FILE"; then
    if [ "$URL_VAR" != "VR_BASE_URL" ] && grep -q "^VR_DEPLOY_URL=" "$ENV_FILE" 2>/dev/null; then
      URL_VAR="VR_DEPLOY_URL"
    else
      MISSING+=("$URL_VAR")
    fi
  fi
fi
```

If `MISSING` is empty: skip to Phase 1.
If `AUTOPILOT=true` and `MISSING` is not empty: stop with
`HUB-E2E-FRONTEND-RESULT.result = "unconfigured"` and include missing vars in
`warnings`.

### 0c: Prompt user

For each var in `MISSING`, ask the user one at a time. Prompt for
`AUTH_MISSING` only later if Phase 3 determines login is required:

- `VR_BASE_URL` → "Local dev server URL? (e.g. http://localhost:3000)"
- `VR_DEVELOPMENT_URL` → "Development deploy URL? (e.g. https://dev.example.com)"
- `VR_STAGING_URL` → "Staging deploy URL? (e.g. https://staging.example.com)"
- `VR_PRODUCTION_URL` → "Production deploy URL? (e.g. https://example.com)"
- `VR_DEPLOY_URL` → "Deployed URL for this feature? (e.g. https://staging.example.com)"
- `VR_USER_EMAIL` → "Test user email?"
- `VR_USER_PASSWORD` → "Test user password?"

After collecting the required vars, ask once:
> "Any extras needed? (API key / org slug / second role) — press enter to skip."

If the user provides extras, ask for each var name + value, append to the collected set.

```bash
# After collecting answers, build the array used by 0d and 0f:
COLLECTED=()
# For each var in MISSING plus any extras, append:
#   COLLECTED+=("${var}=${answer}")
```

### 0d: Persist to .env.local

```bash
touch "$ENV_FILE"
for var_line in "${COLLECTED[@]}"; do
  # var_line is "KEY=VALUE"
  key="${var_line%%=*}"
  # Remove any existing entry for this key, then append
  if [ -f "$ENV_FILE" ]; then
    grep -v "^${key}=" "$ENV_FILE" > "${ENV_FILE}.tmp" || true
    mv "${ENV_FILE}.tmp" "$ENV_FILE"
  fi
  echo "$var_line" >> "$ENV_FILE"
done
```

### 0e: Ensure .gitignore

```bash
if ! grep -qxF ".env.local" .gitignore 2>/dev/null; then
  echo ".env.local" >> .gitignore
  git add .gitignore
fi
```

### 0f: Add pointer to CLAUDE.md or AGENTS.md

Detect which file the repo uses:

```bash
AGENT_DOC=""
[ -f CLAUDE.md ] && AGENT_DOC="CLAUDE.md"
[ -z "$AGENT_DOC" ] && [ -f AGENTS.md ] && AGENT_DOC="AGENTS.md"
[ -z "$AGENT_DOC" ] && AGENT_DOC="CLAUDE.md"  # create if neither exists
```

Build the pointer line from the actual vars collected (the `COLLECTED` list):

```
Visual regression credentials live in .env.local (gitignored). Vars: VR_BASE_URL, VR_USER_EMAIL, VR_USER_PASSWORD[, VR_DEPLOY_URL, ...extras].
```

Append the pointer only if not already present:

```bash
VAR_LIST=$(printf '%s\n' "${COLLECTED[@]}" | cut -d= -f1 | paste -sd, -)
POINTER="Visual regression credentials live in .env.local (gitignored). Vars: ${VAR_LIST}."
if ! grep -qF "Visual regression credentials live in .env.local" "$AGENT_DOC" 2>/dev/null; then
  printf "\n%s\n" "$POINTER" >> "$AGENT_DOC"
  git add "$AGENT_DOC"
fi
```

### 0g: Commit bootstrap changes

```bash
git commit -m "chore(e2e): bootstrap visual regression credentials" || true
```

(The `|| true` covers the case where nothing changed on a re-run.)

---

## Frontend-Change Detection (shared)

Callers use this block to decide whether to invoke `/hub-e2e-frontend`.

```bash
FRONTEND_GLOBS='\.(tsx|jsx|vue|svelte|astro)$|^(app|pages|src/components|src/pages|src/routes)/'
BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||')
BASE_BRANCH=${BASE_BRANCH:-main}
BASE=$(git merge-base HEAD "$BASE_BRANCH")
CHANGED=$(git diff "$BASE"...HEAD --name-only | grep -E "$FRONTEND_GLOBS" || true)

if [ -z "$CHANGED" ]; then
  echo "hub-e2e-frontend: no frontend changes, skipping"
  exit 0
fi
```

If `$CHANGED` is non-empty, invoke `/hub-e2e-frontend` with a handoff payload.
Callers may override `FRONTEND_GLOBS` by reading `.visual-regression.json` if
present.

---

## Phase 1: Pre-flight

### 1a: Verify Playwright

```bash
npx playwright --version 2>/dev/null
```

- If not installed, offer to install:
  ```bash
  npm install -D @playwright/test
  npx playwright install chromium
  ```
- If user declines: STOP. Cannot run without Playwright.
- In `autopilot=true`, never offer installation. Return
  `result=unconfigured` with a warning naming the missing Playwright dependency.

### 1b: Check playwright.config.ts

```bash
test -f playwright.config.ts || test -f playwright.config.js
```

If missing, generate `playwright.config.ts` with these defaults.

In `mode="post_deploy"`, do not generate config. Return
`result=unconfigured` in autopilot mode, or ask the user to add Playwright
config in interactive mode.

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: 'e2e',
  snapshotDir: 'e2e/__snapshots__',
  fullyParallel: true,
  retries: 1,
  reporter: 'html',
  use: {
    baseURL: process.env.BASE_URL || process.env.VR_DEPLOY_URL || process.env.VR_DEVELOPMENT_URL || process.env.VR_BASE_URL || 'http://localhost:3000',
    screenshot: 'only-on-failure',
    viewport: { width: 1280, height: 720 },
    reducedMotion: 'reduce',
  },
  expect: {
    toHaveScreenshot: {
      maxDiffPixelRatio: 0.01,
    },
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: true,
    timeout: 30_000,
  },
});
```

Adjust `baseURL` and `webServer.command` based on the project:
- Read `.env.local` (VR_BASE_URL) or `.env`/`CLAUDE.md` for dev server URL
- Read `package.json` scripts for the correct dev command
- If no `dev` script exists: ask the user for the start command. In
  `autopilot=true`, return `result=unconfigured` instead.

Commit if generated:
```bash
git add playwright.config.ts
git commit -m "chore: add Playwright config for frontend E2E"
```

### 1c: Verify dev server

```bash
curl -s -o /dev/null -w "%{http_code}" --max-time 5 "<baseURL>"
```

- If reachable (200): continue.
- If not reachable: check if `playwright.config.ts` has a `webServer` block.
  - If yes: Playwright will auto-start it. Continue.
  - If no: add a `webServer` block (see template above) and commit.
  - If `package.json` has no `dev` script: ask the user for the start command.
    In `autopilot=true`, return `result=unconfigured` instead.

### 1d: Ensure .gitignore entries

Check that these are in `.gitignore` (add if missing):

```
test-results/
playwright-report/
blob-report/
```

Commit if modified:
```bash
git add .gitignore
git commit -m "chore: add Playwright entries to .gitignore"
```

### 1e: Storybook detection

```bash
STORYBOOK=false
test -d .storybook && STORYBOOK=true
```

Store flag for Phase 3 (component test generation strategy).

---

## Phase 2: Analyze Changes

### 2a.0: Check for structured handoff

When invoked by another skill (e.g., `hub-driven`, `hub-bugfix`), the
caller may pass a handoff payload:

```json
{
  "description": "<human-readable feature or bug description>",
  "changed_files": ["src/app/billing/page.tsx", "..."],
  "affected_urls": ["/billing", "..."],
  "mode": "pre_merge_local",
  "target_env": "local",
  "base_url": "http://localhost:3000",
  "autopilot": false
}
```

Scoping priority:

1. **Handoff present** → use `changed_files` directly; skip git diff. Use
   `affected_urls` as the initial route list if provided.
2. **User description present** → infer routes by grepping the repo for the
   described feature, then fall through to git diff for file list.
3. **Neither** → git diff fallback (existing 2a logic).

Mode policy:

- `pre_merge_local`: may generate missing tests, may update intentional
  baselines after user approval, and may commit generated E2E files.
- `post_deploy`: never generates new tests, never updates baselines, never
  prompts in autopilot mode, and treats visual differences as regressions.

Record scoping source, mode, target env, and base URL in the final report and
artifact.

### 2a: Detect changed frontend files

```bash
# Inherit BASE from /hub-ship when invoked as sub-skill.
# Compute when standalone.
if [ -z "$BASE" ]; then
  BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||')
  BASE_BRANCH=${BASE_BRANCH:-development}
  BASE=$(git merge-base HEAD "$BASE_BRANCH")
fi
git diff "$BASE"...HEAD --name-only | grep -E '\.(tsx|jsx)$'
```

Only `.tsx` and `.jsx` files are mapped to E2E tests. Changes to `.ts`, `.css`,
and `.scss` are tested indirectly through the pages/components that import them.

### 2b: Map files to test paths

Apply these rules in order:

| Source pattern | Test path |
|---------------|-----------|
| `src/app/<route>/page.tsx` | `e2e/<route>.spec.ts` |
| `src/app/<route>/layout.tsx` | `e2e/<route>-layout.spec.ts` |
| `src/components/**/<Name>.tsx` | `e2e/components/<name>.spec.ts` |
| No match | Skip — note in report |

**Monorepo check:** if changed files are under `packages/` or `apps/`:
> "Monorepo detected. Run `/hub-e2e-frontend` from the relevant package directory
> (e.g., `cd packages/web && /hub-e2e-frontend`). Each package needs its own
> `playwright.config.ts` and E2E tests."
> STOP. If invoked as a sub-skill by `/hub-ship`, return control —
> `/hub-ship` continues with backend E2E in Phase 4.

### 2c: Check existing tests

```bash
for test_path in <mapped_paths>; do
  test -f "$test_path" && echo "EXISTS: $test_path" || echo "MISSING: $test_path"
done
```

Report the analysis:
> "Found N changed frontend files. M have E2E tests, K need tests, J skipped (no mapping)."

---

## Phase 3: Generate Missing Tests

If `mode="post_deploy"`, skip this phase entirely. Post-deploy verification
must validate the already-merged test suite against the deployed URL; generating
new tests after deploy hides missing pre-merge coverage. Record missing mapped
tests as warnings in the final result.

For each changed file with no corresponding E2E test:

1. **Read the source file** to understand its purpose.

2. **Determine how to reach it in the browser:**
   - **Pages** → derive route from file path in `src/app/`
   - **Components with Storybook** (`STORYBOOK=true`) → check if a
     `.stories.tsx` file exists for the component. If yes, use Storybook URL.
   - **Components without Storybook** → search for imports of the component
     across pages, navigate to the first page that uses it. If no page imports
     the component, skip it and note in the report.

3. **Check for / generate login helper:**

   ```bash
   test -f e2e/login.helper.ts
   ```

   - If it exists → skip to step 4 (import it in the generated spec).
   - If missing AND the app likely requires login (check for `middleware.ts`,
     auth providers, or login routes like `/login`, `/signin`):
     - If `autopilot=true` and any `AUTH_MISSING` vars remain, return
       `result=unconfigured` with warnings listing the missing vars.
     - Ask the user once for the login selectors:
       > "I need to generate a login helper. What selectors identify:
       > - the email input? (default: `input[type='email']`)
       > - the password input? (default: `input[type='password']`)
       > - the submit button? (default: `button[type='submit']`)
       > - the login URL? (default: `/login`)"
     - Generate `e2e/login.helper.ts`:

       ```typescript
       import { Page } from '@playwright/test';

       export async function login(page: Page) {
         const url = process.env.BASE_URL || process.env.VR_DEPLOY_URL || process.env.VR_DEVELOPMENT_URL || process.env.VR_BASE_URL || '';
         await page.goto(url + '<LOGIN_URL>');
         await page.fill('<EMAIL_SELECTOR>', process.env.VR_USER_EMAIL || '');
         await page.fill('<PASSWORD_SELECTOR>', process.env.VR_USER_PASSWORD || '');
         await page.click('<SUBMIT_SELECTOR>');
         await page.waitForLoadState('networkidle');
       }
       ```

       Substitute the placeholders with the collected selectors.
     - Commit:
       ```bash
       git add e2e/login.helper.ts
       git commit -m "test(e2e): add login helper for visual regression"
       ```
   - If missing AND the app has no login flow → continue without helper.

4. **Generate the test file** following this convention:

```typescript
import { test, expect } from '@playwright/test';
// If e2e/login.helper.ts exists, include this import and the beforeEach:
// import { login } from './login.helper';

test.describe('<Page/Component Name>', () => {
  // test.beforeEach(async ({ page }) => { await login(page); });

  test('renders correctly', async ({ page }) => {
    await page.goto('<route>');
    // functional assertions
    await expect(page.getByRole('heading')).toBeVisible();
    // visual snapshot
    await expect(page).toHaveScreenshot('<name>.png');
  });

  test('<specific functional behavior>', async ({ page }) => {
    await page.goto('<route>');
    // interaction + assertion
  });
});
```

5. After generating all missing test files, **commit them in a single batch:**
```bash
git add e2e/
git commit -m "test(e2e): add frontend E2E tests for <feature>"
```

If all tests already exist: skip to Phase 4.

**Note:** generated tests are a starting point (green-first baseline), not a
TDD substitute. Developers should expand them during `/tdd` cycles.

---

## Phase 4: Run Tests

Resolve the effective base URL:

1. `input.base_url`
2. `.env.local` value selected in Phase 0 (`VR_BASE_URL`,
   `VR_DEVELOPMENT_URL`, `VR_STAGING_URL`, `VR_PRODUCTION_URL`, or
   `VR_DEPLOY_URL`)
3. `BASE_URL`

In `post_deploy`, the effective base URL must not be localhost. If it is missing
or points at `localhost`, return `result=unconfigured` in autopilot mode; in
interactive mode, ask for the deploy URL once.

```bash
BASE_URL="$EFFECTIVE_BASE_URL" npx playwright test --reporter=html 2>&1
```

### Functional failures

- In `pre_merge_local`: analyze each failure as feature bug or test bug, fix,
  commit, and re-run. Max 3 iterations.
- In `post_deploy`: do **not** edit code or tests from this skill. Report the
  failure as a deployed regression and return `result=fail`. The caller
  (`hub-ship` or `hub-driven-autopilot`) decides whether to enter a
  fix/redeploy loop.

### Visual failures (screenshot mismatch)

- Report which screenshots differ, the diff percentage, and diff image
  locations: `test-results/<test-name>/`
- In `pre_merge_local`, ask the user:
  > "These screenshots have visual differences: [list]. Diff images are in
  > `test-results/`. Are these changes intentional?"
- If intentional: update only the affected baselines
  ```bash
  npx playwright test "<specific-test-file>" --update-snapshots
  git add e2e/__snapshots__/
  git commit -m "test: update visual baselines for <feature>"
  ```
- If not intentional: report as regression, user must fix before proceeding.
- In `post_deploy`, never update baselines. Any screenshot mismatch is a
  deployed regression and returns `result=fail`.
- In `autopilot=true`, never ask. Treat screenshot mismatch as `result=fail`.

### After 3 failed iterations

STOP and report:
> "After 3 attempts, these tests are still failing: [list]. I need your input."

---

## Phase 5: Report

Write an artifact for every run:

```text
docs/hub-e2e-frontend-runs/YYYY-MM-DDTHH-MM-SSZ-<mode>-<target_env>.json
```

Include parsed input, effective base URL, changed files, mapped tests, command
run, Playwright summary, visual diff counts, report paths, warnings, and final
`result`.

```
/hub-e2e-frontend result:

Mode:               pre_merge_local | post_deploy
Target env:         local | development | staging | production | deploy
Base URL:           <url>
Result:             pass | fail | skipped | unconfigured
Tests:              X passed, Y failed, Z skipped
Visual baselines:   N new, M updated, P unchanged
Skipped files:      [files with no mapping rule, if any]
Report:             playwright-report/index.html
Diff images:        test-results/ (only if visual failures)
Artifact:           docs/hub-e2e-frontend-runs/...
Duration:           Xs

Regressions:        [list if any, or "none"]
```

Then emit this machine-readable block exactly:

```text
HUB-E2E-FRONTEND-RESULT:
{
  "result": "pass | fail | skipped | unconfigured",
  "mode": "pre_merge_local | post_deploy",
  "target_env": "local | development | staging | production | deploy",
  "base_url": "https://dev.example.com",
  "changed_files": ["src/app/checkout/page.tsx"],
  "affected_urls": ["/checkout"],
  "tests": {
    "passed": 0,
    "failed": 0,
    "skipped": 0
  },
  "visual": {
    "new": 0,
    "updated": 0,
    "diffs": 0
  },
  "report_path": "playwright-report/index.html",
  "diff_path": "test-results/",
  "artifact_path": "docs/hub-e2e-frontend-runs/...",
  "warnings": []
}
```

Fields that do not apply must be `null`, not omitted. Callers parse the
`HUB-E2E-FRONTEND-RESULT:` marker.

---

## Common Mistakes

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Testing `.ts`/`.css` files directly | These aren't renderable — tests will fail or be meaningless | Only map `.tsx`/`.jsx` to E2E tests |
| Auto-accepting screenshot diffs | Visual regressions ship silently | Always show diffs and ask the user |
| Skipping auth fixture setup | All tests fail with login redirect | Check for auth requirements in Phase 3 |
| Running only new tests in Phase 4 | Existing E2E regressions go undetected | Run the full Playwright suite |
| Hardcoding `localhost:3000` as baseURL | Tests fail in CI or against deployed env | Read baseURL from config/env |
| Generating tests for unexposed components | Tests can't navigate to the component | Skip components not imported by any page |

## Important Rules

- **Evidence before claims**: Never say "tests pass" without showing actual Playwright output.
- **Full suite always**: Run ALL Playwright tests, not just the ones you generated.
- **Visual diffs need user approval**: Never auto-accept screenshot mismatches.
- **Post-deploy is verification only**: Do not generate tests, update baselines,
  or edit code in `post_deploy` mode.
- **Autopilot never prompts**: Missing URLs, auth config, or visual diffs become
  structured `unconfigured` / `fail` results.
- **Max 3 retries**: Stop and ask for help after 3 failed fix attempts.
- **Commit between phases**: In `pre_merge_local`, each phase that edits files
  ends with a clean git state. In `post_deploy`, do not edit or commit files.
- **Don't test non-renderable files**: Only `.tsx`/`.jsx` get mapped to E2E tests.
