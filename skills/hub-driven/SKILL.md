---
name: hub-driven
description: >
  End-to-end orchestrator for large tasks with quality assurance. Chains
  brainstorming, hub-aware enrichment, planning, execution, and
  hub-scan validation with automatic hub-fix if needed.
  Triggers on: hub-driven, hub driven, build with quality,
  tarea con hub, quality-assured task, full quality pipeline.
user_invocable: true
---

# Hub-Driven

<!-- SYNC: hub-driven depends on:
     - hub-aware: stateless enrichment (invoked on spec + plan)
     - hub-scan: 12-dimension validation (invoked post-execution)
     - hub-fix: correction with --skip-rescan (invoked if domains < B)
     - hub-loop: referenced as alternative in Final Report
     - ship-with-review: PR creation + fresh-agent code review (Step 8)
     - hub-e2e-frontend: invoked at Step 5.5 when frontend files touched (handoff with target_env=local)
     - hub-driven-autopilot: autonomous sibling (separate entry point, same sub-skills)
     Update if dimension count, flag contracts, or pipeline order change. -->

**Announce at start:** "Running /hub-driven — full quality-assured pipeline for this task."

> **For fully autonomous execution (no human in the loop), use `/hub-driven-autopilot` instead.** That skill resolves every human gate with deterministic rules + logged AI judgment.

## Project Tracking & Governance

Source of truth for how Hub work is tracked lives in `_jockey/`:

