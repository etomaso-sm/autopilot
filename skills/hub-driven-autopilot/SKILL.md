---
name: hub-driven-autopilot
description: >
  Use when a feature task should run hands-off through the hub pipeline,
  especially from a scheduler or explicit autopilot request where no human
  can answer gates during the run. Triggers on: hub-driven-autopilot,
  autopilot, autopilot hub, autopilot driven, ship autopilot.
user_invocable: true
---

# Hub-Driven Autopilot

<!-- SYNC: hub-driven-autopilot depends on:
     - superpowers:brainstorming (Step 1, with autopilot preamble)
     - hub-aware (Steps 2 and 4, unchanged)
     - superpowers:writing-plans (Step 3, with autopilot preamble)
     - superpowers:subagent-driven-development (Step 5, always chosen)
     - hub-e2e-frontend (Step 5.5, conditional, target_env=local)
     - hub-scan (Step 6, partial on affected domains)
     - hub-fix (Step 7, max 1 cycle, --skip-rescan)
     - ship-with-review (Step 8, REFERENCED for body/prompt templates but NOT invoked;
       autopilot inlines the PR-creation and review-dispatch so it can append autopilot
       sections without plumbing params into ship-with-review)
     - Step 8.5 inline review-autofix loop: max 1 cycle, applies only low-risk
       (nit/minor) findings already in PR scope, no re-scan — same invariant as
       hub-fix. Re-dispatches the fresh-agent review exactly once when fixes
       are applied. No new escape hatch; budget review_fix_cycles_used caps it.
     - Step 8.7 full-mode auto-merge: opt-in via input.full_mode (default false).
       Evaluates 4 gates (approve verdict, quality pass, no unresolved findings,
       CI green; CI absence only passes for local targets) and runs `gh pr merge --rebase`
       when all pass. The autopilot branch is retained as an evidence branch so
       post-merge deploy state can be pushed. Three non-fatal hatches
       (14 ci_timeout, 15 merge_command_failed, 16 rebase_not_allowed). Never
       stuck — degrades to "PR stays open".
     - Step 8.8 hub ship verification: when full_mode merged into a
       non-local target_env, invokes hub-check-deploy and post-deploy hub-e2e-frontend
       through hub-ship's hub contract. Delivery is verified only when
       deploy + required post-deploy E2E pass.
     External deps used in bash snippets: gh, git, jq, openssl.
     Expected callers: /schedule, direct human invocation.
     Cross-ref: hub-driven (interactive sibling).
     Update if sub-skill names, escape hatches, state schema, or the PR/review
     inline logic change.

     Hardening additions (post-batch-A-B-C 2026-04-29):
     - `## Strict compliance` section between Bootstrap and State Management
       enumerates mandatory steps, forbidden self-justifications, allowed
       warnings, and the subagent-context exception rule.
     - Bootstrap step 5 RUN_ID now includes `$$` (PID) and an existence check
       to prevent parallel-run id collisions.
     - Step 8.4 has explicit "inline review is FORBIDDEN" language.
     - Two new fatal escape hatches: 17 (`mandatory_step_skipped`) and 18
       (`fabricated_run_id`). Both trip at archive time if state shape betrays
       the contract violation. -->

**Announce at start:** "Running /hub-driven-autopilot — fully autonomous quality-assured pipeline."

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

End-to-end orchestrator that runs the same pipeline as `/hub-driven` — brainstorming, hub-aware enrichment, writing-plans, execution, hub-e2e-frontend, hub-scan, hub-fix, ship-with-review — but **resolves every human-in-the-loop gate with deterministic rules plus logged AI judgment**.

Intended outputs:
- **Success:** a feature branch pushed to GitHub with an open PR whose body includes the spec, plan, quality grades, and autopilot decision log; a fresh-agent code review already dispatched.
- **Stuck:** a branch named `autopilot/<date>-<slug>` with all artefacts committed and a state file archived to `docs/autopilot-runs/<run_id>.json`. No PR. The stuck reason and recovery hints are included in the final report.

**Relation to `/hub-driven`:** `/hub-driven` is unchanged and remains the interactive variant. This skill is a separate entry point, usually invoked by a `/schedule`-dispatched agent or a human who wants hands-off execution.

**Transparency:** every decision the AI makes at a would-be human gate is written to three places — the spec (`## Autopilot Q&A`), the plan (`## Autopilot decisions`), and the state file. The fresh-agent reviewer in Step 8 is told explicitly that no human reviewed the spec/plan, and its verdict replaces that missing gate.

## Activation and Inputs

### Invocation

Freeform:
```
/hub-driven-autopilot <task description>
/hub-driven-autopilot --from-ticket docs/tickets/YYYY-MM-DD-<slug>.md
```

Structured (preferred when dispatched by another agent):
```
/hub-driven-autopilot { "goal": "...", "description": "...", "constraints": [...], "success_criteria": [...], "scope_boundaries": [...], "ticket_id": "IMP-123" }
```

### Input mode detection

If the argument parses as JSON and has a `description` field, treat it as structured input. Otherwise, the entire argument is the freeform `description` and every other field defaults.

**Freeform `--full` flag.** In freeform mode, the literal token `--full` anywhere in the argument — delimited by whitespace — sets `full_mode=true`. The parser strips that token before treating the remainder as `description`. Example: `/hub-driven-autopilot --full add retry button` parses as `{description: "add retry button", full_mode: true}`. In structured JSON mode, pass `"full_mode": true` on the input object.

**Ticket file input.** If the argument includes `--from-ticket <path>`, read the
YAML frontmatter from that ticket and construct the structured input from:

```text
goal, description, constraints, success_criteria, scope_boundaries, ticket_id,
base_branch, max_wall_clock_min, target_env, full_mode, require_deploy
```

If `ticket_id` is absent but frontmatter `id` is set, use `id` as `ticket_id`.
Do not read arbitrary body prose except the `## Hub autopilot input` JSON
block when frontmatter is missing or incomplete. If both frontmatter and JSON
block exist, frontmatter wins and the JSON fills only missing fields.

### Input schema

| Field | Default | Purpose |
|---|---|---|
| `goal` | first sentence of `description` | Outcome that defines success for the ticket. |
| `description` | required | Task to execute. |
| `constraints` | `[]` | Technical/business rules the AI must respect. |
| `success_criteria` | `[]` | Bullets the resulting spec must satisfy. |
| `scope_boundaries` | `[]` | Anti-scope — what NOT to do. |
| `ticket_id` | `null` | Included in the PR body as `Closes <id>`. |
| `base_branch` | `main` | Branch to fork from. |
| `max_wall_clock_min` | `60` | Wall-clock escape hatch ceiling. |
| `target_env` | `"local"` | Passed to hub-e2e-frontend in Step 5.5. |
| `full_mode` | `false` | When `true`, Step 8.7 evaluates merge gates and auto-merges the PR with `gh pr merge --rebase`. Default off. |
| `require_deploy` | `true` when `target_env != "local"`, else `false` | When true, a merged full-mode run must verify deploy + post-deploy checks before delivery is considered verified. |

## Bootstrap

Before Step 1 of the pipeline. Every step below checks a condition; when the condition fails, the agent executes the **Abort Procedure** in `## Escape Hatches` with the indicated `stuck_reason`. There is no shell function called `exit_stuck` — it is the agent's responsibility to run the abort sequence.

1. **Dependency check.**
   ```bash
   command -v gh  >/dev/null                || stuck=missing_deps
   command -v git >/dev/null                || stuck=missing_deps
   command -v jq  >/dev/null                || stuck=missing_deps
   gh auth status >/dev/null 2>&1           || stuck=missing_deps
   ```
   Run each line. If `stuck` is set, execute the Abort Procedure with `stuck_reason=missing_deps`.

2. **Branch safety.**
   ```bash
   CURRENT=$(git rev-parse --abbrev-ref HEAD)
   case "$CURRENT" in
     main|master|develop|production|staging) stuck=protected_branch ;;
   esac
   ```
   If `stuck=protected_branch`, abort. Do not create a new branch.

