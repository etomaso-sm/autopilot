---
name: hub-scan
description: >
  Use when you want a macro health check of the entire codebase: quality
  grades per domain, duplicated patterns, boundary validation, and tech debt.
  Triggers on: hub-scan, code health, quality scan, tech debt,
  codebase audit, quality grades, repo health.
user_invocable: true
---

<!-- SYNC: hub-scan is the source of truth for dimension definitions (13 agents).
     When changing agents, update ALL dependents:
     - hub-aware: dimension tables (specs + plans), count in heading/description
     - hub-fix: priority order, fix type classification, dimension names
     - hub-bugfix: conditional partial scan rules for elevated/sensitive fixes
     - hub-loop: strategy levels, agent count in scan logic
     - hub-driven: references hub-scan for validation, hub-loop as fallback -->

# Hub Scan

**Announce at start:** "Running /hub-scan — scanning codebase health and quality."

## Overview

Macro-level codebase health scan. Grades each project domain, detects
duplicated patterns, validates boundaries, and tracks tech debt.
Generates a persistent `docs/QUALITY_SCORE.md` file committed to git.

Complementary to `simplify` (micro: changed files) — this is macro (whole repo).

### Partial Scan

Supports scanning specific domains: `/hub-scan billing users`

When domains are specified:
- Skip Phase 1 domain detection — use the provided domain names
- Match domain names against the project structure (fuzzy: `billing` matches `app/billing/`, `src/features/billing/`, etc.)
- Only grade the specified domains in Phase 2
- Tech Debt summary (Phase 3) is still full-repo
- Quality Score (Phase 4) updates only the specified domains, preserving previous grades for unscanned domains
- Report (Phase 5) shows only scanned domains but includes the full grade table for context

## Pre-flight

```bash
echo "Scanning repository"
```

Parse arguments: if domain names were provided (e.g., `/hub-scan billing users`),
store them as the target domain list. Otherwise, scan all domains.

Read `docs/QUALITY_SCORE.md` if it exists to compare with previous scan.
For partial scans, preserve the previous grades for domains not being re-scanned.

## Phase 1: Identify Domains

Scan the project structure to identify business domains using multiple strategies
in priority order:

### Strategy 1: Architecture doc (highest priority)
If `ARCHITECTURE.md`, `docs/ARCHITECTURE.md`, or `SPEC.md` exists, use its domain
map. These are authoritative — skip other strategies.

### Strategy 2: Framework-aware detection
Detect the project type and scan accordingly:

```bash
# Django: look for apps directories
find . -name "apps.py" -not -path "*/node_modules/*" -not -path "*/.venv/*" | head -20

# Python packages: look for __init__.py at consistent depths
find . -maxdepth 3 -name "__init__.py" -not -path "*/node_modules/*" -not -path "*/.venv/*" | head -30

# Frontend: features, pages, or modules directories
ls -d src/features/*/ src/pages/*/ src/modules/*/ app/*/ 2>/dev/null

# FastAPI/general Python: look for routers or route modules
find . -name "router*.py" -o -name "routes*.py" -not -path "*/.venv/*" | head -20

# Monorepo: look for packages/services with their own package.json or pyproject.toml
find . -maxdepth 3 \( -name "package.json" -o -name "pyproject.toml" -o -name "Cargo.toml" -o -name "go.mod" \) -not -path "*/node_modules/*" | head -20
```

### Strategy 3: Import graph inference (fallback)
If Strategies 1-2 yield fewer than 2 domains, analyze import patterns:
- Group files by shared imports to identify clusters
- Files that import each other heavily likely belong to the same domain
- Files that are imported from many places are likely shared infrastructure, not a domain

### Output
List all domains found with their root paths. Example:
```
Domains detected (Strategy 2 — Django apps):
  - users: app/users/
  - billing: app/billing/
  - notifications: app/notifications/
  - frontend/dashboard: src/features/dashboard/
  - frontend/auth: src/features/auth/
```

## Phase 2: Grade Each Domain

For each domain, evaluate using 12 parallel agents:

### Agent 1: Test Coverage
- Count test files vs source files in the domain
- **Identify test files** using these patterns (check all):
  - Files matching `*.test.*`, `*.spec.*`, `*_test.*`, `test_*.*`
  - Files inside `tests/`, `__tests__/`, `test/` directories
  - Co-located tests (e.g., `Component.test.tsx` next to `Component.tsx`)
- Check if key files (models, views, serializers, components) have corresponding tests
- Grade: A (>90%), B (70-90%), C (50-70%), D (30-50%), F (<30%)

### Agent 2: DRY — Duplication Detection
- Search for similar function names, patterns, and implementations across the domain
- Look for helpers/utilities reimplemented in multiple places
- Look for copy-pasted logic with minor variations (same structure, different names)
- Check for repeated query patterns, validation logic, or error handling
- **Grading**: Scale relative to domain size (source files, excluding tests):
  - **Small domain** (1-10 files): A (0), B (1), C (2), D (3), F (>3)
  - **Medium domain** (11-30 files): A (0), B (1-2), C (3-4), D (5-7), F (>7)
  - **Large domain** (31+ files): A (0-1), B (2-3), C (4-6), D (7-10), F (>10)
  - Count source files first, then apply the appropriate scale
- **Report each finding with:** file paths, line numbers, and what's duplicated

### Agent 3: Boundary Validation
- Check API inputs: do they use schema validation (serializers, Zod, etc.)?
- Check responses: are they typed/serialized or raw dicts/objects?
- Check inter-module boundaries: are imports going in the right direction?
- Grade: A (all validated), B (1 gap), C (2-3), D (4-5), F (>5)

### Agent 4: Documentation, Accuracy & Complexity

Two sub-evaluations: **Complexity** (static) and **Accuracy** (dynamic).

**Complexity:**
- Are there files > 300 lines? Functions > 50 lines?
- Count TODOs, FIXMEs, HACKs

**Accuracy — does documentation reflect reality?**
- **README/SPEC.md**: Do they list features, endpoints, skills, or components that
  actually exist? Cross-reference against the file system.
  ```bash
  # Example: if README lists "Available Skills", check each skill name exists
  # If SPEC.md lists endpoints, verify the routes exist in code
  ```
