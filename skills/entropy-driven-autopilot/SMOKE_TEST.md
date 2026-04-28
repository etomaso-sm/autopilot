# Smoke test — entropy-driven-autopilot

Run this in a scratch repo or sandbox worktree. Do NOT run in `ai-skills` itself.

## Setup

```bash
mkdir -p /tmp/autopilot-smoke
cd /tmp/autopilot-smoke
git init
git commit --allow-empty -m "init"
git checkout -b main 2>/dev/null || git branch -m main
echo "# Smoke" > README.md
git add . && git commit -m "seed"
```

Symlink the local ai-skills repo so the skill is available:
```bash
mkdir -p .claude/skills
ln -s $HOME/.superset/projects/impactia-dev-cookbook/skills/entropy-driven-autopilot .claude/skills/
```

## Invoke

Freeform:
```
/entropy-driven-autopilot Add a simple CLI that prints "hello <name>" when run.
```

## Expected trace

1. Bootstrap creates branch `autopilot/YYYY-MM-DD-add-a-simple-cli-...`.
2. `docs/autopilot-state.json` is committed with `current_step: "bootstrap"`.
3. Brainstorming writes a spec at `docs/superpowers/specs/YYYY-MM-DD-*-design.md` with an `## Autopilot Q&A` section and an `## Approach` section naming three options and a chosen one.
4. `entropy-aware` enriches the spec.
5. `writing-plans` writes a plan at `docs/superpowers/plans/YYYY-MM-DD-*.md`.
6. `entropy-aware` enriches the plan.
7. `subagent-driven-development` runs the plan tasks. A CLI file is created with tests.
8. `entropy-scan` runs on the affected domain.
9. If grades < B, `entropy-fix` runs once.
10. State file is archived to `docs/autopilot-runs/<run_id>.json`.
11. `ship-with-review` opens a PR with the three autopilot sections appended.
12. Fresh-agent review dispatches and returns a JSON block with `verdict` and `findings`.
13. **If** `verdict == "request-changes"` **and** ≥1 finding passes low-risk classification (nit/minor, in-diff, no new deps, no spec/plan changes): autopilot applies the fixes, commits as `fix(autopilot-review): ...`, pushes, and re-dispatches the reviewer exactly once. `budgets.review_fix_cycles_used` is incremented to 1. A fourth `## Autopilot review autofix` section is appended to the PR body.
14. Final report prints with `Status: shipped`. If autofix ran, the report's `Review autofix` line shows applied/unresolved counts; otherwise it shows `none`.

## Success criteria

- Branch exists and is pushed.
- `docs/autopilot-runs/<run_id>.json` exists, is valid JSON, and contains ≥1 entry in `decisions`.
- A PR is open on GitHub with "Autopilot run", "Key autopilot decisions", and "What to double-check" sections.
- The review agent returned a verdict (any of `approve`, `request-changes`, `comment`) AND a `findings` array (possibly empty).
- `budgets.review_fix_cycles_used` is `0` or `1` (never higher).
- If autofix ran: there is exactly one `fix(autopilot-review): ...` commit, the PR body has a `## Autopilot review autofix` section, and `state.review_history` contains the prior verdict/findings.
- Findings not auto-fixed (severity ≥ moderate, out-of-diff, etc.) appear in the final report under "Unresolved review findings".

## Failure injection tests

Run once each and verify the stuck handling:

1. **Protected branch:** invoke from `main` directly (bypass the check only for this test). Expected: `stuck_reason: protected_branch`, no branch created.
2. **Dirty tree:** leave an unstashable conflict. Expected: `stuck_reason: dirty_tree`.
3. **Timeout:** pass `max_wall_clock_min: 1`. Expected: `stuck_reason: timeout`, partial state committed.
4. **Empty spec:** stub brainstorming to produce an empty file. Expected: `stuck_reason: empty_spec`.

## Review-autofix behaviour tests (not stuck paths)

5. **Reviewer returns nit + minor findings only, all in-diff:** Expected: autopilot applies all, commits `fix(autopilot-review): ...`, pushes, re-dispatches. `review_fix_cycles_used == 1`. `review_history` has 1 entry. PR body has autofix section.
6. **Reviewer returns a `moderate` finding:** Expected: no autofix applied for that finding (severity gate). If no other finding is auto-fixable, Step 8.5 is a no-op and the PR ships with the original `request-changes` verdict. The moderate finding appears in the final report as unresolved.
7. **Reviewer returns a `minor` finding whose `location` is outside the PR diff:** Expected: classified non-auto-fixable (in-diff gate), surfaced as unresolved.
8. **Second review after autofix still `request-changes`:** Expected: NO second autofix cycle (budget cap). `review_verdict` in final report = latest (second) verdict. Status remains `shipped`; PR stays open for the human.

For each, verify:
- State file archived to `docs/autopilot-runs/<run_id>.json`.
- Branch exists locally and on origin.
- No PR created.
- Final report includes the `Next step` line.

## Full mode tests (opt-in auto-merge)

Each case invokes with `--full` (or `"full_mode": true`). The spec/plan/code flow is unchanged; only the post-review behavior differs.

9. **Full mode success — CI absent:** `full_mode=true`, reviewer returns `approve`, quality `pass`, no unresolved findings, repo has no CI configured. Expected: `merge_status="merged"`, `merged=true`, branch deleted on origin, `merge_sha` populated in the final report and machine-readable JSON. The "Autopilot full mode" section appears in the (now merged) PR body with all four gates `✅`.

10. **Full mode skipped — request-changes after autofix:** `full_mode=true`, reviewer returns `request-changes` and after one autofix cycle the second review still returns `request-changes`. Expected: `merge_status="skipped_review_not_approved"`, `merged=false`, PR open, gates G2–G4 recorded as `null` (short-circuited), final report shows `Merge: skipped: skipped_review_not_approved`.

11. **Full mode skipped — CI red:** `full_mode=true`, reviewer approves, CI is configured and one check fails on HEAD. Expected: `merge_status="skipped_ci_failed"`, G1–G3 `true`, G4 `false`, `ci_status="present"`.

12. **Full mode skipped — rebase blocked:** `full_mode=true`, all gates pass, but `gh repo view --json rebaseMergeAllowed` returns `false`. Expected: `merge_status="skipped_rebase_not_allowed"`, no merge attempted, PR open.

13. **Full mode disabled (default):** omit the flag. Expected: `merge_status=null`, `merged=false`, `full_mode=false` in the returned JSON, **no change** vs. current behavior. Step 8.7 does not execute — no new decision entries for `merge_gate_*`.

For each case, verify:
- State file archived to `docs/autopilot-runs/<run_id>.json` contains the expected `merge_status`.
- `state.merge_gates_evaluated` reflects the short-circuit (gates not evaluated are `null`).
- The machine-readable JSON returned to the caller matches the state file for `merge_status`, `merged`, `merge_sha`, and `merge_gates`.
- **On merge success (case 9):** the branch no longer appears on origin (`gh api repos/:owner/:repo/branches --jq '.[].name' | grep -c "^<BRANCH>$"` returns `0`).

## Teardown

```bash
cd ~ && rm -rf /tmp/autopilot-smoke
```
