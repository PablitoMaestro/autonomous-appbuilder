---
name: 00-build-app
description: Build a complete production-ready app from an idea, an empty repo, or a partially-completed one. Orchestrates 7 phases (idea→PRD, design fanout, .ai/ canon scaffold, env provisioning, parallel-subagent execution of every plan task, optional autonomous review loop, smoke test) by composing single-purpose skills. Resumable from any phase via artifact-detection or `phase=` override. Two modes — supervised (3 strategic checkpoints) or fully autonomous (`gates=none`, agent decides everything, asks at most once). Use whenever the user says "build me an app", "ship the whole thing", "complete this project", "0 to hero", "finish the app", "run the master plan end-to-end", "I have an idea for X", "build a flashcard / game / dashboard / tool", or wants autonomous app generation — even if they don't name a stack or specify which phase.
---

# 00 — Build App (Orchestrator)

Take any web (or non-web) idea — or a partially-completed repo — through PRD, design, scaffolding, env provisioning, autonomous execution, optional review, and smoke test, until a production-ready app is running locally. One Claude session, two modes (supervised by default; fully autonomous when invoked with `gates=none`), resumable from any phase.

This skill is a **thin orchestrator**. All phase logic lives in seven sub-skills. **You MUST invoke each sub-skill via the `Skill` tool — never write phase deliverables yourself in main context.**

| # | Phase | Sub-skill (MUST be invoked) |
|---|---|---|
| 1 | Discovery (idea → PRD) | `Skill('01-discover-idea')` |
| 2 | Design exploration | `Skill('02-design-create-n-htmls')` |
| 3 | Scaffold the `.ai/` canon | `Skill('03-scaffold-ai-canon')` |
| 4 | Environment provisioning | `Skill('04-bootstrap-env')` |
| 5 | Autonomous execution | `Skill('05-plan-executor')` |
| 6 | Autonomous review loop (optional, off by default) | `Skill('06-plan-reviewer')` |
| 7 | Smoke test | `Skill('07-smoke-test-app')` |

The orchestrator owns: arg parsing, repo-state detection, mode selection, **routing every phase through the correct `Skill()` invocation**, the 3 strategic checkpoints, the autonomous-decision log, the final summary. **Nothing else.**

## ⚠️ Compliance contract — read this first

This is the rule that overrides anything else in this document. You will be tempted to "save time" by writing PRDs, design picks, or page code yourself in main context. That temptation is wrong every time, and you must catch it.

**Hard rules** (apply in every mode, including `gates=none`):

1. **Every phase MUST be executed by invoking its sub-skill via the `Skill` tool.** Phase 1 = `Skill('01-discover-idea')`. Phase 2 = `Skill('02-design-create-n-htmls')`. Etc. There is no version of this orchestrator where phase artifacts are produced any other way.
2. **NEVER write `.ai/PRD.md`, `.ai/functional-spec.md`, `.ai/site-map.md`, `.ai/techstack.md`, `.ai/environments.md`, `.ai/design.md`, expanded `.ai/master-plan.md` (phases 1+), or any page/component file in `apps/web/src/` directly from main context.** Those are sub-skill outputs.
3. **NEVER bundle multiple master-plan tasks into a single agent.** One task = one agent dispatch. `05-plan-executor` enforces this; you do not bypass it.
4. **NEVER dispatch ≥2 parallel agents into the same working tree.** Every parallel agent runs in its own `git worktree`. `05-plan-executor` enforces this; you do not bypass it.
5. **NEVER mark a phase complete without proof:** the corresponding `Skill()` tool call appears in your message history AND the master-plan checkbox flipped to `[x]` AND the artifact file(s) exist on disk.
6. **NEVER skip the per-task card lifecycle.** Every task in the master-plan has a card at `.ai/master-plan-points/<id>.md`. Cards are created at scaffold time and filled by executor agents. A task without a filled card is not `[x]`.
7. **Decision log is the audit trail in autonomous mode.** Every `Skill()` invocation MUST be logged with timestamp + sub-skill name + return summary. Phases without a corresponding log entry are not closed.

If you catch yourself starting to write a phase artifact in main context, **STOP**. Restart the phase via the proper `Skill()` invocation.

## 🚩 Red flags — stop and self-check

These thoughts mean you are about to violate the compliance contract:

| Pokusa (rationalization) | Rzeczywistość (reality) |
|--------------------------|-------------------------|
| "User already gave me context, I can write the PRD inline." | NO. Sub-skill consumes the same context and produces the canonical artifact + audit log + checkbox flip. |
| "Foundation files (lib/site.ts, primitives, layout components) are lightweight, I'll do them in main context to save time." | NO. Foundation files are master-plan tasks 1.1–1.7. Each gets its own agent. The "save time" argument is the deviation. |
| "6 multi-page agents (e.g., one agent builds /surowce + /oddzialy + 7 branch pages) is faster than 30 single-task agents." | NO. Speed is not the goal. Granularity of audit, isolation via worktree, and one-task-one-PR are the goals. |
| "Sub-skill `03-scaffold-ai-canon` is overkill for this small project — I'll just write the canon files myself." | NO. The sub-skill exists. Use it. Small projects benefit MORE from discipline, not less. |
| "Designs are already on disk, the design pick step is just decision-logging — I can skip the formal step." | NO. The pick step writes `design.md` token distillation, the autonomous-decisions log entry, and flips the checkbox. Skipping it leaves the system in an inconsistent state. |
| "Phase 7 smoke via chrome-devtools-mcp inline is more practical than calling the smoke skill." | NO. `Skill('07-smoke-test-app')` runs its own gates, writes its own report, knows where logs go. Inline smoke is unaudited. |
| "I'll just do this one phase inline, the rest via Skill()." | NO. One inline phase corrupts the audit trail. All-or-nothing. If `Skill()` is somehow broken for one phase, abort and report — do not work around it. |
| "The master-plan-points cards take a long time, I'll skip creating them for this run." | NO. Cards ARE the per-task contract. Without them, executor agents have no input and reviewers have nothing to verify against. |
| "Worktrees are heavyweight for a marketing site, single tree is fine." | NO. Worktrees are how parallel agents coexist without trampling each other. The cost is `cp .env.local`, not heavyweight. |
| "I remember what `01-discover-idea` does, I don't need to invoke it." | NO. The sub-skill version evolves. Invoking via `Skill()` loads the current authoritative behavior. |

