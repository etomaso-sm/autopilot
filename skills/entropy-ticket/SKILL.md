---
name: entropy-ticket
description: >
  Use when a rough idea needs code-aware discovery, goal definition, and one or
  more delivery-ready entropy tickets for local tracking, Linear, autopilot, or
  draft-only planning. Triggers on: entropy-ticket, entropy-intake, armar ticket,
  new ticket, nuevo ticket, draft ticket, crear ticket, idea to ticket.
user_invocable: true
---

# Entropy Ticket — Code-Aware Intake Gate

<!-- SYNC: entropy-ticket depends on:
     - entropy-driven-autopilot input schema:
       goal, description, constraints, success_criteria, scope_boundaries,
       ticket_id, base_branch, max_wall_clock_min, target_env, full_mode,
       require_deploy
     - entropy-linear-autopilot: reads Linear issue content and should preserve
       the "Entropy autopilot input" JSON block when present.
     - docs/TICKETS.md local tracker contract below.
     Update this skill when any input field, delivery status, or tracker field
     changes. -->

**Announce at start:** "Running /entropy-ticket — code-aware entropy intake and ticket shaping."

## Overview

Turn an idea into one or more ticket artifacts after inspecting the relevant
code. Even when the idea sounds clear, first perform code discovery: the code
decides whether the work is one ticket, several tickets, or a spike.

Ticket detail lives in:

```text
docs/tickets/YYYY-MM-DD-<slug>.md
```

Tracking can live in:

- `docs/TICKETS.md` (`tracker.source=local`)
- Linear (`tracker.source=linear`)
- both (`tracker.source=both`; Linear is the operational tracker,
  `docs/TICKETS.md` mirrors status)
- nowhere yet (`tracker.source=none`; draft only)

This skill does not implement or ship tickets. It creates the goal contract that
`/entropy-driven-autopilot --from-ticket ...` uses to judge success.

The ticket file is canonical for the entropy contract (`goal`,
`success_criteria`, `scope_boundaries`, and autopilot JSON). Linear and
`docs/TICKETS.md` are tracking surfaces for that contract.

## Entropy Contract

Required final marker:

```text
ENTROPY-TICKET-RESULT:
{
  "ticket_status": "ready | tracked | pushed | mirrored | aborted",
  "tracker": {
    "source": "local | linear | both | none",
    "path": "docs/TICKETS.md",
    "linear_project": "Impactia"
  },
  "tickets": [
    {
      "local_id": "ET-20260427-001",
      "path": "docs/tickets/2026-04-27-temperature-tuning.md",
      "linear_id": "IMP-123",
      "linear_url": "https://linear.app/...",
      "goal": "...",
      "autopilot_command": "/entropy-driven-autopilot --from-ticket docs/tickets/..."
    }
  ],
  "warnings": []
}
```

Fields that do not apply must be `null`, not omitted.

## Invocation

```text
/entropy-ticket <rough task description>
/entropy-ticket --local <rough task description>
/entropy-ticket --manual <rough task description>
/entropy-ticket --tracker=local|linear|both|none <rough task description>
```

Flags:

| Flag | Effect |
|---|---|
| `--local` | Sets `target_env=local`, `full_mode=false`, `require_deploy=false`. |
| `--manual` | Keeps deploy target but sets `full_mode=false`; a human runs/merges later. |
| `--full` | Sets `full_mode=true`, `require_deploy=true`, `target_env=development` unless the blob says production/staging. |
| `--tracker=<mode>` | Skips the tracker prompt. `local`, `linear`, `both`, or `none`. |

If no argument is given, ask: "¿Qué idea querés convertir en tickets?"

## Frontmatter Schema

Each generated ticket file must use this frontmatter:

