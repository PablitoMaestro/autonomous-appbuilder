---
name: 03-scaffold-ai-canon
description: Expand the master-plan with phases 1+ and write the lighter `.ai/` planning docs (`site-map.md`, `functional-spec.md`, `environments.md`, `techstack.md` light) from a locked PRD + design HTMLs. Heavy docs like `.ai/design.md`, `database-architecture.md`, detailed techstack rationale are NOT generated here — they become Phase 1 master-plan tasks 1.1 (design pick), 1.2 (design.md), 1.3 (database), 1.4 (techstack rationale) executed by dedicated agents with fresh context. Master-plan expansion is dispatched as its own subagent (heaviest single output). Use whenever the user says "scaffold the planning docs", "expand the master plan", "create the .ai canon", "set up the project docs", "expand the PRD into a roadmap", or has a locked PRD + design HTMLs and wants the full execution plan.
---

# 03 — Scaffold .ai/ Canon

> **Invocation contract:** This skill is invoked via `Skill('03-scaffold-ai-canon')` — typically by `00-build-app` at Phase 3, but also standalone. The caller MUST NOT write `functional-spec.md`, `site-map.md`, `techstack.md`, `environments.md`, or expand `master-plan.md` themselves. The expansion is dispatched as a dedicated subagent here; the caller never handles it inline. If a Waves table is missing or illogical after the expansion, this skill auto-dispatches a wave-fixer subagent — the caller does NOT edit the table inline either.

Take a locked `PRD.md` + locked `design.md` + the master-plan stub (created by `01-discover-idea` with Phase 0 only) and produce:
- The lighter `.ai/` planning docs (site-map, functional-spec, environments, techstack-light)
- An EXPANDED master-plan with phases 1+ added (Phase 0 stays as-is, written by `01-discover-idea`)
- Per-task cards in `master-plan-points/`

Then flip master-plan task `0.4 Plan scaffolded` to `[x]`.

The skill's value isn't the templates — it's:
1. **Knowing what NOT to write here.** Heavy docs (design pick, `.ai/design.md`, `database-architecture.md`, detailed techstack rationale) are scheduled as Phase 1 master-plan tasks, not produced here. Each gets its own dedicated subagent during execution, with fresh context for focused thinking.
2. **Master-plan expansion as a dedicated subagent.** It's the heaviest single output (50-200 lines + N task cards) and deserves fresh context.
3. **Cross-referencing.** Every fact lives in one doc; siblings link.

## Inputs

The orchestrator (or standalone caller) must provide:
- `.ai/PRD.md` exists and is locked
- `docs/design-ideas/<timestamp>_<page-type>-explorations/design-*.html` exist (N self-contained design HTMLs from `02-design-create-n-htmls`). For non-UI projects (CLIs, libraries) skip the HTML requirement and the master-plan won't include design tasks 1.1+1.2.
- `.ai/design.md` does NOT need to exist — it's a Phase 1 deliverable, not an input
- `.ai/master-plan.md` exists with Phase 0 (created by `01-discover-idea`)
- Project language (English by default; Polish if `CLAUDE.md` or PRD signals)

If any is missing, stop and report concretely: "Need `.ai/PRD.md` + design HTMLs in `docs/design-ideas/<...>` + `.ai/master-plan.md` (Phase 0 stub) before scaffolding can run. Run `/01-discover-idea` then `/02-design-create-n-htmls` first."

## Workflow at a glance

1. **Read inputs** — PRD + design + existing master-plan + project language.
2. **Generate light docs in parallel** — `techstack-light` + `site-map` + `environments` as parallel subagents. Wait for all.
3. **Generate `functional-spec`** — depends on site-map; runs after parallel batch.
4. **EXPAND master-plan** — dispatch as its OWN subagent with fresh context (heaviest single output). It reads PRD + design HTMLs + functional-spec and adds phases 1+ to the existing Phase 0. Includes (in order): task 1.1 = "Pick design direction from HTMLs (write PICK.md)", task 1.2 = "Generate `.ai/design.md` from picked design", task 1.3 = "Write database-architecture.md", task 1.4 = "Write detailed techstack rationale", then 1.5+ = migrations, types, etc. — these are NOT written now; the master-plan just defines them.
5. **Generate per-task cards** — parallel subagents, one per task in the new phases 1+.
6. **Cross-link** — every doc references siblings instead of duplicating.
7. **Flip 0.4 → `[x]`** in master-plan.
8. **Show + confirm** — supervised mode presents the expanded master plan for trim/reorder. Autonomous mode logs and proceeds.

