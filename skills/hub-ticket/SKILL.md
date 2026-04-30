---
name: hub-ticket
description: >
  Use when a rough idea needs code-aware discovery, goal definition, and one or
  more delivery-ready hub tickets for local tracking, autopilot, or
  draft-only planning. Triggers on: hub-ticket, hub-intake, armar ticket,
  new ticket, nuevo ticket, draft ticket, crear ticket, idea to ticket.
user_invocable: true
---

# Hub Ticket — Code-Aware Intake Gate

<!-- SYNC: hub-ticket depends on:
     - hub-driven-autopilot input schema:
       goal, description, constraints, success_criteria, scope_boundaries,
       ticket_id, base_branch, max_wall_clock_min, target_env, full_mode,
       require_deploy
     - docs/TICKETS.md local tracker contract below.
     Update this skill when any input field, delivery status, or tracker field
     changes. -->

**Announce at start:** "Running /hub-ticket — code-aware hub intake and ticket shaping."

## Project Tracking & Governance

Source of truth for how Hub work is tracked lives in `_jockey/`:

- `_jockey/CONVENTIONS.md` — code, session, deploy, D1, verification rules. New conventions append under `## C## additions` sections.
- `_jockey/DECISIONS.md` — `DEC-C##-NN` decision log; cite when conventions reference a DEC.
- `_jockey/STATE.md` — current Control + active phase.
- `_jockey/LOCKS.md` — file-lock manifest for shared / collision-prone files. **Read this before editing application code.** If a target file is claimed by another run, abort the edit and coordinate (post `[BLOCKED]` in #hub-dev). When you start editing a shared file, append a row claiming it; remove the row after the commit is pushed. `routes/agents.js` is in the **Shared (coordinate first)** lane and additionally requires a pre-edit ping in #hub-dev (Aaron's operating doc rule #4).
- Session prompts live in `_jockey/queue/` until fired, then move to `_jockey/archive/fired/`. Naming: `C[N]-S[#]v[ver]-[name].md`.
- New behavioral conventions or decisions that emerge from this skill's run must land in `_jockey/DECISIONS.md` (DEC entry) and a `## C## additions` block in `_jockey/CONVENTIONS.md`.
- When a ticket produces a CC session prompt, place it in `_jockey/queue/` with the `C[N]-S[#]v[ver]-[name].md` naming — never inline-edit code or fire from `~/Downloads` without staging.

> **Coexists with skill-level tracking — neither invalidates the other.** This is program-level governance. The skill's own tracking artifacts (`docs/TICKETS.md`, `docs/QUALITY_SCORE.md`, `docs/hub-loop-state.json`, evidence files in `docs/`, etc.) remain the authoritative source for the skill's operational state. `_jockey/` is the program-level layer (Control, conventions, decisions). Both must coexist.

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
- nowhere yet (`tracker.source=none`; draft only)

This skill does not implement or ship tickets. It creates the goal contract that
`/hub-driven-autopilot --from-ticket ...` uses to judge success.

The ticket file is canonical for the hub contract (`goal`,
`success_criteria`, `scope_boundaries`, and autopilot JSON). `docs/TICKETS.md`
is a tracking surface for that contract.

## Hub Contract

Required final marker:

```text
HUB-TICKET-RESULT:
{
  "ticket_status": "ready | tracked | aborted",
  "tracker": {
    "source": "local | none",
    "path": "docs/TICKETS.md"
  },
  "tickets": [
    {
      "local_id": "ET-20260427-001",
      "path": "docs/tickets/2026-04-27-temperature-tuning.md",
      "goal": "...",
      "autopilot_command": "/hub-driven-autopilot --from-ticket docs/tickets/..."
    }
  ],
  "warnings": []
}
```

Fields that do not apply must be `null`, not omitted.

## Invocation

```text
/hub-ticket <rough task description>
/hub-ticket --local <rough task description>
/hub-ticket --manual <rough task description>
/hub-ticket --tracker=local|none <rough task description>
```

Flags:

| Flag | Effect |
|---|---|
| `--local` | Sets `target_env=local`, `full_mode=false`, `require_deploy=false`. |
| `--manual` | Keeps deploy target but sets `full_mode=false`; a human runs/merges later. |
| `--full` | Sets `full_mode=true`, `require_deploy=true`, `target_env=development` unless the blob says production/staging. |
| `--tracker=<mode>` | Skips the tracker prompt. `local` or `none`. |

If no argument is given, ask: "¿Qué idea querés convertir en tickets?"

## Frontmatter Schema

Each generated ticket file must use this frontmatter:

```yaml
---
# Ticket metadata
id: ET-20260427-001
created: 2026-04-27
status: ready                 # draft | ready | in-progress | pr-open | verified | blocked | deferred
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

# hub-driven-autopilot input
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
  deploy_gate: hub-check-deploy
  frontend_gate: hub-e2e-frontend

tracker:
  source: local               # local | none
  local_id: ET-20260427-001
  tracker_path: docs/TICKETS.md
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
- if `kind=bug`, note that bugs route to `/hub-bugfix` rather than
  `/hub-driven-autopilot`.

If validation fails, re-ask only the offending field.

### Step 8: Build Ticket Files

Each body must include:

1. `## Goal`
2. `## Code discovery`
3. `## Success criteria`
4. `## Scope boundaries`
5. `## Hub autopilot input`

The autopilot input block must be valid JSON:

````markdown
## Hub autopilot input
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

> "Tracking? docs/TICKETS.md / draft"

Map answers:

| Answer | `tracker.source` | Behavior |
|---|---|---|
| `docs/TICKETS.md`, `local` | `local` | Update local tracker only. |
| `draft` | `none` | No tracker update; ticket files only. |

### Step 10: Update docs/TICKETS.md

When `tracker.source=local`, create or update `docs/TICKETS.md`.

Canonical format:

````markdown
# Tickets

<!-- HUB-TICKETS:START -->
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
      "goal": "...",
      "target_env": "development",
      "full_mode": true,
      "require_deploy": true,
      "autopilot_command": "/hub-driven-autopilot --from-ticket docs/tickets/2026-04-27-temperature-tuning.md",
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
<!-- HUB-TICKETS:END -->

| ID | Status | Priority | Title | Tracker | PR | Delivery |
|---|---|---|---|---|---|---|
| ET-20260427-001 | ready | medium | Temperature tuning for deterministic agent paths | local | - | - |
````

Rules:

- Treat the JSON block as source of truth; regenerate the table from it.
- Preserve existing tickets unless updating the same `id`.
- Stage `docs/TICKETS.md`.
- Do not parse the markdown table as state.

## Final Report

Print:

```text
hub-ticket: ready
  Tracker: <local | none>
  Tickets:
    - <id>: <goal>
      Path: <path>
      Autopilot: /hub-driven-autopilot --from-ticket <path>
```

Then emit `HUB-TICKET-RESULT:` exactly as defined above.

## Integration Notes

### hub-driven-autopilot

Preferred direct handoff:

```text
/hub-driven-autopilot --from-ticket docs/tickets/YYYY-MM-DD-<slug>.md
```

The resolver reads frontmatter and uses only the autopilot input fields. Pass
`id` as `ticket_id`.

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
- File write failure → abort with stderr and no partial result claim.

## Important Rules

- **Goal is mandatory.** It is the external definition of autopilot success.
- **Success criteria are mandatory.** Never guess them silently.
- **Ticket is not spec.** Capture goal, evidence, constraints, boundaries, risks,
  and open questions. Leave approach selection to brainstorming/spec unless the
  codebase has one obvious local pattern.
- **Always inspect code.** A clear idea can still be miscut without repository
  evidence.
- **Local ticket file is canonical for the hub contract.** `docs/TICKETS.md`
  tracks that contract.
- **JSON blocks must stay valid.** `hub-driven-autopilot` depends on them.
- **Do not invoke implementation.** This skill ends at intake.
- **Do not commit.** Stage changed files only; the user decides when to commit.