```yaml
---
# Ticket metadata
id: ET-20260427-001
created: 2026-04-27
status: ready                 # draft | ready | in-progress | pr-open | verified | blocked | deferred
linear_url: null
priority: medium              # null | low | medium | high
kind: feature                 # feature | bug | chore | spike

# success contract
goal: >
  <clear outcome that defines success>
description: >
  <context and problem framing; not the full solution design>
constraints: []
success_criteria: []          # >= 1 required; objectively checkable
scope_boundaries: []

# code-aware intake evidence
code_discovery:
  searched:
    - "<rg query or path inspected>"
  touched_areas:
    - "<file or directory likely affected>"
  observations:
    - "<fact learned from code, not speculation>"
  open_questions: []

# entropy-driven-autopilot input
ticket_id: ET-20260427-001
base_branch: development
max_wall_clock_min: 60
target_env: development       # local | development | staging | production
full_mode: true
require_deploy: true

# test/deploy guidance
tests:
  unit: true
  visual: false
  e2e: false
  notes: null

delivery:
  expected_status: verified
  deploy_gate: entropy-check-deploy
  frontend_gate: entropy-e2e-frontend

tracker:
  source: local               # local | linear | both | none
  local_id: ET-20260427-001
  tracker_path: docs/TICKETS.md
  linear_id: null
  linear_url: null
---
```

## Workflow

### Step 1: Intake

Accept the freeform blob, strip supported flags, and refuse empty input.

Infer `kind`:
- Bug if labels/words include `bug`, `fix`, `broken`, `regression`, `error`,
  `crash`, `500`, `not working`.
- Chore if words include `chore`, `cleanup`, `docs`, `maintenance`.
- Spike if the user asks for analysis, scoping, feasibility, or "take your
  opinion" and implementation is not yet obviously bounded.
- Otherwise feature.

### Step 2: Code Discovery

Always inspect code before shaping tickets.

Use `rg`, `rg --files`, and targeted file reads to answer:

- Which files/routes/modules are likely involved?
- Is there one natural outcome or multiple separable outcomes?
- Is the request implementation-ready or does it need a spike first?
- What code facts should constrain brainstorming later?

Record only evidence:

- `searched`: queries and paths inspected.
- `touched_areas`: likely affected files/directories.
- `observations`: facts learned from code.
- `open_questions`: unknowns that brainstorming/spec should resolve.

Do not write solution architecture here. The ticket is allowed to name an
obvious local pattern, but approach selection belongs to brainstorming/spec.

### Step 3: Ticket Shaping

Decide the cut:

| Discovery result | Output |
|---|---|
| One bounded outcome | One implementation ticket. |
| Multiple independent outcomes | One ticket per outcome. |
| High uncertainty or broad retrofit | One spike ticket first, plus optional deferred implementation tickets. |
| Mixed small fix + larger retrofit | Ship the small fix ticket; create separate spike/retrofit tickets for the rest. |

For each ticket, define:

- `goal`: the outcome that determines whether autopilot succeeded.
- `success_criteria`: objective checks for the goal.
- `scope_boundaries`: what this ticket must not change.
- `code_discovery`: evidence from Step 2.

Goal is mandatory. Success criteria are mandatory. Do not guess them silently:
if code discovery cannot make the goal/checks concrete, ask the user.

### Step 4: IDs, Slugs, and Paths

Local IDs use the date plus a sequence from `docs/TICKETS.md`:

```text
ET-YYYYMMDD-001
```

If `docs/TICKETS.md` does not exist, start at `001`. Otherwise read the JSON
block and increment the highest existing sequence for today.

Slug from the ticket goal:

```bash
SLUG=$(printf '%s' "$GOAL" \
       | tr '[:upper:]' '[:lower:]' \
       | sed -E 's/[^a-z0-9]+/-/g; s/^-+//; s/-+$//' \
       | cut -c1-40)
DATE=$(date -u +%Y-%m-%d)
PATH_OUT="docs/tickets/${DATE}-${SLUG}.md"
mkdir -p docs/tickets
```

If a file exists, ask before overwriting. In non-interactive future batch mode,
append `-v2`, `-v3`, etc.

### Step 5: Inference Pass

Use these defaults unless the blob, code discovery, or flags clearly say
otherwise:

| Field | Inference |
|---|---|
| `base_branch` | `development` |
| `target_env` | `production` if prod words; `staging` if staging words; `local` if `--local`; otherwise `development`. |
| `full_mode` | `false` with `--manual` or `--local`; otherwise `true`. |
| `require_deploy` | `false` only when `target_env=local`; otherwise same as `full_mode` unless user says no deploy. |
| `max_wall_clock_min` | `60` |
| `tests.unit` | `true` |
| `tests.visual` | true for UI/layout/component/page/screen/design words or frontend files. |
| `tests.e2e` | true for flow/login/checkout/payment/onboarding/cross-component words. |
| `priority` | `high` for urgent/blocker/customer-down words; otherwise ask. |

