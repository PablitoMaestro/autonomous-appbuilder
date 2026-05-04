---
name: 06-plan-reviewer
description: Review every completed task of a markdown checklist plan via one subagent per task. Each subagent re-runs baseline tests, walks the feature in a browser, applies surgical improvements (DRY, glue, UX, thin tests), and reports a verdict. Symmetric to `05-plan-executor` — same plan resolution and conventions detection, different lifecycle (review vs build). Use whenever the user says "review the plan", "audit completed tasks", "DRY pass over what shipped", "quality-check the build", "review every [x] task", "improve the master plan", or wants per-task post-build review — even if they don't name the plan file.
---

# Plan Reviewer

> **Invocation contract:** This skill is invoked via `Skill('06-plan-reviewer')` — typically by `00-build-app` at Phase 6 (off by default; on with `review=on` or `review=full`), but also standalone. The caller MUST NOT dispatch review subagents directly from main context. This skill owns the per-task audit lifecycle: each `[x]` task gets one review subagent that re-runs gates, verifies 3-tier test coverage, runs `simplify` (and `frontend-design` for UI), applies surgical fixes within budget, and merges its own PR if changes were needed.

Drive a markdown-checklist implementation plan into a clean state by reviewing each completed task. Symmetric to `05-plan-executor`:

- **Mode A — Full-plan orchestration.** Main agent stays on `main`, dispatches one review subagent per `[x]` task, parses returning reports, watches for cross-task signals.
- **Mode B — Single-task review.** Main agent reviews one named task end-to-end, in-process.

Same Step 0 dual-mode choice, same plan resolution as `05-plan-executor`. Different lifecycle: review-shaped, not execute-shaped.

## When to Use

Trigger this skill when the user asks to:
- Review / audit / improve "the plan" or all completed tasks (Mode A)
- DRY pass / quality pass over a finished or near-finished build (Mode A)
- Run a per-task review loop with surgical improvements (Mode A)
- Review one named task (e.g., "review task 4.1") (Mode B)

Do NOT trigger this skill for:
- Plans that are still being authored (use a planning skill)
- Tasks that haven't been completed yet (use `05-plan-executor`)
- Codebase-wide audits unrelated to a plan (use `pr-review-toolkit:review-pr` or similar)

## Priority — DRY + Simplicity First

Every other check serves DRY. A task that works but is tangled, copy-pasted, or over-engineered is **not** a clean audit.

Fix in this order:
1. **DRY + over-engineering** — duplicated helpers/types/JSX, speculative abstractions, dead flags, `any`, orphan TODOs.
2. **Simplicity** — over-abstracted layers, unnecessary indirection, clever generics. Inline or flatten unless the abstraction pays for itself.
3. **Integration gaps** — feature complete but unreachable, wiring missing, handoffs broken.
4. **Correctness + UX sharp edges** — wrong error surfacing, missing states, backwards handlers.
5. **Test gaps** on risk surface — add a thin regression test.
6. **Stale planning docs** — reconcile `.ai/*.md` with the code.

Quality is a floor, not a ceiling: don't regress UX or break subtle behavior in pursuit of DRY. If a clean refactor would damage the feature, escalate via `issues_found` instead of forcing the change.

## Step 0 — Choose the Mode

Decide before any other work:

- **One task explicitly named** (id, row text, or `<id>.md` card filename) → **Mode B** (single-task, in-process). Skip the cohort loop; jump to *Single-Task Mode* below after Step 1.
- **No task named, or "the plan" / "all completed"** → **Mode A** (orchestration). Run Steps 1–10 below.
- **Ambiguous** ("review the plan starting from 4.1") → ask which mode the user means. Don't assume.

## Step 1 — Resolve the Plan

Same logic as `05-plan-executor`. Default search order if no path given:

1. `./.ai/master-plan.md`
2. `./master-plan.md`
3. `./plans/master-plan.md`
4. `./docs/master-plan.md`
5. `find . -maxdepth 3 -name '*master-plan*.md'`

Multiple distinct candidates → **stop and ask**. Empty plan or no checkboxes → stop and report concretely.

If the user passes an explicit path:
- File path → treat as plan file; cards directory = sibling `<stem>-points/` if present.
- Directory → look for matching `.md` file; cards directory = sibling `<dirname>-points/`.
- Ambiguous → stop and ask.