- **API docs**: Do documented endpoints match actual route definitions?
- **Config docs**: Do documented env vars, settings, or flags match actual usage?
- **Stale references**: Do docs reference files, functions, classes, or modules that
  have been renamed, moved, or deleted?
  ```bash
  # Find file references in docs and verify they exist
  grep -oE '[a-zA-Z_/]+\.(py|ts|tsx|js|jsx|md)' docs/**/*.md 2>/dev/null | while read f; do
    [ ! -f "$f" ] && echo "STALE: $f referenced in docs but doesn't exist"
  done
  ```
- **Count/dimension mismatches**: Do docs claiming "N items/dimensions/agents" match
  the actual count in code? (e.g., "10 agents" when there are actually 12)
- **Changelog/history**: If present, does the latest entry match recent git activity?

Grade:
- A: Documented, accurate, no files >300 lines, no functions >50 lines, 0 TODOs
- B: Documented, 1-2 minor inaccuracies OR 1-2 oversized files OR 1-3 TODOs
- C: Partial docs, some stale references, 3-5 oversized files, 4-8 TODOs
- D: No docs OR multiple stale/inaccurate references, 6+ oversized files, 9-15 TODOs
- F: No docs, widespread inaccuracy, majority of files >300 lines, 15+ TODOs

**Report each finding with:** type (complexity/accuracy), file:line, what's wrong, fix.

Example finding:
| Type | File | Line | Issue | Fix |
|------|------|------|-------|-----|
| Accuracy | README.md | 59 | Lists 15 skills but repo has 20 | Update skills table |
| Accuracy | SPEC.md | 34 | References `app/billing/processor.py` which was renamed to `service.py` | Update path |
| Accuracy | docs/api.md | 12 | Documents `/api/v1/users` but route is now `/api/v2/users` | Update endpoint |
| Complexity | app/billing/views.py | — | 450 lines | Split into viewsets |
| Stale count | docs/hub-spec.md | 25 | Says "10 dimensions" but hub-scan has 12 | Update count |

### Agent 5: Principles Compliance (SOLID & Clean Architecture)

**First:** Read `~/.ai-skills/method/PRINCIPLES.md` to load the team's 10 non-negotiable
principles. If the file doesn't exist, fall back to standard SOLID evaluation.

Evaluate each relevant principle from PRINCIPLES.md against the domain code:

- **Principle 2 — Single Responsibility**: Classes/modules doing more than one thing? Files with mixed concerns (e.g., business logic + HTTP handling + DB queries in the same function)?
- **Principle 3 — Don't Repeat Yourself**: (Cross-checked with Agent 2, but here evaluate at the architectural level — are there duplicate abstractions, redundant service layers, or parallel hierarchies?)
- **Principle 4 — Open/Closed**: Can features be extended without modifying core code? Are there switch/if chains that grow with each new case instead of using polymorphism or registries?
- **Principle 5 — Depend on Abstractions**: Do high-level modules depend on low-level modules directly? Are dependencies injected or hardcoded?
- **Principle 6 — Minimal Surface Area**: Are APIs exposing more than consumers need? Are there public methods/exports that should be private?
- **Principle 9 — Fail Fast and Loud**: Are errors swallowed? Empty catch blocks? Silent failures? Missing error propagation?
- **Principle 10 — Simplicity Over Cleverness**: Over-engineering? Premature abstractions? Unnecessary indirection?

Also check Clean Architecture patterns:
- **Layer separation**: Is business logic mixed with framework code (views calling ORM directly, components with inline API calls)?
- **Use case isolation**: Are business operations in dedicated service/use-case layers, or scattered across views/controllers?
- **Framework independence**: Could you swap Django for FastAPI or React for Vue without rewriting business logic?

Grade: A (all principles followed), B (1-2 minor violations), C (3-5 violations), D (widespread violations), F (no structure)

**Report each violation with:** principle number + name, file:line, what's wrong, and how to fix it.

Example finding:
| Principle | File | Line | Issue | Fix |
|-----------|------|------|-------|-----|
| P2 — Single Responsibility | app/billing/views.py | 45 | View contains business logic, ORM queries, and email sending | Extract to BillingService |
| P5 — Depend on Abstractions | app/billing/tasks.py | 12 | Hardcoded import of StripeClient | Inject via constructor parameter |
| P9 — Fail Fast and Loud | app/auth/middleware.py | 78 | Empty except block swallows auth errors | Raise or log with context |

### Agent 6: Design Patterns
- **Missing patterns**: Where would a pattern simplify the code? (e.g., repeated object creation → Factory, state machines → State pattern, notifications → Observer)
- **Anti-patterns detected**: God objects, anemic domain models, service locator abuse, circular dependencies, deep inheritance hierarchies
- **Pattern misuse**: Over-engineering with unnecessary abstractions, patterns applied where a simple function would suffice

Grade: A (clean, patterns used where appropriate), B (1-2 opportunities missed), C (3-5 issues), D (anti-patterns widespread), F (no discernible structure)

**Report each finding with:** location, what pattern is missing/misused, and concrete suggestion.

### Agent 7: Security Hygiene (Principle 7)

**First:** Read `~/.ai-skills/method/PRINCIPLES.md` Principle 7 — Security by Default.

Evaluate the domain against security best practices:

- **Authentication & Authorization**: Do all endpoints have explicit permission checks? Are there views/routes missing `@login_required`, `permission_classes`, or equivalent guards?
- **Input Validation**: Are all external inputs validated at the boundary? Look for unvalidated request data, missing serializer validation, raw `request.POST`/`request.GET` access without cleaning.
- **SQL/ORM Safety**: Any raw SQL queries? String formatting/f-strings in queries? `.extra()` or `.raw()` calls without parameterization?
- **Secret Management**: Hardcoded API keys, passwords, or tokens in source code? Secrets in settings files that should be environment variables?
- **Output Escaping**: Are responses properly serialized? Any `mark_safe()`, `dangerouslySetInnerHTML`, or template `|safe` usage without justification?
- **CORS/CSRF/Headers**: Is CORS configured restrictively? CSRF protection enabled? Security headers present (CSP, X-Frame-Options, etc.)?
- **Dependency Risks**: Are there known-vulnerable dependencies? (Check if `safety`, `npm audit`, or equivalent has been run recently)
- **Dependency Health**: Are dependencies actively maintained? Check for:
  - Outdated major versions (>2 major versions behind = risk)
  - Abandoned packages (no commits in 12+ months, archived repos)
  - License compliance (copyleft licenses in proprietary projects, missing license declarations)
  - Dependency bloat (unused dependencies still in requirements/package.json)
  ```bash
  # Python: check outdated
  pip list --outdated 2>/dev/null | head -20
  # Node: check outdated
  npm outdated 2>/dev/null | head -20
  # Check for unused deps (Python)
  grep -r "^import\|^from" --include="*.py" . | awk '{print $2}' | sort -u > /tmp/used-imports.txt
  ```