### Step 6: Selective Interview

Ask only for fields that are missing or ambiguous. Keep questions short.

| Field | When to ask | Prompt |
|---|---|---|
| `goal` | Discovery cannot express a clear outcome | "Cuál es el goal exacto de este ticket?" |
| `success_criteria` | Always unless already objective from user/code | "Dame >= 1 success criterion (una linea por criterio; `.` para terminar)." |
| `constraints` | No clear constraint in blob/code | "Constraints tecnicas o de negocio? (una por linea; `.` para ninguna)." |
| `scope_boundaries` | Blob lacks explicit non-goals | "Algo que NO entra en este ticket? (una por linea; `.` para ninguno)." |
| `priority` | Not inferred | "Prioridad? low/medium/high/--" |
| `target_env/full_mode` | User requested autopilot but target unclear | "Debe cerrar en development deployado y verificado? y/n" |

Do not ask twice about the same field. If the user presses enter, keep the
inferred default unless the field is mandatory.

### Step 7: Validate

Before writing, validate each ticket:

- `goal` non-empty
- `description` non-empty
- `success_criteria` has at least one item
- `target_env in {local, development, staging, production}`
- `full_mode`, `require_deploy`, and `tests.*` are booleans
- if `target_env != local` and `full_mode=true`, then `require_deploy=true`
- `code_discovery.observations` has at least one item, unless the repo has no
  relevant code yet; then add a warning.
- if `kind=bug`, note that `entropy-linear-autopilot` routes it to `/entropy-bugfix`

If validation fails, re-ask only the offending field.

### Step 8: Build Ticket Files

Each body must include:

1. `## Goal`
2. `## Code discovery`
3. `## Success criteria`
4. `## Scope boundaries`
5. `## Entropy autopilot input`

The autopilot input block must be valid JSON:

````markdown
## Entropy autopilot input
```json
{
  "goal": "...",
  "description": "...",
  "constraints": [],
  "success_criteria": [],
  "scope_boundaries": [],
  "ticket_id": "ET-20260427-001",
  "base_branch": "development",
  "max_wall_clock_min": 60,
  "target_env": "development",
  "full_mode": true,
  "require_deploy": true
}
```
````

Preview the generated ticket set and ask:

> "Guardar estos tickets? (y/n/edit)"

On save:

```bash
git add docs/tickets/
```

Do not commit automatically.

### Step 9: Tracking Choice

If `--tracker` was not provided, ask:

> "Tracking? docs/TICKETS.md / Linear / both / draft"

Map answers:

| Answer | `tracker.source` | Behavior |
|---|---|---|
| `docs/TICKETS.md`, `local` | `local` | Update local tracker only. |
| `Linear` | `linear` | Push issues to Linear; no local queue row unless a ticket file points to Linear. |
| `both` | `both` | Push to Linear and update local tracker as mirror. |
| `draft` | `none` | No tracker update; ticket files only. |

### Step 10: Update docs/TICKETS.md

When `tracker.source in {local, both}`, create or update `docs/TICKETS.md`.

Canonical format:

````markdown
# Tickets

<!-- ENTROPY-TICKETS:START -->
```json
{
  "version": 1,
  "updated_at": "2026-04-27T18:20:00Z",
  "tickets": [
    {
      "id": "ET-20260427-001",
      "source": "local",
      "status": "ready",
      "priority": "medium",
      "kind": "feature",
      "title": "Temperature tuning for deterministic agent paths",
      "path": "docs/tickets/2026-04-27-temperature-tuning.md",
      "linear_id": null,
      "linear_url": null,
      "goal": "...",
      "target_env": "development",
      "full_mode": true,
      "require_deploy": true,
      "autopilot_command": "/entropy-driven-autopilot --from-ticket docs/tickets/2026-04-27-temperature-tuning.md",
      "run_id": null,
      "pr_url": null,
      "delivery_status": null,
      "deploy_status": null,
      "post_deploy_e2e": null,
      "stuck_reason": null,
      "state_file": null,
      "updated_at": "2026-04-27T18:20:00Z"
    }
  ]
}
```
<!-- ENTROPY-TICKETS:END -->

