# Master Plan Grammar

The exact format `master-plan.md` MUST follow. Both `05-plan-executor` and `06-plan-reviewer` parse the plan file using these rules — drift breaks the whole pipeline.

## Status codes

Each task row carries one inline status code:

| Code | Meaning | Set by |
|---|---|---|
| `[ ]` | Not started | scaffold (default) |
| `[~]` | In progress | executor subagent at start of work |
| `[c]` | Code done, tests not yet | executor subagent after impl |
| `[t]` | Tests passing, awaiting visual e2e | executor subagent after unit/integration |
| `[x]` | Done — code merged + tests green + visual e2e green | executor subagent after final verify |
| `[-]` | Doc-only task; tests not applicable | scaffold or executor |

Reviewer (`06-plan-reviewer`) preserves status — `[x]` stays `[x]`. Review never downgrades.

The grammar `\[[ ~ctx-]\]` (regex) matches any valid status. Use this for queue-building.

## Task ID convention

- Format: `<phase>.<task>` (e.g. `0.1`, `1.1`, `2.4`, `8.6`).
- Phase numbers are 0..N. Phase 0 is the **discovery & scaffolding journey** (auto-tracked by orchestrator); phases 1..N are the actual app work.
- Task numbers within a phase start at 1 and increment.
- IDs are **immutable** once the plan is locked. Renumbering breaks task cards (`master-plan-points/<id>.md`) + findings logs.
- New tasks added during a run get the next free number in their phase OR go in a side-plan (`.ai/plans/<date>-<name>/`).

## Phase 0 — special semantics

Phase 0 is reserved for orchestrator-tracked work and follows fixed conventions:
- `0.1` — Tool preflight passed (set by `00-build-app` Step 0.5)
- `0.2` — PRD locked (set by `01-discover-idea`)
- `0.3` — Design locked (set by `02-design-create-n-htmls`)
- `0.4` — Plan scaffolded (set by `03-scaffold-ai-canon` after expansion)
- `0.5` — Env bootstrapped + runbook (set by `04-bootstrap-env`)
- `0.6` — CLAUDE.md skeleton + AGENTS.md/GEMINI.md pointers initialized (set by `01-discover-idea` after writing the skeleton + thin pointers; enriched by `03-scaffold-ai-canon` and `04-bootstrap-env` — but stays `[x]` after the skeleton exists). AGENTS.md and GEMINI.md are 3-line pointers to CLAUDE.md so the canonical doc stays the single source of truth and cross-tool consumers (Codex, Gemini CLI) find the right file by convention without duplicating content.

These task IDs are hard-referenced by the orchestrator's phase-detection. Do not rename, renumber, or reorder them.

`05-plan-executor` does not pick up Phase 0 tasks during its loop — they're set by orchestrator phases, not by feature work.

## Master-plan phase number ≠ orchestrator phase number

Two numbering systems coexist:
- **Orchestrator phases** (1-7): describe what the agent does (discover, design, scaffold, bootstrap, execute, review, smoke).
- **Master-plan phases** (0-N): describe what gets built (Phase 0 = discovery journey, Phase 1+ = app work).

They map loosely (orchestrator Phase 1 produces master-plan task 0.2; orchestrator Phase 5 executes master-plan tasks 1.1+) but they're not the same axis. When confusion is possible, prefix: "orchestrator Phase 5" vs "master-plan Phase 1".

## Plan-row format

One task per row:

```
- [<status>] **<id>** — <one-line description> · <est> · <tags>
```

- `<est>` is optional; use `q` for quick wins (~30 min), `s` for substantial (~2-4 h), or wall-clock estimate.
- `<tags>` are optional; common ones: `e2e` (touches e2e tests), `db` (schema migration), `breaking` (API change).

Examples:
```
- [ ] **1.1** — Initialize Next.js + Supabase + Vercel skeleton · s
- [x] **2.3** — Add `users` table + RLS policies · q · db
- [~] **4.2** — Wire up admin layout shell
- [-] **9.6** — Update CLAUDE.md with deploy notes
```

## Waves table

The plan starts with a `### Waves` section that defines parallelization order. Without it, executor falls back to plan-row order.

```
### Waves

| Wave | Tasks | Notes |
|---|---|---|
| W1 | 1.1 → 1.2 → 1.3 | foundation, sequential |
| W2 | 2.1 ∥ 2.2 ∥ 2.3 | independent table migrations |
| W3 | (3.1 ∥ 3.2) → 3.3 | UI primitives then layout |
| W4 | 4.1 ∥ 4.2 ∥ 4.3 ∥ 4.4 | feature implementations |
| ... |
```

- `∥` means parallel (executor can dispatch concurrently, subject to its concurrency caps).
- `→` means sequential (later task depends on earlier completion).
- Parens group sub-flows.

Use Polish `### Fale` if `CLAUDE.md` says project language is Polish — both readers (`05-plan-executor`, `06-plan-reviewer`) accept either heading.

## Top-of-file structure

```markdown
# Master Plan — <Project Name>

> Source of truth for scope and progress. Status codes: `[ ]` → `[~]` → `[c]` → `[t]` → `[x]` (or `[-]` for doc-only). Numbering is immutable.

## Phase 1 — Foundation

- [ ] **1.1** — ...
- [ ] **1.2** — ...

## Phase 2 — Schema

- [ ] **2.1** — ...

...

## Phase 9 — Deploy + handoff

- [ ] **9.1** — ...

---

### Waves

| Wave | Tasks | Notes |
|---|---|---|
| W1 | 1.1 → 1.2 → 1.3 | foundation |
| W2 | 2.1 ∥ 2.2 ∥ 2.3 | schema |
| ... |

### Plany dodatkowe / Side plans

| Plan | Status | Notes |
|---|---|---|
| (none yet) | | |
```

## Side-plan registration

When a side plan is created (e.g., `.ai/plans/2026-05-04-review-fixes/`), it must be registered in the master plan's "Plany dodatkowe / Side plans" footer table:

```
| 2026-05-04-review-fixes | in progress | from Phase 6 review loop, 12 high-severity findings |
```

This keeps the master plan as the single index for all plan files.

## Doc-only tasks

Tasks that only update documentation (e.g. `9.5 — Update site-map.md after route renames`) use `[-]` for tests because there's nothing to test. They still go through executor lifecycle (worktree, commit, PR, merge), just without the test gates.

## Anti-patterns

- ❌ Renumbering tasks ("oh let's make 4.1 → 4.2 because this should be later"). Use `### Waves` to reorder execution; never mutate IDs.
- ❌ Mixing status grammars within a file (e.g. some rows use `[x]`, others use `✓`). Pick one and stick to it.
- ❌ Plan-row descriptions longer than ~120 chars. Move detail to the task card (`master-plan-points/<id>.md`).
- ❌ Tasks without IDs ("- [ ] add login button"). Executor can't track without an ID.
- ❌ Adding tasks to the master plan that reference files / tables that don't exist in `.ai/database-architecture.md` or `.ai/site-map.md` yet. Update those docs first.

## Why this grammar

- **Inline status code** = grep-able. `rg "^\s*-\s*\[\s\]" .ai/master-plan.md | wc -l` tells you how many tasks remain.
- **Immutable IDs** = stable references across cards, findings logs, PRs (`feature/4.1-flashcard-create`), branch names, commit messages.
- **Waves table separate** = scope (the task list) and execution order (the waves) evolve at different rates; coupling them in one structure makes both harder to edit.
- **Side-plan footer** = one place to find every plan that exists, even when 5 side-plans are spawned during a long autonomous run.
