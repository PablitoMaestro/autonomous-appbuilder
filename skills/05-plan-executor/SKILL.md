---
name: 05-plan-executor
description: Execute one task or every remaining task from a markdown checklist plan. Use when the user asks to "execute the plan", "run the master plan", "finish all remaining tasks in plan X", "ship the plan in parallel", "execute task 4.1", "do plan point 5.2", or invokes a slash command of a similar shape. Defaults to a `master-plan.md` discovered in the repo, but accepts an explicit plan path or a specific task id. Repo-agnostic — works in any codebase that has a markdown checklist plan with tasks.
---

# Plan Executor

> **Invocation contract:** This skill is invoked via `Skill('05-plan-executor')` — typically by `00-build-app` at Phase 5, but also standalone. The caller MUST NOT dispatch page-builder or task-implementing subagents directly from main context. This skill owns the cohort loop, the worktree orchestration, the per-task card lifecycle, the test-coverage contract, and the PR-merge-to-main step. Bypassing it means subagents that don't write tests, don't run `simplify`, don't merge their PRs — i.e., a broken audit trail.

Drive a markdown-checklist implementation plan forward in two modes:

- **Mode A — Full-plan orchestration.** The main agent stays on `main` (or `master`) and dispatches **one subagent per remaining task** (never bundles), each in its own `git worktree`, each merging its own PR back to `main`. The main agent does not edit product code itself.
- **Mode B — Single-task execution.** The main agent itself executes one named task end-to-end, in-process, no subagents. Same lifecycle as `references/subagent-prompt.md`.

Pick the mode in Step 0.

## ⚠️ Hard contracts — read first

These rules are non-negotiable. They make the difference between "executor that ships" and "executor that produces a tangle".

### Granularity contract — one task = one agent

**Every parallel agent dispatch handles EXACTLY ONE task ID from the plan. Bundling multiple tasks into one agent dispatch is FORBIDDEN.**

If the plan has tasks `3.3`, `3.4`, `3.5`, you dispatch THREE agents — not one agent that "does all the catalog pages". The dispatching agent must keep the granularity of the plan.

Why this matters:
- Audit trail: one task → one card update → one branch → one PR → one merged commit
- Rollback: a bad agent affects exactly one task, not seven
- Worktree isolation: 1 worktree per task means concurrent agents never trample each other
- Per-task card lifecycle stays atomic — a card is either filled or not, never half-filled by a multi-task agent

Edge case: if two tasks **must** ship together because one is meaningless without the other (a Server Action and the form that calls it), they're either (a) one task in the plan — collapse them at scaffold time, or (b) sequenced via `→` in the wave (one ships first, the other follows in the next cohort). Never bundled into one agent.

### Isolation contract — worktree per parallel + per important task

**Every agent that runs in parallel with another agent MUST be in its own `git worktree`.** Parallel work without isolation = file races, merge conflicts, broken local stacks.

**Every code-touching task in master-plan phases 1+ MUST be in its own worktree** — even if it's currently the only one in flight. These are the audited deliverables; one task = one branch = one PR = one merge to `main`.

**Worktree is OPTIONAL (allowed in main working tree)** for these classes of subagent — they're sequential, single-purpose, and don't compete for files:

| Subagent class | Worktree? | Why |
|----------------|-----------|-----|
| Cleanup / recovery (rebase orphan branches, close stale PRs, clean `[gone]` branches) | optional | Operates on git metadata, not source files; single subagent at a time |
| Wave-fixer (re-edits `master-plan.md` `## Recommended order — Waves` section only) | optional | One file, one section, no other agent touches it |
| Card-generator (creates `master-plan-points/<id>.md` from template) | optional | Disjoint files (one per ID); but if dispatched in parallel ≥2 at once, then yes |
| Log-fixer / log-backfill (touches `.ai/agent-logs/*.md` only) | optional | Audit-only files |
| Doc-sync (touches `.ai/*.md` planning docs only after a code task already merged) | optional | Documentation, not code |
| PR-finisher (just runs `gh pr merge` on a stuck PR) | optional | Git operation, no file edits |
| Browser-investigation (read-only, may take screenshots) | optional | No file writes at all |