3. **Working tree.** If dirty, try to stash:
   ```bash
   if [ -n "$(git status --porcelain)" ]; then
     git stash push -u -m "autopilot: pre-bootstrap" || stuck=dirty_tree
   fi
   ```
   Untracked-only trees are acceptable because `git stash -u` includes them; an outright failure (merge conflicts, submodule issues) is `dirty_tree`.

4. **Create the autopilot branch** from `base_branch`. The slug is produced inline — no external `slugify` command is required:
   ```bash
   SLUG=$(printf '%s' "$description" \
          | tr '[:upper:]' '[:lower:]' \
          | sed -E 's/[^a-z0-9]+/-/g; s/^-+//; s/-+$//' \
          | cut -c1-40)
   DATE=$(date -u +%Y-%m-%d)
   BRANCH="autopilot/${DATE}-${SLUG}"
   git checkout "$base_branch"
   # Pull is best-effort: in worktrees or local-only repos it may fail; record a warning and continue.
   git pull --ff-only 2>/dev/null || echo "autopilot: ff-pull skipped (no upstream or non-fast-forward)"
   git checkout -b "$BRANCH"
   ```

   If invoked with `--from-ticket`, update the ticket file status to
   `in-progress`, set/confirm `ticket_id`, and commit that change on the
   autopilot branch with:
   ```bash
   git add docs/tickets/<ticket>.md
   git commit -m "chore(autopilot): mark ticket in progress"
   ```
   If the ticket file is not writable, continue and record a warning; the
   ticket file remains the source for dispatch status.

5. **Generate `run_id`:**
   ```bash
   RUN_ID="$(date -u +%Y-%m-%dT%H-%M-%SZ)-$$-$(openssl rand -hex 3)"
   # `$$` is the autopilot process PID. Including it guarantees that two
   # autopilots dispatched to the same repo at the same wall-clock second
   # cannot collide on `run_id` even if the random suffix matches. This was
   # added after a parallel batch produced 3 different runs with an identical
   # `run_id` because subagents short-circuited the random call.
   if [ -e "docs/autopilot-runs/${RUN_ID}.json" ]; then
     # Path already exists — extend the random suffix and retry once.
     RUN_ID="$(date -u +%Y-%m-%dT%H-%M-%SZ)-$$-$(openssl rand -hex 6)"
     [ -e "docs/autopilot-runs/${RUN_ID}.json" ] && stuck=run_id_collision
   fi
   ```
   You MUST run the command above and use its actual stdout. You MUST NOT
   provide a `run_id` from memory or from any example in this document.
   The `run_id` is the post-mortem key — a fabricated id silently overwrites
   another run's archived state file.

6. **Initialize the state file** (schema in `## State Management`) at `docs/autopilot-state.json` with `current_step: "bootstrap"`. Populate `input.full_mode` from the parsed flag/JSON (default `false`). Initialize `merge_status: null`, `merged: false`, `merge_sha: null`, and all four `merge_gates_evaluated.*` fields as `null`. Then commit:
   ```bash
   mkdir -p docs
   # write state file (schema below) using jq or a Python stdlib script; never string-interpolate user input.
   git add docs/autopilot-state.json
   git commit -m "chore(autopilot): initialize run ${RUN_ID}"
   ```

7. Proceed to Step 1 of the pipeline.

## Strict compliance — what you cannot self-justify

This skill is a rigid pipeline. Several behaviors that "feel reasonable" for a small fix are explicitly forbidden because they erode the autopilot contract: every gate this pipeline enforces is replacing a human gate that does not exist. Downgrading any of them voids the contract.

### Mandatory steps — no self-skipping

The following pipeline steps are MANDATORY. They MUST be invoked exactly as written. They MUST NOT be skipped, inlined, or "evaluated by the orchestrator" in lieu of the underlying sub-skill, regardless of how narrow, surgical, trivial, security-scoped, or self-evidently-aware the diff appears.

| Step | Sub-skill | Skip allowed when |
|---|---|---|
| 2 | `/hub-aware` on spec | Never. On non-zero exit, retry once → fatal hatch 13 (`hub_aware_failed`) on second failure. |
| 3 | `superpowers:writing-plans` | Never. Empty plan trips fatal hatch 5 (`empty_plan`). |
| 4 | `/hub-aware` on plan | Never. Same retry-once → hatch 13 rule as Step 2. |
| 6 | `/hub-scan` | Never. The grades are part of the autopilot output the human reviewer needs. |
| 7 | `/hub-fix` | Allowed when `state.quality_gate == "pass"` after Step 6 (no domain below B). NOT allowed for any other reason. |
| 8.4 | Agent-tool dispatch of fresh-agent review | Never silently. If the Agent tool genuinely fails, trip hatch 11 (`review_dispatch_failed`) and record `review_verdict=not-dispatched`. **Inline review is forbidden.** |

### Forbidden self-justifications (these all trip hatch 17)

If you find yourself reasoning along any of these lines, STOP. Either invoke the step as written or trip the corresponding escape hatch — there is no third path.

- "This is a narrow / surgical / trivial / 2-line / security-only fix, so hub-aware would be a no-op."
- "The spec already encodes hub conventions inline, so hub-aware-spec adds nothing."
- "The plan is one task, so hub-aware-plan is unnecessary."
- "The diff is config-only, so hub-scan does not apply to this domain."
- "hub-scan would surface findings out of scope per the no-refactor constraint, so I'll skip it."
- "I'll evaluate the 13 quality dimensions inline in the main agent."
- "I'll do the review inline because the Agent tool is not available in this subagent context." (See next subsection — this is almost always wrong.)
- "I'll provide a structured verdict that matches the rubric without dispatching a fresh agent."

These are not warnings. These are fatal hatch 17 (`mandatory_step_skipped`). Self-skipping is never a degraded mode — it is a contract violation, and a run that self-skipped a mandatory step ships a PR with no audit trail proving the gate was satisfied.

### Allowed warnings (these are fine and expected)

These are the only `step_history[-1].warning` entries the pipeline supports for steps in the table above:

- `hub-e2e-frontend: no frontend files changed in diff` — Step 5.5 is conditional on a file-change check; this is a legitimate skip, not in the mandatory table.
- `hub-e2e-frontend: local server not reachable` — Step 5.5's only documented degraded mode (per the section).
- `hub-fix: skipped — quality_gate already pass` — Step 7 is conditional on Step 6's result; this is a legitimate skip.

Anything else logged as a "warning" against a mandatory step is a hatch 17 trip.

### Subagent context is not an exception

If you were dispatched as a subagent (your caller invoked you via the Agent tool with `subagent_type: general-purpose`), you have the **same toolset and the same obligations** as a top-level invocation. The Agent tool IS available to nested `general-purpose` subagents — the tool list for that type is `*`.

Before claiming "Agent tool not available in this subagent context" and falling back to inline review, you MUST actually attempt to invoke the Agent tool. Only if the attempt itself fails (the tool returns an error, not "I assume it would fail") may you trip hatch 11. A run that records `review_verdict=approve` from an inline self-review is a contract violation; record `not-dispatched` instead so the human reviewer knows the gate was not satisfied.

### Why this exists

A prior autopilot batch shipped 9 PRs claiming `approve` from "inline self-review" because the dispatched subagents assumed (without testing) that the Agent tool was unavailable. A 10th PR honestly tripped hatch 11 and was clearly flagged for human review — that one was correct behavior; the other 9 misled the caller. This section is the corrective.

## State Management

The state file is written **at every transition** and committed. It is both the runtime source of truth and the post-mortem artefact.

**Path during a run:** `docs/autopilot-state.json`
**Path after the run (success or stuck):** `docs/autopilot-runs/<run_id>.json`, archived in the last commit before either opening the PR or finishing with `stuck`.

**Schema:**