Grade: A (all secure, deps healthy), B (1 minor gap), C (2-3 gaps), D (4-5 gaps including auth/input or multiple outdated/abandoned deps), F (critical vulnerabilities — raw SQL, hardcoded secrets, missing auth, known CVEs in deps)

**Report each finding with:** category, file:line, what's exposed, and how to fix it.

Example finding:
| Category | File | Line | Issue | Fix |
|----------|------|------|-------|-----|
| Authorization | app/billing/views.py | 12 | ViewSet missing permission_classes | Add IsAuthenticated + object-level perm |
| Input Validation | app/users/api.py | 45 | Raw request.data passed to ORM filter | Validate through serializer first |
| Secrets | app/settings/local.py | 23 | STRIPE_SECRET_KEY hardcoded | Move to environment variable |

### Agent 8: Git Health (Churn & Coupling)

Analyze the git history to surface risk signals invisible in static analysis.

- **Churn hotspots**: Which files in this domain change most frequently?
  ```bash
  git log --since="6 months ago" --name-only --pretty=format: -- <domain-path> | sort | uniq -c | sort -rn | head -15
  ```
  Files with high churn are maintenance risks — they likely have unclear boundaries or are doing too much.

- **Temporal coupling**: Which files outside this domain change together with files inside it?
  ```bash
  # For each of the top 10 most-changed files in the domain, find co-changed files
  git log --since="6 months ago" --name-only --pretty=format: -- <file> | sort | uniq -c | sort -rn | head -5
  ```
  Files that frequently change together but live in different domains signal hidden coupling or misplaced boundaries.

- **Tech debt age**: How old are the TODOs, FIXMEs, and HACKs?
  ```bash
  # For each TODO/FIXME found, check when it was introduced
  git log -1 --format="%ai" -S "TODO" -- <file>
  ```
  Debt older than 3 months is likely forgotten. Debt older than 6 months is structural.

- **Author concentration**: Is knowledge concentrated in one person?
  ```bash
  git shortlog -sn --since="6 months ago" -- <domain-path> | head -5
  # For young repos (<6 months), use full history:
  # git shortlog -sn -- <domain-path> | head -5
  ```
  If >80% of commits come from one author, that's a bus-factor risk.

**Young repos** (<6 months of history): adjust `--since` to the repo's age
(`git log --reverse --format=%ai | head -1`). Grade relative to available history
rather than penalizing for short history.

Grade:
- A: Low churn, no temporal coupling leaks, fresh debt (<3mo), distributed knowledge
- B: Moderate churn (1-2 hotspots), minor coupling, some old debt
- C: High churn (3-5 hotspots), clear coupling leaks, debt >3mo old
- D: Very high churn, significant cross-domain coupling, debt >6mo, knowledge concentrated
- F: Critical — files changing daily, domains entangled, ancient debt, single-author domain

**Report each finding with:** type, file/domain, metric, and what it signals.

Example finding:
| Type | Location | Metric | Signal |
|------|----------|--------|--------|
| Churn hotspot | app/billing/views.py | 47 commits in 6mo | Likely doing too much — consider splitting |
| Temporal coupling | app/billing/views.py ↔ app/users/permissions.py | 12 co-changes | Hidden dependency — billing shouldn't know about user permissions directly |
| Stale debt | app/billing/models.py:23 | TODO from 2025-08-14 (7mo) | Forgotten — either fix or remove |
| Bus factor | app/billing/ | 1 author: 92% of commits | Knowledge concentration risk |

### Agent 9: Testability

Beyond test coverage (Agent 1 counts tests), this agent evaluates whether the code
is *structurally testable* — can you write good tests for it without heroic effort?

- **Hardcoded dependencies**: Functions/classes that instantiate their own dependencies
  instead of receiving them. Look for `SomeService()` created inside functions rather
  than passed as parameters.
  ```python
  # Untestable — hardcoded
  def process_payment(order):
      client = StripeClient(settings.STRIPE_KEY)
      return client.charge(order.total)

  # Testable — injected
  def process_payment(order, payment_client):
      return payment_client.charge(order.total)
  ```

- **Side effects in constructors/module scope**: Code that runs on import (API calls,
  DB queries, file I/O at module level). These make testing painful because importing
  the module triggers the side effect.

- **Global/singleton state**: Mutable module-level variables, singleton registries that
  accumulate state between tests. Look for patterns like `_cache = {}` at module level
  that aren't cleared between test runs.

- **Deep call chains without seams**: Functions that call 5+ other functions internally
  with no way to inject or intercept intermediate steps. Testing requires either mocking
  everything (fragile) or running the full chain (slow/flaky).

- **Mixed I/O and logic**: Functions that interleave business logic with I/O (DB, HTTP,
  filesystem). The logic can't be tested without the I/O. Look for functions where
  removing I/O lines would make the logic trivial to test.

Grade:
- A: Dependencies injected, no side-effect imports, no global state, clean seams
- B: 1-2 hardcoded dependencies, minor global state
- C: 3-5 testability issues, some mixed I/O and logic
- D: Widespread hardcoded deps, global state, deep chains
- F: Untestable without major refactoring — can't write a unit test without mocking everything

**Report each finding with:** type, file:line, what blocks testing, and how to fix it.

Example finding:
| Type | File | Line | Blocker | Fix |
|------|------|------|---------|-----|
| Hardcoded dep | app/billing/service.py | 15 | `StripeClient()` created inside function | Accept as parameter, default to real client |
| Side-effect import | app/notifications/sms.py | 3 | `twilio_client = Client(...)` at module level | Lazy-init or move to factory function |
| Global state | app/cache/registry.py | 8 | `_providers = {}` mutated at import time | Use dependency injection or reset in test fixtures |
| Mixed I/O | app/reports/generator.py | 34 | DB query + calculation + file write in one function | Split into query, calculate, write |

### Agent 10: Observability

