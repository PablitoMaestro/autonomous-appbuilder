# Build-app skills — index

The seven skills in this directory compose into the `/00-build-app` orchestrator. Each is also independently invocable as `Skill('<name>')` for targeted use without the full pipeline.

> **TL;DR for the impatient:** start with `Skill('00-build-app')` and let it route. Sub-skills are invoked from there. Don't read all the SKILL.md files end-to-end — let the orchestrator load them as it needs.

## The seven skills

| # | Skill | What it owns | Invoked from |
|---|-------|--------------|--------------|
| 0 | [`00-build-app`](./00-build-app/SKILL.md) | Orchestrator. 7 phases, 2 modes, decision-log audit, 3 strategic checkpoints. | User (slash command or natural language) |
| 1 | [`01-discover-idea`](./01-discover-idea/SKILL.md) | `.ai/PRD.md` + master-plan stub (Phase 0) + CLAUDE.md skeleton | `00-build-app` Phase 1 |
| 2 | [`02-design-create-n-htmls`](./02-design-create-n-htmls/SKILL.md) | N parallel HTML design explorations under `docs/design-ideas/` | `00-build-app` Phase 2 |
| 3 | [`03-scaffold-ai-canon`](./03-scaffold-ai-canon/SKILL.md) | Light `.ai/` docs + EXPANDED master-plan with Waves table + per-task cards (auto-chained dispatch) | `00-build-app` Phase 3 |
| 4 | [`04-bootstrap-env`](./04-bootstrap-env/SKILL.md) | Local dev stack boots green; `.env.local` populated; runbook seeded | `00-build-app` Phase 4 |
| 5 | [`05-plan-executor`](./05-plan-executor/SKILL.md) | Per-task subagent dispatch, worktrees, 3-tier tests, simplify pass, PR-merge-to-main | `00-build-app` Phase 5 |
| 6 | [`06-plan-reviewer`](./06-plan-reviewer/SKILL.md) | Per-`[x]`-task audit subagents, coverage check, surgical fixes | `00-build-app` Phase 6 (opt-in) |
| 7 | [`07-smoke-test-app`](./07-smoke-test-app/SKILL.md) | 4 narrow gates: dev boots, hydrates, auth round-trips, golden CRUD round-trips | `00-build-app` Phase 7 |

## Universal contracts (shared across all skills)

These rules apply to every skill in the pipeline. Each skill restates the parts relevant to it; the canonical statement is here.

### 1. Invocation contract

**Every skill is invoked via the `Skill` tool. The caller never reproduces the skill's work inline.** Each skill's `> Invocation contract:` block at the top of its `SKILL.md` documents what the caller MUST NOT do. The orchestrator's compliance audit (Step 6 of `00-build-app`) verifies every phase has its `Skill('XX')` invocation in the message history.

### 2. Test-coverage contract (Phase 5 + 6)

**Every code-shipping task delivers three test tiers:** unit + integration + e2e. Skip only via whitelisted reason (`pure-config-no-logic`, `pure-presentation-no-boundaries`, `no-user-visible-surface`, `docs-only-task`). Defined in `05-plan-executor/SKILL.md § Test-coverage contract`; enforced by `06-plan-reviewer` Step 5a.

### 3. Worktree + PR contract (Phase 5 + 6)

**Every parallel agent OR every code-touching task in master-plan phases 1+ runs in its own `git worktree` and merges its own PR to `main`.** Cleanup / recovery / wave-fix / log / doc-sync subagents may share the main tree (sequential, single-purpose). Defined in `05-plan-executor/SKILL.md § Isolation contract` and `§ Integration contract`.

### 4. Granularity contract (Phase 5)

**One master-plan task = one agent dispatch.** Bundling tasks is forbidden. The dispatching agent keeps the granularity of the plan.

### 5. Required skill invocations inside subagents

Every implementing or reviewing subagent MUST invoke:

- `Skill('simplify')` on its diff, before commit. Non-negotiable.
- `Skill('frontend-design:frontend-design')` if its diff touches `.tsx`/`.jsx`/`.css`/`.scss`/`.html` files.

These are the only two `Skill()` calls expected from subagents; the orchestrator handles all phase-level skill routing.

## Reading order if you ARE going to dive in

For someone new to this skill set who wants to understand it, not just run it:

1. `00-build-app/SKILL.md` — the spine. Compliance contract + Red Flags table at the top tells you what NOT to do; the phase loop tells you what each `Skill('XX')` produces.
2. `05-plan-executor/SKILL.md` — the longest phase, the most contracts. Granularity / Isolation / Integration / Card-lifecycle / Test-coverage / Wave contracts all live here.
3. `05-plan-executor/references/subagent-prompt.md` — the 13-step lifecycle every code-shipping subagent follows. This is the contract that makes a task `[x]`.
4. `03-scaffold-ai-canon/SKILL.md § REQUIRED — Waves table` — the format the executor consumes.
5. `00-build-app/SKILL.md § Hooks layer` — what `.claude/settings.json` observes and how the audit works.

The other four skills (01, 02, 04, 07) are short and single-purpose; read their `SKILL.md` only when the orchestrator routes through them.

## Hooks layer (`.claude/settings.json`)

Five hooks observe `/00-build-app` runs and write JSONL to `.ai/agent-logs/hooks-<YYYY-MM-DD>.jsonl`:

| Event matcher | What gets logged |
|---------------|------------------|
| `PostToolUse / Skill` | Every `Skill('XX')` invocation |
| `PostToolUse / Write\|Edit` | Only writes to canonical artifacts (`.ai/{PRD,master-plan,functional-spec,site-map,techstack,environments,design,database-architecture}.md` or `apps/web/src/app/**/*.tsx`) |
| `PostToolUse / Agent` | Every subagent dispatch |
| `PostToolUse / Bash` | Only `git commit/push/merge/rebase`, `gh pr *`, `gh repo *` |
| `Stop` | One-line audit summary at end of turn (silent if no activity) |

The `Stop` hook prints `[build-app audit] Skill=N · canon-edits=N · subagents=N · git-ops=N` when there's any activity. Mismatched numbers (e.g., `canon-edits=8 but Skill=0`) signal main-context deviation — the audit is what makes silent shortcuts visible.

**Hook activation caveat:** the settings watcher only watches `.claude/` directories that existed at session start. After first creation of `.claude/settings.json`, open `/hooks` once OR restart the session for hooks to fire.

## Key conventions

- **Project language for `.ai/` docs follows `CLAUDE.md`.** Code, identifiers, commit messages stay English.
- **Master-plan checkboxes are the resume state.** Killing the orchestrator mid-run, switching contexts, or re-invoking days later all resume cleanly because artifacts ARE state.
- **`.ai/agent-logs/`** holds (in increasing time-resolution):
  - `run-summary-<YYYY-MM-DD-HHMM>.md` — **chronological orchestration timeline** for one `/00-build-app` run. ASCII tree, one line per dispatch/return, scannable top-to-bottom. The "what happened, in what order" view (format spec in `00-build-app/SKILL.md` § Step 2.6).
  - `autonomous-decisions-<date>.md` — load-bearing decisions with WHY (autonomous mode only).
  - `findings-<date>-<plan-slug>.md` — cohort engineering findings from `05-plan-executor`.
  - `review-findings-<date>.md` — `06-plan-reviewer` audit results.
  - `smoke-<date>.md` — `07-smoke-test-app` evidence.
  - `hooks-<date>.jsonl` — programmatic observability (every Skill/Agent/canon-edit/git-op tool call). Machine-readable.

  Markdown logs are project history (committed). The JSONL hook log is gitignored — it's a per-session firehose.

## Standalone use

Each skill works without the orchestrator:

- `Skill('01-discover-idea')` — write a PRD for a fresh project
- `Skill('02-design-create-n-htmls')` — N HTML design explorations for any page
- `Skill('05-plan-executor')` Mode B — execute one task from any markdown plan
- `Skill('06-plan-reviewer')` Mode B — review one shipped task
- `Skill('07-smoke-test-app')` — narrow runtime sanity check

`03-scaffold-ai-canon` and `04-bootstrap-env` are technically standalone-invocable too, but they assume locked PRD / techstack inputs respectively.

## What this directory does NOT contain

- Stack-specific code generation guidance — that lives in plugin skills (`vercel:nextjs`, `vercel:shadcn`, `supabase:supabase`, etc.) which subagents invoke when their task touches the relevant stack.
- Tool wrappers — `chrome-devtools-mcp`, `claude-in-chrome`, etc. are MCP servers, not skills. Subagents call them directly per the project's `CLAUDE.md § Browser tooling` table.
- Project-specific config — that lives in this project's `CLAUDE.md` and `.ai/*.md` docs, not here.