```json
{
  "run_id": "2026-04-16T14-30-00Z-a1b2c3",
  "started": "2026-04-16T14:30:00Z",
  "last_update": "2026-04-16T14:47:12Z",
  "input": {
    "goal": "...",
    "description": "...",
    "constraints": [],
    "success_criteria": [],
    "scope_boundaries": [],
    "ticket_id": "IMP-123",
    "base_branch": "main",
    "max_wall_clock_min": 60,
    "target_env": "local",
    "full_mode": false,
    "require_deploy": false
  },
  "branch": "autopilot/2026-04-16-add-foo",
  "current_step": "execution",
  "step_history": [
    {"step": "bootstrap",     "status": "done", "started": "...", "ended": "..."},
    {"step": "brainstorming", "status": "done", "started": "...", "ended": "..."}
  ],
  "artefacts": {
    "spec_path":    "docs/superpowers/specs/2026-04-16-add-foo-design.md",
    "plan_path":    "docs/superpowers/plans/2026-04-16-add-foo.md",
    "quality_path": "docs/QUALITY_SCORE.md"
  },
  "decisions": [
    {
      "step": "brainstorming",
      "gate": "approach_choice",
      "options": ["A: monolith", "B: service split", "C: hybrid"],
      "chosen": "B",
      "confidence": "high",
      "rationale": "constraint X rules out A; B matches neighboring patterns in src/..."
    }
  ],
  "budgets": {
    "hub_fix_cycles_used":    0,
    "test_retries_used":          0,
    "implementer_answers_given":  0,
    "review_fix_cycles_used":     0
  },
  "stuck_reason":    null,
  "quality_gate":    null,
  "ticket_goal_satisfied": null,
  "success_criteria_satisfied": null,
  "scope_boundaries_respected": null,
  "pr_url":          null,
  "review_verdict":  null,
  "review_findings": [],
  "review_history":  [],
  "merge_status":    null,
  "merged":          false,
  "merge_sha":       null,
  "merge_gates_evaluated": {
    "G1_review_approve": null,
    "G2_quality_pass":   null,
    "G3_no_unresolved":  null,
    "G4_ci_green":       null,
    "ci_status":         null
  },
  "delivery_status": null,
  "deploy_status": null,
  "deploy_result": null,
  "post_deploy_e2e": null,
  "frontend_e2e_result": null,
  "ship_result": null
}
```

**Transitions:** after every step, update `step_history`, `last_update`, `artefacts`, `decisions`, and `budgets`, then commit:

```bash
git add docs/autopilot-state.json
git commit -m "chore(autopilot): <step> done"
```

**Archive:** immediately before opening the PR (or immediately before reporting `stuck`):

```bash
mkdir -p docs/autopilot-runs
git mv docs/autopilot-state.json "docs/autopilot-runs/${run_id}.json"
git commit -m "chore(autopilot): archive state ${run_id}"
```

This prevents subsequent runs from overwriting the record.

**Resume is not an objective of v1.** A failed or killed run leaves the state file as evidence; the human inspects it and re-invokes from scratch.

**No secrets in state.** `input.description` is assumed to be non-sensitive (it came from the user or a ticket file). Do not write credentials, tokens, or environment variables into the state file.

**`merge_status` values** (all `null` when `input.full_mode == false`):

| Value | Meaning |
|---|---|
| `null` | Full mode disabled or Step 8.7 not reached |
| `pending_merge` | Gates passed, merge command about to run |
| `merged` | Merge succeeded |
| `skipped_review_not_approved` | G1 failed |
| `skipped_quality_failed` | G2 failed |
| `skipped_unresolved_findings` | G3 failed |
| `skipped_ci_failed` | G4 failed — CI red |
| `skipped_ci_timeout` | G4 failed — `gh pr checks --watch` timed out |
| `skipped_rebase_not_allowed` | Repo blocks rebase merge |
| `skipped_timeout` | Wall-clock already exhausted at Step 8.7 entry |
| `merge_command_failed` | `gh pr merge` returned non-zero |

`merged` is the convenience boolean `merge_status == "merged"`.

## File-locking gate (LOCKS.md)

This skill writes application code. Before any tool call that edits a repo-tracked file, enforce the file-lock convention defined in `_jockey/LOCKS.md`. The rules and lock format live in that file; this section is the per-skill enforcement contract.

### Pre-edit gate (every code-modifying tool call)

For each file you intend to edit:

1. **Check.** Read `_jockey/LOCKS.md`. Inspect the "Active Locks" table for a row matching the target file path.
2. **Blocked path** — file is claimed by a different run/skill:
   - If the timestamp is ≤ 24h old: post `[BLOCKED]` in `#hub-dev` (channel `C0B18NVCR5E`) with the file, your run id, and the conflicting owner. Wait for release. Do not edit.
   - If the timestamp is > 24h old: stale per LOCKS.md rule. Overwrite the row with your claim and add `replaces stale claim from <prev owner>` to the notes column.
3. **`routes/agents.js` special case** — even when unclaimed, this file is the **"Shared (coordinate first)"** lane in LOCKS.md ownership lanes. Aaron's operating doc rule #4 requires a pre-edit ping. Post a `[STATUS]` one-liner in `#hub-dev` announcing the edit (file, run id, intent) before claiming. Wait for an ack or 5 min, whichever comes first.
4. **Claim.** Append a row to the "Active Locks" table:

   ```md
   | <file path> | <skill> run <run_id-or-branch> | <ISO-8601 UTC> | <one-line purpose> |
   ```

   Commit it as its own commit before any application-code edit:

   ```bash
   git add _jockey/LOCKS.md
   git commit -m "chore(locks): claim <file> for <skill> run <id>"
   ```

### Release (post-completion)

After this run's final application-code commit on the claimed file is pushed AND the work using that file is complete (PR merged, run archived, or `done`):

1. Remove your row from `_jockey/LOCKS.md`.
2. Commit: `chore(locks): release <file> after <skill> run <id>`.
3. Push.

If this run aborts mid-flight (escape hatch fired, manual stop), the row remains as evidence; the next run's stale-lock check (step 2 above) will reclaim it after 24h. Do not pre-emptively release on abort — leaving the row makes the conflict visible.

### What counts as a "shared file"

LOCKS.md applies to any repo-tracked file that more than one engineer or autopilot run might touch. In practice that's the entire `routes/`, `lib/`, `migrations/`, `src/components/`, `src/pages/`, `worker-api.js`, every `wrangler*.toml`, `_jockey/*` (excluding `LOCKS.md` itself), and `package.json` / `package-lock.json`. Skill-private artifacts (`docs/autopilot-runs/<run_id>.json`, `docs/QUALITY_SCORE.md`, scratch files under `/tmp/`) do **not** need a claim.

`routes/agents.js` is the canonical collision-prone file and always requires the pre-edit ping per step 3 above.

## Pipeline

Steps 1–7 invoke existing sub-skills with this autopilot preamble prepended to the prompt (Step 8 is inlined and does not receive the preamble):

> **Autopilot mode.** No human is available. Resolve every gate using, in priority order: structured input → repo conventions (CLAUDE.md, neighboring code) → PRINCIPLES.md → reasoned default. Log each resolved gate with `{step, gate, options?, chosen, confidence, rationale}` into `docs/autopilot-state.json` under `decisions` while that file exists. After the archive in Step 8 the runtime path becomes `docs/autopilot-runs/<run_id>.json` — sub-skills invoked before Step 8 should not need to know about the archive.

### Step 1: Brainstorming

Invoke `superpowers:brainstorming` with the autopilot preamble plus the full input object.

Per-gate policy:

| Gate | Policy |
|---|---|
| Visual companion offer | Skip — no viewer. |
| Clarifying questions | AI generates each question and answers it from `input` → repo → PRINCIPLES.md → default. Every Q/A appended to the spec under `## Autopilot Q&A`. |
| 2–3 approach choice | AI generates all three options, picks the one best matching `constraints` + `success_criteria`, logs the choice and writes all three into the spec under `## Approach` with the chosen one marked. |
| Per-section approval | Self-approve if every section contains concrete content (no `TBD`, no literal `<placeholder>`, no empty lists). |
| Final design approval | Self-approve. |
| Spec self-review | Run the brainstorming checklist (placeholders, contradictions, ambiguity, scope). Fail → retry once. Second failure → `stuck_reason: spec_review_failed`. |