Evaluate whether the domain is debuggable and monitorable in production — can you
diagnose issues without reading source code?

- **Structured Logging**: Are log statements present at key decision points (request
  handling, error paths, external service calls)? Are they structured (JSON/key-value)
  or unstructured (print/f-string)? Look for:
  ```python
  # Bad — unstructured, no context
  print(f"Error processing order {order_id}")

  # Good — structured, contextual
  logger.error("order_processing_failed", order_id=order_id, error=str(e), customer_id=customer.id)
  ```

- **Error Context**: When errors are caught, do they include enough context to diagnose?
  Look for bare `except:` or `except Exception:` without logging, re-raises without
  context, and error messages that don't identify what failed or why.

- **Request Tracing**: For web services, can a single request be traced end-to-end?
  Look for correlation IDs, request IDs, or tracing middleware (OpenTelemetry, Sentry,
  Datadog). Without tracing, debugging distributed calls is guesswork.

- **Health Checks**: Does the domain expose health/readiness endpoints? Can monitoring
  systems verify the service is working without manual inspection?

- **Metrics & Alerting**: Are business-critical operations instrumented with metrics
  (counters, histograms, gauges)? Look for metric libraries (prometheus_client,
  statsd, Sentry performance) or custom metric collection. Key question: would you
  know if this domain silently stopped working?

- **Log Levels**: Are log levels used appropriately? DEBUG for development, INFO for
  normal operations, WARNING for recoverable issues, ERROR for failures. Look for
  ERROR-level logs on non-error paths or missing ERROR logs on actual failures.

Grade:
- A: Structured logging, request tracing, health checks, metrics on critical paths, appropriate log levels
- B: Logging present but not fully structured, basic tracing, some metrics
- C: Logging exists but inconsistent (mix of print/logger), no tracing, no metrics
- D: Minimal logging, no structure, no tracing, no health checks
- F: No logging, no observability — blind in production

**Report each finding with:** category, file:line, what's missing, and how to fix it.

Example finding:
| Category | File | Line | Issue | Fix |
|----------|------|------|-------|-----|
| Structured logging | app/billing/service.py | 45 | Uses `print()` for error output | Replace with `logger.error()` with structured fields |
| Error context | app/billing/views.py | 78 | Bare `except Exception: pass` swallows errors | Log with context, re-raise or return error response |
| Request tracing | app/billing/ | — | No correlation ID middleware | Add request ID middleware, propagate to logs |
| Health check | app/billing/ | — | No health endpoint | Add `/health` endpoint checking DB + external deps |
| Metrics | app/billing/service.py | — | Payment processing has no metrics | Add counter for payments processed/failed |

### Agent 11: Frontend Quality (conditional — frontend domains only)

**Applicability:** Only run this agent on domains that contain frontend code (`.tsx`, `.jsx`,
`.vue`, `.svelte`, `.css`, `.scss` files, or a `package.json` with frontend framework deps).
Skip entirely for backend-only domains.

```bash
# Detect frontend files in domain
find <domain-path> \( -name "*.tsx" -o -name "*.jsx" -o -name "*.vue" -o -name "*.svelte" -o -name "*.css" -o -name "*.scss" \) -not -path "*/node_modules/*" | head -5
```

Evaluate the domain against frontend-specific quality criteria:

- **Design System Compliance**: Is there a design system (tokens, theme, style guide)?
  Are components using it consistently?
  - Look for: theme files, CSS custom properties / design tokens, Tailwind config,
    styled-system, or equivalent
  - Check if components use hardcoded values (`color: #3b82f6`, `padding: 16px`)
    instead of tokens (`var(--color-primary)`, `theme.spacing.4`, `p-4`)
  - Look for spacing inconsistencies: mixed units (`px` vs `rem`), arbitrary values
    vs scale-based values
  - Typography: are font sizes, weights, and line heights from a defined scale?
  ```bash
  # Find hardcoded color values in components
  grep -rn "#[0-9a-fA-F]\{3,8\}\|rgb(\|rgba(" --include="*.tsx" --include="*.jsx" --include="*.vue" <domain-path> | head -20
  # Find hardcoded spacing
  grep -rn "margin:\|padding:\|gap:" --include="*.tsx" --include="*.jsx" --include="*.css" <domain-path> | grep -v "var(\|theme\.\|rem\|em" | head -20
  ```

- **Component Architecture**: Are components well-structured?
  - **Separation of concerns**: Presentational vs container components? Or are
    components mixing data fetching, business logic, and rendering in one file?
  - **Composition over inheritance**: Components using composition (`children`,
    slots, render props) vs deep inheritance chains?
  - **Component size**: Components over 200 lines? These likely need splitting.
  - **Prop drilling**: Are props passed through 3+ levels without being used?
    Signals need for context, state management, or composition refactoring.
  - **Reusability**: Are similar UI patterns reimplemented in multiple components
    instead of extracted into shared components?
  ```bash
  # Find large components
  find <domain-path> \( -name "*.tsx" -o -name "*.jsx" -o -name "*.vue" \) -not -path "*/node_modules/*" -exec wc -l {} + | sort -rn | head -10
  ```

- **Accessibility (a11y)**: Can all users interact with the UI?
  - Interactive elements (`div`, `span`) used as buttons without `role`, `tabIndex`,
    or keyboard event handlers
  - Images without `alt` attributes
  - Forms without `label` elements or `aria-label`
  - Missing `aria-*` attributes on dynamic content (modals, dropdowns, alerts)
  - Color contrast: are there low-contrast text combinations?
  - Focus management: is focus trapped in modals? Does focus move logically?
  ```bash
  # Find clickable divs/spans without role
  grep -rn "onClick" --include="*.tsx" --include="*.jsx" <domain-path> | grep -i "<div\|<span" | grep -v "role=" | head -15
  # Find images without alt
  grep -rn "<img" --include="*.tsx" --include="*.jsx" <domain-path> | grep -v "alt=" | head -10
  ```