- `_jockey/CONVENTIONS.md` — code, session, deploy, D1, verification rules. New conventions append under `## C## additions` sections.
- `_jockey/DECISIONS.md` — `DEC-C##-NN` decision log; cite when conventions reference a DEC.
- `_jockey/STATE.md` — current Control + active phase.
- `_jockey/LOCKS.md` — file-lock manifest for shared / collision-prone files. **Read this before editing application code.** If a target file is claimed by another run, abort the edit and coordinate (post `[BLOCKED]` in #hub-dev). When you start editing a shared file, append a row claiming it; remove the row after the commit is pushed. `routes/agents.js` is in the **Shared (coordinate first)** lane and additionally requires a pre-edit ping in #hub-dev (Aaron's operating doc rule #4).
- Session prompts live in `_jockey/queue/` until fired, then move to `_jockey/archive/fired/`. Naming: `C[N]-S[#]v[ver]-[name].md`.
- New behavioral conventions or decisions that emerge from this skill's run must land in `_jockey/DECISIONS.md` (DEC entry) and a `## C## additions` block in `_jockey/CONVENTIONS.md`.
- Session prompts produced by this orchestrator must follow the queue → archive flow; do not fire from `~/Downloads` or scratch paths without staging through `_jockey/queue/` first.

> **Coexists with skill-level tracking — neither invalidates the other.** This is program-level governance. The skill's own tracking artifacts (`docs/TICKETS.md`, `docs/QUALITY_SCORE.md`, `docs/hub-loop-state.json`, evidence files in `docs/`, etc.) remain the authoritative source for the skill's operational state. `_jockey/` is the program-level layer (Control, conventions, decisions). Both must coexist.

## Overview

End-to-end orchestrator that takes a large task and delivers it with quality
assured by `/hub-scan`. Chains existing skills in a fixed pipeline:

```
brainstorming → hub-aware → writing-plans → hub-aware → execution → hub-e2e-frontend (if frontend) → hub-scan → hub-fix → ship-with-review
```

Each skill runs its full process. `hub-aware` enriches artefacts between
phases. `hub-scan` validates the final result. `hub-fix` corrects
findings if needed (max 1 cycle).

## File-locking gate (LOCKS.md)

This skill orchestrates sub-skills (`hub-fix`, `subagent-driven-development`, etc.) that write code. The file-lock convention is enforced by those sub-skills directly — see their `## File-locking gate (LOCKS.md)` sections. As the orchestrator, your responsibilities are:

1. **Verify the sub-skill enforces the gate.** Every Hub sub-skill that writes application code (`hub-fix`, `hub-bugfix`, `hub-driven-autopilot`'s execution step) carries the per-skill gate from this convention. If you dispatch a non-Hub sub-skill (e.g. `superpowers:subagent-driven-development`) to edit Hub repo files, prepend a one-line reminder to the dispatched prompt:
   > Before any Edit/Write on repo-tracked files, check `_jockey/LOCKS.md`. If your target is claimed by another run, abort and post `[BLOCKED]` in #hub-dev. Otherwise claim the row before editing and release it after push. `routes/agents.js` requires a pre-edit ping in #hub-dev regardless.
2. **Surface lock conflicts.** If a sub-skill aborts with a lock conflict, do not retry blindly — propagate the failure with the conflicting owner / file in the final report so the human can coordinate.
3. **Do not claim locks at the orchestrator level.** Locks live with whoever actually edits the file, so the sub-skill claims and releases. Orchestrator-level claims would risk dangling locks if the sub-skill releases and the orchestrator doesn't, or vice versa.

## Pipeline

### Step 1: Brainstorming

Invoke `superpowers:brainstorming` with the user's task description.

Let brainstorming run its full process: context exploration, clarifying questions,
approach proposals, design presentation, and spec writing. Do not shortcut any
phase — the spec quality directly affects everything downstream.

**Output:** spec file at `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`

### Step 2: Hub-Aware on Spec

Invoke `hub-aware` on the spec file produced in Step 1.

```
/hub-aware docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md
```

This enriches the spec with any missing hub-scan dimensions. No user
interaction — inline correction.

**Output:** same spec file, corrected inline

### Step 3: Writing Plans

Invoke `superpowers:writing-plans` using the enriched spec as input.

Let writing-plans run its full process: scope check, file structure mapping,
task decomposition with TDD steps, self-review.

**Output:** plan file at `docs/superpowers/plans/YYYY-MM-DD-<topic>.md`

### Step 4: Hub-Aware on Plan

Invoke `hub-aware` on the plan file produced in Step 3.

```
/hub-aware docs/superpowers/plans/YYYY-MM-DD-<topic>.md
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
[Frontend-Change Detection block in `hub-e2e-frontend`](../hub-e2e-frontend/SKILL.md#frontend-change-detection-shared).

- If no frontend changes → log `hub-e2e-frontend: no frontend changes, skipping` and continue to Step 6.
- If frontend changes detected → invoke `/hub-e2e-frontend` with handoff:

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

### Step 6: Hub Scan

After execution completes, run a partial hub-scan targeting only the domains
affected by the task:

```
/hub-scan <affected-domain-1> <affected-domain-2>
```

Determine affected domains from:
- The spec's architecture section (what domains it touches)
- The plan's file paths (which domain directories were modified)
- Git diff since the plan started (which directories have changes)

**Output:** `docs/QUALITY_SCORE.md` with grades for affected domains

### Step 7: Hub Fix (Conditional)

Check the hub-scan results:

- **All affected domains >= B:** `QUALITY_GATE=pass`. Proceed to Step 8.
- **Any affected domain < B:** Invoke `/hub-fix` to correct findings.
  This is limited to **1 cycle only**.

After hub-fix completes, check grades again:

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
  hub-driven.
- **`fail`**: `ship-with-review` runs in **ask mode** — it presents the
  failing grades and asks the user whether to (a) open PR + review anyway,
  (b) skip PR and iterate, or (c) open PR without review. User decides.

**Output:** PR URL + review report, or "deferred" if the user opted out.

## Final Report

Present the final summary:

```
hub-driven: task complete
  Spec: docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md
  Plan: docs/superpowers/plans/YYYY-MM-DD-<topic>.md

  Hub-scan results:
  | Domain   | Grade | Status |
  |----------|-------|--------|
  | <domain> | <grade> | <pass/needs-work> |

  hub-aware enrichments:
    Spec: <N> dimensions added/enriched
    Plan: <M> dimensions added/enriched

  Status: <All domains >= B. Done. | Some domains < B after 1 fix cycle. See findings above.>

  Ship: <PR URL + review verdict from Step 8 | deferred by user | skipped (quality gate not met)>
```

If any domain is still < B:
> "Some domains didn't reach grade B after one correction cycle. You can:
> 1. Run `/hub-fix` again for another cycle
> 2. Run `/hub-scan <domain>` to inspect specific findings
> 3. Run `/hub-loop` to autonomously iterate until all domains reach grade A
> 4. Accept the current state and iterate later"

## Important Rules

- **Full skill processes**: Each invoked skill runs its complete process. Do not shortcut brainstorming questions, skip writing-plans self-review, or bypass hub-fix test gates.
- **Max 1 hub-fix cycle**: After hub-scan, run hub-fix at most once. If grades still < B, report and stop. The user decides next steps.
- **Partial scan only**: Step 6 scans only affected domains, not the entire codebase. This keeps the feedback loop fast.
- **Preserve user interaction**: Steps 1 (brainstorming) and 5 (execution choice) involve user interaction. Do not auto-pilot these — the user's input shapes the outcome.
- **Artefact chain**: Each step builds on the previous. spec → enriched spec → plan → enriched plan → code → validation. If any step fails, stop and report.
