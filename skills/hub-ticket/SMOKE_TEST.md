# Smoke test — hub-ticket

Run this in a scratch repo or sandbox worktree. Do NOT run in `ai-skills` itself.

## Setup

```bash
mkdir -p /tmp/hub-ticket-smoke/src
cd /tmp/hub-ticket-smoke
git init
git commit --allow-empty -m "init"
git checkout -b main 2>/dev/null || git branch -m main
cat > src/agents.js <<'JS'
export async function classify(model, input) {
  return model.generate({ system: "classify", input });
}
export async function draft(model, input) {
  return model.generate({ system: "draft", input });
}
JS
git add src/agents.js && git commit -m "fixture"
```

Symlink the local ai-skills repo:

```bash
mkdir -p .claude/skills
ln -s $HOME/.superset/projects/impactia-dev-cookbook/skills/hub-ticket .claude/skills/
```

## Cases

### 1. Code-aware single ticket, local tracker

Invoke:

```text
/hub-ticket --tracker=local Set deterministic temperature for classification without changing generation
```

Expected:
- Skill inspects code before drafting; `code_discovery.observations` mentions the classification/generation call shape.
- One ticket is generated under `docs/tickets/`.
- Frontmatter contains non-empty `goal`.
- `success_criteria` has at least one objective criterion.
- `docs/TICKETS.md` is created with an `HUB-TICKETS` JSON block and a table.
- Tracker row includes `autopilot_command: /hub-driven-autopilot --from-ticket ...`.
- `git status` shows `docs/tickets/...` and `docs/TICKETS.md` staged, but no commit.

### 2. Broad request splits into spike or multiple tickets

Invoke:

```text
/hub-ticket --tracker=local Analyze structured outputs and versioned prompts from the current agent code and make the tickets needed
```

Expected:
- Skill inspects relevant files with `rg`.
- If implementation scope is broad, it creates either:
  - one `kind: spike` ticket first, or
  - multiple bounded tickets with separate goals.
- No ticket contains solution architecture as a hard contract.
- Each ticket has `goal`, `success_criteria`, `scope_boundaries`, and `code_discovery`.

### 3. Linear tracker

Invoke case 1 with `--tracker=linear`.

Expected:
- Skill creates Linear issue(s).
- On success, ticket frontmatter sets `tracker.source: linear`, `tracker.linear_id`, `linear_url`, and `ticket_id` to the Linear ID.
- `docs/TICKETS.md` is not required.
- Linear issue body includes the exact `## Hub autopilot input` JSON block.

### 4. Both trackers

Invoke case 1 with `--tracker=both`.

Expected:
- Skill creates Linear issue(s) and updates `docs/TICKETS.md`.
- Local `id` remains stable as `ET-...`.
- `ticket_id` in the autopilot JSON becomes the Linear ID so PRs can close Linear.
- `docs/TICKETS.md` row includes both local id and Linear URL.

### 5. Draft only

Invoke case 1 with `--tracker=none`.

Expected:
- Ticket file is generated and staged.
- No Linear issue is created.
- `docs/TICKETS.md` is not created or updated.
- `tracker.source: none`.

### 6. Validation failure

Force an empty goal or empty success criteria during the interview.

Expected:
- Skill re-prompts up to 3 times.
- On repeated failure, aborts with `ticket_status=aborted`.
- No partial success claim is made.

## Success Criteria

Across all cases:
- Every ticket has a non-empty `goal`.
- Every ticket has at least one `success_criteria` item.
- Every ticket has `code_discovery.searched`, `touched_areas`, `observations`, and `open_questions`.
- Every ticket body includes valid `## Hub autopilot input` JSON with `goal`.
- `docs/TICKETS.md`, when present, has a valid JSON block between `HUB-TICKETS:START` and `HUB-TICKETS:END`.
- Files are staged but never committed.

## Teardown

```bash
cd ~ && rm -rf /tmp/hub-ticket-smoke
```
