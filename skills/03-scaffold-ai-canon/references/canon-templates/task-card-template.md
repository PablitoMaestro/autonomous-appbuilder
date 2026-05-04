# Task Card Template — `master-plan-points/<id>.md`

One file per task in the master plan. The scaffold skill generates these alongside the master plan; executor + reviewer subagents fill in the lower sections during their lifecycles.

## Template

```markdown
# Task <id> — <one-line title from master plan>

**Phase:** <phase number + name>
**Estimate:** <q | s | wall-clock>
**Tags:** <db / e2e / breaking / etc.>
**Status:** `[ ]` (current — sync with master-plan.md row)

## Goal

<2-4 sentences. What this task delivers and why. Reference the PRD's MVP item or NFR this serves.>

## Acceptance criteria

The task is `[x]` only when ALL of these are true:
- [ ] <criterion 1 — testable; e.g. "User can submit /login form with valid creds and lands on /dashboard">
- [ ] <criterion 2>
- [ ] <criterion 3>

## Dependencies

- **Depends on:** <other task IDs that must be `[x]` first; "none" if standalone>
- **Blocks:** <task IDs that depend on this one; for awareness, not enforced>

## Implementation hints

- **Files to touch (best guess):** <list paths>
- **Existing patterns to follow:** <reference to similar task or component>
- **Pitfalls / gotchas:** <e.g. "watch out for race condition with X", "this changes the schema, run pnpm db:types after">

## Test plan

- **Unit/integration:** <which assertions, which file path>
- **E2e (Playwright/equivalent):** <which user flow, what to verify>
- **Visual:** <which page in browser, what to look at, mobile viewport y/n>

## Implementation log

(Filled in by executor subagent during work — append, don't overwrite)

```
### YYYY-MM-DD HH:MM — agent <id>

- Branch: feature/<id>-<slug>
- PR: <url>
- Files touched: <list>
- Decisions: <what + why for non-obvious calls>
- Test results: typecheck=p lint=p build=p test=p e2e=p
- Status flipped: [ ] → [~] → [c] → [t] → [x]
```

## Review log

(Filled in by reviewer subagent during Phase 6 — append-only)

```
### Review — YYYY-MM-DD HH:MM — agent <id>

- Baseline verdict: <green / failures>
- Changes: <surgical fix or "none — audit clean">
- Review addition: <only if verdict=extended; what + why + files>
- Gaps closed: <integration glue added; "none">
- Post-review verdict: clean | improved | extended | issues_found | blocked | failed
```
```

## How to fill at scaffold time

When generating cards from the master plan:

1. **Title + ID** copied from the master-plan row.
2. **Goal** — 2-4 sentences. Pull from the PRD's MVP/NFR section that this task serves. If the task is foundational (e.g., "scaffold Next.js"), explain in terms of what it unlocks.
3. **Acceptance criteria** — 3-5 testable bullets. Use checkbox markdown so reviewer can mark them off.
4. **Dependencies** — derive from the waves table (`→` arrows) and obvious file overlaps.
5. **Implementation hints** — best-guess file paths from `site-map.md` + `database-architecture.md`. OK to say "verify exact path during implementation".
6. **Test plan** — concrete enough that the executor knows what to write. Skip `e2e` for tasks with no UI surface.
7. **Implementation log + Review log** — leave the headers; subagents append.

## Anti-patterns

- ❌ Cards that just restate the title in the Goal section. Add real context.
- ❌ Acceptance criteria like "feature works" — too vague. Use observable behavior.
- ❌ Listing every possible file as a dependency. Be selective.
- ❌ Test plans that say "add tests" without specifying which.
- ❌ Filling Implementation log at scaffold time. That's the executor's job.

## Example — short version

Some tasks are small enough that the full template is overkill. For `q` (quick) tasks, the minimum acceptable card is:

```markdown
# Task 9.6 — Update CLAUDE.md with deploy notes

**Phase:** 9 — Deploy + handoff
**Estimate:** q
**Status:** `[-]`

## Goal

After Phase 9 deploys, capture lessons learned and current deploy procedure in CLAUDE.md so future agents have the right runbook reference.

## Acceptance criteria

- [ ] CLAUDE.md has a "## Deployment" section
- [ ] Section references docs/runbook.md
- [ ] Mentions any post-9.3/9.4 surprises (rollback gotchas, env quirks)

## Implementation hints

- Read docs/runbook.md as the source; CLAUDE.md links to it.

## Test plan

- N/A (doc-only task; status `[-]`)
```