## Phase 1 — What we generate (and what we DON'T)

| Doc | Generated here? | Why / where instead |
|---|---|---|
| `.ai/techstack.md` (LIGHT — names + 1-line each) | ✅ yes | Just enough for env bootstrap; full rationale is Phase 1 task 1.4 |
| `.ai/site-map.md` | ✅ yes | Routes are derivable from PRD MVP + page type from `02-design-create-n-htmls` |
| `.ai/functional-spec.md` | ✅ yes | User stories follow from PRD; cheap to draft |
| `.ai/environments.md` | ✅ yes | Env tier matrix follows from techstack pick |
| `.ai/master-plan.md` (EXPAND existing — add phases 1+) | ✅ yes (as subagent) | Phase 0 already in place; adding phases 1+ is the central scaffold deliverable |
| `.ai/master-plan-points/<id>.md` (cards) | ✅ yes (parallel subagents) | One per task in phases 1+; light placeholders OK |
| `docs/design-ideas/<...>/PICK.md` (design pick rationale) | ❌ NO | Phase 1 task 1.1 — subagent reads N HTMLs, picks best, writes PICK.md with chosen file + rationale + proposed adjustments |
| `.ai/design.md` | ❌ NO | Phase 1 task 1.2 — different subagent reads PICK.md + chosen HTML, distills tokens (colors, typography, spacing, motion) into `.ai/design.md` |
| `.ai/database-architecture.md` | ❌ NO | Phase 1 task 1.3 — dedicated agent with fresh context for schema design |
| `.ai/techstack.md` (DETAILED rationale) | ❌ NO | Phase 1 task 1.4 — dedicated agent expands the light version |

**Why two design tasks (1.1 + 1.2) instead of one:**
- **Audit trail.** Pick is a decision; token distillation is execution. Different decisions deserve different cards.
- **Adjustments.** The pick subagent may propose small adjustments to the chosen HTML (e.g., "design-01 picked, but warm the orange `#FF8A00` → `#FF8500` for slightly higher contrast"). Those adjustments get applied by the design.md subagent. One agent doing both invites it to silently change the design without flagging.
- **Same lifecycle as everything else.** Pick + distillation flow through standard executor (worktree → card → simplify pass → PR → merge) — no inline orchestrator decisions.

**Why design tasks (1.1 + 1.2) come BEFORE database (1.3):** Design tokens inform component-level Definition of Done (e.g., responsive DoD, accessibility floor) and sometimes the schema needs to support theme/preference state surfaced by the design. Picking before schema means schema is informed; picking after means re-work risk.

**Why drop database-architecture from scaffold:** schema design is too important to share an agent with 5 other docs. Cramming it in produces shallow tables. As a Phase 1 task it gets a dedicated agent with full context, full lifecycle (worktree, simplify, PR), and direct feedback from the early data tasks (1.5 migrations, 1.6 types) that follow in the same wave.

## Phase 2 — Dispatch order

The flow is **auto-chained** — once the skill kicks off, the main agent does NOT make decisions between batches. Each batch's completion triggers the next batch automatically. The main agent's only job here is monitoring and surfacing failures.

**Batch 1 (parallel — 3 subagents):** techstack-light + site-map + environments. ~3-5 min each.

**Batch 2 (single subagent):** functional-spec. Depends on site-map (it links stories to routes). Auto-dispatched as soon as Batch 1 returns.

**Batch 3 (single, dedicated subagent):** master-plan EXPANSION. Reads PRD + design + functional-spec; adds phases 1+ to the existing Phase 0; includes the canonical Phase 1 (schema + foundation) per `references/canon-templates/master-plan-template.md`. **MUST produce a Waves table** (see required format below). ~10-15 min.