**Output:** spec at `docs/superpowers/specs/YYYY-MM-DD-<slug>-design.md`. Commit + state update.

### Step 2: Hub-aware on spec

```
/hub-aware docs/superpowers/specs/YYYY-MM-DD-<slug>-design.md
```

Already non-interactive. If the command fails (non-zero exit, stack trace, or the spec file is unchanged after the call), retry once with the error captured as context. If it fails a second time, abort with `stuck_reason=hub_aware_failed` (hatch 13).

### Step 3: Writing-plans

Invoke `superpowers:writing-plans` with the autopilot preamble.

| Gate | Policy |
|---|---|
| Scope concerns flagged by self-review | Resolve by **narrowing scope** — never expand. Log each narrowing as a decision with `gate: scope_adjustment`. |
| Self-review | Run as normal. |

**Output:** plan at `docs/superpowers/plans/YYYY-MM-DD-<slug>.md`. Commit + state update.

### Step 4: Hub-aware on plan

```
/hub-aware docs/superpowers/plans/YYYY-MM-DD-<slug>.md
```

Same failure policy as Step 2: retry once on non-zero exit; second failure → `stuck_reason=hub_aware_failed` (hatch 13).

### Step 5: Execution

Always invoke `superpowers:subagent-driven-development`. Do not offer the inline option.

| Gate | Policy |
|---|---|
| Implementer subagent asks a question | Main agent answers from `spec + plan + state.decisions`. Budget: **3 answers per task** (counter increments on each answer). On the 4th question, check `budgets.implementer_answers_given >= 3` → `stuck_reason: implementer_overloaded` (do NOT answer the 4th). |
| Implementer returns `blocked` | Main agent retries once with the error context. Increment `budgets.test_retries_used` on the retry. On the 2nd `blocked`, check `>= 1` → `stuck_reason: subagent_blocked` (do NOT retry a second time). |
| Branch safety | Already on `autopilot/...` from bootstrap; do not create another branch. |

The two-stage reviews inside `subagent-driven-development` (spec compliance + code quality) are preserved as-is; they help the autopilot output meet its own standards.

### Step 5.5: hub-e2e-frontend (conditional)

Run the shared Frontend-Change Detection block. If frontend files were touched:
invoke `/hub-e2e-frontend` with this structured payload and parse
`HUB-E2E-FRONTEND-RESULT`:

```json
{
  "description": "<from spec title>",
  "changed_files": ["<files>"],
  "affected_urls": ["<inferred>"],
  "mode": "pre_merge_local",
  "target_env": "local",
  "autopilot": true
}
```

If the local dev server is not reachable, **skip with a warning**: set `state.step_history[-1].status = "skipped"` and `state.step_history[-1].warning = "local server not reachable"` (use the same `warning` field name as non-fatal escape hatches — do not invent a `reason` field here). This is not a stuck condition; the fresh-agent reviewer in Step 8 sees the note.

### Step 6: Hub-scan (partial)

Determine affected domains from:
- spec architecture section
- plan file paths
- `git diff $base_branch...HEAD --name-only`

Run:
```
/hub-scan <domain-1> <domain-2> ...
```

Outputs `docs/QUALITY_SCORE.md` + `docs/quality-score.json`. Update `artefacts.quality_path` in state.

### Step 7: Hub-fix (conditional)

Max **1 cycle**.

- All affected domains ≥ B → `state.quality_gate = "pass"`. Proceed.
- Any < B → invoke:
  ```
  /hub-fix <domain-1> <domain-2> --skip-rescan
  ```
  Increment `budgets.hub_fix_cycles_used`. Re-read `docs/quality-score.json`.
  - All ≥ B after fix → `state.quality_gate = "pass"`.
  - Still < B → `state.quality_gate = "fail"`. **Do not abort** — proceed to Step 8.

### Step 8: Open PR + dispatch fresh-agent review (inline, no user prompts)

Autopilot **does not delegate** this step to `ship-with-review`. The logic is inlined here so the PR body and the review prompt can include autopilot-specific sections without plumbing extra parameters into `ship-with-review`. Use `ship-with-review/SKILL.md` as the reference for the PR body structure — we replicate its format and append the three autopilot sections.

8.1 — **Archive the state file** (per `## State Management`):
```bash
mkdir -p docs/autopilot-runs
git mv docs/autopilot-state.json "docs/autopilot-runs/${RUN_ID}.json"
git commit -m "chore(autopilot): archive state ${RUN_ID}"
```

8.2 — **Build the PR body.** Start from the `ship-with-review` template (Summary, Spec & Plan, Changes, Quality Grades, Manual Test Plan, Automated Coverage, Notes for reviewers), then append the three autopilot sections defined in `## Output → PR body additions`. Write the full body to a temp file:
```bash
BODY_FILE=$(mktemp)
# ... agent writes the assembled markdown to $BODY_FILE ...
```

Derivation rules when autopilot fills each section:
- **Manual Test Plan**: pull bullets from `input.success_criteria` (if provided) and from the spec's `## Autopilot Q&A` section. Each bullet must be an action + expected result a human can run locally. Do NOT paraphrase TDD tests here — those are for Automated Coverage. If the diff is purely internal (no user-visible behavior), omit the whole section per `ship-with-review`'s omit rule.
- **Automated Coverage**: pull unit/integration counts and commands from the `subagent-driven-development` run, and visual-regression pass/fail from Step 5.5 `/hub-e2e-frontend` if it was invoked. Hub-scan grades from Step 6 already render in `## Quality Grades` above — do NOT duplicate them here. If a bullet has no data, write `"not run"` — never invent numbers.

8.3 — **Open the PR:**
```bash
git push -u origin "$BRANCH"
PR_URL=$(gh pr create \
  --base "$base_branch" \
  --title "$PR_TITLE" \
  --body-file "$BODY_FILE" \
  | tail -n1)
# `gh pr create` emits the URL on the last line; `tail -n1` strips any
# warnings/notices printed before it. Alternatively, use `--json url --jq .url`
# on newer gh versions.
```
If `gh pr create` exits non-zero, abort with `stuck_reason=pr_creation_failed`.

8.4 — **Dispatch the fresh-agent review** with the Agent tool (subagent_type: `general-purpose`, fresh context). The prompt is the standard `ship-with-review` Step 4 template **plus** the autopilot context block defined in `## Output → Fresh-agent review prompt — autopilot context block`.

The reviewer MUST return the ticket-contract verdict as the last block of its response — a single JSON object:

```json
{
  "verdict": "approve | request-changes | comment",
  "ticket_goal_satisfied": true,
  "success_criteria_satisfied": true,
  "scope_boundaries_respected": true,
  "findings": [
    {
      "severity":      "nit | minor | moderate | major | blocker",
      "category":      "bug | style | naming | edge-case | perf | security | docs | scope | architecture | other",
      "location":      "path/to/file.py:123",
      "description":   "what's wrong",
      "suggested_fix": "concrete patch idea, one paragraph max",
      "auto_fixable":  true
    }
  ]
}
```

The `auto_fixable` flag is the reviewer's first-pass judgment; Step 8.5 applies the strict classification rules regardless and may override it. `findings` may be `[]` (empty) for clean reviews or `comment`-only verdicts with no actionables.

Capture all five fields. Write `state.review_verdict`,
`state.review_findings`, `state.ticket_goal_satisfied`,
`state.success_criteria_satisfied`, and `state.scope_boundaries_respected`.
If any ticket-contract boolean is `false` and no finding explains the gap,
append a synthetic `scope` finding before writing state. Commit the state
update.

If the Agent tool fails (hatch 11 `review_dispatch_failed`), record
`review_verdict: not-dispatched`, `review_findings: []`,
`ticket_goal_satisfied: null`, `success_criteria_satisfied: null`, and
`scope_boundaries_respected: null`, then skip directly to Step 8.6 — the PR is
already open.