If any of these fire, **stop, log the temptation in the decision log, and route through the proper sub-skill instead.**

## Invocation

```
/00-build-app                      # default: detect repo state, supervised mode
/00-build-app "<idea>"             # idea as first positional arg
/00-build-app gates=none           # autonomous mode — no prompts, decisions logged
/00-build-app gates=all            # also gate Phase 4 (env provisioning) approval
/00-build-app phase=design         # force entry at Phase 2 (re-run design fanout)
/00-build-app task=4.1             # delegate to 05-plan-executor Mode B (single task)
/00-build-app from=scratch         # ignore artifacts, run all phases (refuses overwrites without force=yes)
/00-build-app review=on            # enable Phase 6 (autonomous review loop)
/00-build-app review=full iter=2   # review incl. security review, run twice (fix → re-review)
/00-build-app deploy=prod          # add a final prod-deploy pass (opt-in)
```

Args are `key=value` pairs in any order. The first non-key positional is the idea string.

## Step 0 — Parse args

Extract:
- `idea` — first positional or empty
- `mode` — `supervised` (default) | `autonomous` if `gates=none`
- `gates` — `default` (3 checkpoints) | `none` (zero) | `all` (4 checkpoints, adds Phase 4)
- `phase` — explicit entry phase (overrides detection)
- `task` — single-task mode (delegates entirely to `05-plan-executor` Mode B)
- `from` — `scratch` for forced full run
- `review` — `off` (default) | `on` | `full` (incl. security review)
- `iter` — review iteration cap (default 1)
- `deploy` — `local` (default) | `prod`
- `force` — `no` (default) | `yes` (allows from=scratch on existing artifacts)

If `task=` is set, skip Step 0.5 + Step 1 — invoke `Skill('05-plan-executor')` with `mode=B` and the task ID, then return its summary.

## Step 0.5 — Tool preflight

Verify universal tools (CLIs + MCP servers + auth) before any phase runs. Catching `chrome-devtools-mcp` missing now is much cheaper than catching it 30 min into the run when Phase 7 needs it.

Universal checks (run now):
- `node --version` ≥ 20, `pnpm --version` ≥ 10, `git --version` — hard blockers
- MCP `chrome-devtools-mcp` available — hard blocker (Phase 7 smoke depends on it)
- `gh --version` + `gh auth status` — hard blocker (every parallel-agent worktree merges via `gh pr merge`)

Stack-specific checks (deferred to Phase 4): docker running, supabase CLI, vercel CLI + auth. `04-bootstrap-env` already handles these once the stack is known.

Print a one-block readiness report. Format + critical-vs-deferrable rules in `references/tool-preflight.md`.

- **Supervised mode:** show report; proceed if no hard blockers.
- **Autonomous mode:** log report; proceed if no hard blockers; HARD BLOCK on any blocker (write to decision log + pause).

When preflight passes, flip master-plan task `0.1 Tool preflight passed` to `[x]`. If master-plan doesn't exist yet, note "preflight green" — `01-discover-idea` will create the master plan with `0.1 → [x]` pre-filled.

**Also at this step (idempotent — skip if already done):** initialize the agent-logs directory + drop the navigation README so a human reading any later log file knows where to start. One-shot recipe:

```bash
mkdir -p .ai/agent-logs
[ -f .ai/agent-logs/README.md ] || cp .claude/skills/00-build-app/references/agent-logs-readme.md .ai/agent-logs/README.md
```

The template at `references/agent-logs-readme.md` explains all six log file types (run-summary, autonomous-decisions, findings, review-findings, smoke, hooks JSONL) with a "start here" pointer. Without it, users opening a fresh project's `.ai/agent-logs/` later see five files and don't know which to read first. This is the only setup step the orchestrator does inline because it's a one-line `cp` and produces no audit risk.

## Step 1 — Detect entry phase from master-plan checkboxes

Single source of truth: `.ai/master-plan.md` Phase 0 task statuses.

```
if not exists .ai/master-plan.md:
    entry_phase = 1   # nothing started yet; discover-idea creates the plan
else:
    read Phase 0 task statuses (grep '^\s*-\s*\[(.)\]\s*\*\*0\.\d+\*\*')
    entry_phase = first phase whose Phase 0 task is not [x]
    if all Phase 0 done:
        any [ ]/[~]/[c]/[t] in Phases 1+? → entry_phase = 5
        else: smoke log within 24h with ✅? → entry_phase = "done"
        else: entry_phase = 7
```

