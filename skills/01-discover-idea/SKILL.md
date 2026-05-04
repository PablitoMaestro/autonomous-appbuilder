---
name: 01-discover-idea
description: Generate a `.ai/PRD.md` from a one-line idea, an empty repo, or a partially-described project. Picks the tech stack, defines scope, MVP, target users, NFRs, and success criteria. Detects existing intent from `CLAUDE.md` / `README.md` before asking; in autonomous mode, asks once and proceeds with sensible defaults. Use whenever the user says "draft a PRD", "I have an idea for an app", "let's plan a new project", "I want to build X", "scope this out", "help me start a new app", or starts brainstorming a new product — even if they don't explicitly ask for a PRD.
---

# 01 — Discover Idea

Produce a single concise `.ai/PRD.md` that downstream phases (design, scaffolding, execution, smoke test) can rely on as the source of truth for product intent. Detect what the repo already says before interrogating the human; when the repo is silent, ask a tight one-shot intake.

> **Invocation contract:** This skill is invoked via `Skill('01-discover-idea')` — typically by the `00-build-app` orchestrator at Phase 1, but also standalone. The caller MUST NOT write `.ai/PRD.md` themselves and then "skip" this skill — the PRD is THIS skill's output. If the file already exists from a previous run, this skill detects that and resumes instead of starting fresh.