**Batch 3.5 (conditional, single subagent — wave-fixer):** **Auto-dispatched immediately after Batch 3 returns** if any of these hold:
- Master-plan lacks a `## Recommended order — Waves` section
- Waves table uses any symbols other than `∥` (parallel) and `→` (sequence)
- A non-`[x]` task in phases 1+ doesn't appear in any wave
- Phase 1 design tasks 1.1 → 1.2 are marked `∥` instead of `→` (illogical — design.md depends on pick)
- Phase 1 schema tasks (1.3 → 1.4 → 1.5 → 1.6) are marked `∥` instead of `→` (illogical — types depend on migrations depend on schema)
- Any wave has > 5 parallel tasks AND those tasks plausibly share files

The wave-fixer subagent reads the master-plan, the functional-spec, and the per-task summaries from Batch 3, then re-edits ONLY the `## Recommended order — Waves` section to fix the issue. Does NOT touch task definitions. Returns a one-line summary of the fix. **The main agent does NOT re-edit the Waves table inline — always dispatch the wave-fixer.**

**Batch 4 (parallel — N subagents, one per new task — auto-dispatched as soon as Batch 3 [or 3.5 if it ran] returns):** per-task cards for every task in phases 1+. One subagent per card. Light placeholders are fine — `05-plan-executor` subagents fill the implementation log later. **The main agent does NOT create these cards inline; they are always dispatched from this batch.**

**Important:** Batches 1→2→3→(3.5)→4 are sequential at the batch level, parallel within. The main agent dispatches each batch as soon as the prior one returns — there is no "wait for user" or "re-read transcripts" step in between. If a batch fails, the main agent dispatches a recovery subagent (not inline retry).

## Phase 2 — Document specs

Each doc has a clear shape. Detailed templates are in `references/canon-templates/` — read the relevant template before generating each doc.

### `.ai/techstack.md` (LIGHT version)
- Frontend / backend / database / hosting / auth / monitoring / testing — each with just the picked tool name + a single line saying what it does.
- Detailed rationale ("why this stack") is deferred to Phase 1 task 1.4.
- Cross-reference: PRD's "Tech stack" section is the *what*; this doc is just the canonical names list. The *why* comes later.

### `docs/design-ideas/<...>/PICK.md` and `.ai/design.md` — NOT WRITTEN HERE

These are Phase 1 master-plan tasks 1.1 and 1.2 — the FIRST execute tasks, before schema.

- **Task 1.1 — Pick design direction.** Subagent reads all N HTMLs in `docs/design-ideas/<timestamp>_<page-type>-explorations/`, evaluates each against PRD criteria (target users, tone, brand constraints, conversion goals, accessibility floor), picks the best fit. Writes `docs/design-ideas/<...>/PICK.md` with: chosen file path, rationale (3-6 sentences referencing PRD criteria), proposed adjustments (optional list of small tweaks), and "what to distill into design.md" (key tokens, primitives, signatures the design.md subagent should extract).
- **Task 1.2 — Generate `.ai/design.md`.** Different subagent reads PICK.md + the chosen HTML, distills the design system into `.ai/design.md` — tokens (colors, typography, spacing, motion), responsive patterns, primitives, visual signatures. Applies any adjustments from PICK.md.

### `.ai/database-architecture.md` — NOT WRITTEN HERE
- This doc becomes Phase 1 master-plan task 1.3.
- Scheduled after design (1.1, 1.2) because design tokens sometimes shape schema (theme persistence, user preferences, etc.).
- A dedicated agent in `05-plan-executor` writes it with fresh context — schema design needs full attention.

### `.ai/site-map.md`
- Route inventory — every URL the MVP exposes, with auth gating + primary components.
- Path conventions (e.g., `(marketing)/` group, `(app)/` group, `admin/` flat).
- Placeholders for Phase 2+ routes (so future work has a slot).

### `.ai/functional-spec.md`
- User stories (one per primary feature from PRD), key flows, task-to-need mapping.
- Format: "As a <user>, I want to <action> so that <outcome>." → links to the master-plan task that delivers it.

### `.ai/master-plan.md` (EXPAND, don't create)
- **The single source of truth for scope and progress.** Already exists with Phase 0 from `01-discover-idea`.
- Add phases 1+ following the canonical template at `references/canon-templates/master-plan-template.md`.
- **Phase 1 is fixed and ordered** (canonical sequence — don't reorder, don't drop):
  - `1.1` = Pick design direction from HTMLs → writes `docs/design-ideas/<...>/PICK.md`
  - `1.2` = Generate `.ai/design.md` from picked design + adjustments
  - `1.3` = Write `database-architecture.md`
  - `1.4` = Write detailed `techstack.md` rationale
  - `1.5` = Migrations (depend on 1.3)
  - `1.6` = Types regen (depends on 1.5)
  - `1.7+` = Foundation primitives (layout shells, design-token implementation, copy infra) — informed by 1.2 design.md