Phase 0 task → orchestrator phase mapping:
- `0.2` PRD locked → Phase 1
- `0.3` Design locked → Phase 2
- `0.4` Plan scaffolded → Phase 3
- `0.5` Env bootstrapped → Phase 4

Detail rules + edge cases in `references/phase-detection.md`.

Override handling:
- `phase=X` → set `entry_phase = X`, flip the corresponding Phase 0 task back to `[ ]`.
- `from=scratch` → set `entry_phase = 1`, refuse without `force=yes` if artifacts exist.

Report to user (or log in autonomous mode):
```
Detected entry phase: <N> (<reason>). Mode: <mode>. Gates: <gates>. Review: <review>. Will run phases <entry..end>.
```

## Step 2 — Run phases sequentially via `Skill()`

For each phase from `entry_phase` to 7 (or 6 if smoke is the end), run this exact loop:

```
For each phase N:
    1. Verify pre-condition: master-plan task 0.<N> is [ ] or [~]
       (or: detect that we're reentering phase via override)
    2. Invoke Skill('XX-skill-name') with the documented args
    3. Wait for the Skill tool's return
    4. Verify post-condition:
       - All artifact files for phase N exist on disk
       - Master-plan task 0.<N> is [x]
       - In autonomous mode: a decision-log entry exists naming Skill('XX')
    5. If post-condition fails → ABORT, do not proceed. Report what's missing.
    6. Emit one-line summary to user/log: "Phase N complete via Skill('XX'): <artifacts>"
    7. Continue to phase N+1
```

The phase-by-phase invocation contracts:

### Phase 1 — Discover idea

**You MUST invoke `Skill('01-discover-idea')`. Do not write `.ai/PRD.md` yourself.**

- **Args to pass:** `idea`, `mode`, `decision_log_path`, `language` (English default; Polish if `CLAUDE.md` says so or PRD signals it)
- **Expected output (sub-skill produces these):**
  - `.ai/PRD.md` — written
  - `.ai/master-plan.md` — Phase 0 stub written
  - `CLAUDE.md` — skeleton at repo root
  - `AGENTS.md` / `GEMINI.md` — thin pointers to CLAUDE.md (only if missing; never overwrite)
- **Sub-skill flips:** `0.2` → `[x]` (PRD locked), `0.6` → `[x]` (CLAUDE.md initialized)
- **Forbidden in this phase:**
  - Writing PRD content yourself
  - Writing master-plan structure yourself
  - Skipping the `Skill()` call because "the user already gave context"
- **Checkpoint (supervised, gates ≥ default):** present PRD; user approves / redos / aborts.

### Phase 2 — Design exploration

**You MUST invoke `Skill('02-design-create-n-htmls')`. Do not dispatch design subagents inline yourself. Do not pick the best HTML inline. Do not write `.ai/design.md` inline.**

- **Args to pass:** `n` (default 10), `page_type` (from PRD; "landing page" by default), `decision_log_path`
- **Expected output:**
  - `docs/design-ideas/<timestamp>_<page-type>-explorations/design-*.html` (N self-contained HTMLs)