**"Failure" means an actual tool error**, not an assumption. If you are running as a subagent and you suspect the Agent tool may not be available, you MUST attempt the dispatch first. The subagent_type `general-purpose` has full tool access including the Agent tool, so the dispatch will normally succeed. Only when the attempted call returns an actual error may hatch 11 fire.

**Inline review as a fallback is FORBIDDEN.** You MUST NOT, under any circumstances, generate a verdict yourself and write it to `state.review_verdict` as if a fresh-agent had produced it. Doing so produces a `approve` value that the caller cannot distinguish from a true fresh-agent approval, which silently defeats the autopilot's only safeguard against the missing human gate. If the Agent tool is unavailable, `review_verdict` MUST be `not-dispatched` — that is what tells the human reviewer to look closely. A run that records `approve` from an inline self-review trips fatal hatch 17 (`mandatory_step_skipped`) at archive time and the autopilot reports `status: stuck` rather than `shipped`.

8.5 — **Triage findings and apply low-risk autofix (at most once).**

**Gate — skip this step if any of these hold:**
- `review_verdict == "approve"` AND `review_findings` is empty AND all three
  ticket-contract booleans are `true`
- `review_findings` is empty
- `budgets.review_fix_cycles_used >= 1` (should never happen in a clean run; defensive)

**Low-risk classification — ALL must hold for a finding to be auto-fixed:**
- `severity ∈ {nit, minor}` — never `moderate`, `major`, or `blocker`
- `location` is a file that appears in `git diff $base_branch...HEAD --name-only` (i.e., already within PR scope)
- `suggested_fix` requires NO new dependencies, NO new files outside the existing diff, and NO changes to spec/plan/architecture/public APIs
- The fix fits in a single commit (no structural refactor)

Re-classify every finding against these rules; override the reviewer's `auto_fixable` value. If **no findings pass**, skip to Step 8.6 — the PR stays as-is with the current verdict and the human merges after review.

**If at least one finding is auto-fixable, run exactly one autofix cycle:**

1. For each auto-fixable finding, apply the fix directly with Edit/Write. **Do NOT** invoke `subagent-driven-development` for this — fixes are scoped, fast, and must stay in the main agent. For each applied fix, append one entry to `state.decisions`:
   ```json
   {
     "step": "ship",
     "gate": "review_autofix",
     "options": null,
     "chosen": "<finding title>",
     "confidence": "high",
     "rationale": "<location> — <one-line action taken>"
   }
   ```

2. Commit the batch:
   ```bash
   git commit -m "fix(autopilot-review): apply ${N} low-risk review finding(s)"
   ```

3. Push:
   ```bash
   git push origin "$BRANCH"
   ```

4. Increment `budgets.review_fix_cycles_used` (now equals 1). Commit the state update.

5. **Re-dispatch the fresh-agent review exactly once.** Use the same Agent tool call as Step 8.4, with the same prompt template plus this note prepended:
   > Previous review requested changes; autopilot applied N low-risk fixes in commit `<sha>`. Verify them and re-review.

   Before overwriting, append the previous `{verdict, findings,
   ticket_goal_satisfied, success_criteria_satisfied,
   scope_boundaries_respected}` set to `state.review_history`. Then overwrite
   `state.review_verdict`, `state.review_findings`,
   `state.ticket_goal_satisfied`, `state.success_criteria_satisfied`, and
   `state.scope_boundaries_respected` with the new reviewer output. Commit the
   state update.

6. **Do NOT re-run `hub-scan`.** The autofix is narrow and bounded; same invariant as `hub-fix --skip-rescan`.

7. Regardless of the second verdict, proceed to Step 8.6. If the second review still returns `request-changes`, the PR stays open with the updated verdict in the final report; the human merges after inspecting the remaining findings.

**Findings that did NOT pass low-risk classification** are left unresolved and surfaced in the final report under "Unresolved review findings" (see `## Output → Final report`).

8.6 — **Record outcome in the archived state file** safely (no shell interpolation inside Python). Use `jq`:
```bash
jq --arg pr "$PR_URL" \
   --arg v "$VERDICT" \
   --argjson goal_ok "$TICKET_GOAL_SATISFIED" \
   --argjson criteria_ok "$SUCCESS_CRITERIA_SATISFIED" \
   --argjson scope_ok "$SCOPE_BOUNDARIES_RESPECTED" \
   --arg now "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
   '.pr_url = $pr
    | .review_verdict = $v
    | .ticket_goal_satisfied = $goal_ok
    | .success_criteria_satisfied = $criteria_ok
    | .scope_boundaries_respected = $scope_ok
    | .last_update = $now' \
   "docs/autopilot-runs/${RUN_ID}.json" > "docs/autopilot-runs/${RUN_ID}.json.tmp" \
   && mv "docs/autopilot-runs/${RUN_ID}.json.tmp" "docs/autopilot-runs/${RUN_ID}.json"

git add "docs/autopilot-runs/${RUN_ID}.json"
git commit -m "chore(autopilot): record PR ${PR_URL} and verdict"
git push origin "$BRANCH"
```

Set `TICKET_GOAL_SATISFIED`, `SUCCESS_CRITERIA_SATISFIED`, and
`SCOPE_BOUNDARIES_RESPECTED` to `true`, `false`, or `null` from the latest
review output before running jq. `review_findings`, `review_history`, and
`budgets.review_fix_cycles_used` are already in the archived state file — they
were written during Steps 8.4 and 8.5 and committed as they changed. This
final jq call records the PR URL, the latest verdict, and the latest ticket
contract booleans.

### Step 8.7: Full-mode merge (conditional)

Runs only when `state.input.full_mode == true`. Skipped entirely when `full_mode == false` — no gate evaluation, no decision entries, no state changes.

**Wall-clock precheck:**
```bash
elapsed=$(( $(date +%s) - START_EPOCH ))
budget=$(( max_wall_clock_min * 60 - elapsed ))
if [ "$budget" -le 0 ]; then
  merge_status="skipped_timeout"
  # jump to 8.7.5 (record outcome)
fi
ci_timeout=$(( budget < 900 ? budget : 900 ))
```

If `budget <= 0`, skip all gate evaluation and go directly to 8.7.5 with `merge_status=skipped_timeout`.

**Gates (all must pass, evaluated in order, short-circuit on first failure):**

| Gate | Check | Failure → `merge_status` |
|---|---|---|
| G1 | `state.review_verdict == "approve"` AND all three ticket-contract booleans are `true` | `skipped_review_not_approved` |
| G2 | `state.quality_gate == "pass"` | `skipped_quality_failed` |
| G3 | `state.review_findings_unresolved == []` (use Step 8.5 low-risk rules to classify) | `skipped_unresolved_findings` |
| G4 | CI green; absent CI is allowed only for `target_env == "local"` | `skipped_ci_failed` or `skipped_ci_timeout` |

Each **evaluated** gate appends a decision entry:
```json
{"step": "ship", "gate": "merge_gate_G<N>", "options": ["<pass conditions>"], "chosen": "pass | fail", "confidence": "high", "rationale": "<what was observed>"}
```
Gates skipped by short-circuit leave `state.merge_gates_evaluated.G<N>_*` as `null`.

**G4 — CI detection:**
```bash
CI_JSON=$(gh pr checks "$PR_URL" --json state,name 2>&1 || true)
if echo "$CI_JSON" | grep -q "no checks reported\|no required checks"; then
  CI_STATUS="absent"
elif echo "$CI_JSON" | jq -e '. | length == 0' >/dev/null 2>&1; then
  CI_STATUS="absent"
else
  CI_STATUS="present"
fi
# record ci_status into state.merge_gates_evaluated.ci_status
```
- `CI_STATUS="absent"` and `state.input.target_env == "local"` → G4 passes.
- `CI_STATUS="absent"` and `state.input.target_env != "local"` → G4 fails with `merge_status=skipped_ci_failed`. Automated development/staging/production delivery requires CI evidence.
- `CI_STATUS="present"` → wait for checks with bounded timeout:
  ```bash
  timeout "${ci_timeout}s" gh pr checks "$PR_URL" --watch --fail-fast
  rc=$?
  ```
  - `rc == 0` → G4 passes.
  - `rc == 124` (timeout) → `merge_status=skipped_ci_timeout`, `ci_status=timeout`.
  - Any other non-zero → `merge_status=skipped_ci_failed`.