The executor MUST:
1. **For mandatory-worktree subagents:** create the worktree before dispatch, pass the path + branch name, wait for agent's PR-merge, clean up the worktree after merge.
2. **For optional-worktree subagents:** skip worktree creation. In the addendum, tell the agent explicitly: "You are running in the main working tree. Touch ONLY the file(s) named in this prompt. Do not run `git worktree add`, do not push, do not open a PR."

Worktrees do not carry gitignored files. Worktree-bound subagents bootstrap `.env.local`, run `pnpm install`, etc. inside their worktree. This is documented in `references/subagent-prompt.md` Step 2.

If the executor is unsure whether a task needs a worktree: **default to YES**. Worktree overhead is small; collision recovery is expensive.

### Integration contract — every task merges to main via PR

**Every task ends with `gh pr merge` to `main`. There is no "we'll merge it later" branch parking lot.**

The lifecycle is hard-wired:

```
worktree off main → branch → implement → verify → simplify pass → push → gh pr create → gh pr merge --squash --delete-branch → worktree remove
```

If checks fail, the agent fixes them and merges. If the agent cannot merge (genuine block — needs human review per repo policy), it reports `merged: no` with the PR URL and the orchestrator escalates. Branches do not accumulate.

### Card-lifecycle contract — every task has a filled card

**Every task in the plan has a per-task card at `<cards-dir>/<id>.md`. The card is created at scaffold time and filled by the executor agent during work.**

The executor MUST:
1. Before dispatching agent for task X.Y: verify `<cards-dir>/X.Y.md` exists. If not, copy from `_template.md` and pre-fill Description + Acceptance from the plan/functional-spec.
2. Pass the card path to the agent. The agent's prompt addendum says "your task card is at `<cards-dir>/X.Y.md` — read it first, fill it as you go".
3. After the agent reports back, verify card has filled: Implementation, Tests (3 sub-sections — unit/integration/e2e), Changelog entry. If not, the task is not done — re-dispatch with explicit "fill the card sections".

A task is `[x]` only when its card has filled Implementation, all three Tests sub-sections, and a Changelog entry, AND the PR is merged. All gates must pass.

### Test-coverage contract — every task ships with three tiers

**Every code-shipping task delivers three test tiers, written by the SAME agent that shipped the task:**

1. **Unit tests** — pure logic, helpers, validators, formatters
2. **Integration tests** — module boundaries, framework boundaries, DB queries
3. **End-to-end test (≥1 spec)** — exercises the task's Acceptance criteria via the user-visible flow

Why same-agent: separation of concerns is fine in human teams; in agent runs, the only agent with full context for the task's acceptance criteria is the one that just shipped it. Handing off "tests as a follow-up task" means tests that don't actually validate the original intent.

#### Hard checks the executor enforces on each subagent report

Reject the report (and re-dispatch the task) when any of these hold:

| Check | Failure mode | Executor action |
|-------|--------------|-----------------|
| `tests_written.unit` is missing or zero AND `tests_skipped_with_reason.unit` is empty | Agent shipped logic without unit tests | Re-dispatch: "Write unit tests for the logic you added in `<files>`. See subagent-prompt Step 6a." |
| `tests_written.integration` is missing or zero AND no skip reason | Agent shipped boundary code without integration tests | Re-dispatch: "Write integration tests for the boundaries in `<files>`. See subagent-prompt Step 6b." |
| `tests_written.e2e` is zero AND no skip reason | Agent shipped a user-visible feature without an e2e | Re-dispatch: "Write at least one e2e spec validating the card's Acceptance criteria. See subagent-prompt Step 6c." |
| `gates.unit`, `gates.integration`, or `gates.e2e` is `fail` | Tests are present but failing | Re-dispatch: "Tests fail — fix the underlying code or test, do not skip the test to ship." |
| `simplify_ran: no` | Mandatory pass skipped | Re-dispatch with `simplify` reminder |
| `frontend_design_ran: no` AND task touched UI files | Mandatory UI pass skipped | Re-dispatch with `frontend-design:frontend-design` reminder |
| `merged: no` AND no documented human-review reason | PR didn't make it to `main` | PR-finisher subagent OR re-dispatch |
| `card_filled: no` | Implementation/Tests/Changelog sections empty | Re-dispatch: "Fill the card before reporting." |

#### Acceptable test-skip reasons (whitelist — anything else gets rejected)

