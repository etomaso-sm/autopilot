---
name: entropy-driven
description: >
  End-to-end orchestrator for large tasks with quality assurance. Chains
  brainstorming, entropy-aware enrichment, planning, execution, and
  entropy-scan validation with automatic entropy-fix if needed.
  Triggers on: entropy-driven, entropy driven, build with quality,
  tarea con entropy, quality-assured task, full quality pipeline.
user_invocable: true
---

# Entropy-Driven

<!-- SYNC: entropy-driven depends on:
     - entropy-aware: stateless enrichment (invoked on spec + plan)
     - entropy-scan: 12-dimension validation (invoked post-execution)
     - entropy-fix: correction with --skip-rescan (invoked if domains < B)
     - entropy-loop: referenced as alternative in Final Report
     - ship-with-review: PR creation + fresh-agent code review (Step 8)
     - entropy-e2e-frontend: invoked at Step 5.5 when frontend files touched (handoff with target_env=local)
     - entropy-driven-autopilot: autonomous sibling (separate entry point, same sub-skills)
     Update if dimension count, flag contracts, or pipeline order change. -->

**Announce at start:** "Running /entropy-driven — full quality-assured pipeline for this task."

> **For fully autonomous execution (no human in the loop), use `/entropy-driven-autopilot` instead.** That skill resolves every human gate with deterministic rules + logged AI judgment and is the canonical dispatch target for `entropy-linear-autopilot` feature tickets.

## Overview

End-to-end orchestrator that takes a large task and delivers it with quality
assured by `/entropy-scan`. Chains existing skills in a fixed pipeline:

```
brainstorming → entropy-aware → writing-plans → entropy-aware → execution → entropy-e2e-frontend (if frontend) → entropy-scan → entropy-fix → ship-with-review
```

Each skill runs its full process. `entropy-aware` enriches artefacts between
phases. `entropy-scan` validates the final result. `entropy-fix` corrects
findings if needed (max 1 cycle).

## Pipeline

### Step 1: Brainstorming

Invoke `superpowers:brainstorming` with the user's task description.

Let brainstorming run its full process: context exploration, clarifying questions,
approach proposals, design presentation, and spec writing. Do not shortcut any
phase — the spec quality directly affects everything downstream.

**Output:** spec file at `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`

### Step 2: Entropy-Aware on Spec

Invoke `entropy-aware` on the spec file produced in Step 1.

```
/entropy-aware docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md
```

This enriches the spec with any missing entropy-scan dimensions. No user
interaction — inline correction.

**Output:** same spec file, corrected inline

### Step 3: Writing Plans

Invoke `superpowers:writing-plans` using the enriched spec as input.

Let writing-plans run its full process: scope check, file structure mapping,
task decomposition with TDD steps, self-review.

**Output:** plan file at `docs/superpowers/plans/YYYY-MM-DD-<topic>.md`

### Step 4: Entropy-Aware on Plan

Invoke `entropy-aware` on the plan file produced in Step 3.

```
/entropy-aware docs/superpowers/plans/YYYY-MM-DD-<topic>.md
```

This enriches the plan with any missing steps for test coverage, boundary
validation, security, etc.

**Output:** same plan file, corrected inline

### Step 5: Execution

Offer the user the execution choice (same as writing-plans handoff):

> "Plan enriched and ready. Two execution options:
>
> 1. **Subagent-Driven (recommended)** — fresh subagent per task, review between tasks
> 2. **Inline Execution** — execute in this session with checkpoints
>
> Which approach?"

If Subagent-Driven: invoke `superpowers:subagent-driven-development`
If Inline: invoke `superpowers:executing-plans`

**Output:** implemented code, committed to git

### Step 5.5: Visual Regression (frontend only)