- **Performance Patterns**: Is the frontend optimized for user experience?
  - Unnecessary re-renders: missing `useMemo`, `useCallback`, `React.memo` on
    expensive computations or frequently-rendered components
  - Missing lazy loading: large routes/components loaded eagerly instead of
    `React.lazy()`, dynamic `import()`, or equivalent
  - Bundle impact: are heavy libraries imported when lighter alternatives exist?
    (e.g., full `lodash` instead of `lodash-es` or individual imports, `moment`
    instead of `date-fns` or `dayjs`)
  - List rendering: large lists without virtualization (`react-window`,
    `@tanstack/virtual`, etc.)
  - Unoptimized images: no `loading="lazy"`, no responsive sizes, large images
    served without optimization
  ```bash
  # Check for full lodash/moment imports
  grep -rn "import.*from ['\"]lodash['\"]" --include="*.ts" --include="*.tsx" --include="*.js" <domain-path> | head -5
  grep -rn "import.*from ['\"]moment['\"]" --include="*.ts" --include="*.tsx" --include="*.js" <domain-path> | head -5
  ```

- **State Management**: Is state well-organized?
  - Local state where it should be global (user auth, theme, locale repeated in
    multiple components)
  - Global state where it should be local (form field values in Redux/Zustand)
  - Prop drilling through 3+ levels as proxy for missing state management
  - State shape: normalized vs denormalized data in stores
  - Side effects: are async operations (API calls) mixed into components or
    properly separated (hooks, middleware, services)?

- **Responsive Design & i18n Readiness**:
  - Are breakpoints used consistently from a defined scale?
  - Is there a mobile-first approach or are mobile styles an afterthought?
  - Hardcoded strings in UI: any user-facing text not going through i18n?
  ```bash
  # Find hardcoded user-facing strings (rough heuristic)
  grep -rn ">[A-Z][a-z].*</" --include="*.tsx" --include="*.jsx" <domain-path> | grep -v "i18n\|t(\|Trans\|FormattedMessage\|intl" | head -15
  ```

Grade:
- A: Design system followed, clean component architecture, a11y compliant, performant, good state management, responsive, i18n-ready
- B: Design system mostly followed (1-2 hardcoded values), minor a11y gaps, components well-structured
- C: Inconsistent design tokens, some a11y issues, 2-3 oversized components, some perf issues
- D: No design system, multiple a11y violations, prop drilling, no lazy loading, hardcoded strings
- F: No structure — inline styles everywhere, zero a11y, god components, no performance consideration

**Report each finding with:** category, file:line, what's wrong, and how to fix it.

Example finding:
| Category | File | Line | Issue | Fix |
|----------|------|------|-------|-----|
| Design System | src/features/billing/InvoiceCard.tsx | 12 | Hardcoded `color: #3b82f6` | Use `var(--color-primary)` or `text-primary` token |
| Component Size | src/features/billing/BillingDashboard.tsx | — | 380 lines, mixes data fetching + rendering + filtering | Split into BillingDashboard (layout), BillingFilters, BillingTable |
| Accessibility | src/features/billing/PayButton.tsx | 28 | `<div onClick={pay}>` without role or keyboard handler | Use `<button>` element or add `role="button"` + `onKeyDown` |
| Performance | src/features/billing/TransactionList.tsx | 5 | `import _ from 'lodash'` (full bundle) | `import groupBy from 'lodash/groupBy'` |
| State Mgmt | src/features/billing/InvoiceForm.tsx | 15 | Form state stored in Redux global store | Move to local `useState` or form library (react-hook-form) |
| i18n | src/features/billing/StatusBadge.tsx | 22 | Hardcoded `"Paid"`, `"Pending"`, `"Overdue"` | Use `t('billing.status.paid')` |

### Agent 12: Repo Hygiene

**Scope distinction from Agent 4:** Agent 4 evaluates doc *content accuracy* (is what
the doc says true?). Agent 12 evaluates doc *lifecycle* (should this doc still exist?
is it in the right place? is it linked?). A doc can be accurate but orphaned (Agent 12),
or linked but stale (Agent 4). If both apply, Agent 12 takes precedence (delete > fix).

Evaluate whether the repository is clean of residual, orphaned, or misplaced files
that accumulate over time from development workflows.

- **Orphaned plan/spec files**: Plans and specs for features that are already
  implemented and merged. Compare plan files against git log — if the plan's tasks
  are all reflected in committed code, the plan is complete and should be archived
  or removed.
  ```bash
  # List plan files and check if they reference completed work
  find docs/ -name "*.md" -path "*/plans/*" -o -path "*/specs/*" 2>/dev/null
  ```

- **Deprecated file locations**: Files in old directory structures that have been
  superseded. Common patterns:
  - `docs/plans/` when the project now uses `docs/superpowers/plans/`
  - Config files in root when they should be in `.claude/` or `docs/`
  - Duplicated files across locations (same content, different paths)

- **State/temp files committed to repo**: Files that should be ephemeral:
  - `hub-loop-state.json` after the loop is complete (graduated/needs_human)
  - Test baseline files (`/tmp/hub-baseline-*.txt`)
  - Build artifacts or cache files not in `.gitignore`
  ```bash
  # Find potential temp/state files
  find . -name "*-state.json" -o -name "*.tmp" -o -name "*.bak" \
    -not -path "*/node_modules/*" -not -path "*/.git/*" 2>/dev/null
  ```

- **Unused dependencies or config files**: Package configs, CI workflows, or tool
  configs for tools that are no longer used in the project.

- **Dead documentation**: Docs that aren't linked from anywhere (no README reference,
  no ARCHITECTURE.md reference, no skill reference). Orphaned docs are invisible docs.
  ```bash
  # Find doc files and check if they're referenced anywhere
  for doc in docs/**/*.md; do
    basename=$(basename "$doc")
    refs=$(grep -rl "$basename" --include="*.md" --include="*.py" --include="*.ts" . 2>/dev/null | grep -v "$doc" | wc -l)
    [ "$refs" -eq 0 ] && echo "ORPHANED: $doc (0 references)"
  done
  ```

- **Accumulation rate**: How many plan/spec files are created per month? If >5/month
  without cleanup, flag as a hygiene concern.

Grade:
- A: Clean repo — no orphans, no deprecated locations, no stale state files, docs linked
- B: 1-2 minor orphans or one deprecated location still in use
- C: 3-5 orphaned files, mix of old and new directory structures
- D: 6-10 orphans, state files committed, dead docs, deprecated locations
- F: >10 orphans, multiple deprecated structures, temp files committed, no cleanup process

**Report each finding with:** type, file path, why it's residual, and recommended action.