- For non-UI projects (CLIs, libraries, no `02-design-create-n-htmls` output): skip 1.1 + 1.2; renumber so database-architecture is 1.1.
- Phases 2+ are project-specific — pull features from the PRD's MVP + functional-spec.
- Inline status code per task: `[ ]` → `[~]` → `[c]` → `[t]` → `[x]` (with `[-]` for doc-only). Phase 0 statuses stay as the discover-idea skill set them.
- Each task has a one-line description in the plan; details live in `.ai/master-plan-points/<id>.md`.
- See `references/master-plan-grammar.md` for status semantics and the Phase 0 special conventions.

#### REQUIRED — Waves table (the executor's iteration unit)

The expanded master-plan MUST contain a `## Recommended order — Waves` section. This is not optional. `05-plan-executor` reads this table to determine cohort dispatch. **Without it the executor stops and refuses to dispatch.**

Format (verbatim shape — do not invent variants):

```markdown
## Recommended order — Waves

**Starting point:** _W1 — <wave-name>_

- **W0** — Phase 0 (planning, sequential, completed by orchestrator)
- **W1** — <wave-name>: <task-id> ∥ <task-id> ∥ <task-id> → <task-id> ∥ <task-id>
- **W2** — <wave-name>: <task-id> ∥ <task-id> ...
- **W3** — ...
```

Symbols:
- `∥` — parallel: tasks dispatched concurrently in one cohort (executor caps at 2-3 if shared state).
- `→` — sequence: wait for everything to the left to merge before dispatching the right side.

Hard rules for waves:
1. **Every non-`[x]` task in phases 1+ MUST appear in exactly one wave.** Tasks not in waves are invisible to the executor.
2. **Foundation goes in W1.** Layout primitives, design tokens, copy infrastructure, brand assets — all in W1 because everything later depends on them.
3. **W2+ should rarely have more than 5 parallel tasks.** Above 5, you'll hit shared-resource collisions (DB seeds, ports, type-package edits).
4. **Use `→` honestly.** A task that imports from another task's file MUST be sequenced after it, not parallel.
5. **Phase 1 design + schema chain is sequential.** 1.1 → 1.2 → 1.3 → 1.4 → 1.5 → 1.6 — design.md depends on pick, schema depends on design tokens, types depend on migrations. No parallelism in this chain.

Example shape (real project):

```
- **W1** — Foundation: 1.1 → 1.2 → 1.3 → 1.4 → 1.5 → 1.6 ∥ 1.7 ∥ 1.8 ∥ 1.9
- **W2** — Marketing core: 2.1 ∥ 2.2 ∥ 2.3
- **W3** — Treść: 3.1 → 3.2 ∥ 3.3 ∥ 3.4 → 3.5 (×7) ∥ 3.6 ∥ 3.7
```

If the master-plan you generate lacks a Waves table or uses different symbols than `∥`/`→`, the orchestrator will reject it and re-dispatch this skill.

**Dispatch as its own subagent.** Master-plan expansion is the heaviest single output (50-200 lines + reasoning about waves). Give it fresh context. The subagent's prompt should be ~150 lines: the canonical template + PRD + design + functional-spec, with a clear "EXPAND, don't create" instruction. Returns: completion summary + path. Don't return the full file contents (already on disk).

### `.ai/master-plan-points/<id>.md`
One card per task, generated alongside the master plan. Each card has:
- Task description (verbose form of the plan-row line)
- Acceptance criteria (testable bullets)
- Dependencies (other task IDs)
- Files to touch (best-guess)
- Test plan (unit / integration / e2e)
- Status changelog (filled in during execution)

Generate cards for ALL tasks at scaffold time — even if some are placeholder. Empty cards are better than missing cards (subagents can fill them in).

### `.ai/environments.md`
- Environment matrix (local / staging / prod) with services per env.
- Git branching strategy (trunk-based vs gitflow — `04-bootstrap-env` enforces).
- Test user credentials (placeholder format, real values in `.env.local`).
- Cross-reference: `apps/*/.env.example` lists keys; this doc lists *how they're sourced and rotated*.