- **After sub-skill returns:** Phase 2 ends. Pick + token-distillation are deferred to Phase 5 master-plan tasks 1.1 and 1.2 (defined in `03-scaffold-ai-canon` canonical Phase 1). Each task gets its own subagent, worktree, card, and PR — same audit trail as every other master-plan task.
- **Supervised mode checkpoint:** present the gallery for user awareness ("here are the 10 directions; the picking subagent in W1 task 1.1 will choose autonomously OR you can write `docs/design-ideas/<...>/PICK.md` now to pin the choice"). User can either trust the W1 pick or pre-write PICK.md to constrain it.
- **Phase 0 task `0.3` flip semantics:** `0.3` is named **"Design HTMLs generated"** (NOT "pick made") and flips to `[x]` as soon as the N HTMLs exist on disk. The pick + design.md are master-plan tasks tracked separately, not Phase 0 state.
- **Forbidden in this phase:**
  - Dispatching the N parallel HTML agents directly from main context (the sub-skill owns that)
  - Picking the best HTML inline in main context
  - Writing `.ai/design.md` inline (that's W1 task 1.2, not orchestrator work)
  - Distilling tokens from memory or compressing the pick into a chat decision (every choice goes through a subagent → card → PR for the same reason every code task does)

### Phase 3 — Scaffold `.ai/` canon

**You MUST invoke `Skill('03-scaffold-ai-canon')`. Do not write `functional-spec.md`, `site-map.md`, `techstack.md`, `environments.md`, or expand `master-plan.md` yourself.**

- **Args to pass:** `language`, `mode`, `decision_log_path`
- **Expected output:**
  - `.ai/site-map.md`
  - `.ai/functional-spec.md`
  - `.ai/techstack.md` (light)
  - `.ai/environments.md`
  - `.ai/master-plan.md` — EXPANDED with phases 1+, **including a Waves table (W0..WN with `∥`/`→` cohort markers)**
  - `.ai/master-plan-points/<id>.md` — one card per task in phases 1+
  - `CLAUDE.md` — Planning Structure table enriched
  - **NOT:** `.ai/database-architecture.md` (deferred to Phase 1 master-plan task 1.1)
- **Sub-skill flips:** `0.4` → `[x]`
- **Forbidden in this phase:**
  - Writing canon docs yourself
  - Producing a master-plan without an explicit Waves table (the executor needs it)
  - Skipping per-task card creation
- **Checkpoint (supervised, gates ≥ default):** present master plan; user trims / reorders / approves.

### Phase 4 — Bootstrap env

**You MUST invoke `Skill('04-bootstrap-env')`. Do not run `pnpm install` / write `.env.local` / scaffold layout yourself.**

- **Args to pass:** `mode`, `decision_log_path`, `gates=all` flag if requested
- **Expected output:**
  - Working dev stack — `pnpm dev` (or stack equivalent) boots green
  - `apps/*/.env.local` populated with placeholder values appropriate for v1
  - `docs/runbook.md` seeded
  - `CLAUDE.md` — "Local Development" section populated with real commands
- **Sub-skill flips:** `0.5` → `[x]` when dev server boots green
- **Forbidden in this phase:**
  - Running `pnpm install` yourself before invoking the sub-skill
  - Writing `globals.css` design-tokens block yourself (that's master-plan task 1.x done by an executor agent in Phase 5 — the env phase only verifies dev boots, it does not implement design)
- **Checkpoint (supervised, gates=all only):** present provisioning summary; user approves.

### Phase 5 — Execute master plan

**You MUST invoke `Skill('05-plan-executor')` in Mode A. Do not dispatch page-builder subagents directly from main context.**

- **Args to pass:** `mode=A`, `plan_path=.ai/master-plan.md`, `cards_dir=.ai/master-plan-points/`, project conventions (auto-detected from `CLAUDE.md`)
- **Expected behavior of `05-plan-executor`:**
  - Reads the Waves table from `master-plan.md`
  - Iterates wave-by-wave (W1, W2, …)
  - Within each wave, dispatches **one subagent per task ID** (never bundles)
  - Every subagent runs in **its own `git worktree`** off `main`
  - Every subagent merges its PR back to `main` before reporting `merged: yes`
  - Writes its own findings log to `.ai/agent-logs/findings-<date>-<plan-slug>.md`
- **Orchestrator's job during Phase 5:** wait for `Skill('05-plan-executor')` to return. Don't try to inspect or micromanage.
- **Forbidden in this phase:**
  - Dispatching page-builder agents directly from main context
  - Bundling multiple master-plan tasks into one agent ("agent for /surowce + /oddzialy + 7 branch pages" — NO, that's seven distinct tasks = seven agents)
  - Writing foundation files (lib/, components/ui/, layout/) yourself in main context — those are master-plan tasks 1.1–1.7, executed by 7 dedicated agents
  - Skipping worktree creation per agent
  - Skipping per-task PR-merge-to-main
- **Checkpoint:** none — Phase 5 is unattended by design.
- **Re-entry:** if `05-plan-executor` returns with tasks remaining as `blocked` or `failed` despite retry, pause and report.

### Phase 6 — Review loop (optional, off by default)

**Skip if `review=off` (default). Otherwise, you MUST invoke `Skill('06-plan-reviewer')` in Mode A.**

- **Args to pass:** `plan_path`, `security_review=true` if `review=full`, `iter` cap
- **Expected behavior:** `06-plan-reviewer` audits each `[x]` task with one review-subagent per task in worktrees, surfaces `improved`/`extended`/`issues_found` per task, and (for issues_found) queues new master-plan tasks for `05-plan-executor` to handle on a follow-up pass.
- **Output:** `.ai/agent-logs/review-findings-<date>.md`, optional `.ai/plans/<date>-review-fixes/`
- **Forbidden:** running `06-plan-reviewer` against the same working tree as the orchestrator; reviewer subagents must use their own worktrees.

### Phase 7 — Smoke test

**You MUST invoke `Skill('07-smoke-test-app')`. Do not run inline smoke probes via curl/chrome-devtools yourself.**

- **Args to pass:** `repair=true` if `mode=autonomous` (lets the smoke skill auto-fix small issues like missing imports or missing assets); `repair=false` in supervised mode unless user opts in
- **Expected output:** `.ai/agent-logs/smoke-<date>.md` with all gates explicit
- **Forbidden:** marking the build "complete" without `Skill('07-smoke-test-app')` returning a green report.

### (Optional) Final pass — Deploy to prod

- Runs only if `deploy=prod`.
- Invokes `Skill('vercel:deploy')` (or stack-equivalent). Waits for green deploy URL.
- Reports the prod URL in the final summary.

## Step 2.5 — Main agent commit cadence (after every phase)

**After each completed phase, the main agent MUST commit and push.** This is non-negotiable. The orchestrator runs for hours; an unpushed working tree is one Ctrl-C away from being lost.

What gets committed by the main agent (vs by subagents via their PRs):

| Phase | Main agent commits | Path | Why a main-agent commit (not a PR) |
|-------|-------------------|------|------------------------------------|
| 1 | `.ai/PRD.md`, `.ai/master-plan.md` (Phase 0 stub), `CLAUDE.md` skeleton, `AGENTS.md`/`GEMINI.md` pointers, decision-log entry | repo root + `.ai/` | Sub-skill writes them on `main`; not feature code, no PR needed |
| 2 | Picked-design tokens distilled into `.ai/design.md`, decision-log entry, optional design HTMLs in `docs/design-ideas/` | `.ai/` + `docs/` | Same — planning artifacts live on `main` |
| 3 | `.ai/{site-map,functional-spec,techstack,environments}.md`, expanded `master-plan.md`, all `master-plan-points/<id>.md` cards | `.ai/` | Planning expansion |
| 4 | `apps/*/.env.example` updates, `docs/runbook.md`, CLAUDE.md "Local Development" section | `apps/` + `docs/` + repo root | Env scaffolding, no product code |
| 5 | Per-cohort findings log + run-summary append | `.ai/agent-logs/findings-<date>-<slug>.md`, `.ai/agent-logs/run-summary-<...>.md` | Phase 5 is unattended; subagent PRs ARE the work, but main agent still commits its own audit logs |
| 6 | Per-cohort review-findings log + run-summary append + any review-fix side plans | `.ai/agent-logs/review-findings-<date>.md`, `.ai/agent-logs/run-summary-<...>.md`, `.ai/plans/<date>-review-fixes/` | Same — review subagents merge their own PRs; main agent only commits its log |
| 7 | `.ai/agent-logs/smoke-<date>.md` + screenshots + run-summary final summary | `.ai/agent-logs/` | Smoke evidence + final timeline closure |

> Every phase 1–7 ALSO appends to `.ai/agent-logs/run-summary-<YYYY-MM-DD-HHMM>.md` per the format in Step 2.6. That file is in every phase's `git add` recipe.

**Cadence:**
- After each phase completes its post-condition (artifacts on disk + checkbox flipped to `[x]`), main agent runs:

  ```bash
  git add -A .ai/ docs/ apps/web/.env.example CLAUDE.md AGENTS.md GEMINI.md  # phase-specific
  git commit -m "phase-<N>(<sub-skill>): <one-line summary>"
  git push origin main
  ```

- Phase 5 commits are smaller and per-cohort (just the findings log appended), but still pushed regularly so a session interruption doesn't lose the orchestration audit.

**Forbidden cadence anti-patterns:**
- Letting all 7 phases finish before any push — one crash = whole run gone
- Main agent waiting until the user "asks" for a push — autonomous mode means autonomous, including pushing
- Pushing only at the final summary — checkpoints (after Phases 1, 2, 3, 4 in supervised mode) need a fresh push so the user reviews exactly what's on `origin`

**Why this matters more in autonomous mode:** the user isn't watching. If the run crashes at Phase 6 and Phase 1-5 work isn't pushed, the user wakes up to a half-empty repo and doesn't know what to trust. Pushing per phase makes the run resumable from `origin` — kill the agent, re-run `/00-build-app`, detection-from-master-plan-checkboxes resumes cleanly.

## Step 2.6 — Run summary log (chronological narrative)

The main agent maintains **one human-readable timeline file per run** at `.ai/agent-logs/run-summary-<YYYY-MM-DD-HHMM>.md` (timestamp = run start). This is the play-by-play view a human reader scans top-to-bottom to understand "what happened, in what order, why".

**Distinction from sibling logs:**
- `hooks-<date>.jsonl` — raw event stream (every tool call). Machine-readable.
- `autonomous-decisions-<date>.md` — load-bearing decisions with WHY. Sparse, narrative.
- `findings-<date>-<plan-slug>.md` — per-cohort engineering findings from `05-plan-executor`.
- **`run-summary-<...>.md` — chronological orchestration log**. ASCII tree of dispatches, returns, decisions, recoveries. The "what I orchestrated, in what order".

### Format (ASCII tree, one line per event)

```markdown
# Run summary — 2026-05-04 13:42 (Europe/Warsaw)

> Chronological log of `/00-build-app gates=none`. Read top-to-bottom.

**Idea:** "Replace example.com with a new marketing site"
**Mode:** autonomous · **Started:** 2026-05-04 13:42 · **Finished:** 14:38 (~56 min)

---

## Phase 1 — Discover idea

```
13:42 → Skill('01-discover-idea')        · write PRD from idea + extra-context.md
13:48 ← returned                         · .ai/PRD.md (124 lines), 0.2 → [x]
```

## Phase 2 — Design exploration

```
13:48 → Skill('02-design-create-n-htmls') · n=10, page=landing
        ├─ 13:48 ┬→ 10 parallel design subagents dispatched
        ├─ 13:55 ┴← all returned · 10 HTMLs in docs/design-ideas/<...>
13:55   0.3 → [x] (HTMLs generated)
```

## Phase 5 — Execute master plan

```
14:05 → Skill('05-plan-executor') Mode A
        │
        ├─ W1 — Foundation (sequential 1.1→1.2→1.3→1.4→1.5→1.6 ∥ 1.7..1.9)
        │   ├─ 14:06 → 1.1  Pick design direction       · worktree feature/1.1
        │   │   14:08 ← merged PR #12                   · picked design-01 + 2 adjustments
        │   ├─ 14:08 → 1.2  Generate .ai/design.md      · worktree feature/1.2
        │   │   14:11 ← merged PR #13
        │   ├─ 14:11 → 1.3  database-architecture.md    · worktree feature/1.3
        │   │   14:16 ← merged PR #14                   · 8 tables, 4 ENUMs, RLS
        │   ├─ ... (1.4 → 1.5 → 1.6)
        │   └─ 14:30 W1 closed [x]                      · 9 tasks, 9 PRs
        │
        ├─ W2 — Marketing core (3 parallel)
        │   ├─ 14:30 ┬→ 2.1 Homepage                    · worktree feature/2.1
        │   │        ├→ 2.2 /audience-a                 · worktree feature/2.2
        │   │        └→ 2.3 /audience-b                 · worktree feature/2.3
        │   ├─ 14:38 ┼← 2.2 merged PR #19
        │   ├─ 14:39 ┼← 2.1 merged PR #18
        │   ├─ 14:40 ┴← 2.3 merged PR #20
        │   └─ 14:40 W2 closed
        │
        └─ ... (W3..W6 similar)
```

## Phase 7 — Smoke test

```
15:32 → Skill('07-smoke-test-app')
15:35 ← all 4 gates green                · 18/18 routes 200, console clean
```

## Recovery log

```
14:52 ⚠ task 4.2 report rejected         · simplify_ran=no
14:52 → re-dispatched 4.2                · explicit "Skill('simplify') is mandatory"
14:55 ← merged PR #29
```

---

## Final summary

| Metric | Value |
|--------|-------|
| Phases run | 7 / 7 |
| Subagents dispatched | 38 |
| PRs merged to main | 27 |
| Tasks shipped (`[x]`) | 27 |
| Recoveries | 1 (re-dispatch on missing simplify) |
| Total duration | 56 min |
```

### Cadence — when to append

Append at these moments (NOT after every single tool call — that's what hooks JSONL is for):

| Append a line for... | Granularity |
|----------------------|-------------|
| Each `Skill('XX')` invocation + its return | One `→` line + one `←` line |
| Each subagent dispatch in Phase 5/6 | One indented `├─` line per dispatch |
| Each subagent return (with PR # if merged) | One indented line under same parent |
| Each wave open / close | Bracket the wave with header + footer |
| Each phase open / close | Major section header |
| Mishap / recovery | `⚠` line in the Recovery log section |
| Master-plan checkbox flip | One inline note (`0.X → [x]`) |
| Final summary table | At the very end, after Phase 7 |

### Conventions

- **Time format:** `HH:MM` local timezone (project's working timezone). 24-hour, no seconds.
- **Glyphs:** `→` outbound dispatch, `←` return, `┬─/├─/└─` tree branches, `⚠` mishap.
- **Brevity is the rule.** One line per event. Subagent purpose = max 5–7 words. PR number + commit-line verb only. **Detail belongs in card / decision-log / findings-log; this is a TIMELINE.**
- **Visual rhythm matters.** Use the ASCII tree consistently — readers scan it like a stack trace, not a paragraph.
- **No prose paragraphs in the body.** Header lines, code blocks with tree, tables. That's it.

### Where to write — main agent only

The main agent owns this file. **Subagents do NOT write to it.** A subagent's findings live in its card + the cohort findings log. The orchestrator is the only entity with full chronological view.

**Cadence + commit:** the run-summary is committed by the main agent in Step 2.5 — same per-phase push as other planning artifacts. Add it to the per-phase `git add -A` recipe.

## Step 3 — Checkpoint UX (supervised mode, default 3 gates)

After each gated phase (1, 2, 3 — and 4 if `gates=all`), present:

```
**Phase <N> — <name>** complete.

<one-line summary of what was produced>
<2-3 bullets of key decisions>

Reply:
- "ok" / "go" → proceed to Phase <N+1>
- "redo: <feedback>" → re-dispatch this phase with your feedback prepended
- A specific edit (e.g., "change MVP item 3 to X") → I'll apply inline if possible; redo if structural
- "abort" → stop here; artifacts left in place
```

Watch for inline edits via `git diff` between checkpoint emit and user reply — if the user edited the artifact directly, treat as approval (they voted by editing).

## Step 4 — Autonomous mode UX (`gates=none`)

1. **Up front, send one structured context-gathering message:**

   ```
   I'll build "<idea or 'an app per repo signals'>" autonomously.

   Optional context — answer any of these or skip; I'll proceed in 5 minutes regardless:
   - Target users (who's the primary audience?)
   - Particular features you want in MVP
   - Aesthetic direction (calm / playful / bold / institutional / agent's choice)
   - Stack preference (or agent picks)
   - Anything off-limits

   I'll log every load-bearing decision to .ai/agent-logs/autonomous-decisions-<date>.md so you can audit afterward. Every phase will go through its dedicated sub-skill and corresponding agent dispatches will use git worktrees + PR merges; no shortcuts.
   ```

2. Whether or not the user replies within 5 minutes, proceed.

3. Every load-bearing decision (stack pick, MVP scope choice, design pick from N variants, env provisioning compromises, task ordering, review verdicts) is appended to the decision log:

   ```
   ## YYYY-MM-DD HH:MM — Phase N (<sub-skill>), autonomous mode
   - Skill('XX-skill-name') invoked
     - Args: { ... }
     - Returned: <summary>
   - <Decision>: <value>
     - Why: <rationale referencing a signal — idea shape, default rule, repo evidence>
   ```

4. Pause only on:
   - **Hard block** — missing API credential the agent cannot self-source.
   - **Double-failure on a phase** — after 2 retries, present structured "what worked / what didn't / proposed remediation" report.
   - **Compliance violation** — orchestrator detects it produced a phase artifact without a corresponding `Skill()` invocation. Self-report, restart the phase correctly.

5. Final summary at the end with link to the decision log.

## Step 5 — Failure recovery

- **Per-phase max retries: 2.** After the second failure, pause regardless of mode and present the structured report.
- **No token budget enforcement.** Quality > frugality. Cost controls live in sub-skills.
- **Phase 5 + 6 recovery is delegated** to `05-plan-executor` / `06-plan-reviewer` (their own re-plan logic).

## Step 6 — Self-audit before final summary

Before emitting the final summary, the orchestrator MUST verify compliance:

1. For each phase 1..7 that ran: did its `Skill()` invocation appear in the message history? If not, that phase is unaudited — restart it.
2. For each phase: does the decision log have a corresponding "Skill('XX') invoked" entry (autonomous mode)? If not, the log is incomplete — append the missing entries from message history.
3. Master-plan checkboxes 0.1–0.6 + every Phase 1+ task: are they consistent with the artifacts on disk? If a task is `[x]` but no card exists, OR if a card has no Implementation / Tests (3 sub-sections) / Changelog filled in, the task is not actually done — fix or downgrade.
4. For Phase 5: every shipped task should have a closed PR linked from its card. If `05-plan-executor` returned but cards lack PR URLs, surface that gap.
5. **Test-coverage audit** — for every Phase 1+ task marked `[x]`: does its card have all three Tests sub-sections filled (or whitelisted skip reasons documented)? If any tier is missing without justification, dispatch an executor re-do for that task. Don't ship a build where some tasks are marked `[x]` but have zero unit/integration/e2e coverage.

If the audit finds issues, fix them (or report them) before the final summary. **A finished build with broken audit trail is not finished.**

## Step 7 — Final summary

```
**Build complete (or partial).**

Idea: "<verbatim>"
Mode: <supervised|autonomous>
Stack: <picked stack>

Phases run (each via Skill() invocation):
1. ✅ Skill('01-discover-idea') — PRD: <path>
2. ✅ Skill('02-design-create-n-htmls') — picked: <design-NN.html>, tokens in .ai/design.md
3. ✅ Skill('03-scaffold-ai-canon') — master plan: <N tasks across M waves>
4. ✅ Skill('04-bootstrap-env') — local stack working
5. ✅ Skill('05-plan-executor') — <X PRs merged across N worktrees, Y tasks shipped>
6. <skipped|✅ Skill('06-plan-reviewer') — <N improvements, M issues found>>
7. ✅ Skill('07-smoke-test-app') — all green / <N> warnings

App running locally at: http://localhost:<port>

Audit:
- All <N> task cards have Implementation, Tests, and Changelog sections filled.
- All <X> PRs merged to main; no orphan branches.
- Decision log: .ai/agent-logs/autonomous-decisions-<date>.md (<L> entries)

Next steps:
- Open http://localhost:<port> in a browser
- For prod deploy: `/00-build-app deploy=prod`
- For autonomous-mode audit trail: see .ai/agent-logs/autonomous-decisions-<date>.md
- For review log: see .ai/agent-logs/review-findings-<date>.md
```

## Universal quality rails

1. **Detection beats interrogation.** Always check what the master-plan already says before asking. Resume cheaply.
2. **Master-plan checkboxes are the state.** Phase 0 tracks the planning journey; Phases 1+ track app work. One file, one truth.
3. **Sub-skills do the work; orchestrator routes via `Skill()`.** This file should never contain phase-specific logic.
4. **Autonomous mode logs every load-bearing decision AND every `Skill()` invocation.** "I made this choice; here's why" — every time. The audit trail is the contract that makes autonomy trustworthy.
5. **Failures pause, never auto-skip.** If a phase double-fails in autonomous mode, the agent yields control. Silent failure poisons the whole run.
6. **One task = one agent = one worktree = one PR = one merge to main.** This is `05-plan-executor`'s contract; the orchestrator does not bypass it.

## Lean main agent — context discipline + delegate everything

The orchestrator runs in the user's main session. Without discipline its context bloats across 7 phases. Mitigations:

1. **File-as-state, not memory-as-state.** Each phase reads what it needs from files (PRD, master-plan checkboxes, design.md, etc.). The orchestrator never has to "remember" what Phase 1 said in conversation history — Phase 4 can re-read `.ai/PRD.md`. This is why master-plan checkboxes are the only state.
2. **One-line phase summaries.** When a phase completes, the orchestrator emits one line: "Phase N complete via Skill('XX'): <what was produced>". The full output stays on disk; only the summary lives in chat.
3. **Heavy work goes to sub-skills' subagents.** Each sub-skill internally dispatches `general-purpose` subagents (with their own fresh contexts) for heavy work. `01-discover-idea` does this for the intake itself; `03-scaffold-ai-canon` does it for master-plan generation; `05-plan-executor` does it per-task; `06-plan-reviewer` does it per-review.
4. **Sub-skills loaded sequentially.** When the orchestrator invokes a sub-skill via the `Skill` tool, that sub-skill's body loads into context. Only ONE sub-skill is active at a time — the previous one's content is no longer needed (its output is on disk). Don't re-invoke prior sub-skills from main context unless necessary; re-read the artifact file instead.

The single most context-protective rule: **after each phase, the orchestrator's next action should be `Read .ai/master-plan.md`, NOT a re-read of the prior phase's transcript.** That single discipline keeps the orchestrator lean.

### Delegate everything that isn't routing

The main agent's job is **routing + auditing**, not doing. When ANYTHING beyond a one-line state flip needs to happen — dispatch a subagent. Even small things that "feel quick to do inline" eat the main agent's context and corrupt the audit trail.

**Always dispatch a subagent for:**

| Situation | Subagent type / task |
|-----------|---------------------|
| Master-plan-point cards need creating after `master-plan.md` is written | Card-generator subagent (auto-chained inside `03-scaffold-ai-canon`, not main-agent's job) |
| Waves table is missing or looks illogical | Wave-fixer subagent (re-edits the master-plan's Waves section in isolation) |
| Branches accumulated, didn't merge cleanly | Cleanup subagent (rebases / merges / closes orphans) |
| Merge conflict on `main` from a Phase 5 task | Conflict-resolver subagent in a fresh worktree |
| `pnpm install` left a broken state in some worktree | Worktree-recovery subagent |
| `gh pr merge` failed for one task | PR-finisher subagent for that single PR |
| A subagent reported `merged: no` and human-review isn't required | Re-dispatch the same task after correcting the failure cause |
| Findings log is malformed / missing entries | Log-fixer subagent |
| Decision log entries are missing for a phase that ran | Log-backfill subagent |
| `.ai/*.md` planning doc is stale after some task touched it | Doc-sync subagent |
| Need to confirm a subtle visual bug in the running app | Browser-investigation subagent (chrome-devtools-mcp specialist) |

**Inline is OK only for these tiny, routing-grade actions:**
- Reading a file to detect state (one or two reads, no edits)
- Flipping a single master-plan checkbox (one `Edit` call)
- Appending one line to the decision log
- Emitting a one-line phase summary to the user

If you start typing more than ~30 lines of edit, you've already overshot — stop and dispatch a subagent instead.

### Not every subagent needs a worktree

Worktree creation is mandatory only when:
- The subagent runs **in parallel** with other subagents (worktree isolation prevents file races)
- The subagent works on a **W1+ master-plan task that touches code** (those are tracked, audited, PR-merged per the executor contract)

Worktree is **optional / not needed** for:
- Cleanup, recovery, wave-fixing, log-fixing subagents (sequential, single-purpose, often work in `.ai/*.md` only)
- Doc-sync subagents that touch only `.ai/*.md` or `docs/*.md`
- Single-PR-finisher subagents that just run `gh pr merge` on a stuck PR
- Browser-investigation subagents that read but don't write

The rule: **isolation cost should match the risk of the work**. A cleanup agent that rewrites a malformed `findings-*.md` doesn't need a branch — but a Phase 5 page-builder absolutely does.

When dispatching a non-worktree subagent, tell it explicitly: "You are running in the main working tree. Touch only the file(s) named in this prompt. Do not run `git worktree add`, do not push, do not open a PR. Edit the file, return a one-line summary, exit."

## Hooks layer — programmatic observability (`.claude/settings.json`)

This project ships with `.claude/settings.json` hooks that log every Skill invocation, every canonical-artifact Write/Edit, every subagent dispatch (Agent), and every git/gh operation to `.ai/agent-logs/hooks-<YYYY-MM-DD>.jsonl`. The `Stop` hook prints a one-line audit summary at end of turn (`[build-app audit] Skill=N · canon-edits=N · subagents=N · git-ops=N`) so deviations from the compliance contract surface immediately.

**Hook scope (observability, not blocking):**
- `PostToolUse / Skill` — every `Skill('XX')` call logged with timestamp + tool_input
- `PostToolUse / Write|Edit` — only writes to `.ai/{PRD,master-plan,functional-spec,site-map,techstack,environments,design,database-architecture}.md` or `apps/web/src/app/**/*.tsx` are logged (filter inside the hook to avoid noise)
- `PostToolUse / Agent` — every subagent dispatch logged
- `PostToolUse / Bash` — only `git commit|push|merge|rebase`, `gh pr *`, `gh repo *` are logged (filter inside)
- `Stop` — emits `{systemMessage: "[build-app audit] Skill=N · canon-edits=N · subagents=N · git-ops=N"}` if today's log has any activity, otherwise silent

**How the orchestrator uses the log:**
- Step 6 self-audit can `grep` the log to verify each phase produced its `Skill('XX')` invocation. Missing entry = unaudited phase = restart.
- Final summary cites the audit numbers as proof of compliance.
- A user reading the audit message at end of turn sees at a glance whether the run was sane (e.g., "canon-edits=8 but Skill=0" = direct main-context edits = deviation).

**Important caveat:** the settings.json watcher only watches `.claude/` directories that existed at session start. If this is the first time `.claude/settings.json` was created in the current session, the user must open `/hooks` once (reloads config) or restart the session for hooks to fire. After that, hooks fire automatically across all subsequent turns.

The log files match `.ai/agent-logs/hooks-*.jsonl` and are gitignored — they're per-session artifacts, not durable history.

## Reference files

- `references/tool-preflight.md` — Step 0.5 universal vs deferred checks; readiness-report format
- `references/phase-detection.md` — single-rule checkbox-based detection + edge cases
- `references/checkpoint-prompts.md` — exact wording of the 3 supervised gates
- `references/autonomous-decisions.md` — decision-log template with examples
- `references/eval-strategy.md` — 3-tier test plan (deferred from v0; for future iteration)

## Why this skill is shaped this way

- **Composition over monolith.** Each sub-skill is independently usable (`/01-discover-idea` works without the orchestrator). The orchestrator becomes optional, not foundational.
- **Detection-driven resume.** Killing the agent mid-run, switching contexts, or re-invoking days later all resume cleanly because the artifacts ARE the state.
- **Two modes, one shape.** Supervised and autonomous are the same flow with different checkpoint behavior — not separate code paths.
- **Phase 5 + 6 unattended by design.** These are the long phases. The orchestrator hands off to `Skill('05-plan-executor')` / `Skill('06-plan-reviewer')` and waits; doesn't try to micromanage. The executor's cohort-loop logic runs there, not here.
- **Smoke test as the contract for "production-ready, running locally".** Without runtime verification, the claim is empty. `Skill('07-smoke-test-app')` is the only allowed way to claim Phase 7 done.
- **Compliance contract above all.** The Red Flags table exists because the failure mode of this orchestrator is "main agent feels qualified and writes phase artifacts inline". That failure is what the contract makes unrecoverable without a self-audit.