**If any gate fails:** record `merge_status`, skip to 8.7.5. **Do not abort the run** — it is shipped.

**8.7.1 — Pre-merge state commit:**
```bash
jq --argjson gates "$(build_gates_json)" \
   '.merge_status = "pending_merge" | .merge_gates_evaluated = $gates | .last_update = "'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'"' \
   "docs/autopilot-runs/${RUN_ID}.json" > tmp && mv tmp "docs/autopilot-runs/${RUN_ID}.json"
git add "docs/autopilot-runs/${RUN_ID}.json"
git commit -m "chore(autopilot): merge gates passed, merging"
git push origin "$BRANCH"
```

**8.7.2 — Method check:**
```bash
REPO_MERGE=$(gh repo view --json mergeCommitAllowed,squashMergeAllowed,rebaseMergeAllowed)
if ! echo "$REPO_MERGE" | jq -e '.rebaseMergeAllowed' >/dev/null; then
  merge_status="skipped_rebase_not_allowed"
  # jump to 8.7.5
fi
```

No fallback to squash/merge — silent method switching would surprise the repo owner.

**8.7.3 — Merge:**
```bash
if gh pr merge "$PR_URL" --rebase; then
  MERGE_SHA=$(gh pr view "$PR_URL" --json mergeCommit --jq '.mergeCommit.oid')
  merge_status="merged"
else
  merge_status="merge_command_failed"
fi
```

**8.7.4 — Post-merge state record:**

The remote branch is intentionally retained as an evidence branch. After any
merge outcome, commit the state update and push `origin "$BRANCH"` so the
archived state file is inspectable even after the PR is merged.

```bash
jq --arg ms "$merge_status" \
   --arg sha "${MERGE_SHA:-}" \
   --arg now "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
   '.merge_status = $ms | .merge_sha = (if $sha == "" then null else $sha end) | .merged = ($ms == "merged") | .last_update = $now' \
   "docs/autopilot-runs/${RUN_ID}.json" > tmp && mv tmp "docs/autopilot-runs/${RUN_ID}.json"
git add "docs/autopilot-runs/${RUN_ID}.json"
git commit -m "chore(autopilot): merge outcome: ${merge_status}"
git push origin "$BRANCH"
```

**8.7.5 — Record outcome when short-circuited:**

Reached when any gate failed, wall-clock was exhausted, or method check failed. Same jq recipe as 8.7.4, except `MERGE_SHA` is empty (so `merge_sha=null`) and `merged=false`. Push the branch so the human can inspect.

Proceed to Step 8.8.

### Step 8.8: Hub ship verification (conditional)

Runs only when all of these are true:

- `state.input.full_mode == true`
- `state.merged == true`
- `state.input.require_deploy == true` OR `state.input.target_env != "local"`

If any condition is false, set:
- `delivery_status = "pr_open"` when `merged == false`
- `delivery_status = "merged_unverified"` when `merged == true` but deploy verification is not required

Commit and push that archived state update to `origin "$BRANCH"`, then proceed
to the final report.

When the conditions are true, invoke `/hub-ship` in hub autopilot mode:

```json
{
  "mode": "autopilot",
  "branch": "<BRANCH>",
  "base_branch": "<base_branch>",
  "environment": "<state.input.target_env>",
  "pr_url": "<PR_URL>",
  "changed_files": ["<git diff base_branch...HEAD files captured before merge>"],
  "quality_gate": "<state.quality_gate>",
  "review_verdict": "<state.review_verdict>",
  "deploy_only": true,
  "require_deploy": true,
  "require_post_deploy_e2e": true
}
```

`hub-ship` must parse and return `HUB-SHIP-RESULT`. Copy the relevant
fields into the archived autopilot state:

- `ship_result`
- `deploy_status`
- `deploy_result`
- `post_deploy_e2e`
- `frontend_e2e_result`
- `delivery_status`

Mapping:

| `HUB-SHIP-RESULT.ship_status` | Autopilot `delivery_status` |
|---|---|
| `verified` | `verified` |
| `partial` | `merged_unverified` |
| `failed` | `deploy_failed` |
| `blocked` | `deploy_blocked` |

Commit the updated archived state file and push `origin "$BRANCH"`. If
`delivery_status != "verified"`, the run is still `status: shipped` because
the PR was opened/merged, but callers must not mark the ticket Done unless they
explicitly accept unverified delivery.

Proceed to the final report section (`## Output → Final report`).

## Escape Hatches

Hard rules. When any of these triggers, the skill commits whatever artefacts exist, archives the state file, pushes the branch, and returns the final report to the caller. Hatches marked "non-fatal" report but do not abort the remainder of the run.

| # | Trigger | Detection | `stuck_reason` | Fatal? |
|---|---|---|---|---|
| 1 | Protected branch at bootstrap | `current_branch ∈ {main, master, develop, production, staging}` | `protected_branch` | yes |
| 2 | Unstashable dirty tree | `git stash push -u` fails | `dirty_tree` | yes |
| 3 | Missing dependencies | `gh auth status` fails | `missing_deps` | yes |
| 4 | Empty spec after brainstorming | `wc -c < spec_path < 500` or `grep -c '^## ' spec_path == 0` | `empty_spec` | yes |
| 5 | Empty plan after writing-plans | `grep -c '^- \[ \]' plan_path == 0` | `empty_plan` | yes |
| 6 | Spec self-review fails twice | second failure | `spec_review_failed` | yes |
| 7 | Implementer asks a 4th question in one task | `budgets.implementer_answers_given >= 3` at the moment the 4th question arrives (counter increments on each answer given, so after 3 answers it equals 3 and the next question trips) | `implementer_overloaded` | yes |
| 8 | Subagent returns `blocked` a 2nd time | `budgets.test_retries_used >= 1` at the moment the 2nd `blocked` arrives (counter increments on retry, so after the one allowed retry it equals 1 and the next block trips) | `subagent_blocked` | yes |
| 9 | Wall-clock exceeded | `(now - started) > max_wall_clock_min * 60` checked at every step entry | `timeout` | yes |
| 10 | `gh pr create` fails | non-zero exit | `pr_creation_failed` | yes |
| 11 | Fresh-agent review dispatch fails | Agent tool error | `review_dispatch_failed` | **no** — PR already exists, record `review_verdict=not-dispatched` and continue |
| 12 | Step stuck in a loop with no progress | `step_history` shows the same `step` entered >2 times AND no new `artefacts` AND no new `decisions` since the last entry | `loop_detected` | **no** — skip the stuck step; see Non-fatal procedure for the escalation rule |
| 13 | `/hub-aware` fails twice on the same artefact | non-zero exit twice in a row on the same path | `hub_aware_failed` | yes |
| 14 | Full mode: CI timeout waiting for green | `gh pr checks --watch` exceeds `ci_timeout` (derived from remaining wall-clock, capped at 900s) | `merge_status=skipped_ci_timeout` | **no** — ship without merge |
| 15 | Full mode: `gh pr merge` returns non-zero | non-zero exit from the merge command | `merge_status=merge_command_failed` | **no** — PR stays open with approve verdict |
| 16 | Full mode: repo does not allow rebase merge | `gh repo view --json rebaseMergeAllowed` returns `false` | `merge_status=skipped_rebase_not_allowed` | **no** — ship without merge |
| 17 | Mandatory step self-skipped | Any of Step 2/3/4/6/7 (when `quality_gate != "pass"`)/8.4 was logged as `skipped` or `inlined` against the strict-compliance rules in `## Strict compliance — what you cannot self-justify`, OR `state.review_verdict ∈ {"approve","request-changes","comment"}` was written without a successful Agent-tool dispatch in `step_history` | `mandatory_step_skipped` | yes |
| 18 | Fabricated `run_id` | `state.run_id` does not match the format `<UTC timestamp>-<PID>-<hex>` produced by the bootstrap command | `fabricated_run_id` | yes |

