---
name: entropy-aware
description: >
  Post-process a spec or plan to align it with entropy-scan's 13 quality
  dimensions. Reads the artefact, evaluates gaps, and corrects inline.
  Triggers on: entropy-aware, entropy aware, quality check spec,
  quality check plan, enrich spec, enrich plan.
user_invocable: true
---

# Entropy Aware

<!-- SYNC: entropy-aware and entropy-scan share dimension definitions.
     Update both when changing criteria. -->

**Announce at start:** "Running /entropy-aware — enriching artefact with entropy-scan quality dimensions."

## Overview

Stateless post-processor. Takes a spec or plan file, evaluates it against the
13 dimensions that `/entropy-scan` grades, and corrects it inline — adding what's
missing without removing what's already there.

Composable: any skill can invoke `entropy-aware` after producing an artefact.
Does NOT run scans or tests — works purely on document content.

## Phase 1: Read and Classify

Read the artefact at the provided path. Classify it:

- **Spec**: lives in `docs/superpowers/specs/` OR contains design/architecture
  sections (## Architecture, ## Solution, ## Design, ## Domain Model, etc.)
- **Plan**: lives in `docs/superpowers/plans/` OR contains checkbox tasks (`- [ ]`)
  with step-by-step implementation instructions
- **Ambiguous**: treat as spec (specs are more permissive to enrich)

Also read `~/.ai-skills/method/PRINCIPLES.md` if it exists — dimensions 5 and 7
reference specific principles.

### Detect frontend involvement

Check whether the artefact involves frontend code:
- **Explicit signals**: spec/plan mentions React, Vue, Svelte, Angular, Next.js,
  Tailwind, CSS, components, UI, frontend, or similar
- **File path signals**: plan tasks reference `src/features/`, `src/components/`,
  `src/pages/`, `app/`, or files with `.tsx`, `.jsx`, `.vue`, `.svelte` extensions
- **Project context**: check if `package.json` exists with frontend framework deps

If frontend involvement is detected, dimension 11 applies. Otherwise, skip it.

## Phase 2: Evaluate Against 12 Dimensions

For each dimension, check whether the artefact already covers it. Track:
- **Covered**: the dimension is adequately addressed — do nothing
- **Partial**: some coverage but gaps exist — enrich the existing section
- **Missing**: no coverage at all — add a new section or steps

### For Specs

| # | Dimension | What must be present |
|---|-----------|---------------------|
| 1 | Test Coverage | Testing strategy: what gets tested, approach (unit/integration/e2e), coverage expectations. Not just "we'll write tests" — specifics. |
| 2 | DRY | Shared logic identified and centralized in the design. No parallel implementations of the same concept. Reusable components/services explicitly listed. |
| 3 | Boundaries | Input validation and output serialization defined at every system edge (API endpoints, module interfaces, external service calls). Schema/type definitions referenced. |
| 4 | Docs & Complexity | Complexity budget: no single file/class should exceed 300 lines by design. Documentation expectations for public interfaces. |
| 5 | Principles (SOLID) | Design adheres to PRINCIPLES.md — specifically P2 (SRP), P4 (Open/Closed), P5 (DIP), P6 (Minimal Surface Area), P9 (Fail Fast), P10 (Simplicity). Each component has one responsibility. Dependencies are injected, not hardcoded. |
| 6 | Design Patterns | Appropriate patterns chosen for the problem. No god objects, no circular dependencies, no deep inheritance. If the spec introduces a pattern, it should be justified. |
| 7 | Security | Auth model defined. Input validation at boundaries. Secret management via env vars. No hardcoded credentials. CORS/CSRF policy if web-facing. Dependency health: no abandoned or known-vulnerable deps. License compliance considered. References P7 from PRINCIPLES.md. |
| 8 | Git Health | Ownership model: who owns what. Commit strategy: atomic, well-scoped. Knowledge distribution: no single-person bottlenecks by design. |
| 9 | Testability | Dependency injection in the architecture. No side-effect constructors. Business logic separated from I/O. Clean seams for testing at every layer. |
| 10 | Observability | Logging strategy defined: structured logging at key decision points, error context, request tracing (correlation IDs). Health check endpoints. Metrics on critical business operations. Log levels used appropriately. |
| 11 | Frontend Quality | *(Only if artefact involves frontend code.)* Design system/tokens referenced for colors, spacing, typography. Component architecture: presentational vs container separation, composition over inheritance, max ~200 lines per component. Accessibility: semantic HTML, ARIA attributes, keyboard navigation, focus management. Performance: lazy loading strategy, bundle optimization, list virtualization for large datasets. State management: local vs global state boundaries defined. Responsive breakpoints from defined scale. i18n: all user-facing strings through translation system. |
| 12 | Repo Hygiene | Doc accuracy plan: how will documentation stay in sync with code? (e.g., README updated as part of feature tasks, SPEC.md reviewed quarterly). Cleanup strategy: plan/spec files archived or deleted after implementation. No deprecated file locations used. State files (entropy-loop-state.json) cleaned after completion. |
| 13 | CI/CD & Deployment | Spec covers the Impactia 5-stage pipeline: (1) PR gate with lint+typecheck+unit tests as required check, (2) build producing SHA-pinned artifact (Docker image for Hetzner, build output for Vercel), (3) explicit branch→env mapping (`main`→prod, `develop`/`dev`→staging), (4) **post-deploy E2E smoke against the deployed URL** (not localhost) — minimum health check + one critical flow, (5) documented rollback path. Secrets referenced via `secrets.*` and listed in README/`docs/ENV.md` with owner. Target declared (Hetzner or Vercel or justified alternative). |

### For Plans

| # | Dimension | What must be present |
|---|-----------|---------------------|
| 1 | Test Coverage | TDD steps: failing test → implementation → passing test. Every task that writes code must start with a test. |
| 2 | DRY | No duplicated logic across tasks. If two tasks need the same utility, one task creates it and the other references it. |
| 3 | Boundaries | Explicit steps for input validation and output typing at API/module edges. Serializer/schema creation steps where needed. |
| 4 | Docs & Complexity | No task should produce a file >300 lines. If a task creates a large file, it should be split. Documentation steps for public APIs. |
| 5 | Principles (SOLID) | Tasks reference PRINCIPLES.md where relevant. Implementation follows SRP (one concern per file), DIP (inject dependencies), Open/Closed (extend via composition). |
| 6 | Design Patterns | Implementation uses appropriate patterns. No tasks that create god objects or introduce circular dependencies. |
| 7 | Security | Tasks include auth setup, input validation steps, secret management via env vars. No hardcoded credentials in code blocks. Dependency audit step if new deps are introduced. |
| 8 | Git Health | Commits are atomic (one concern per commit), well-scoped, with descriptive messages following conventional commits. |
| 9 | Testability | Implementation structure uses injected dependencies, isolated I/O, and clean seams. No hardcoded service instantiation inside functions. |
| 10 | Observability | Tasks include structured logging setup, error context in catch blocks, request tracing middleware if web service, health check endpoint, and metrics for critical operations. |
| 11 | Frontend Quality | *(Only if plan involves frontend code.)* Tasks use design tokens (no hardcoded colors/spacing). Component tasks respect ~200 line limit, split large components. Accessibility steps: semantic HTML, ARIA for dynamic content, keyboard handlers on interactive elements. Performance steps: lazy loading for routes, individual imports for heavy libs, virtualization for lists. State tasks: local state for forms, global for shared data. i18n: all strings through translation function. Responsive: mobile-first with breakpoint scale. |
| 12 | Repo Hygiene | Plan includes doc update steps alongside code changes (README, SPEC.md, API docs). Cleanup task at end: delete/archive plan file after implementation is verified. No tasks that create files in deprecated locations. |
| 13 | CI/CD & Deployment | Plan covers all 5 Impactia pipeline stages as concrete tasks: (1) update/create `.github/workflows/ci.yml` with lint+typecheck+unit test jobs on `pull_request` and mark them as required checks in branch protection; (2) build task producing SHA-pinned artifact (`image: ...:${{ github.sha }}` for Hetzner, Vercel build for frontends); (3) deploy job with explicit `if: github.ref == 'refs/heads/main'` (prod) / `develop` (staging) and `needs:` on build; (4) **post-deploy smoke job** that runs AFTER deploy and hits the deployed URL — Playwright with `baseURL=${{ secrets.DEPLOY_URL }}` or `curl <url>/health` with assertion; (5) rollback step or README section. New secrets: add `secrets.*` reference AND a task to document them in `docs/ENV.md`. No `:latest` tags. No localhost-based smoke. |

## Phase 3: Correct Inline

For each dimension rated **Partial** or **Missing**:

**For specs:**
- Add a new subsection under the most relevant existing section, or create a new
  `## Entropy Dimensions` section at the end if no natural home exists
- Write concrete requirements, not vague statements
- Reference specific principles by number (P1-P10) where applicable

**For plans:**
- Add missing steps to existing tasks where they naturally belong
- If a whole concern is missing (e.g., no security steps at all), add a new task
- Ensure TDD order: test → implement → verify → commit
- Add concrete code/commands, not placeholders

**Preservation rule:** Never remove or rewrite existing content. Only add or
augment. The original author's intent is preserved.

## Phase 4: Report

After correcting, output a brief summary to the terminal:

```
entropy-aware: corrected <path>
  + Added: <what was added> (dim N)
  + Added: <what was added> (dim N)
  ~ Enriched: <what was augmented> (dim N)
  Dimensions already covered: N/13
  Dimensions added/enriched: M/13
```

## Important Rules

- **Never delete content**: Only add or augment. The artefact's original structure and intent must be preserved.
- **Concrete over vague**: "Add input validation using Zod schema for CreateUserRequest" not "add validation". "Write pytest for BillingService.charge() with mocked payment client" not "write tests".
- **Reference PRINCIPLES.md**: When adding dimension 5 or 7 content, cite specific principle numbers (P1-P10).
- **No scans or tests**: This skill works on document content only. It does not run entropy-scan, tests, or any code.
- **Idempotent**: Running entropy-aware twice on the same artefact should not duplicate content. Check before adding.