| Tier | Acceptable skip reason value (use exact strings) |
|------|--------------------------------------------------|
| `unit` | `pure-config-no-logic` (next.config.ts, env files, lockfiles) |
| `integration` | `pure-presentation-no-boundaries` (rare — even prop pass-through has boundaries) |
| `e2e` | `no-user-visible-surface` (lib utilities, internal type packages, build scripts) |
| any | `docs-only-task` (the master-plan marks this with `[-]` from scaffold time) |

If the agent invents a different skip reason, reject. Whitelist exists to prevent "tests felt expensive" rationalizations.

### Wave contract — execute wave-by-wave per the plan's Waves table

**The plan's `Waves` table is the cohort iteration unit. Read it first; iterate through it.**

Every well-formed master-plan has a `Waves` section like:

```
**Starting point:** _W1 — Foundation_

- **W0** — _Phase 0 (planning, sequential, already done)_
- **W1** — Foundation: 1.1 ∥ 1.2 ∥ 1.3 → 1.4 ∥ 1.5
- **W2** — Marketing core: 2.1 ∥ 2.2 ∥ 2.3
- **W3** — Treść i lokalność: 3.1 ∥ 3.2 → 3.3 ∥ 3.4 ∥ 3.5 → 3.6 ∥ 3.7
...
```

Inside a wave, `∥` = parallel (dispatch concurrently), `→` = sequence (wait for left to merge before right starts). Across waves: never start W(N+1) until W(N) is fully `[x]`. Skip a wave only if the user explicitly says "stop after wave M".

If the plan has NO Waves table, you stop and ask `03-scaffold-ai-canon` to add one — don't guess wave structure on the fly.

### What the executor MUST NOT do

- ❌ Write product code itself in Mode A (orchestration mode). The main agent only orchestrates. All code happens in subagents.
- ❌ Bundle 2+ task IDs into one agent dispatch
- ❌ Dispatch parallel agents into the same working tree
- ❌ Mark a task `[c]` or `[x]` if its card lacks Implementation/Tests/Changelog
- ❌ Skip the simplify pass before commit (every subagent runs `Skill('simplify')` on its diff)
- ❌ Skip the frontend-design pass for any task with `.tsx`/`.jsx`/`.css` changes (every UI subagent runs `Skill('frontend-design:frontend-design')` before commit)
- ❌ Skip PR merge — branches do not accumulate
- ❌ Broad-kill processes (`pkill`, `docker rm <wildcard>`) — you'll wipe other agents' servers/containers

## When to Use

Trigger this skill when the user asks to:
- Execute / finish / ship "the plan" or "the master plan" (Mode A)
- Run all remaining tasks in a specific plan file (Mode A)
- Orchestrate parallel work on plan tasks via subagents (Mode A)
- Resume an in-progress plan execution (Mode A)
- Execute a single named task / plan point (e.g. "do task 4.1", "execute point 5.2") (Mode B)

Do NOT trigger this skill for:
- Plans that are still being authored (use a planning skill instead)
- Code review of an already-completed plan (use `06-plan-reviewer` instead)
- Ad-hoc work that isn't tied to a plan task

## Step 0 — Choose the Mode

Decide before any other work:

- **One task explicitly named** (id, row text, or `<id>.md` card filename) → **Mode B** (single-task, in-process). Skip the cohort-loop steps; jump to *Single-Task Mode* below after Step 1.
- **No task named, or "the plan" / "all of it" / "remaining tasks"** → **Mode A** (orchestration). Run Steps 1–10 below.
- **Ambiguous** ("execute the plan starting from 4.1") → ask which mode the user means. Don't assume.

Both modes share Step 1 (resolve the plan), Step 3 (detect project conventions), and the lifecycle in `references/subagent-prompt.md`. The mode only changes who runs the lifecycle and how many tasks at once.

## Step 1 — Resolve the Plan

The plan file is the authoritative scope-and-sequencing document for this run. Resolve it before doing anything else.

### Default (no path given)

Search for a master plan in this order. Stop at the first hit:

1. `./.ai/master-plan.md`
2. `./master-plan.md` (repo root)
3. `./plans/master-plan.md`
4. `./docs/master-plan.md`
5. Any `*master-plan*.md` reachable via `find . -maxdepth 3` if none of the above hit