**Abort procedure (fatal):**

1. Set `state.stuck_reason`, `state.last_update`.
2. Commit: `chore(autopilot): stuck — <reason>`.
3. Archive: `git mv docs/autopilot-state.json docs/autopilot-runs/<run_id>.json`; commit.
4. `git push -u origin <branch>` — branch stays for inspection. **No PR.**
5. Return the final report (see `## Output`) with the reason and the archived file path.

**Non-fatal procedure (hatches 11, 12, 14, 15, 16):**

Write `state.step_history[-1].warning = "<problem description>"`. At the end of the pipeline, the final report lists these warnings so the caller surfaces them.

Hatch 11 specifically: set `state.review_verdict = "not-dispatched"` and continue to the final report.

Hatch 12 specifically (loop detected): skip the offending step and advance to the next pipeline stage. Record `state.step_history[-1].status = "skipped"` with the warning text. **Escalation rule:** if the skipped step is a producer of an artefact the next step requires (no spec after Step 1, no plan after Step 3, no code changes after Step 5), escalate to fatal — flip the hatch to `stuck_reason=loop_detected` and execute the Abort Procedure. The skip-and-continue path only applies when a later step can compensate.

**Loop detection heuristic.** Treat "no progress" as: the last two `step_history` entries are the same `step` **and** no new files were added to `artefacts` **and** no new entries were added to `decisions`. This avoids false positives when a step is legitimately re-entered after a retry.

Hatches 14–16 specifically (full-mode failures): these never alter `stuck_reason`. They set `state.merge_status` to the appropriate skipped/failed value, push the branch (since the merge did not complete and the branch still exists remotely), and let the pipeline fall through to the final report with `status: shipped`. The PR stays open for the human to merge manually.

**Review-autofix invariant (no new escape hatch).** Step 8.5 is capped by `budgets.review_fix_cycles_used` (max 1), mirroring `hub-fix`. It never loops, never re-runs `hub-scan`, and never escalates severity. If the second review after an autofix still returns `request-changes`, the PR is left open with the updated verdict and the human merges after inspecting — this is *not* a stuck condition. Anything the reviewer flagged as severity ≥ `moderate` is never auto-fixed.

## Output

### PR body additions

After `ship-with-review` builds its standard PR body, append these three sections. Include the **optional fourth section** (`## Autopilot review autofix`) only when `budgets.review_fix_cycles_used >= 1`.

```markdown
## Autopilot run
- Run ID: `<run_id>`
- State file: `docs/autopilot-runs/<run_id>.json`
- Branch created from: `<base_branch>`
- Wall-clock: `<minutes>m`
- Ticket goal: `<input.goal>`
- Goal satisfied: `true | false | n/a`
- Success criteria satisfied: `true | false | n/a`
- Scope boundaries respected: `true | false | n/a`
- Quality gate: `pass | fail`
- Stuck reason: `<none | reason>`

## Key autopilot decisions
<top 3–5 entries from state.decisions, formatted as:
 - **<gate>** → chose **<chosen>** (confidence: <c>). <rationale>>

## What to double-check
<bullets auto-generated from state:
 - every decision with confidence: low
 - every task that retried a test
 - every domain with grade < B
 - every scope_adjustment (narrowed scope)
 - every review_autofix decision (autopilot applied a fix the reviewer flagged)
 - any success criterion or scope boundary that is not clearly satisfied>
```

Optional, append only when the autofix cycle ran:

```markdown
## Autopilot review autofix
- Cycles used: `<N>/1`
- Prior verdict: `<previous verdict>` → current verdict: `<current verdict>`
- Applied fixes:
  - `<finding title>` at `<location>` — <one-line action>
- Unresolved findings (not auto-fixable, left for human review):
  - `<severity>` — `<finding title>` at `<location>` — <description, 1 line>
```

Optional, append only when `input.full_mode == true`:

```markdown
## Autopilot full mode
- Merge status: `<merged | skipped_* | merge_command_failed>`
- Merge SHA: `<sha | n/a>`
- Delivery status: `<verified | merged_unverified | deploy_failed | deploy_blocked | pr_open>`
- Deploy status: `<verified | failed | timeout | unconfigured | skipped>`
- Post-deploy E2E: `<pass | fail | skipped | unconfigured>`
- Gates evaluated:
  - G1 review approve + ticket contract: ✅ / ❌ / —
  - G2 quality pass: ✅ / ❌ / —
  - G3 no unresolved findings: ✅ / ❌ / —
  - G4 CI green: ✅ / ❌ / — (ci_status: present | absent | timeout | n/a)
```

Where `—` means "not evaluated" (short-circuited by a prior gate failure or by wall-clock timeout).

The "Key autopilot decisions" selection rule: prefer (a) approach_choice, (b) scope_adjustment entries, (c) any decision with confidence: low, (d) any `review_autofix` entries, (e) fill up to five from the most recent decisions if fewer than five are found above.

### Fresh-agent review prompt — autopilot context block

Append this block to the existing `ship-with-review` Step 4 prompt template, **before** the `## Your task` line. The outer fence below is `~~~` so the inner ```json block can be rendered verbatim in the prompt:

~~~
## Context: this PR was produced by autopilot
- No human guided brainstorming or reviewed the spec/plan.
- Decisions log (from docs/autopilot-runs/<run_id>.json):

<full JSON array state.decisions, pretty-printed>

- Escape hatches that did NOT trigger — verify the AI's judgment on:
  (a) approach choice — was the chosen option really the best given constraints?
  (b) scope boundaries — did execution respect them?
  (c) any decision marked confidence: low
- State file (same run): docs/autopilot-runs/<run_id>.json

Be especially rigorous: the usual human gate was bypassed.
Your verdict replaces it.

Also verify the ticket contract:
- Is the ticket goal satisfied by the diff?
- Are all success criteria satisfied?
- Were scope boundaries respected?

## Required output format
End your response with a single fenced JSON block that the autopilot parses:

```json
{
  "verdict": "approve | request-changes | comment",
  "ticket_goal_satisfied": true,
  "success_criteria_satisfied": true,
  "scope_boundaries_respected": true,
  "findings": [
    {
      "severity":      "nit | minor | moderate | major | blocker",
      "category":      "bug | style | naming | edge-case | perf | security | docs | scope | architecture | other",
      "location":      "path/to/file.ext:LINE",
      "description":   "what is wrong, one or two sentences",
      "suggested_fix": "concrete fix idea in one paragraph (what to change, not prose rationale)",
      "auto_fixable":  true
    }
  ]
}
```

Rules for the JSON block:
- If the PR is clean, `verdict` is `approve` and `findings` is `[]`.
- `ticket_goal_satisfied`, `success_criteria_satisfied`, and
  `scope_boundaries_respected` must be boolean values. If any is false, return
  at least one finding explaining the gap.
- Set `auto_fixable: true` ONLY when ALL of these hold:
  (1) severity is `nit` or `minor`,
  (2) the fix is confined to files already in this PR's diff,
  (3) the fix needs no new deps / new files / API changes / spec or plan edits,
  (4) it fits in a single commit (no structural refactor).
  Otherwise set `auto_fixable: false`.
- Autopilot will re-classify `auto_fixable` against the same rules and may override your value. Your job is to be accurate, not lenient.
- Anything severity >= moderate is always `auto_fixable: false` — human judgment required.

## Context: autopilot may apply your low-risk fixes and re-request review
If you mark `request-changes` with one or more `auto_fixable: true` findings, autopilot will apply them in a single commit (one cycle max), push, and ask you to re-review. On the second pass you will see a note like "Previous review requested changes; autopilot applied N low-risk fixes in commit <sha>". Evaluate the result normally. If remaining issues exist, return them in `findings` with accurate severities — autopilot will NOT run a second autofix cycle; anything left over is handed to the human.
~~~

### Final report (returned to the caller)

```
hub-driven-autopilot: done
  Run ID: <run_id>
  Branch: <branch>
  Duration: <Xm>
  Status: shipped | stuck

  Spec: <artefacts.spec_path | none>
  Plan: <artefacts.plan_path | none>
  PR:   <pr_url | none>
  Review verdict: <approve | request-changes | comment | skipped | not-dispatched>
  Review autofix: <N applied, M unresolved | none>
  Ticket goal satisfied: <true | false | n/a>
  Success criteria satisfied: <true | false | n/a>
  Scope boundaries respected: <true | false | n/a>
  Quality gate:   <pass | fail | n/a>
  Full mode:      <enabled | disabled>
  Merge:          <merged at <sha> | skipped: <reason> | merge_command_failed | n/a>
  Delivery:       <verified | merged_unverified | deploy_failed | deploy_blocked | pr_open | n/a>
  Deploy:         <verified | failed | timeout | unconfigured | skipped | n/a>
  Post-deploy E2E:<pass | fail | skipped | unconfigured | n/a>
  Stuck reason:   <none | reason>

  Autopilot decisions: <len(state.decisions)> logged
  Budgets used: hub-fix=<X>/1, test-retries=<Y>/1, implementer-answers=<Z>/3, review-fix=<W>/1
  Warnings: <count from non-fatal hatches>

  Non-fatal warnings (if any):
    - <step>: <warning text>

  Unresolved review findings (if any — left for human merger):
    - <severity> — <title> at <location>

  State file: <archived path>