Example finding:
| Type | File | Why Residual | Action |
|------|------|-------------|--------|
| Orphaned plan | docs/plans/2026-03-03-deploy-verification.md | Feature implemented on 2026-03-15 | Archive or delete |
| Deprecated location | docs/plans/ (10 files) | Superseded by docs/superpowers/plans/ | Move or delete |
| Dead doc | docs/superpowers/specs/2026-04-01-hub-aware-driven-design.md | Not referenced by any skill or README | Link from relevant docs or delete |
| Completed state | docs/hub-loop-state.json | All domains graduated | Delete |
| Accumulation | docs/superpowers/plans/ | 4 plans in 2 weeks, none cleaned up | Establish cleanup process |

### Agent 13: CI/CD & Deployment

Evaluate whether the repository has a healthy, documented deployment pipeline
that matches the **Impactia standard pipeline shape**. Repo-level dimension
(like Agent 12): every domain gets the same grade, since CI/CD is shared
infrastructure.

Impactia targets are **Hetzner** (backend services via GitHub Actions + SSH/
Docker Compose) and **Vercel** (frontends/landings). This dimension is opinionated
toward those targets — other stacks (Modal, Fly, Render) are acceptable but must
still satisfy the same stage checklist. Related skills: `hetzner-cloud`,
`deploy-landing`, `wand-environments`, `hub-check-deploy`, `hub-e2e-frontend`, `run-e2e`.

#### The Impactia standard pipeline (required stages)

Every deployable repo MUST implement all five stages:

1. **PR gate** (`pull_request` trigger): lint + typecheck + unit tests. Must be
   blocking (branch protection or required check). No merge without green.
2. **Build**: runs on merge to the deploy branch. Produces the artifact
   (Docker image for Hetzner, build output for Vercel). Build failure
   blocks deploy.
3. **Deploy**: branch→environment mapping is explicit.
   - Hetzner: SSH + `docker compose pull && up -d` (or equivalent) on the
     target host. Image tag pinned to the commit SHA, not `latest`.
   - Vercel: deploy via `vercel` CLI or Git integration, with preview for PRs
     and production for main.
4. **Post-deploy smoke** (REQUIRED, not optional): E2E smoke against the
   **deployed URL**, not localhost. Minimum: health check + one critical
   user flow (login, landing load, API ping). Runs automatically after deploy.
   Failure must either auto-rollback or open a Sentry/Linear incident.
5. **Rollback path**: documented revert — either automated (re-run previous
   workflow with pinned SHA) or a one-command manual revert recorded in
   README/ARCHITECTURE.md.

If any of the five stages is missing, the dimension grade drops by at least
one letter per missing stage.

#### Detection

```bash
# Pipeline files present
ls -la .github/workflows/ 2>/dev/null
# Vercel projects
ls vercel.json .vercel/project.json 2>/dev/null
# Hetzner deploy signals
grep -rlE 'hetzner|HETZNER_|appleboy/ssh-action|docker compose' \
  .github/workflows/ 2>/dev/null
# Unit test invocation inside PR workflow
grep -rE 'pytest|vitest|jest|ruff|tsc --noEmit|eslint' \
  .github/workflows/ 2>/dev/null
# Post-deploy smoke signals (Playwright/Cypress against deployed URL)
grep -rE 'playwright|cypress|curl.*(staging|production|\.vercel\.app|\.impactia)' \
  .github/workflows/ 2>/dev/null
# SHA pinning (not :latest)
grep -rE 'image:.*:latest' .github/workflows/ docker-compose*.yml 2>/dev/null
# Branch → env mapping
grep -rE 'branches:|environment:|if:.*github\.ref' \
  .github/workflows/ 2>/dev/null
# Secrets referenced
grep -rE 'secrets\.[A-Z_]+' .github/workflows/ 2>/dev/null
# Hardcoded credential-looking values
grep -rE '(api[_-]?key|token|password|secret)\s*[:=]\s*["'\''][A-Za-z0-9]{16,}' \
  .github/workflows/ 2>/dev/null
```

#### Stage-by-stage checks

- **Stage 1 (PR gate)**: workflow triggered by `pull_request` runs `lint`,
  `typecheck`, and `unit tests`. No tests = stage missing.
- **Stage 2 (Build)**: Hetzner repos build + push a Docker image tagged with
  `${{ github.sha }}`. Vercel repos have `next build`/`vite build` succeeding
  as part of the Vercel pipeline. Using `:latest` as the deploy tag = stage failing.
- **Stage 3 (Deploy)**: explicit env mapping:
  - `main` → production, `develop`/`dev` → staging (Impactia convention).
  - Vercel: production branch set to `main`, preview for all others.
  - Hetzner: target host and compose file path declared in workflow.
  - Deploy step runs only after Stages 1-2 pass (`needs:`).
- **Stage 4 (Post-deploy smoke)**: a job runs AFTER deploy completes, hitting
  the deployed URL (not localhost). Examples that count:
  - Playwright/Cypress project configured with `baseURL` = deployed URL.
  - `curl <url>/health` + assertion on response.
  - `hub-e2e-frontend` or `run-e2e` skill invocation against the deployed env.
  Smoke that runs on localhost BEFORE deploy does NOT count for this stage.
- **Stage 5 (Rollback)**: README/ARCHITECTURE.md section titled "Rollback" or
  "Recovery", OR a `rollback` workflow in `.github/workflows/`. One sentence
  ("re-run previous green workflow") is enough; absence is not.

#### Secrets hygiene (hard requirement)

- Every referenced `secrets.*` MUST be listed in README or `docs/ENV.md` with
  purpose and owner.
- Hardcoded credentials in workflow YAML = automatic **F** regardless of other
  stages. Also flag for rotation if already in git history.
- Hetzner SSH keys, Vercel tokens, Docker registry creds, DB URLs must all be
  via `secrets.*`.

#### Stale workflow steps

Workflow references scripts/commands that no longer resolve (e.g.,
`npm run build:legacy` not in `package.json`, deleted make targets).

#### Grade

- **A**: All 5 stages present and healthy. Secrets documented. SHA-pinned
  deploys. Post-deploy smoke against deployed URL. Rollback documented.
  No stale steps.
- **B**: 4/5 stages present; the missing one is Stage 5 (rollback) only.
  Minor gaps (1 undocumented secret, 1 stale step).