After Step 5 commits, check whether frontend files changed using the
[Frontend-Change Detection block in `entropy-e2e-frontend`](../entropy-e2e-frontend/SKILL.md#frontend-change-detection-shared).

- If no frontend changes → log `entropy-e2e-frontend: no frontend changes, skipping` and continue to Step 6.
- If frontend changes detected → invoke `/entropy-e2e-frontend` with handoff:

```json
{
  "description": "<from Step 1 spec title>",
  "changed_files": ["<files from detection block>"],
  "affected_urls": ["<inferred from spec + file paths>"],
  "mode": "pre_merge_local",
  "target_env": "local"
}
```

Runs against `VR_BASE_URL` (local dev server). Failures are reported alongside
Step 6 scan grades but do not block the pipeline — the user decides whether to
update snapshots or fix the UI.

**Output:** visual test results appended to the Step 6 report.

---

### Step 6: Entropy Scan

After execution completes, run a partial entropy-scan targeting only the domains
affected by the task:

```
/entropy-scan <affected-domain-1> <affected-domain-2>
```

Determine affected domains from:
- The spec's architecture section (what domains it touches)
- The plan's file paths (which domain directories were modified)
- Git diff since the plan started (which directories have changes)

**Output:** `docs/QUALITY_SCORE.md` with grades for affected domains

### Step 7: Entropy Fix (Conditional)

Check the entropy-scan results:

- **All affected domains >= B:** `QUALITY_GATE=pass`. Proceed to Step 8.
- **Any affected domain < B:** Invoke `/entropy-fix` to correct findings.
  This is limited to **1 cycle only**.

After entropy-fix completes, check grades again:

- **All >= B:** `QUALITY_GATE=pass`. Proceed to Step 8.
- **Still < B:** `QUALITY_GATE=fail`. Proceed to Step 8 anyway — the user decides.

### Step 8: Ship With Review (always runs)

Invoke `ship-with-review`, passing:

- `SPEC_PATH` = spec file from Step 1
- `PLAN_PATH` = plan file from Step 3
- `QUALITY_PATH` = `docs/QUALITY_SCORE.md` from Step 6
- `QUALITY_GATE` = `pass` | `fail` (from Step 7)

Behavior depends on `QUALITY_GATE`:

- **`pass`**: `ship-with-review` runs in **auto mode** — it creates the PR
  with the fixed body format and dispatches the fresh code-review agent
  without asking. No prompts. The PR URL and review verdict come back to
  entropy-driven.
- **`fail`**: `ship-with-review` runs in **ask mode** — it presents the
  failing grades and asks the user whether to (a) open PR + review anyway,
  (b) skip PR and iterate, or (c) open PR without review. User decides.

**Output:** PR URL + review report, or "deferred" if the user opted out.

## Final Report

Present the final summary:

```
entropy-driven: task complete
  Spec: docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md
  Plan: docs/superpowers/plans/YYYY-MM-DD-<topic>.md

  Entropy-scan results:
  | Domain   | Grade | Status |
  |----------|-------|--------|
  | <domain> | <grade> | <pass/needs-work> |

  entropy-aware enrichments:
    Spec: <N> dimensions added/enriched
    Plan: <M> dimensions added/enriched

  Status: <All domains >= B. Done. | Some domains < B after 1 fix cycle. See findings above.>

  Ship: <PR URL + review verdict from Step 8 | deferred by user | skipped (quality gate not met)>
```

If any domain is still < B:
> "Some domains didn't reach grade B after one correction cycle. You can:
> 1. Run `/entropy-fix` again for another cycle
> 2. Run `/entropy-scan <domain>` to inspect specific findings
> 3. Run `/entropy-loop` to autonomously iterate until all domains reach grade A
> 4. Accept the current state and iterate later"

## Important Rules

- **Full skill processes**: Each invoked skill runs its complete process. Do not shortcut brainstorming questions, skip writing-plans self-review, or bypass entropy-fix test gates.
- **Max 1 entropy-fix cycle**: After entropy-scan, run entropy-fix at most once. If grades still < B, report and stop. The user decides next steps.
- **Partial scan only**: Step 6 scans only affected domains, not the entire codebase. This keeps the feedback loop fast.
- **Preserve user interaction**: Steps 1 (brainstorming) and 5 (execution choice) involve user interaction. Do not auto-pilot these — the user's input shapes the outcome.
- **Artefact chain**: Each step builds on the previous. spec → enriched spec → plan → enriched plan → code → validation. If any step fails, stop and report.
