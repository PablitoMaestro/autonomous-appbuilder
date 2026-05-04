# Autonomous Decision Log Format

When `gates=none`, every load-bearing decision the agent makes on the human's behalf MUST be appended to `.ai/agent-logs/autonomous-decisions-<YYYY-MM-DD>.md`. Without this audit trail, autonomous mode is a black box and the human has no way to understand what shipped.

## File location

- Path: `.ai/agent-logs/autonomous-decisions-<YYYY-MM-DD>.md`
- Create the `.ai/agent-logs/` directory if missing.
- One file per calendar day. If multiple autonomous runs happen on the same day, append to the same file (don't overwrite).
- The orchestrator + every sub-skill it invokes append to the same log file.

## Entry format

Each load-bearing decision is one block:

```markdown
## YYYY-MM-DD HH:MM — Phase N (<sub-skill>), autonomous mode

- <Decision>: <chosen value>
  - **Why:** <rationale referencing a signal — idea shape, default rule, repo evidence>
- <Decision>: <chosen value>
  - **Why:** <rationale>

### Open question surfaced (skipped — no reply)
- <question> → defaulted to <default>
```

Every "Why" line MUST reference a *signal*: the user's idea text, a default rule from a sub-skill, an existing repo file's contents, or an explicit fallback. "Because it seemed best" is not a valid Why.

## What counts as load-bearing

Log these decisions:
- **Stack picks** (full stack + each component if it deviates from the default).
- **Design picks** (which of the N HTML variants was chosen, with one-line rationale).
- **MVP scope choices** (any feature added or dropped from the user's idea).
- **Out-of-scope decisions** (anything explicitly deferred to Phase 2+).
- **Env provisioning compromises** (skipped remote provisioning because no creds; chose region; picked DB tier).
- **Master-plan ordering decisions** (wave layout, task numbering, deferred tasks).
- **Reviewer verdicts** (when Phase 6 runs: any `extended` verdicts that added features).
- **Smoke test failures** (which checks failed, what the repair plan is).
- **Hard blocks** (when the agent paused waiting for human input it never got).

Do NOT log:
- Routine implementation details (what file got edited, what test passed).
- Anything that's already in commit messages, the master plan, or task cards.
- Decisions that reduce to "default per the recipe" with nothing surprising.

## Examples

### Phase 1 (discover-idea)

```markdown
## 2026-05-03 14:32 — Phase 1 (01-discover-idea), autonomous mode

- Idea input: "a flashcard app for medical students"
- Stack picked: Next.js (App Router) + Supabase + Vercel + shadcn/Tailwind
  - **Why:** SaaS web shape (default rule); no contradictory signals in idea or repo
- Audience inferred: medical students preparing for licensing exams (Polish-speaking)
  - **Why:** idea text mentions "medical students"; nearest adjacent audience signal in repo CLAUDE.md is non-English; defaulted to Polish per language inference rule
- Out-of-scope auto-set: payments, multi-tenancy, mobile-native app
  - **Why:** not mentioned in idea; default MVP fence

### Open question surfaced (skipped — no reply)
- Specific exam content focus (LEK / specialization / continuing ed)? → defaulted to general medical school content
```

### Phase 2 (design-create-n-htmls)

```markdown
## 2026-05-03 14:51 — Phase 2 (02-design-create-n-htmls), autonomous mode

- Design pick: design-04.html (concept: "editorial serif restraint with single bold accent")
  - **Why:** highest signal-to-noise per universal quality rails; restraint matches medical-context tone in PRD
- Discarded: design-07 (numbered-sections + diagram-as-headline)
  - **Why:** too academic-paper feel; loses approachability for student users
```

### Phase 4 (bootstrap-env) compromise

```markdown
## 2026-05-03 15:10 — Phase 4 (04-bootstrap-env), autonomous mode

- Local provisioning: ✅ Supabase + Next dev (localhost:3000)
- Remote provisioning: ⚠️ skipped Vercel project creation
  - **Why:** VERCEL_TOKEN not provided; recipe rule says skip remote in autonomous mode when creds missing
- Domain: not configured
  - **Why:** requires manual DNS; logged for human follow-up

### Open question surfaced (skipped — no reply)
- Supabase region preference? → defaulted to eu-central-1 (closest to Polish audience inferred in Phase 1)
```

### Phase 6 (plan-reviewer) extended verdict

```markdown
## 2026-05-04 02:14 — Phase 6 (06-plan-reviewer), autonomous mode

- Task 4.2 (admin dashboard) verdict: extended
  - Added: empty-state CTA on /admin when no content exists
  - **Why:** task as shipped left admin landing visually broken; documented in master-plan-points/4.2.md as Review addition
- Task 5.1 (recommendations) verdict: issues_found (escalation)
  - **Why:** algorithm produces empty recommendations for users with < 3 watch events; not in surgical budget to fix; flagged for human review
```

### Hard block

```markdown
## 2026-05-04 03:22 — Phase 4 (04-bootstrap-env), autonomous mode

- HARD BLOCK: cannot proceed without GITHUB_TOKEN
  - **Why:** the picked stack uses GitHub Actions for CI; without auth, repo can't be created and Phase 5's PR-per-task workflow won't have a remote
  - **Action:** paused; will resume on next invocation if `gh auth status` shows authenticated
```

## Reading the log

The end-of-run summary should always include a link: `Decision log: .ai/agent-logs/autonomous-decisions-<date>.md`.

The human reading the log should be able to:
1. Skim the timestamps to see how long each phase took.
2. Search for "Why:" to find every rationale.
3. Search for "Open question" to find every implicit-default the agent picked.
4. Search for "HARD BLOCK" to find any incomplete state.

If the log is hard to skim by these queries, the entries are too verbose. Tighten future entries.

## Anti-patterns

- ❌ "Picked Next.js because it's the best framework." (no signal)
- ✅ "Picked Next.js because: SaaS web shape (default rule); no contradictory signals."
- ❌ "Created Supabase project successfully." (not a decision — that's a routine action; goes in commit log if anywhere)
- ✅ "Picked Supabase region eu-central-1 because: PRD audience inferred as Polish."
- ❌ Logging 200 routine task completions during Phase 5. (Master plan checkboxes already track that.)
- ✅ Logging 1 cross-task signal during Phase 5 ("noticed 4.1 and 4.6 both reach for `formatDuration`; extracted to lib/format.ts").