A repo without per-task cards is supported — plan-file rows are the task ledger. State this up front: "Proceeding without per-task cards; using plan-file rows."

## Step 2 — Build the Review Queue

1. Read the plan in full.
2. Read repo guidelines: `CLAUDE.md` / `AGENTS.md` / `GEMINI.md`.
3. Build the queue: every **`[x]`** task in the plan, in plan-row order (or wave order if the plan has its own `### Fale` / `### Waves` table).
4. Skip tasks the plan marks as human-gated, deferred, exempt, or explicitly excluded by `$ARGUMENTS`.

Note: review queue is the **opposite** of the execute queue. Executor builds non-`[x]`; reviewer audits `[x]`.

## Step 3 — Detect Project Conventions

Before dispatching, detect:

- **Package manager** (lockfile-based): `pnpm`, `npm`, `yarn`, `bun`, `uv`, `pip`, `cargo`, `go`.
- **Verification commands** from `package.json` scripts: `typecheck`, `lint`, `build`, `test`, `test:e2e`. Note what's available — pass actual script names.
- **Branch policy**: `main` vs `master`; PR target.
- **Worktree convention**: project's `CLAUDE.md` may dictate (e.g., `../<repo>-<id>`). Default: `../<repo-name>-review-<task-id>`.
- **Per-worktree gitignored files**: `.env.local`, SDK tokens — list these so subagents bootstrap correctly.
- **Shared local infrastructure**: docker compose, local DBs, dev-server ports — these constrain concurrency.

Pass relevant findings verbatim to each subagent in its addendum.

## Step 4 — Pick the Next Cohort

From the head of the queue, pick 1–10 tasks under these caps:

- **Cap at 2 concurrent for shared-state e2e** (single shared DB). Leader runs reset; follower uses a project-defined skip-reset env var (e.g., `SKIP_DB_RESET=1` — exact name documented in `.ai/environments.md`) and a unique `PORT`.
- **Cap at 3 for non-e2e reviews** (static checks + browser spot-check only).
- **Cap at 2 when tasks share files** (shared layouts, type packages).
- **When in doubt, fewer agents.** Merge conflicts and broken local stacks cost more than serial execution.
- **Never broad-kill processes** (`pkill`, `docker rm <wildcard>`).

## Step 5 — Dispatch the Cohort

Spawn the cohort in a single message with parallel `Agent` tool calls. For each subagent:

1. The full **review subagent prompt template** from `references/review-subagent-prompt.md` — verbatim.
2. The resolved plan path, cards directory path (or "none"), plan slug.
3. The task id.
4. Sequencing constraint if any ("wait for X to merge before pushing").
5. Detected project conventions (package manager, verify commands, gitignored env files, worktree convention, port assignment).
6. `$ARGUMENTS` overrides verbatim.

## Step 6 — Collect Reports

Each subagent ends with a `=== REVIEW SUBAGENT REPORT ===` block (format in `references/review-subagent-prompt.md`). Parse it. Update an in-memory ledger of task → verdict → branch → PR → merged.

Verdict vocabulary (`clean / improved / extended / issues_found / blocked / failed`) defined in `references/review-subagent-prompt.md` § Verdicts — single source of truth.

If `merged: yes`, the main branch advanced — warn next cohort to rebase.

## Step 7 — Write Findings Log

Append to `.ai/agent-logs/review-findings-<YYYY-MM-DD>-<plan-slug>.md` (or `agent-logs/...` if no `.ai/`). **Main agent only.** Audience: human operator. Keep it terse, decision-grade.

Each cohort entry:

```markdown
## <ISO timestamp> — review cohort <W?>.<n> (tasks: a, b, c)

**Verdicts:** <one line per task>
**Improvements:** <1-2 sentences per `improved` task>
**Gaps closed:** <bullets — glue added per task; "none">
**Features added (`extended`):** <bullets — task + feature + why + card-updated; "none">
**Issues found but not fixed:** <bullets — what the human should look at>
**Cross-task signals:** <duplication patterns, shared-helper extractions; "none">
**Concurrency notes:** <e2e serialization, port issues, reset races>
**Progress:** <e.g., "W3 done; W4 next">
```

Commit directly on the integration branch (no PR — main-agent log).

## Step 8 — Cross-Task DRY Watch