```

The `Full mode` line shows `disabled` (and `Merge: n/a`) when `input.full_mode == false`, to keep the report shape constant.

If `status == "stuck"`, add a final line:
```
  Next step: read the state file, identify the gap, re-invoke with refined input.
```

### Machine-readable return (for calling agents)

After the human-readable report, also emit this JSON block — this is the contract any dispatcher parses. The dispatched subagent must return this block verbatim as the last thing in its Agent-tool response.

```json
{
  "status": "shipped | stuck",
  "run_id": "<run_id>",
  "branch": "<branch>",
  "pr_url": "<url | null>",
  "review_verdict": "approve | request-changes | comment | skipped | not-dispatched | null",
  "ticket_goal_satisfied": true,
  "success_criteria_satisfied": true,
  "scope_boundaries_respected": true,
  "quality_gate": "pass | fail | null",
  "stuck_reason": "<one of the escape hatch reasons | null>",
  "state_file": "docs/autopilot-runs/<run_id>.json",
  "decisions_count": 0,
  "budgets": {
    "hub_fix_cycles_used": 0,
    "test_retries_used": 0,
    "implementer_answers_given": 0,
    "review_fix_cycles_used": 0
  },
  "review_autofixes_applied": 0,
  "review_findings_unresolved": [
    {"severity": "moderate", "title": "...", "location": "path:line"}
  ],
  "warnings": ["<step>: <warning text>", "..."],
  "full_mode": false,
  "merged": false,
  "merge_sha": null,
  "merge_status": null,
  "merge_gates": {
    "G1_review_approve": null,
    "G2_quality_pass":   null,
    "G3_no_unresolved":  null,
    "G4_ci_green":       null,
    "ci_status":         null
  },
  "delivery_status": "verified | merged_unverified | deploy_failed | deploy_blocked | pr_open | null",
  "deploy_status": "verified | failed | timeout | unconfigured | skipped | null",
  "deploy_result": null,
  "post_deploy_e2e": "pass | fail | skipped | unconfigured | null",
  "frontend_e2e_result": null,
  "ship_result": null
}
```

Fields that do not apply (e.g., `pr_url` on a stuck run, `stuck_reason` on a shipped run, `merge_sha` when `merged == false`) MUST be `null`, not omitted. This keeps the parser trivial for the caller. `review_autofixes_applied` is always a non-negative integer; `review_findings_unresolved` is always an array (possibly empty). When `input.full_mode == false`, `merge_status` is `null`, `merged` is `false`, all `merge_gates.*` values are `null`, and delivery fields are `null` unless a caller explicitly requested deploy verification.

## Integration Notes

### /schedule

For unattended runs, `/schedule` fires this skill (or a caller that invokes it). The state file and archived runs make it safe to run unattended: every failure leaves debug evidence on the autopilot branch without touching shared branches.

### hub-driven (interactive sibling)

This skill **does not delegate** to `hub-driven`. Both skills invoke the same sub-skills directly. The difference is the preamble passed to those sub-skills and the deterministic policy at each gate.

## Important Rules

- **Log every non-deterministic decision.** Every time the AI picks among options (clarifying question, approach, scope narrowing, implementer answer), append to `state.decisions` before proceeding. The PR body and review prompt rely on this record.
- **Ticket goal is the success contract.** A shipped run is not complete unless
  `ticket_goal_satisfied`, `success_criteria_satisfied`, and
  `scope_boundaries_respected` are all true. Tests and deploy are necessary but
  not sufficient.
- **Never delegate to `hub-driven`.** Invoke sub-skills directly with the autopilot preamble. Delegating to an interactive orchestrator defeats the autopilot contract.
- **Never skip ship-with-review's fresh-agent review** unless hatch 11 fires after an actual Agent-tool dispatch failure. Inline self-review masquerading as a fresh-agent verdict trips fatal hatch 17 (`mandatory_step_skipped`). The review verdict replaces the missing human gate; faking it voids the contract.
- **Mandatory pipeline steps cannot be self-skipped.** Steps 2, 3, 4, 6, 7 (when quality_gate is not pass), and 8.4 are mandatory. Justifications like "narrow fix", "surgical change", "spec already aware", "Agent tool unavailable in subagent context" are not allowed degraded modes — they are hatch 17. See `## Strict compliance — what you cannot self-justify` for the full list.
- **`run_id` must come from the bootstrap command, not from memory.** The bootstrap RUN_ID line includes `$$` (PID) so parallel autopilots cannot collide. A `run_id` that does not match the `<timestamp>-<PID>-<hex>` shape trips fatal hatch 18 (`fabricated_run_id`).
- **Always open a PR on success**, regardless of `quality_gate`. The reviewer agent sees the grades and decides.
- **Merge only under `full_mode: true`, only when all 4 gates pass (approve verdict + ticket contract, quality pass, no unresolved findings, CI green; CI absence is allowed only for local targets).** Without `full_mode`, the skill opens and reviews; merging is a human decision. Never merge from any path other than Step 8.7. Use `gh pr merge --rebase`; keep the autopilot branch as the evidence branch until delivery state has been pushed. Skip (do not fall back) if the repo blocks rebase.
- **Development delivery requires deploy evidence.** For non-local `target_env`,
  CI absence is not green, and a merged PR is not considered delivered until
  Step 8.8 returns `delivery_status=verified`.
- **Full mode never escalates severity.** Step 8.5 autofix rules are unchanged — 1 cycle max, only nit/minor, in-diff, no spec/plan/arch changes. Full mode does NOT relax them to force a merge.
- **Never skip hooks, never force-push.** Inherited from `ship-with-review`.
- **Max 1 hub-fix cycle.** Do not loop — that is `/hub-loop`'s job. If `quality_gate == "fail"` after one cycle, ship anyway and let the review flag concerns.
- **Max 1 review-autofix cycle.** Apply only findings that pass the Step 8.5 low-risk rules (nit/minor, in-diff, no new deps, no spec/plan/arch changes, single commit). Re-dispatch the reviewer exactly once to confirm. Never run a second autofix cycle. Anything severity ≥ `moderate` is never auto-fixed — that is always a human decision. Do NOT re-run `hub-scan` after autofix.
- **Commit state after every transition.** A crash must not lose the decision log.
- **Branch stays on stuck.** Push the branch even on failure so humans can inspect locally with `git checkout autopilot/...`.
- **Do not store secrets in state.** `input.description` is assumed non-sensitive.
