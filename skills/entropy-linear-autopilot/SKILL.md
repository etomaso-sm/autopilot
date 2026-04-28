---
name: entropy-linear-autopilot
description: >
  Use when Linear tickets in a configured project should be processed
  automatically through entropy workflows, especially scheduled ticket queues or
  autopilot batches. Triggers on: entropy-linear-autopilot, process tickets,
  autopilot, auto tickets, procesar tickets, tickets automáticos.
user_invocable: true
---

# Entropy Linear Autopilot

**Announce at start:** "Running /entropy-linear-autopilot — fetching and processing Linear tickets through quality pipelines."

## Overview

Fetches tickets in a configured status from a Linear project, classifies each
as bug or feature, and dispatches parallel agents to process them through
`/entropy-bugfix` (bugs) or `/entropy-driven-autopilot` (features). Updates
ticket status in Linear as processing progresses.

## Prerequisites

- Linear MCP server connected and accessible via OAuth (see `/linear` skill for setup)
- `entropy-linear-autopilot.json` at repo root with project configuration
- Access to `/entropy-driven-autopilot` and `/entropy-bugfix` skills

## Phase 0: Setup

1. Read `entropy-linear-autopilot.json` from the repo root.
   - If missing, stop and report:
     > "No `entropy-linear-autopilot.json` found at repo root. Create one with at minimum: `{ "project": "<your-project-name>" }`."
   - Apply defaults for any missing fields:
     - `sourceStatus`: `"Todo"`
     - `inProgressStatus`: `"In Progress"`
     - `doneStatus`: `"Done"`
     - `bugLabels`: `["bug", "bugfix", "fix"]`
     - `maxTicketsPerRun`: `3`
     - `baseBranch`: `"development"`
     - `targetEnv`: `"development"`
     - `fullMode`: `true`
     - `requireDeploymentVerification`: `true`
     - `schedule`: `null`
     - `labels`:
       ```json
       {
         "dispatched": "autopilot",
         "shipped": "autopilot:shipped",
         "stuck": "autopilot:stuck"
       }
       ```
     Disabling rules:
     - If `labels` is **omitted** from config, the full default object above is
       applied (labels are ON).
     - If `labels` is **explicitly `null`** in config, skip the entire label
       flow — no auto-create, no attach attempts, no log lines about labels.
     - If any individual sub-key (`dispatched`/`shipped`/`stuck`) is `null`
       or missing, skip that specific label while keeping the others.

2. Authenticate Linear MCP if needed. If a Linear MCP call fails with an auth
   error, call `mcp__claude_ai_Linear__authenticate` and retry once.

## Phase 0.5: Ensure labels exist (once per run)

2a. If `labels` config is `null`, skip this phase entirely.

2b. Otherwise, for each non-null `labels.{dispatched|shipped|stuck}` value,
    check whether the label exists in the Linear workspace/team that owns
    the configured `project`. Use a natural-language MCP query:

    > "List labels for project {project} matching name '{labelName}'"

    - If it exists → record its ID in an in-memory map `LABEL_IDS[kind] = id`.
    - If it does not exist → create it:

      > "Create label '{labelName}' in the workspace/team for project {project}"

      On success, record the returned ID. On failure (permissions, auth,
      race with another run), log a warning:

      > "autopilot: could not ensure label '{labelName}' ({error}); continuing without it"

      and set `LABEL_IDS[kind] = null` so downstream steps skip it without
      aborting. **This phase is best-effort and never blocks dispatch.**

2c. Log the resolved map using label **names** (readable in scheduler logs),
    falling back to `"disabled"` when a label was null/missing or failed to
    resolve:
    > "Labels ready: dispatched={labels.dispatched or "disabled"}, shipped={labels.shipped or "disabled"}, stuck={labels.stuck or "disabled"}"

## Phase 1: Fetch

3. Call `list_issues` to fetch tickets matching the configured project and
   `sourceStatus`. Use a natural-language query:

   > "List issues in project {project} with status {sourceStatus}"

4. If no tickets are returned, report:
   > "No tickets in '{sourceStatus}' status for project '{project}'. Nothing to process."
   Then skip to Phase 5 (Schedule Offer).

5. Sort tickets by priority (urgent → high → medium → low → none).
   Take the first `maxTicketsPerRun` tickets. Log how many were found vs. how
   many will be processed:
   > "Found {total} tickets in '{sourceStatus}'. Processing top {count} by priority."

## Phase 2: Classify

6. For each ticket, determine if it is a **bug** or **feature**:

   **Step A — Label check:**
   Check the ticket's labels against the `bugLabels` array from config.
   If any label matches (case-insensitive), classify as **bug**.

   **Step B — Content fallback:**
   If no label matched, read the ticket title and description. Classify based on:
   - **Bug signals:** error, broken, regression, crash, fails, not working, fix,
     incorrect, wrong, unexpected, 500, timeout, exception
   - **Feature signals:** add, implement, create, new, support, enable, build,
     design, integrate, migrate

   If ambiguous or unclear, default to **feature** (entropy-driven is the safer
   pipeline for unknowns — it includes brainstorming to clarify scope).

  **Step C — Entropy input block:**
   Check the issue description for this section:
   ~~~markdown
   ## Entropy autopilot input
   ```json
   { ... }
   ```
   ~~~
   If present and valid JSON, store it as `ENTROPY_INPUT`. It is the source of
   truth for feature dispatch fields, including `goal`. Config defaults fill
   only missing fields.
   If invalid, post a warning comment and fall back to `{title}\n\n{description}`.