The skill's value isn't the PRD template — it's the discipline of producing a PRD that's *specific enough to constrain* (so subsequent phases don't drift) without *over-specifying* (so the agent can still make sensible local decisions inside it).

## Workflow at a glance

1. **Detect** intent from existing repo files — `CLAUDE.md`, `README.md`, `.ai/*.md`, `package.json`, `docs/*.md`. Skip files that don't exist.
2. **Choose mode** — supervised (default) asks clarifying questions one at a time; autonomous (`mode=autonomous` in the addendum from the orchestrator) asks once and proceeds in 5 minutes regardless of reply.
3. **Pick the stack** based on the idea shape + repo signals + the tier rules below.
4. **Write** `.ai/PRD.md` using the template at the end of this file.
5. **Show + confirm** in supervised mode (one structured turn — accept / edit field / abort). Skip in autonomous mode; log decisions instead.

## Phase 1 — Detect

Read these in parallel and synthesize:

| Signal | Files |
|---|---|
| Existing project intent | `CLAUDE.md`, `AGENTS.md`, `README.md`, `.ai/*.md`, `docs/*.md` |
| Stack hints | `package.json` (deps), lockfiles (`pnpm-lock.yaml`, `bun.lockb`, `Cargo.lock`, `go.mod`, `pyproject.toml`), framework configs (`next.config.*`, `vite.config.*`, `astro.config.*`) |
| Brand / domain | `package.json` `name` + `description` + `keywords`, `LICENSE` org name |
| Audience cues | Visible language in any existing UI files; locale config; README language |

If the user passed an idea string (positional arg or in the conversation), treat it as a hard signal that supersedes detection conflicts.

**If the repo is sparse and no idea was given:** fall back to the intake (Phase 2). Don't fabricate a PRD from nothing.

## Phase 2 — Intake (only when detection is insufficient)

Present a single structured intake, multiple-choice where possible:

```
I'd like to ground the PRD before drafting. Answer any of these — skip what you don't care about:

1. **Product type** (multi-pick OK):
   [ ] SaaS web app   [ ] static content / docs   [ ] internal tool   [ ] CLI / library
   [ ] browser game   [ ] desktop app             [ ] mobile             [ ] other: ___

2. **Who uses it** (one line): ___

3. **Top 1–3 things it must do** (MVP scope): ___

4. **Aesthetic direction** (or "agent's choice"): ___

5. **Stack preference** (or "agent picks based on type"): ___

6. **Any non-obvious constraints** (compliance, target deployment, language, integrations): ___

Reply with answers, "go" to use defaults, or skip — autonomous mode will proceed in 5 minutes regardless.
```

In **supervised mode**, wait for a reply or "go". In **autonomous mode**, send this once, set a 5-minute soft timer in your reasoning, then proceed with defaults whether or not the user replies.

## Phase 3 — Pick the stack

Apply tier rules in order; first match wins:

| Idea / signal | Stack |
|---|---|
| Repo non-empty + framework detected | Use detected stack — adapt, don't replace |
| Idea mentions a stack explicitly ("Phaser game", "Tauri app", "Rails API") | That wins |
| SaaS / web app / unspecified default | **Next.js (App Router) + Supabase + Vercel + shadcn/Tailwind** |
| Static content / blog / docs site | Astro |
| Browser game | Phaser + Vite |
| Desktop app | Tauri + Vite |
| CLI / library | Bun + TypeScript |
| Mobile | Expo (React Native) |

In autonomous mode, log the pick + the rule that matched to `.ai/agent-logs/autonomous-decisions-<date>.md`.

## Phase 4 — Dispatch artifact-writer subagent

Once the dialogue is complete and decisions are made, the heavy formatting work (PRD body + master-plan stub + CLAUDE.md from template + AGENTS.md/GEMINI.md pointers) is dispatched to a single `general-purpose` subagent. Why dispatch:

- The PRD is a ~200-line structured doc — heaviest single output of this skill. Formatting it inline bloats the orchestrator's context with content that's about to live on disk anyway.
- CLAUDE.md is filled from a ~170-line template (`references/claude-md-template.md`). The subagent reads the template, substitutes project-specific bits, and writes — the orchestrator never needs to see the template body.
- All 5 files are independent enough to write in one subagent invocation; coordination cost is small.

### Subagent dispatch

Spawn one `general-purpose` subagent with this prompt:

```
You are dispatched to write 5 files at the start of a new project. The user just finished a discovery dialogue with the orchestrator; you have all decisions below. Your job is to format the artifacts and write them to disk. Do NOT re-litigate decisions — they're locked.

DECISIONS FROM DIALOGUE (verbatim):
- Idea: <verbatim user input>
- Product type: <e.g., SaaS web app>
- Audience (primary): <…>
- Audience (secondary, optional): <…>
- Problem: <…>
- Solution shape: <…>
- MVP scope (3-7 features): <list>
- Out of scope (Phase 2+): <list>
- Success criteria: <list>
- Non-functional requirements:
  - Performance: <…>
  - Accessibility: <…>
  - Compliance: <…>
  - Auth: <…>
  - Localization: <…>
  - Hosting: <…>
- Tech stack:
  - Frontend / Backend / Database / Hosting (one line each, with picked tool)
  - Why this stack (one sentence)
- Assumptions: <list>
- Open questions: <list — or "agent decided: X; Why: Y">
- Project name: <inferred from idea>
- Project language for .ai/ docs: <English | Polish | …>

FILES TO WRITE:

1. `.ai/PRD.md` — use the PRD template structure below. Write in <project language>. Don't pad; every line should add a constraint or a fact.

   Template:
   ```markdown
   # PRD — <Product Name>
   > One-line elevator pitch.
   ## Product
   ## Target users
   ## Problem
   ## Solution
   ## MVP scope
   ## Out of scope (Phase 2+)
   ## Success criteria
   ## Non-functional requirements
   ## Tech stack
   ## Assumptions
   ## Open questions
   ```

2. `.ai/master-plan.md` — Phase 0 stub only. Phases 1+ are stubbed for `03-scaffold-ai-canon` to expand.

   Use this exact body (substitute <Product Name>):
   ```markdown
   # Master Plan — <Product Name>

   > Source of truth for scope and progress. Status codes: `[ ]` → `[~]` → `[c]` → `[t]` → `[x]` (or `[-]` for doc-only). See `.ai/README.md` for cross-doc conventions.

   ## Phase 0 — Discovery & scaffolding

   - [x] **0.1** — Tool preflight passed (CLIs + MCP servers + auth) · q
   - [x] **0.2** — PRD locked at `.ai/PRD.md` · q
   - [ ] **0.3** — Design locked at `.ai/design.md` · q
   - [ ] **0.4** — Plan scaffolded (this file expanded with phases 1+) · q
   - [ ] **0.5** — Env bootstrapped + runbook seeded · q
   - [x] **0.6** — CLAUDE.md initialized + AGENTS.md/GEMINI.md pointers · q

   ## Phases 1+ — to be expanded by `03-scaffold-ai-canon`

   (stub — `03-scaffold-ai-canon` expands this section after design is locked)

   ---

   ### Waves

   | Wave | Tasks | Notes |
   |---|---|---|
   | W0 | 0.1 → 0.2 → (0.3 ∥ 0.6) → 0.4 → 0.5 | bootstrap journey; CLAUDE.md (0.6) parallel with design pick (0.3) |

   ### Plany dodatkowe / Side plans

   | Plan | Status | Notes |
   |---|---|---|
   | (none yet) | | |
   ```

3. `CLAUDE.md` (at repo root) — **DETECTION-FIRST: if it exists already, DO NOT overwrite. Skip and report "preserved existing CLAUDE.md".**

   If missing, the action depends on whether the picked stack is the default (Next.js + Supabase + Vercel monorepo) or something else:

   **Default stack (Next.js + Supabase + Vercel):**
   Read the canonical template at `<skill-path>/references/claude-md-template.md` (~170 lines). Substitute the `_TODO_` markers with project-specific facts:
   - "Project overview" — one-line product description from PRD
   - "Project language" — set to <project language>
   - Keep everything else as-is. The template's repo layout, local-dev commands, status workflow, browser tooling tiers, DRY principles, and "most important notes" sections are battle-tested.

   **Non-default stack (e.g., Vite SPA, Astro static, Bun CLI, Phaser game, Tauri desktop, etc.):**
   Use the canonical template as STRUCTURAL inspiration (same section ordering, same voice), but adapt:
   - **Preserve verbatim** (these are universal across stacks): `Project overview`, `Project language`, `Planning structure`, `Git branching`, `Task status workflow`, `DRY & simplicity`, `Keeping documentation up to date`, `Most important notes`.
   - **Rewrite for the picked stack**:
     - `Repo layout` — show the real directory structure for the stack (no monorepo if not applicable; no `apps/web/` if it's a flat Vite SPA; etc.).
     - `Local development` — real commands the stack uses (`bun dev` for Bun; `pnpm astro dev` for Astro; `npm start` for plain CLI; etc.). Match scripts the project will actually have in `package.json` after Phase 4.
     - `Key directories & files` — drop Next.js / Supabase paths; list paths real to the picked stack. For non-web stacks (CLI, library), this section may shrink to just `src/`, `package.json`, `README.md`.
     - `Browser tooling for frontend work` — DROP this entire section if the stack has no frontend (CLI, library, server-only). Keep + adapt for browser games (mention canvas-debugging notes), keep as-is for SPA/SSR web stacks.
   - For exotic stacks (Rust + Cargo, Go modules, Python with uv, etc.) — the universal sections still apply. Adapt the others using `package.json`-equivalent (e.g., `Cargo.toml`, `pyproject.toml`).

   Either way, the result is a stack-appropriate CLAUDE.md ~120-200 lines that downstream subagents can rely on. The template references `.ai/database-architecture.md`, `.ai/site-map.md`, etc. — those don't exist yet, but they will after Phase 3 (or won't, for stacks without DBs/routes; in that case omit the row from the Planning structure table). Forward references are fine; CLAUDE.md is for the entire project lifecycle.

4. `AGENTS.md` (at repo root) — **DETECTION-FIRST: skip if exists.**

   If missing, write this 5-line pointer (DRY: CLAUDE.md owns content, AGENTS.md just redirects):
   ```markdown
   # AGENTS.md

   This project's agent guidelines live in [CLAUDE.md](./CLAUDE.md).

   All agents (Claude Code, Codex, Gemini CLI, generic LLM tools) should read CLAUDE.md as the single source of truth for project conventions, planning structure, tech stack, and workflow rules. AGENTS.md exists only to redirect tools that look for this filename by convention.
   ```

5. `GEMINI.md` (at repo root) — **DETECTION-FIRST: skip if exists.**

   Identical to AGENTS.md, just rename the heading to `# GEMINI.md`.

WORKFLOW:
1. Check existence of CLAUDE.md, AGENTS.md, GEMINI.md (Bash: `ls -la CLAUDE.md AGENTS.md GEMINI.md 2>&1`).
2. Read `<skill-path>/references/claude-md-template.md` ONLY IF you'll need to write CLAUDE.md (i.e. it's missing).
3. Write all 5 files (skipping any that exist for #3-5).
4. Return ONE-LINE summary listing what was written/preserved + total line counts. Do NOT echo file contents.

Output format:
=== WRITER SUBAGENT REPORT ===
prd: written | <line count> lines
master_plan: written | <line count> lines
claude_md: written from template | preserved existing | <line count>
agents_md: written | preserved existing
gemini_md: written | preserved existing
language: <project language>
=== END REPORT ===
```

The orchestrator parses this report, sets `0.2` and `0.6` to `[x]` in the master-plan, then proceeds to Phase 5 confirmation.

## Phase 5 — Confirm (supervised mode only)

Present the draft PRD to the user as one structured message:

```
**PRD draft written to .ai/PRD.md.**

Key decisions made:
- Product type: <…>
- Stack picked: <…>  (rule: <…>)
- MVP scope: <N features>
- Out of scope: <N items>

Reply:
- "ok" / "go" → lock the PRD and proceed to Phase 2 (design exploration)
- "redo: <feedback>" → I'll regenerate with your feedback
- A specific edit ("change MVP item 3 to X", "add NFR for offline mode") → I'll apply and re-show
- "abort" → stop here
```

In **autonomous mode**, skip this step. Append a structured "Phase 1 decisions" block to `.ai/agent-logs/autonomous-decisions-<date>.md` with every load-bearing choice + a `Why:` line.

## Universal quality rails

1. **PRD is in the project's primary language.** If `CLAUDE.md` says "all `.ai/` docs are in Polish", the PRD is in Polish. If the project is English-first, write English. The orchestrator passes this signal in the addendum.
2. **No marketing fluff.** "Revolutionary platform that transforms how X" is noise. State what the thing does.
3. **MVP scope is what ships first, not what could ship eventually.** Move ambition to "Out of scope (Phase 2+)".
4. **Stack rationale fits in one sentence.** If you need a paragraph, you're justifying — pick a different stack.
5. **Open questions are flagged, not hidden.** If you decided something on the user's behalf, write `agent decided: <X>; Why: <Y>` so they can override.

## When the orchestrator invokes this skill

The orchestrator passes an addendum:
- `mode`: `supervised` | `autonomous`
- `idea`: the user's positional arg (may be empty)
- `language`: project language for the PRD (default: English; Polish if repo signals)
- `decision_log_path`: where to append autonomous decisions

When invoked standalone (not from the orchestrator), assume `mode=supervised`, ask the user for the idea if not in conversation, and skip the decision log.

## Reference files

- `references/intake-questions.md` — branching follow-ups when round 1 leaves gaps + autonomous defaults
- `references/claude-md-template.md` — the canonical CLAUDE.md template (project-language-agnostic, battle-tested across multiple projects). The Phase 4 artifact-writer subagent reads this and substitutes `_TODO_` markers.

## Why this skill is shaped this way

- **Detection beats interrogation.** Most repos with any prior thought have signals. Asking 20 questions to a user who already wrote a 3-line README is rude.
- **One-shot intake when needed.** A single structured intake respects the user's time more than a 10-turn dialogue. If they want depth, they'll redirect.
- **Autonomous mode is real autonomy.** "Ask once, proceed in 5 minutes" is not a gate — it's a courtesy. The agent must be ready to ship without a reply.
- **Heavy formatting → subagent.** The PRD body + CLAUDE.md template substitution + 3 small pointer files = ~400 lines of output. Doing this inline would bloat the orchestrator's context with content that's about to live on disk. Dispatching to a single fresh-context subagent keeps the main agent lean.
- **CLAUDE.md from a proven template, not from scratch.** The repo's `app-template/CLAUDE.md` (also vendored at `references/claude-md-template.md`) captures conventions battle-tested across multiple projects. Substituting 2-3 `_TODO_` markers beats writing CLAUDE.md from scratch every project.
- **Detection-first for CLAUDE.md / AGENTS.md / GEMINI.md.** Mature repos may already have these. Never overwrite — only fill missing slots.