| ID | Status | Priority | Title | Tracker | PR | Delivery |
|---|---|---|---|---|---|---|
| ET-20260427-001 | ready | medium | Temperature tuning for deterministic agent paths | local | - | - |
````

Rules:

- Treat the JSON block as source of truth; regenerate the table from it.
- Preserve existing tickets unless updating the same `id`.
- Stage `docs/TICKETS.md`.
- Do not parse the markdown table as state.

### Step 11: Linear Push

When `tracker.source in {linear, both}`, call Linear `create_issue` for each
ticket:

- title: first non-empty line of `goal`, max 80 chars
- description:
  - local path
  - goal
  - code discovery summary
  - exact `## Entropy autopilot input` JSON block
  - success criteria
  - scope boundaries
- labels:
  - `bug` when `kind=bug`
  - `autopilot` when `full_mode=true`

On success:

1. Set frontmatter `linear_url`, `tracker.linear_id`, and `tracker.linear_url`.
2. If `tracker.source=linear`, set `id` and `ticket_id` to the Linear identifier.
3. If `tracker.source=both`, keep local `id` stable and set `ticket_id` to the
   Linear identifier so PRs can close the Linear issue.
4. Update the JSON block's `ticket_id`.
5. Update `docs/TICKETS.md` mirror row when `source=both`.
6. Stage changed files.

On failure, keep local files valid and report a warning. Linear push never
blocks local ticket creation.

## Final Report

Print:

```text
entropy-ticket: ready
  Tracker: <local | linear | both | none>
  Tickets:
    - <id>: <goal>
      Path: <path>
      Linear: <id | none>
      Autopilot: /entropy-driven-autopilot --from-ticket <path>
```

Then emit `ENTROPY-TICKET-RESULT:` exactly as defined above.

## Integration Notes

### entropy-driven-autopilot

Preferred direct handoff:

```text
/entropy-driven-autopilot --from-ticket docs/tickets/YYYY-MM-DD-<slug>.md
```

The resolver reads frontmatter and uses only the autopilot input fields. If the
ticket has `tracker.linear_id`, pass that as `ticket_id`; otherwise pass `id`.

### entropy-linear-autopilot

When Linear issue text contains `## Entropy autopilot input`,
`entropy-linear-autopilot` should parse that JSON and use it as the structured
input. Config defaults fill only missing fields. This preserves goal, success
criteria, scope boundaries, and deploy intent from the ticket.

### Local tracking

There is no separate local dispatcher yet. `docs/TICKETS.md` provides the
autopilot command per ticket. Run the command for the next `ready` ticket when
you want to execute it.

## Escape Hatches

- Empty idea after prompt → `ticket_status=aborted`
- No relevant code can be inspected → continue only if the user accepts a
  warning; otherwise abort.
- User refuses save → no file written; `ticket_status=aborted`
- Missing goal or success criteria after 3 attempts → abort; these define
  success for autopilot.
- Linear MCP unavailable → keep local files, emit warning; if tracker was
  `linear`, downgrade result to `ready` with `tracker.source=none` unless user
  chooses local tracking.
- File write failure → abort with stderr and no partial result claim.

## Important Rules

- **Goal is mandatory.** It is the external definition of autopilot success.
- **Success criteria are mandatory.** Never guess them silently.
- **Ticket is not spec.** Capture goal, evidence, constraints, boundaries, risks,
  and open questions. Leave approach selection to brainstorming/spec unless the
  codebase has one obvious local pattern.
- **Always inspect code.** A clear idea can still be miscut without repository
  evidence.
- **Local ticket file is canonical for the entropy contract.** Linear and
  `docs/TICKETS.md` track or mirror that contract.
- **JSON blocks must stay valid.** `entropy-linear-autopilot` and
  `entropy-driven-autopilot` depend on them.
- **Do not invoke implementation.** This skill ends at intake.
- **Do not commit.** Stage changed files only; the user decides when to commit.
