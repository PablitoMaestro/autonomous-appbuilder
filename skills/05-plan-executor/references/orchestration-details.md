# Orchestration Details

Operational details the main agent needs while running a plan-execution cohort: concurrency caps, findings-log format, resource conflicts, recovery patterns.

## Concurrency Caps

Default: 1–10 concurrent subagents. Apply these caps based on what the cohort touches.

| Trigger | Cap | Why |
|---|---|---|
| Any subagent runs e2e tests against a shared local database | 3 | DB resets corrupt each other when run in parallel against one Postgres / one Mongo / one Redis. |
| Two tasks touch the same source files (shared layout, types package, config) | 2 | Merge conflicts on the same lines waste more time than serial execution. |
| Cohort needs unique dev-server ports | Number of free ports | If you only have 4 free ports configured, cap at 4. |
| Tasks have explicit prereq chains within the wave | Number of leaves | Don't dispatch a task whose prereq is also in this cohort. |
| Uncertain | Smaller | Merge conflicts and broken local stacks cost more than serial execution. |

### Port assignment for parallel dev servers

When dispatching ≥2 subagents that each run a dev server, assign each a unique `PORT`:

```
Subagent 1: PORT=3004
Subagent 2: PORT=3008
Subagent 3: PORT=3012
```

Pass the `PORT` value in the subagent's addendum. Forbid broad process kills (`pkill -f "next dev"`, `docker rm <wildcard>`) — they wipe other agents' servers.

### Shared-state e2e tests

If the project's e2e suite runs a `db reset` before tests (Supabase, Postgres, etc.), only **one** subagent in the cohort should run the reset; followers should opt into a "skip reset" mode if the project supports it. Pass this in the addendum:

> "You are subagent 2 of 3 running e2e in this cohort. Subagent 1 will run the DB reset; wait until it reports the reset has completed, then run your e2e with `<SKIP_RESET_ENV_VAR>=1`."

If the project doesn't have a skip-reset mode, **serialize** the e2e portion: cap concurrent e2e at 1 even if the cohort is larger.

## Findings-Log Format

Append to `.ai/agent-logs/findings-<YYYY-MM-DD>-<plan-slug>.md` (or repo-root `agent-logs/...` if no `.ai/`). Create the directory if missing. **Only the main agent writes here.**

Audience: the human operator reading tomorrow morning, not future agents. Be terse and decision-grade.

### Per-cohort entry

```markdown
## <ISO-8601 timestamp> — cohort <W?>.<n> (tasks: 4.1, 4.6, 4.8) — plan: <plan-slug>

**What shipped:** <1–2 sentences per task — what now works end-to-end>

**Key findings:**
- <gotcha / hidden constraint / surprising dependency>
- <library quirk, env issue, flaky test root cause>
- <decision I made without asking — trade-off taken and why>
- <something the human should verify manually before trusting>

**Cross-task signals:** <things that affect later waves — e.g. "4.1 `VideoPlayer` wrapper now authoritative; 5.2 must reuse"; "none" if strictly local>

**Risks / open questions:** <what I'm worried about, what I skipped, what needs human input>

**Progress:** <e.g., "W3 complete; W4 starts next" — one line>
```

### Final session summary (after the queue is exhausted)

```markdown
## <ISO-8601 timestamp> — session summary

**Waves completed:** W1, W2, W3, W4
**Tasks closed (`[x]`):** 4.1, 4.6, 4.8, 5.1, 5.2, …
**Tasks skipped (with justification):**
- 9.3 — human go/no-go checkpoint, deferred per plan exemption list
- 9.4 — pilot onboarding, requires real users

**Top cross-task patterns observed:** <2–3 bullets the human should know about going forward>
**Outstanding risks:** <2–3 bullets>
```

### Commit policy for the log

Commit findings logs directly on the integration branch (no PR — they're a main-agent log, not product code). Use a terse commit message:

```
chore(agent-log): cohort <W>.<n> — <plan-slug>
```

If the project requires PRs even for non-code commits, follow that convention instead — but optimize for not blocking the next cohort behind a PR review.

## Resource-Conflict Quick Reference

| Resource | Conflict risk | Mitigation |
|---|---|---|
| Single local DB instance (Postgres, Mongo, MySQL) | DB reset wipes all workers' state | Serialize e2e runs; only one reset per cohort; use skip-reset env if available |
| Dev server | Port collision on default port | Unique `PORT=` per worktree |
| Browser test runner caches | Browser download / artifact dir collisions | Each worktree has its own `node_modules` → its own cache. Fine in parallel. |
| Integration branch HEAD (`main`) | Advances on each merge | Later subagents must rebase before push. Warn the next cohort when a merge lands mid-flight. |
| Plan file inline statuses | Multiple subagents updating different rows | Each subagent updates only its own row. Merge conflicts here are the canary for cohort oversizing. |
| Docker containers | Shared between workers | Never broad-kill (`docker rm $(...)`). Target the specific container by name/id. |
| Per-worktree gitignored env files | Don't carry across worktrees | Subagent re-creates them in step 2 of its lifecycle. |

## Recovery Patterns

### Subagent reports `blocked` on a prereq

1. Note the prereq task id from the report.
2. Check whether the prereq is in the queue. If yes, dispatch it next (or alongside, if compatible). If no, ask the user where it stands.
3. After the prereq merges, re-dispatch the blocked task with a fresh worktree.

### Subagent reports `failed` on a task bug

1. Read the `issues` block — what specifically broke?
2. Re-dispatch with a pointer to the failure: "Task 4.6 failed last attempt because `<X>`. Investigate why and fix it." Don't blindly re-run the same prompt.
3. If the second attempt also fails, escalate: report to the user with both subagent reports and ask for guidance.

### Subagent reports `failed` on a genuinely impossible requirement

1. Confirm by reading the task card / plan row — is the requirement genuinely impossible, or is the subagent giving up too early?
2. If genuinely impossible: skip with written justification in the task card / plan row. Mark status as deferred/skipped per the plan's grammar.
3. If subagent gave up too early: re-dispatch with explicit pushback ("the requirement is X; here's why your last attempt didn't satisfy it; try approach Y").

### Cohort-wide pattern duplication

If two reports show subagents both creating helpers/components that already exist (or that overlap with each other), demand a follow-up dedup pass:

1. Don't accept `merged: yes` as final state — file a follow-up task in the queue.
2. Dispatch a cleanup subagent: "Tasks 4.1 and 4.6 both added `formatDuration()` helpers. Consolidate into a single shared module under `<conventional-shared-path>` and update both call sites."

### Mid-flight merge to integration branch

When `merged: yes` lands while other cohort subagents are still working:

1. Note the merge in your ledger.
2. Warn the next cohort in their addendum: "Integration branch advanced during your dispatch. Rebase on push before opening PR."
3. If subagents are far enough along that rebase risks conflict, let them push to their feature branches; resolve at PR time.

## Stop Conditions

Stop dispatching new cohorts when any of these happen:

- The queue is empty (final sweep applies — see SKILL.md step 10).
- The user explicitly stops the run.
- A failure cascade: 3+ subagents report `failed` in a single cohort with overlapping root causes — pause and ask the user before continuing. Likely the plan or the local stack has a deeper issue.
- Local infrastructure is broken (Docker daemon down, database unresponsive) and not recovering. Ask the user to repair before continuing — never broad-kill containers as a "fix."