## Phase 3 — Cross-linking discipline + CLAUDE.md enrichment

After generating every doc, run a final pass that:
- Replaces any duplicated fact with a cross-reference.
- Adds back-references (`See also: .ai/X.md`) at the bottom of related docs.
- Verifies every master-plan task has a corresponding card file.
- **Enriches CLAUDE.md** (created as a skeleton by `01-discover-idea`):
  - Updates the "Planning Structure" table to list ALL generated `.ai/*.md` docs with their purposes (PRD, design, techstack-light, site-map, functional-spec, environments, master-plan, master-plan-points/).
  - Adds a "Repo Layout" section if the project has a defined directory structure.
  - Adds any project-specific conventions surfaced in the PRD or design (e.g., specific naming patterns, route conventions).
  - **Never overwrites existing user content** — only ADDS missing sections or updates the Planning Structure table. Use Edit (not Write) so existing sections are preserved.

DRY rule: if a fact appears in two docs, pick the one where it's most natural and link from the other. Common DRY violations:
- Stack listed in PRD AND techstack → keep both, but PRD has 1-line summary, techstack has the rationale.
- Routes listed in site-map AND functional-spec → site-map owns the inventory; functional-spec links to it per story.
- Schema in database-architecture AND task cards → database-architecture owns; cards link.

## Phase 4 — Confirmation (supervised mode only)

Present the generated canon to the user as one structured turn:

```
**`.ai/` canon scaffolded.**

Generated:
- techstack.md (light — names + 1-line each)
- site-map.md — <N routes>
- functional-spec.md — <N user stories>
- master-plan.md — <N tasks across M waves>
- master-plan-points/ — <N cards>
- environments.md — <local + staging + prod>

(Design pick + `.ai/design.md` are Phase 1 master-plan tasks 1.1 + 1.2; `database-architecture.md` is task 1.3; detailed `techstack.md` rationale is task 1.4 — all written by `05-plan-executor` agents with fresh context, not here.)

The master plan has <N> tasks. Estimated execution time: <Y hours> with parallel subagents.

Reply:
- "ok" / "go" → lock the canon and proceed to Phase 4 (env provisioning)
- "trim: <task IDs>" → remove specific tasks (and their cards) before locking
- "redo master-plan: <feedback>" → regenerate just the master plan with feedback
- "abort" → stop
```

In **autonomous mode**, skip this step and append decisions (number of tasks, waves, any tasks deliberately deferred) to `.ai/agent-logs/autonomous-decisions-<date>.md`.

## Universal quality rails

1. **One source of truth per fact.** If you write a thing in two places, link don't duplicate.
2. **Cards are exhaustive** — every plan task has a card, even if minimal. Easier to fill in than to create later.
3. **Master plan is realistic.** Don't list 200 tasks for a 5-feature MVP. Aim for 20-60 tasks across 3-7 waves.
4. **Wave parallelization is honest.** Mark `∥` only when tasks truly don't share files; mark `→` when they do.
5. **Project language wins for `.ai/` docs.** Code/comments stay English; planning docs match the project's primary language.

## When the orchestrator invokes this skill

The orchestrator passes:
- `language`: project language for `.ai/` docs
- `decision_log_path`: where to append autonomous decisions
- `mode`: `supervised` | `autonomous`

Standalone invocation: assume `mode=supervised`, English language unless `CLAUDE.md` says otherwise, no decision log.

## Reference files

- `references/canon-templates/` — one template per doc with the exact structure
- `references/master-plan-grammar.md` — status codes, wave format, ID convention

## Why this skill is shaped this way

- **Dependency order is non-negotiable.** Generating master-plan before techstack means tasks reference tools that haven't been picked yet.
- **Master plan immutability matters.** Once locked, downstream phases (execute, review) treat the numbered IDs as identity. Renumbering breaks every task card and findings log.
- **Cards generated upfront** avoid the "execute phase invents structure on the fly" anti-pattern. Subagents fill cards with details; they don't create them.
- **Bilingual support** is built in. The convention: `.ai/` planning docs in the project's primary language (whatever the team and stakeholders share); code, comments, and commit messages in English. The orchestrator passes the `language` signal — sub-skills honor it.
