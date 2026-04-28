---
name: entropy-bugfix
description: Use when fixing any bug, test failure, regression, or unexpected behavior, including flaky tests, integration failures, and performance issues treated as bugs, especially when the fix may cross boundaries or touch sensitive domains
user_invocable: true
---

<!-- SYNC: entropy-bugfix depends on entropy-scan grades and entropy-fix strategy
     semantics. Update this skill if entropy-scan grade thresholds or
     entropy-fix invocation rules change.
     Also invokes entropy-e2e-frontend post-deploy when frontend files are touched
     (handoff with mode=post_deploy, target_env=deploy). Update this skill if the handoff
     contract changes. -->

# Entropy Bugfix

**Announce at start:** "Running /entropy-bugfix — root-cause-first bugfix with conditional entropy validation."

## Overview

Default bug workflow: investigate the root cause, encode a failing repro, apply
the smallest fix that addresses the cause, then escalate to entropy validation
only when the change is elevated or sensitive.

## When to Use

Use for:
- bugs
- test failures
- regressions
- unexpected behavior
- flaky tests
- integration failures
- performance issues being treated as bugs

Do not use for:
- net-new features
- codebase-wide quality improvement campaigns
- broad refactors that the user already framed as architecture work

## Phase 1: Root Cause Investigation

Before proposing or applying any fix:

1. Read the error, trace, or failing assertion carefully.
2. Reproduce the issue consistently.
3. Check recent code, config, or dependency changes.
4. Gather evidence at component boundaries when the issue crosses layers.
5. Trace the bad state or value back to its origin.
6. State one concrete root-cause hypothesis.

No fixes before this phase is complete.

## Phase 2: Failing Repro

Invoke `tdd` once the issue is reproducible enough to encode.

- Prefer a focused automated test.
- If no test harness fits, write a one-off repro script.
- Verify the repro fails for the expected reason before changing production code.

## Phase 3: Minimal Fix

- Change one concern at a time.
- Fix the identified cause, not nearby cleanup.
- Re-run the repro and make it pass.
- Stop if the first fix fails and return to investigation with the new evidence.

## Risk Classification

Classify the final fix before closing:

| Class | Trigger | Next Step |
|------|---------|-----------|
| **local** | Exactly 1 changed file, non-sensitive area, no boundary or shared-config change | Skip entropy gate |
| **elevated** | 2+ changed files, or one-file change at a boundary, shared utility, config, or integration seam | Run partial `entropy-scan` |
| **sensitive** | Auth, permissions, billing/payments, security, secrets, validation, serialization, persistence/data integrity, CI/CD, deploy, shared infra | Run partial `entropy-scan` |

If in doubt, classify upward.

## Entropy Gate

Only run this phase for `elevated` or `sensitive` fixes.

1. Determine affected domains from the touched paths and bug context.
2. Run `/entropy-scan <affected-domain-1> <affected-domain-2>`.
3. If all affected domains are `>= B`, continue to closeout.
4. If any affected domain is `< B`, run:

```text
/entropy-fix <affected-domain-1> <affected-domain-2> --strategy=simple
```

5. Re-check once.
6. If any affected domain is still `< B`, report the findings and stop.

Do not escalate to aggressive or rewrite strategy from inside a normal bugfix.
That is `entropy-loop` or explicit follow-up work.

## Closeout

Always report:
- root cause
- repro artefact
- changed files or affected domains
- risk class
- whether the entropy gate ran
- scan results if it ran

Verification must include:
- the failing repro now passes
- directly relevant tests pass
- no new failures were introduced in the affected area

## Post-Deploy Visual Check (frontend only)

Run after the fix is deployed to staging/preview. Skip if there is no deploy
step for the current branch.

1. Run the [Frontend-Change Detection block](../entropy-e2e-frontend/SKILL.md#frontend-change-detection-shared) against the bugfix branch.
2. If no frontend files touched → skip silently.
3. If frontend files touched → invoke `/entropy-e2e-frontend` with handoff:

```json
{
  "description": "<bug description / issue title>",
  "changed_files": ["<files from detection block>"],
  "mode": "post_deploy",
  "target_env": "deploy",
  "autopilot": false
}
```

Runs against `VR_DEPLOY_URL` from `.env.local`. If missing, `entropy-e2e-frontend`'s
Phase 0 will prompt once and persist it.

A visual failure here indicates either (a) the fix regressed something else, or
(b) an intentional UI change needs a baseline update. Report back to the user
for the decision.

---

## Common Mistakes

- Fixing the symptom before reproducing and tracing the cause
- Running `entropy-scan` on every tiny one-file local bug
- Skipping the scan for a one-file fix in a sensitive domain
- Turning a bugfix into an architecture rewrite
- Letting `entropy-fix` expand scope beyond one simple correction cycle