If multiple distinct candidates exist, **stop and ask the user which to use** — do not guess.

### Explicit path given

When the user passes a path (e.g. `.ai/plans/2026-04-21-launch-completeness/`), resolve it like this:

| Input shape | Resolution |
|---|---|
| Path is a `.md` file | Treat as the plan file. Cards directory (if any) = sibling `<stem>-points/`. |
| Path is a directory | Look for a single `.md` file whose stem matches the directory name. Cards directory = sibling `<dirname>-points/` if present. |
| Ambiguous / no match | **Stop and ask the user to clarify.** Do not guess. |

The "plan slug" = the plan-file stem (or directory name). Use it for the findings-log filename and worktree branches.

### Required-files report

If any of the following are missing or wrong, **STOP and tell the user concretely** before dispatching anything:

- The plan file doesn't exist → `"I couldn't find a plan file. I searched: <list>. Run /03-scaffold-ai-canon first."`
- The plan file exists but is empty or contains no checkboxes → `"The plan file at <path> has no [ ]/[x] tasks. I can't determine scope."`
- The plan exists but has **no Waves table** → `"Plan exists but lacks a Waves table. The executor needs explicit cohort structure. Either add waves manually or invoke /03-scaffold-ai-canon to regenerate the plan with waves."`
- Cards directory doesn't exist (and the plan references cards) → create it from `_template.md` (one card per task ID) before dispatching.
- A `CLAUDE.md` / `AGENTS.md` / `GEMINI.md` exists at repo root but mentions a plan layout that contradicts what was found → flag the inconsistency to the user before proceeding.

A repo without per-task cards is supported only if the plan file rows contain enough acceptance-criteria text to brief an agent — usually they don't. Default expectation: cards exist.

## Step 2 — Read the Plan and Build the Wave-by-Wave Queue

1. Read the plan file in full.
2. Read repo guidelines: `CLAUDE.md` / `AGENTS.md` / `GEMINI.md` at repo root, if present.
3. Locate the **Waves** section (or "Fale" in Polish projects). This IS the iteration unit.
4. Build the queue **wave-by-wave**: for each wave W1, W2, …
   - Parse `∥` / `→` markers between task IDs
   - Tasks within a `∥` group are dispatched concurrently
   - Tasks separated by `→` are sequenced (wait for left wave to fully merge before dispatching right)
5. Skip tasks already `[x]`.
6. Honor explicit "Prerequisites" / "Depends on" / "Zależności" notes per task (these may add cross-wave constraints).
7. Note tasks the plan itself marks as human-gated / deferred / out-of-scope. These are exempt from auto-execution.

## Step 3 — Detect Project Conventions

Before dispatching any subagent, do a one-time read pass to brief the cohort accurately. Check:

- **Package manager**: `pnpm-lock.yaml` (pnpm), `yarn.lock` (yarn), `package-lock.json` (npm), `bun.lockb` (bun), `pyproject.toml` + `uv.lock` (uv), `requirements.txt` (pip), `Cargo.lock` (cargo), `go.mod` (go).
- **Verification commands**: parse `package.json` scripts for `typecheck`, `lint`, `build`, `test`, `test:e2e`. Note what's available; pass actual script names.
- **Branch policy**: `main` vs `master`; trunk-based vs gitflow. Read repo `git log` briefly. Note which branch PRs target.
- **Worktree convention**: project's `CLAUDE.md` may dictate (e.g. `../<repo>-<id>`). If silent, default to `../<repo-name>-<task-id>`.
- **Per-worktree gitignored files**: any `.env.local` / config / SDK token files that don't carry across worktrees. List them so subagents bootstrap correctly.
- **Shared local infrastructure**: docker compose stacks, local databases, dev-server ports — these constrain concurrency. Note ports the project's dev/test commands use by default.
- **UI surface check**: do any of the plan tasks touch `.tsx`/`.jsx`/`.css`/`.scss`? If yes, every such task's subagent will additionally invoke `Skill('frontend-design:frontend-design')` before committing — this is a project-level fact for the cohort prompts.

Pass the relevant findings to each subagent verbatim in its prompt addendum (Step 5).

## Step 4 — Pick the Next Cohort (within the current wave)

From the head of the current wave's `∥` group (parallel tasks), pick 1–N tasks to dispatch concurrently. **One task = one agent**, no exceptions. Apply these caps:

- **Cap at 3 concurrent when any subagent will run shared-state e2e tests** (e.g. tests that reset a single shared database). Stagger their reset phases via `SKIP_DB_RESET=1` if the project documents that escape hatch.
- **Cap at 2 concurrent when two tasks touch the same files** (shared layout shells, type packages, config). Read the cards before fanning out.
- **When in doubt, fewer agents.** Merge conflicts and broken local stacks cost more than serial execution.
- **Prefer larger / "substantial" tasks first within a wave** — they take longer; quick-wins fill in parallel.
- **Never broad-kill processes** (`pkill`, `docker rm <wildcard>`) — you'll wipe other agents' servers/containers. Only target the specific PID/container you started.

If the current wave's parallel group is fully dispatched and you're now waiting for `→` sequenced tasks, complete the wait before dispatching the next cohort.

## Step 5 — Dispatch the Cohort (one subagent per task)

Spawn the cohort in a single message with multiple `Agent` tool calls in parallel. For each subagent, pass:

1. The full **subagent prompt template** from `references/subagent-prompt.md` — verbatim. This is the lifecycle the subagent must follow.
2. The resolved plan file path, cards directory path, plan slug.
3. **The specific task identifier** — exactly one ID per agent. NEVER multiple.
4. The path to the per-task card: `<cards-dir>/<id>.md`. Verify it exists; if not, create it from `_template.md` with Description + Acceptance pre-filled.
5. The worktree convention to use: `../<repo>-<plan-slug>-<id>` (or whatever `CLAUDE.md` documents).
6. The branch convention to use: `feature/<plan-slug>-<id>-<short-slug>`.
7. Any sequencing constraint ("wait for task 3.4's PR to merge before pushing — your prereq").
8. The detected project conventions from Step 3 (package manager, verification commands, gitignored env files, dev-server port assignment).
9. **Required skills the subagent MUST invoke during its lifecycle:**
   - `Skill('simplify')` — runs on diff before commit. NON-NEGOTIABLE.
   - `Skill('frontend-design:frontend-design')` — runs after implementation if the task touches `.tsx`/`.jsx`/`.css`/`.scss`. The subagent decides based on its own diff.
10. Any user override from the original request (e.g. "skip frontend tasks", "dry run — do not commit", "concurrency = 1").

The dispatching call shape (illustrative — actual call uses the `Agent` tool):

```
Agent({
  description: "Execute task <id>",
  subagent_type: "general-purpose",
  prompt: <subagent-prompt-template> + <addendum with the 9 fields above>
})
```

## Step 6 — Collect Reports

Each subagent ends with a structured `=== SUBAGENT REPORT ===` block (format in `references/subagent-prompt.md`). Parse it. Update an in-memory ledger of task → status → branch → PR → merged → tests-written → card-filled.

Apply the **Test-coverage contract** hard checks (see contract section above) plus:
- `merged: yes` is required for `status: done`.
- `simplify_ran: yes` is required.
- `frontend_design_ran: yes` is required IF the task touched UI files.
- `card_filled: yes` is required (Implementation, Tests with 3 sub-sections, Changelog).
- `tests_written.unit + integration + e2e` totals must be ≥3 (or the corresponding skip reasons present from the whitelist).
- All `gates.*` for tiers that were written MUST be `pass`.

Reject any report that fails these checks and dispatch a re-do per the table in the Test-coverage contract. **Don't paper over failures.** A reported `merged: yes` with `tests_written.e2e=0` and no skip reason is a violation; the task isn't actually done.

If `merged: yes`, the main branch advanced — warn the next cohort to rebase on push (the subagent prompt already handles this; just be aware).

## Step 7 — Write Findings Log

Append cohort findings to `.ai/agent-logs/findings-<YYYY-MM-DD>-<plan-slug>.md` (create if missing). Format details in `references/orchestration-details.md`.

**Only the main agent writes here** — subagents do not. Audience is the human operator. Keep it terse and decision-grade. One block per cohort.