- **C**: 3/5 stages (typically missing post-deploy smoke AND rollback), OR
  deploy uses `:latest` instead of SHA pinning, OR 2-3 undocumented secrets.
- **D**: ≤2/5 stages, OR PR gate is non-blocking (no branch protection /
  no required check), OR env mapping undocumented, OR broken workflow
  (references removed scripts).
- **F**: No CI/CD at all, OR hardcoded credentials in workflow files, OR
  workflows present but never run (no valid triggers), OR deploys happen
  manually with no pipeline record.

**Post-deploy smoke is not optional.** A repo with perfect lint + unit tests
but zero smoke against the deployed URL caps at **C**. The rationale: unit
tests passing locally have never caught a broken prod deploy — only a request
against the real URL does.

**Report each finding with:** stage, file, issue, and recommended action.

Example finding:
| Stage | File | Issue | Action |
|-------|------|-------|--------|
| 1 (PR gate) | .github/workflows/ci.yml | Runs lint only, no unit tests | Add `pytest`/`vitest` job |
| 1 (PR gate) | GitHub branch rules | CI not marked as required check | Enable branch protection for `main` |
| 2 (Build) | .github/workflows/deploy.yml:45 | Image tagged `:latest` | Tag with `${{ github.sha }}` |
| 3 (Deploy) | docs/ | No branch→env mapping documented | Add section to ARCHITECTURE.md |
| 4 (Smoke) | — | No post-deploy smoke job exists | Add Playwright job hitting deployed URL after deploy |
| 4 (Smoke) | .github/workflows/e2e.yml | Runs against `localhost:3000`, not deployed URL | Set `baseURL` to `${{ secrets.DEPLOY_URL }}` |
| 5 (Rollback) | README.md | No rollback instructions | Add "Rollback" section: re-run previous workflow with pinned SHA |
| Secrets | .github/workflows/ci.yml:34 | `TOKEN: abc123...` inline | Move to `secrets.TOKEN`, rotate, document |
| Secrets | .github/workflows/deploy.yml | `secrets.HETZNER_SSH_KEY` not in README | Add to `docs/ENV.md` with owner |
| Stale | .github/workflows/ci.yml:12 | `npm run build:v1` not in package.json | Remove or update |

### Combining Grades

**Letter-to-number mapping:** A=4, B=3, C=2, D=1, F=0

Compute domain grade: sum all agent scores, divide by agent count, round down
to nearest letter.

- **Backend-only domains:** 12 agents (1-10 + 12 + 13). Score range 0-48.
- **Frontend domains:** 13 agents. Score range 0-52.

Agent 11 (Frontend) only contributes for frontend domains.
Agent 12 (Repo Hygiene) always contributes — it evaluates repo-level hygiene
scoped to the domain's directory.
Agent 13 (CI/CD) always contributes — repo-level, same grade across all domains.

Agent 11 subcategories are equally weighted into a single grade:
design system + component architecture + a11y + performance + state management +
responsive/i18n → average (round down) = Agent 11 grade.

**Rounding:** 3.9 → B (3), 2.1 → C (2). Always round down — we don't inflate.

Example (backend-only, 12 agents): A,A,B,A,B,A,A,B,A,B,A,B = 4+4+3+4+3+4+4+3+4+3+4+3 = 43/12 = 3.58 → B

## Phase 3: Tech Debt Summary

```bash
grep -rn "TODO\|FIXME\|HACK\|XXX" \
  --include="*.py" --include="*.ts" --include="*.tsx" --include="*.js" \
  --include="*.jsx" --include="*.vue" --include="*.svelte" --include="*.go" \
  --include="*.rs" --include="*.java" --include="*.rb" --include="*.css" \
  --include="*.scss" \
  . 2>/dev/null | wc -l
```

Categorize by domain and severity.

## Phase 4: Generate Quality Score

Create or update `docs/QUALITY_SCORE.md`:

```markdown
# Quality Score

**Last scan:** YYYY-MM-DD
**Overall grade:** B- (prev: C+, trending: up)

## Domain Grades

| Domain | Tests | DRY | Boundaries | Docs | Principles | Patterns | Security | Git Health | Testability | Observability | Frontend | Hygiene | CI/CD | Overall |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| users | A | A | B | A | B | A | A | B | A | B | — | A | B | A |
| billing | C | D | C | F | D | C | F | D | D | D | — | C | B | D |
| frontend/dashboard | B | C | B | B | C | B | A | B | B | C | C | B | B | B |

## Detailed Findings

### <domain>

#### Principles Compliance Violations

| Principle | File | Line | Issue | Fix |
|-----------|------|------|-------|-----|
| SRP | app/billing/views.py | 45 | View contains business logic, ORM queries, and email sending | Extract to BillingService |
| DIP | app/billing/tasks.py | 12 | Hardcoded import of StripeClient | Inject via constructor parameter |

#### DRY Violations

| Files | Lines | What's duplicated |
|-------|-------|-------------------|
| views.py, api.py | 23, 89 | Permission check logic (same 10-line block) |

#### Design Pattern Opportunities

| Location | Issue | Suggested Pattern |
|----------|-------|-------------------|
| app/billing/processor.py:34 | if/elif chain for 5 payment methods | Strategy pattern |
| app/notifications/ | Direct calls to 4 notification channels | Observer/Event pattern |

#### Anti-Patterns

| Location | Anti-pattern | Impact |
|----------|-------------|--------|
| app/billing/models.py | God object (BillingAccount: 450 lines, 23 methods) | Hard to test, hard to change |

#### Security Findings

| Category | File | Line | Issue | Fix |
|----------|------|------|-------|-----|
| Authorization | app/billing/views.py | 12 | ViewSet missing permission_classes | Add IsAuthenticated |

#### Git Health

| Type | Location | Metric | Signal |
|------|----------|--------|--------|
| Churn hotspot | app/billing/views.py | 47 commits in 6mo | Consider splitting |

#### Testability Issues

| Type | File | Line | Blocker | Fix |
|------|------|------|---------|-----|
| Hardcoded dep | app/billing/service.py | 15 | `StripeClient()` inside function | Accept as parameter |

#### Observability Issues

| Category | File | Line | Issue | Fix |
|----------|------|------|-------|-----|
| Structured logging | app/billing/service.py | 45 | Uses `print()` for error output | Replace with `logger.error()` with structured fields |
| Request tracing | app/billing/ | — | No correlation ID | Add request ID middleware |

#### Frontend Quality Issues (if applicable)

| Category | File | Line | Issue | Fix |
|----------|------|------|-------|-----|
| Design System | src/features/dashboard/Card.tsx | 12 | Hardcoded `#3b82f6` | Use design token `var(--color-primary)` |
| Accessibility | src/features/dashboard/FilterDropdown.tsx | 34 | `<div onClick>` without keyboard support | Use `<button>` or add `role` + `onKeyDown` |
| Performance | src/features/dashboard/DataTable.tsx | 5 | Full lodash import | Import individual functions |