7. Move all selected tickets to `inProgressStatus` via `update_issue`:

   For each ticket:
   > "Update issue {ticket-id} status to {inProgressStatus}"

7a. If `LABEL_IDS.dispatched` is non-null, attach it to each selected ticket:

   > "Add label '{labels.dispatched}' to issue {ticket-id}"

   This marks the ticket as "touched by autopilot" regardless of final
   outcome, so a Linear view filtered by this label shows everything the
   autopilot has processed. On per-ticket add failure, log a warning and
   continue — never abort the run.

8. Log the classification:
   > "Ticket classification:
   > - {TICKET-ID}: {title} → bug
   > - {TICKET-ID}: {title} → feature
   > - {TICKET-ID}: {title} → feature"

## Phase 3: Dispatch

9. Use `dispatching-parallel-agents` to launch one agent per ticket.

   Each agent is dispatched via the `Agent` tool with a self-contained prompt.
   Agents do NOT inherit this session's context.

   **For bug tickets**, the agent prompt is:

   ```
   You are processing a Linear ticket as a bug fix.

   ## Ticket
   - ID: {ticket-id}
   - Title: {title}
   - Description: {description}

   ## Instructions
   1. Invoke `/entropy-bugfix` with the ticket description above as the bug report.
   2. Let entropy-bugfix run its full pipeline: root cause investigation,
      failing repro, minimal fix, risk classification, entropy gate.
   3. When creating a PR (via entropy-bugfix's closeout or ship-with-review),
      include "Closes {ticket-id}" in the PR body.
   4. Return a summary: root cause found, files changed, PR URL (if created),
      or error description if something failed.
   ```

   **For feature tickets**, the agent prompt is:

   ```
   You are processing a Linear ticket as a new feature via autopilot.

   ## Ticket
   - ID: {ticket-id}
   - Title: {title}
   - Description: {description}

   ## Instructions
   1. Invoke `/entropy-driven-autopilot` with this structured JSON. If the
      Linear description contains a valid `## Entropy autopilot input` JSON
      block, use that object first and fill missing fields from the config.
      Otherwise use the fallback below:
      {
        "goal": "{title}",
        "description": "{title}\n\n{description}",
        "ticket_id": "{ticket-id}",
        "base_branch": "{config.baseBranch}",
        "target_env": "{config.targetEnv}",
        "full_mode": {config.fullMode},
        "require_deploy": {config.requireDeploymentVerification}
      }
   2. Let it run its full autonomous pipeline. It will create a branch,
      write a spec and plan, implement, test, open a PR, dispatch a
      fresh-agent code review, optionally merge, and when configured verify
      the development deploy through the entropy ship gates — or stop with a
      `stuck_reason` if an escape hatch triggers.
   3. Return the Machine-readable JSON block defined in
      `skills/entropy-driven-autopilot/SKILL.md → ## Output → Machine-readable
      return`. That block is the canonical contract; entropy-linear-autopilot parses
      `status`, `pr_url`, `review_verdict`, `stuck_reason`, `state_file`,
      `ticket_goal_satisfied`, `success_criteria_satisfied`,
      `scope_boundaries_respected`,
      `delivery_status`, `deploy_status`, `post_deploy_e2e`, `merged`, and
      `merge_status`. Do NOT emit a different shape.
   ```

   Dispatch all agents in a single message with multiple `Agent` tool calls
   so they run in parallel.

10. Wait for all agents to complete. Collect their return summaries.

## Phase 4: Close

11. Parse each agent's return. Features emit the canonical JSON contract
    (see `skills/entropy-driven-autopilot/SKILL.md → ## Output → Machine-readable
    return`); bugs still emit a free-text summary from `/entropy-bugfix`.

    **Feature tickets — fully verified**:
    Condition: `status: shipped`, `delivery_status: verified`,
    `ticket_goal_satisfied: true`, `success_criteria_satisfied: true`, and
    `scope_boundaries_respected: true`.

    - Call `update_issue` to move the ticket to `doneStatus`:
      > "Update issue {ticket-id} status to {doneStatus}"
    - Record the PR URL and delivery status in the final report row.
    - If `LABEL_IDS.shipped` is non-null, attach it:
      > "Add label '{labels.shipped}' to issue {ticket-id}"

    **Feature tickets — `status: shipped` but not fully verified**
    (`delivery_status` is `merged_unverified`, `deploy_failed`,
    `deploy_blocked`, `pr_open`, or `null`, or any ticket-contract boolean is
    false):
    - Leave the ticket in `inProgressStatus`.
    - Record the PR URL in the final report row.
    - Post a comment:
      > "Autopilot produced a PR but did not fully verify completion.
      > PR: {pr_url}
      > Merge status: {merge_status}
      > Goal satisfied: {ticket_goal_satisfied}
      > Success criteria satisfied: {success_criteria_satisfied}
      > Scope boundaries respected: {scope_boundaries_respected}
      > Delivery status: {delivery_status}
      > Deploy status: {deploy_status}
      > Post-deploy E2E: {post_deploy_e2e}
      > State file: {state_file}"
    - If `review_verdict == "not-dispatched"`, include:
      > "Fresh-agent review failed to dispatch; please review manually or
      > re-invoke `/ship-with-review`."
    - If `LABEL_IDS.stuck` is non-null, attach it. The work is not Done until
      delivery is verified.

    **Feature tickets — `status: stuck`** (autopilot aborted before opening
    a PR):
    - Call `create_comment` with:
      > "Autopilot stopped before opening a PR.
      > Reason: {stuck_reason}
      > State file (on branch {branch}): {state_file}
      > Re-invoke with refined input after inspecting the state file."
    - Leave the ticket in `inProgressStatus`.
    - If `LABEL_IDS.stuck` is non-null, attach it:
      > "Add label '{labels.stuck}' to issue {ticket-id}"

    **Bug tickets — success** (free-text from `/entropy-bugfix` that includes
    a PR URL or "no fix needed"):
    - Move to `doneStatus`. Record the PR URL if present.
    - If `LABEL_IDS.shipped` is non-null, attach it.

    **Any other return** (unexpected JSON shape, exception, no return, or a
    bug-ticket agent reporting an error):
    - Call `create_comment` with the raw return text:
      > "Add comment to issue {ticket-id}: Autopilot processing failed. Error: {error-summary}"
    - Leave the ticket in `inProgressStatus`.
    - If `LABEL_IDS.stuck` is non-null, attach it (the ticket did not ship).

    Per-ticket label-attach failures log a warning and continue — never
    override or block the primary Phase-4 status transitions.

12. Print the final report:

    ```
    entropy-linear-autopilot: run complete

    | Ticket | Title | Type | Status | PR | Delivery | Deploy | Stuck reason |
    |--------|-------|------|--------|-----|----------|--------|--------------|
    | {ID}   | {title} | feature | done | {PR-URL} | verified | verified | — |
    | {ID}   | {title} | feature | in-progress | {PR-URL} | deploy_failed | failed | — |
    | {ID}   | {title} | feature | stuck | — | — | — | empty_spec |
    | {ID}   | {title} | bug | shipped | {PR-URL} | n/a | n/a | — |

    Processed: {N} tickets
    Success: {M}
    Errors: {E}
    Remaining in '{sourceStatus}': {R}

    Labels applied this run:
      dispatched: {labels.dispatched or "disabled"} → {N_dispatched_attached} tickets
      shipped:    {labels.shipped or "disabled"}    → {N_shipped_attached} tickets
      stuck:      {labels.stuck or "disabled"}      → {N_stuck_attached} tickets
    Label warnings: {N_label_warnings}
    ```

## Phase 5: Schedule Offer

13. If `schedule` in config is `null`, offer:
    > "Want to set up a schedule to run `/entropy-linear-autopilot` automatically?
    > This uses `/schedule` to create a recurring trigger. Suggested: every 30 minutes.
    >
    > 1. **Yes, every 30 minutes**
    > 2. **Yes, custom interval** (you tell me)
    > 3. **No, I'll run it manually**"

14. If the user accepts:
    - Invoke `/schedule` to create the trigger with the chosen frequency.
    - Update `entropy-linear-autopilot.json` to set `schedule` to the chosen value
      (e.g., `"every 30 minutes"`).
    - Commit the config change.

15. If the user declines or if `schedule` is already set, skip silently.

## Safety Guards

- **Max tickets per run**: never process more than `maxTicketsPerRun` tickets
  in a single invocation, even if more are available.
- **Skip in-progress**: only pick up tickets in `sourceStatus`. Tickets already
  in `inProgressStatus` were either started by a previous run or manually — do
  not touch them.
- **No-op on empty**: if no tickets match, exit after the report. Do not create
  placeholder work.
- **Error isolation**: each agent runs independently. One failure does not affect
  others and does not abort the run.
- **PR traceability**: every PR created by an agent must include the Linear
  ticket identifier (e.g., `Closes IMP-123`) in its body.
- **Done means delivered**: feature tickets move to `doneStatus` only when
  `delivery_status == "verified"` and the ticket goal/success/scope booleans
  are all true. A PR that is merely opened, merged, or deployed but misses the
  ticket contract stays in progress.

## Common Mistakes

- Dispatching agents without moving tickets to `inProgressStatus` first (causes
  duplicate processing if the schedule fires again)
- Classifying ambiguous tickets as bugs when they should default to features
- Forgetting to pass the ticket ID to the agent prompt (breaks PR traceability)
- Moving a feature ticket to Done on `status: shipped` without checking
  `delivery_status: verified` and the ticket contract booleans
- Running without checking `maxTicketsPerRun` (launching too many agents)
- Trying to process tickets that are already in progress