Commit this log directly on the integration branch (no PR — it's a main-agent log, not product code).

## Step 8 — Re-plan on Failure or Block

- **`status: blocked` on a prereq** → launch the prereq first, then retry the blocked task.
- **`status: failed` on a task bug** → re-dispatch with a pointer to the specific failure. Do not blindly re-run the same prompt.
- **`status: failed` on a genuinely impossible requirement** → skip with written justification in the task card / plan row. Only after a real attempt.
- **Compliance failure** (no simplify, no frontend-design, no card fill, no PR merge) → re-dispatch with explicit reminder. Do not paper over it.

## Step 9 — Loop wave-by-wave

After the current cohort's reports are in:
1. If the current wave still has unmerged tasks (the `→` sequenced ones), dispatch them now.
2. If the current wave is fully `[x]`, advance to the next wave.
3. Go back to Step 4.

Do not start wave N+1 until wave N is fully `[x]` (unless the user explicitly says "skip wave N").

## Step 10 — Final Sweep

When the queue is exhausted:
1. Verify plan-file statuses match reality (`grep -E '^\s*-?\s*\[[ ~ct-]\]' <plan-file>` should only show the plan's documented exempt tasks plus any explicit skips).
2. Verify every `[x]` task has a filled card (Implementation, Tests, Changelog).
3. Verify every `[x]` task has a merged PR linked from its card.
4. Append a `## <ISO timestamp> — session summary` block to the day's findings log: wave-by-wave recap + explicit skip list + audit pass/fail.
5. Print the same summary to the user.

## Single-Task Mode (Mode B)

When the user named exactly one task, skip the orchestration loop and execute it yourself. Use the same lifecycle as `references/subagent-prompt.md` — *you* are the worker now. Adjustments from the orchestration flow:

1. **Build full context up front, in parallel.** Read in one batch (parallel tool calls): the task card (`<cards-dir>/<id>.md`), the plan file, `CLAUDE.md` / `AGENTS.md`, plus any `.ai/*.md` the task or repo guidelines reference (PRD, design, schema, env strategy, tech stack). Don't read serially.
2. **Prerequisite handling — warn and ASK, don't auto-block.** If the task lists prereqs that aren't yet `[x]`, surface them to the user and ask whether to proceed anyway. Single-task mode is often used precisely to push past a prereq the user has decided to defer; respect their judgment.
3. **Worktree decision — explain your call.** Default: new worktree if the task touches source code. Exception: pure documentation/planning tasks (only `.ai/*.md`, `docs/*.md`, README updates) can stay on the current branch. State your reasoning either way before starting work.
4. **Run the lifecycle.** Follow `references/subagent-prompt.md` steps 2 (bootstrap) → 11 (commit/push/PR/merge/card-fill) yourself. The whole document applies — verification, simplify pass, frontend-design pass (if UI), status updates, card lifecycle, planning-doc upkeep.
5. **No `=== SUBAGENT REPORT ===` block.** That format exists so the orchestrator can parse a returning subagent. In Mode B there's no orchestrator parsing your output — give the user a normal end-of-turn summary instead: what shipped, key decisions, anything they should verify.

The findings log (Step 7) is **orchestration-only**. Don't write to `.ai/agent-logs/findings-*.md` from single-task mode — but DO update the per-task card.

## DRY — Non-negotiable

- Every subagent runs `Skill('simplify')` on its diff before marking the task done.
- Every UI-touching subagent runs `Skill('frontend-design:frontend-design')` after implementation.
- When dispatching a cohort that touches overlapping files, call it out in each assignment so subagents coordinate via imports rather than copy-paste.
- If a report shows a subagent created a helper that already exists elsewhere, demand a follow-up dedup before accepting `merged`.
- Watch for pattern duplication across reports (two subagents both adding `formatDuration()`) and dispatch a cleanup subagent if needed.

## What to Read Next

- `references/subagent-prompt.md` — the verbatim lifecycle prompt to inject into each subagent.
- `references/orchestration-details.md` — concurrency caps, findings-log template, resource-conflict table, recovery patterns.

## Overrides and Modes

The user may include overrides with the request. Pass these into every subagent's addendum verbatim. Common ones:

- `"skip frontend tasks"` — narrow the queue.
- `"run sequentially only"` — concurrency = 1.
- `"stop after wave 2"` — partial execution.
- `"dry run"` — subagents follow the lifecycle but stop before commit/push/PR.

If an override conflicts with a hard rule (e.g. "disable simplify pass"), flag it to the user and ask before proceeding.