#### Repo Hygiene Issues

| Type | File | Why Residual | Action |
|------|------|-------------|--------|
| Orphaned plan | docs/plans/2026-03-03-deploy-verification.md | Feature implemented | Archive or delete |
| Deprecated location | docs/plans/ (10 files) | Superseded by docs/superpowers/plans/ | Move or delete |
| Stale doc | docs/api.md | References removed endpoints | Update or delete |

#### CI/CD & Deployment Issues

| Type | File | Issue | Action |
|------|------|-------|--------|
| No gates | .github/workflows/deploy.yml | Deploy runs with no test step | Add test job as prerequisite |
| Hardcoded secret | .github/workflows/ci.yml:34 | Credential inline in workflow | Move to `secrets.*` |
| Undocumented secret | .github/workflows/deploy.yml | `HETZNER_SSH_KEY` not in README | Document required secrets |
| Stale step | .github/workflows/ci.yml:12 | Runs script removed from package.json | Update or remove |
| No env mapping | docs/ | Branch→environment mapping undocumented | Document in README or ARCHITECTURE.md |

### <next domain>
...

## Tech Debt

| Type | Count | Prev |
|---|---|---|
| TODO | 12 | 15 |
| FIXME | 3 | 3 |
| HACK | 1 | 2 |

## Top 5 Refactoring Priorities

1. domain: description (impact: high/medium/low)
2. ...

## History

| Date | Overall | Trend |
|---|---|---|
| YYYY-MM-DD | B- | — |
```

Also generate `docs/quality-score.json` with the same data in machine-readable format:

```json
{
  "last_scan": "YYYY-MM-DD",
  "overall_grade": "B-",
  "previous_grade": "C+",
  "trending": "up",
  "scanned_domains": ["users", "billing"],
  "domains": {
    "users": {
      "tests": "A",
      "dry": "A",
      "boundaries": "B",
      "docs_complexity": "A",
      "principles": "B",
      "patterns": "A",
      "security": "A",
      "git_health": "B",
      "testability": "A",
      "observability": "B",
      "frontend": null,
      "hygiene": "A",
      "overall": "A"
    },
    "billing": {
      "tests": "C",
      "dry": "D",
      "boundaries": "C",
      "docs_complexity": "F",
      "principles": "D",
      "patterns": "C",
      "security": "F",
      "git_health": "D",
      "testability": "D",
      "observability": "D",
      "frontend": null,
      "hygiene": "C",
      "overall": "D"
    },
    "frontend/dashboard": {
      "tests": "B",
      "dry": "C",
      "boundaries": "B",
      "docs_complexity": "B",
      "principles": "C",
      "patterns": "B",
      "security": "A",
      "git_health": "B",
      "testability": "B",
      "observability": "C",
      "frontend": "C",
      "hygiene": "B",
      "overall": "B"
    }
  },
  "tech_debt": {
    "todo": 12,
    "fixme": 3,
    "hack": 1,
    "total": 16
  },
  "history": [
    {"date": "YYYY-MM-DD", "overall": "B-", "trend": "up"}
  ]
}
```

For partial scans, merge new domain grades into the existing JSON, preserving
unscanned domains. Update `last_scan` and `overall_grade` accordingly.

Commit the quality score:
```bash
git add docs/QUALITY_SCORE.md docs/quality-score.json
git commit -m "docs: update quality score (overall: <grade>)"
```

## Phase 5: Report

Present the full report to the user (same format as QUALITY_SCORE.md but in console).
Include ALL detailed findings — every SOLID violation, every DRY issue, every pattern
opportunity. Don't summarize or skip findings.

After presenting, ask:
> "Want me to fix these issues? Run `/hub-fix` to create a correction plan and apply fixes."

### Lite Mode

When invoked with `--lite` (e.g., `/hub-scan --lite` or by hub-loop for
spot-checks), run a faster scan:

- Skip detailed findings tables — only compute grades per dimension
- Skip Agent 8 (Git Health) — requires expensive git log analysis
- Skip bash command execution in agents — evaluate from code reading only
- Output only the grade table and overall grade, not the full report
- Still update `docs/QUALITY_SCORE.md` and `docs/quality-score.json`

Lite mode is ~3-5x faster. Use for spot-checks and progress validation.
Full mode is always used for initial scans and final reports.

## Important Rules

- **Grade honestly**: Don't inflate grades. Accurate grading drives improvement.
- **Compare with previous**: Always show trending when previous scan exists.
- **Actionable priorities**: Each refactoring priority must be specific and actionable, not vague.
- **Commit the score**: Always commit both `docs/QUALITY_SCORE.md` and `docs/quality-score.json` after scanning.
- **Full repo scan by default**: Scan ALL domains unless specific domains are provided as arguments.
- **Partial scan preserves history**: When scanning specific domains, preserve previous grades for unscanned domains in both MD and JSON output.
- **Principles are authoritative**: Agent 5 and Agent 7 MUST read `~/.ai-skills/method/PRINCIPLES.md`. Findings must reference specific principle numbers.
- **Security is not optional**: Agent 7 findings rated D or F should be flagged as critical in the Top 5 Refactoring Priorities, regardless of other grades.
- **Git history adds context, not grades inflation**: Agent 8 grades reflect risk signals. High churn alone is not bad — high churn + low test coverage IS bad. Cross-reference with other agents.
- **Corrections are a separate skill**: `/hub-scan` diagnoses. `/hub-fix` corrects. Do not execute fixes from within hub-scan.
- **Doc accuracy is not optional**: Agent 4 must cross-reference documentation claims against actual code/files. "Docs exist" is not the same as "docs are accurate".
- **Hygiene is cumulative**: Agent 12 findings compound over time. Old plans that were acceptable last month become debt this month. Grade based on current state, not intent.
