# Phase Detection — Read Master-Plan Checkboxes

The orchestrator decides where to enter the 7-phase flow by reading **one source of truth**: the Phase 0 task statuses in `.ai/master-plan.md`.

## Single rule

```
if .ai/master-plan.md does not exist:
    entry_phase = 1   # nothing started yet; discover-idea creates the master plan
else:
    read Phase 0 task statuses from master-plan
    entry_phase = first phase whose Phase 0 task is not [x]
                = mapped per the table below
```

## Phase ↔ Phase 0 task mapping

The master plan's Phase 0 tracks the orchestrator's planning/scaffolding work:

| Master-plan task | Set by | Maps to orchestrator phase |
|---|---|---|
| `0.1 Tool preflight passed` | orchestrator Step 0.5 | (gate before Phase 1) |
| `0.2 PRD locked` | `01-discover-idea` after PRD draft accepted | Phase 1 |
| `0.3 Design locked` | `02-design-create-n-htmls` after pick | Phase 2 |
| `0.4 Plan scaffolded (this file expanded)` | `03-scaffold-ai-canon` after expansion | Phase 3 |
| `0.5 Env bootstrapped + runbook seeded` | `04-bootstrap-env` after dev stack works | Phase 4 |
| (Phases 1+) all `[x]` | `05-plan-executor` per-task | Phase 5 |
| smoke log within 24h with ✅ verdict | `07-smoke-test-app` | Phase 7 |

Phase 6 (review) is opt-in and not part of auto-resume. It runs only when `review=on` is passed.

## Detection algorithm

```python
# Pseudocode — actual impl is shell + Read tool
plan_path = ".ai/master-plan.md"

if not exists(plan_path):
    return entry_phase = 1

phase_0_tasks = grep(r"^\s*-\s*\[(.)\]\s*\*\*0\.\d+\*\*", plan_path)
# returns list of (status_char, task_id) tuples

# Find first incomplete Phase 0 task
for status, task_id in phase_0_tasks:
    if status != "x" and status != "-":
        return entry_phase = task_to_phase(task_id)

# All Phase 0 done — check phases 1+
remaining = grep(r"^\s*-\s*\[[ ~ct]\]\s*\*\*\d+\.\d+\*\*", plan_path)
# any non-Phase-0 task that isn't done?
if remaining > 0:
    return entry_phase = 5    # plan-executor

# All tasks done — check smoke
recent_smoke = find(".ai/agent-logs/smoke-*.md", mtime within 24h)
if recent_smoke and contains(recent_smoke, "Verdict: ✅"):
    return entry_phase = "done"

return entry_phase = 7    # need fresh smoke
```

## Why this works (and why we abandoned multi-artifact detection)

The previous design checked 6 separate artifacts (`.ai/PRD.md`, `.ai/design.md`, `.ai/master-plan.md`, env files, plan completion, smoke log). Three problems:

1. **Out of sync.** A `.ai/PRD.md` could exist while task `0.2` is `[ ]` if the orchestrator crashed mid-Phase-1. Two truths, ambiguous.
2. **Brittle validity checks.** "PRD has all required sections" is a heuristic that breaks when sections rename.
3. **No accommodation for partial work.** A heavy doc like `database-architecture.md` (now Phase 1 task 1.1) shouldn't be a phase gate — it's a feature task.

The new design has ONE truth (master-plan checkboxes) and the artifact existence is just a side-effect.

## Edge cases

### Master-plan exists but Phase 0 section missing
- The plan was created by an older skill version OR hand-written. Fall back to "look for any `[ ]`/`[~]`/`[c]`/`[t]` task" detection. If found, entry = Phase 5. If none, entry = Phase 7.
- Warn the user: "Phase 0 not present in master-plan; treating as legacy plan. Recommend running `/03-scaffold-ai-canon` to add Phase 0."

### Phase 0 task `0.1` is `[ ]` but Step 0.5 already ran in this session
- The orchestrator just ran preflight; flip `0.1` to `[x]` before applying detection. (The flip happens at end of Step 0.5.)

### Multiple Phase 0 tasks `[~]` (in progress)
- Means a previous run died mid-phase. Resume at the lowest-numbered `[~]` task (the one that was most recently being worked).

### `phase=X` override
- Skip detection. `entry_phase = X`. Flip the corresponding Phase 0 task back to `[ ]` (the override is asking us to redo it).
- Refuse without `force=yes` if the artifact would be overwritten and existing content would be lost.

### `task=<id>` override (single-task mode)
- Skip detection entirely. Hand off to `05-plan-executor` Mode B.

### `from=scratch`
- `entry_phase = 1`. Refuse without `force=yes` if any artifact exists.
- With `force=yes`: archive existing `.ai/` to `.ai/_archive/<timestamp>/` before running Phase 1.

### Smoke log is `❌` failed verdict
- If `repair=true` was set on the failing run, a `smoke-fixes` side-plan was created → Phase 5 has new tasks → entry = Phase 5.
- Otherwise → entry = Phase 7 (re-run smoke; assumes user fixed something).

## Reporting

After detection, emit one line:
```
Detected entry phase: <N> (<reason>). Mode: <mode>. Will run phases <entry..end>.
```

Examples:
- `Detected entry phase: 1 (no .ai/master-plan.md). Mode: supervised. Will run phases 1..7.`
- `Detected entry phase: 4 (master-plan task 0.5 is [ ]). Mode: autonomous. Will run phases 4..7.`
- `Detected entry phase: 5 (Phase 0 done; 47 incomplete tasks in phases 1+). Mode: autonomous. Will run phases 5..7.`
- `Detected entry phase: done (all tasks [x], smoke ✅ within 24h). Nothing to do.`

In autonomous mode the same line is appended to the decision log instead of presented interactively.