After each cohort, scan reports for patterns:
- Two+ subagents flagging the same duplication → dispatch a dedicated cleanup subagent for one central extraction. Don't let each reviewer extract its own variant.
- Two+ reports flagging the same UX bug → escalate to human; might be a design-system gap.
- Pattern of `clean` verdicts for a whole wave → consider whether the queue is too generous; some tasks may not need review.

## Step 9 — Re-plan on Block / Failure

- **`status: blocked`** on a prereq → launch the prereq's review first, then retry the blocked task.
- **`status: failed`** on a regression introduced by review → revert the review's commit; do not blindly re-dispatch.
- **`status: issues_found`** is a valid terminal state — capture findings, move on.

## Step 10 — Loop and Final Sweep

Loop Steps 4–9 until the queue is exhausted. Then:

1. Verify plan-file statuses: every previously-`[x]` task should still be `[x]` (review never downgrades).
2. Append a `## <ISO timestamp> — session summary` block to the day's findings log: cohort recap + skip list + escalations.
3. Print the same summary to the user.

## Single-Task Mode (Mode B)

When the user named one task, skip the cohort loop and review it yourself.

1. **Build context up front, in parallel** — task card, plan file, `CLAUDE.md` / `AGENTS.md`, plus `.ai/*.md` the task touches.
2. **Worktree decision — explain your call.** Default: review worktree (`../<repo>-review-<id>`). Exception: pure docs reviews can stay on the current branch. State your call before starting.
3. **Run the lifecycle** from `references/review-subagent-prompt.md` yourself (steps 2 → 11). The whole document applies.
4. **No `=== REVIEW SUBAGENT REPORT ===` block.** Give the user a normal end-of-turn summary instead: verdict, what shipped, anything they should verify.

The findings log is **orchestration-only**. Don't write to it from Mode B.

## DRY — Non-negotiable

- Every subagent runs the `simplify` skill on its diff. `simplify_ran: no` in a report = send it back, incomplete.
- Two+ reports flagging the same duplication → cleanup subagent, one central extraction.
- `clean` is valid and expected. If every task returns `improved`/`extended`, subagents are *rewriting*, not reviewing — push back.
- Less code, not differently-organized code. Moving duplication around doesn't count as a fix.

## Cross-cutting issues without fan-out

This skill deliberately does NOT fan out specialized reviewer agents (`pr-review-toolkit:code-reviewer`, `silent-failure-hunter`, `type-design-analyzer`, etc.) over the whole codebase in parallel. Reasons:

- The per-task subagent already sees the whole repo when reviewing its task.
- Cross-task signals are caught by the main agent's cross-cohort DRY watch (Step 8).
- Fan-out reviewers tend to produce churn (each rewrites their slice); per-task review with a surgical change budget produces fewer, higher-confidence improvements.

When a subagent's review checklist warrants it (e.g., security-touching task), the subagent **invokes** the relevant specialized reviewer as a tool inside its lifecycle — not as a parallel process. See `references/review-subagent-prompt.md` step 5.

## Reference Files

- `references/review-subagent-prompt.md` — verbatim lifecycle prompt for each subagent.

## Overrides and Modes

`$ARGUMENTS` overrides pass verbatim into every subagent addendum. Common ones:
- `"only W4 W5"` — narrow queue
- `"no e2e"` — skip e2e gates, keep browser spot-check
- `"serial only"` — concurrency = 1
- `"be ruthless on DRY"` — stretch budget for DRY-only, still no architectural rewrites
- `"dry run"` — run lifecycle but stop before commit/push/PR

If an override breaks a hard rule (e.g., "5 parallel e2e"), refuse and flag.

## Why this skill is shaped this way

- **Symmetric to `05-plan-executor`.** Same plan resolution, same conventions detection, same dispatch shape, same report parsing — only the *intent* differs (build vs audit). Symmetry makes both skills easier to reason about.
- **Per-task review beats codebase fan-out.** Surgical change budget per task prevents reviewer scope-creep; cross-task signals via main-agent watch catch what fan-out would catch, with less churn.
- **`[x]` stays `[x]`.** Review never downgrades status. If a task is broken, escalate via `issues_found`; the human (or executor) handles the actual fix.
- **DRY is the hill.** Every other check serves it. Without DRY discipline, parallel execution produces a codebase full of near-duplicates that no one wants to maintain.
