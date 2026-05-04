# Master Plan — <Project Name>

> Source of truth for scope and progress. Status codes: `[ ]` → `[~]` → `[c]` → `[t]` → `[x]` (or `[-]` for doc-only). Task IDs are **immutable** once locked. See `master-plan-grammar.md` for the full spec.

## Phase 0 — Discovery & scaffolding

This phase tracks the orchestrator's own planning work. Each task here is set by a specific orchestrator phase, not by `05-plan-executor`. Used by `00-build-app` for resume detection.

- [x] **0.1** — Tool preflight passed (CLIs + MCP servers + auth) · q
- [ ] **0.2** — PRD locked at `.ai/PRD.md` · q
- [ ] **0.3** — Design locked at `.ai/design.md` · q
- [ ] **0.4** — Plan scaffolded (this file expanded with phases 1+) · q
- [ ] **0.5** — Env bootstrapped + runbook seeded · q
- [ ] **0.6** — CLAUDE.md skeleton + AGENTS.md/GEMINI.md pointers initialized · q

(`01-discover-idea` flips 0.2 and writes the CLAUDE.md skeleton + AGENTS.md/GEMINI.md as thin pointers to CLAUDE.md (0.6 → [x]); `02-design-create-n-htmls` flips 0.3; `03-scaffold-ai-canon` flips 0.4 and enriches CLAUDE.md's Planning Structure table; `04-bootstrap-env` flips 0.5 and populates CLAUDE.md's Local Development section. Detect-existing-or-create — never overwrites existing files. AGENTS.md / GEMINI.md are pointer-only to keep CLAUDE.md as the single source of truth and avoid DRY violations.)

---

## Phase 1 — Schema + foundation docs

The heaviest planning work — done by dedicated subagents with fresh context, not crammed into the scaffold step. These tasks must complete before any feature work can begin because they define the data shape and tech rationale every feature depends on.

- [ ] **1.1** — Design data model and write `.ai/database-architecture.md` (tables, ENUMs, RLS, Storage buckets, triggers) · s · db
- [ ] **1.2** — Write `.ai/techstack.md` with picked tools + 1-line rationale per choice · q
- [ ] **1.3** — Write initial Supabase migrations from the schema doc · s · db
- [ ] **1.4** — Generate TS types and wire `pnpm db:types` script · q
- [ ] **1.5** — Repo skeleton + base scripts if not already in place · q
- [ ] **1.6** — Wire CI: GitHub Actions running typecheck + build on every PR · q

## Phase 2 — Design system + primitives

Apply design tokens, install component primitives, build the app shell.

- [ ] **2.1** — Apply design tokens from .ai/design.md (CSS vars, Tailwind config) · q
- [ ] **2.2** — Install + configure shadcn/ui (or stack equivalent) · q
- [ ] **2.3** — Build app shell (nav, footer, layout wrappers) · s
- [ ] **2.4** — Add light/dark theme toggle if design specifies · q

## Phase 3 — Auth + access control

- [ ] **3.1** — Wire auth provider (Supabase Auth) — email + password by default · s
- [ ] **3.2** — Create /login + /register pages with form validation · s
- [ ] **3.3** — Implement auth-gated route group (redirect anon → /login) · q
- [ ] **3.4** — Add admin role + admin route protection · q

## Phase 4 — Core features (MVP)

The user-facing functionality from PRD's MVP scope. Pull straight from `.ai/functional-spec.md`.

- [ ] **4.1** — <feature 1 from PRD MVP> · s
- [ ] **4.2** — <feature 2 from PRD MVP> · s
- [ ] **4.3** — <feature 3 from PRD MVP> · s

## Phase 5 — Cross-cutting concerns

- [ ] **5.1** — Wire Sentry (or stack equivalent) for error tracking · q
- [ ] **5.2** — Add Plausible / PostHog / equivalent for product analytics · q
- [ ] **5.3** — Accessibility audit + fixes (WCAG AA per NFR) · s

## Phase 6 — Tests + polish

- [ ] **6.1** — Add unit tests for pure logic helpers · s
- [ ] **6.2** — Add integration tests for Server Actions / API routes · s
- [ ] **6.3** — Add Playwright e2e for the golden path (PRD success criteria) · s · e2e
- [ ] **6.4** — Mobile viewport polish (responsive bug fixes) · q

## Phase 7 — Documentation

- [-] **7.1** — Update README.md with quickstart + architecture overview
- [-] **7.2** — Audit CLAUDE.md for accuracy after Phase 1-6 changes

## Phase 8 — Deploy + handoff

- [ ] **8.1** — Deploy preview to Vercel; verify all routes load · q
- [ ] **8.2** — Configure custom domain + SSL · q
- [ ] **8.3** — Production deploy go/no-go (HUMAN GATE) — needs sign-off
- [ ] **8.4** — Pilot user onboarding (HUMAN GATE) — needs real users
- [ ] **8.5** — Post-deploy smoke check + Sentry alert verification · q
- [-] **8.6** — Update CLAUDE.md with deploy notes + lessons learned

---

### Waves

| Wave | Tasks | Notes |
|---|---|---|
| W0 | 0.1 → 0.2 → (0.3 ∥ 0.6) → 0.4 → 0.5 | bootstrap journey; CLAUDE.md skeleton (0.6) parallel with design pick (0.3) |
| W1 | 1.1 → 1.2 → 1.3 → 1.4 → (1.5 ∥ 1.6) | schema before types; skeleton + CI parallel |
| W2 | (2.1 ∥ 2.2) → 2.3 → 2.4 | tokens + shadcn parallel, then shell, then theme |
| W3 | 3.1 → 3.2 ∥ 3.3 → 3.4 | auth provider first, then UI parallel, then admin gating |
| W4 | 4.1 ∥ 4.2 ∥ 4.3 | core features parallel (different files) |
| W5 | 5.1 ∥ 5.2 → 5.3 | wiring parallel, then audit |
| W6 | 6.1 ∥ 6.2 → 6.3 → 6.4 | unit + integration parallel, then e2e, then polish |
| W7 | 7.1 ∥ 7.2 | docs parallel |
| W8 | 8.1 → 8.2 → [8.3 HUMAN] → [8.4 HUMAN] → 8.5 → 8.6 | deploy chain with human gates |

### Plany dodatkowe / Side plans

| Plan | Status | Notes |
|---|---|---|
| (none yet) | | |

---

## Notes for the scaffold subagent

When generating a real master plan from this template:

1. **Phase 0 stays as-is.** Don't renumber or rename it. The orchestrator and sub-skills hard-reference `0.1` through `0.5`.
2. **Phase 1 also stays mostly as-is.** Tasks 1.1 (database-architecture) and 1.2 (techstack) are doctrine — they're the FIRST things plan-executor picks up because schema + stack rationale are prerequisites for every feature.
3. **Replace Phase 4 placeholders** with concrete features from the PRD's MVP section.
4. **Tune Phases 2-8** to project size — small MVPs may collapse Phase 5+6 into one "polish" phase.
5. **Keep estimates realistic** — `q` for ~30 min, `s` for ~2-4 h. Don't pad.
6. **Phase 8 always has 2 HUMAN GATES** (go/no-go + pilot). These are exempt from auto-execution.
7. **The waves table is your reasoning made visible** — every `∥` and `→` is a real claim about file overlap and dependency. Get it wrong and executor fights merge conflicts.

## Why Phase 1 is "schema + foundation docs"

The previous design crammed `database-architecture.md` and `techstack.md` rationale into the scaffold phase, alongside 4 other docs. That overloaded one agent with 6 design surfaces.

Now: scaffold writes lighter docs (site-map, functional-spec, environments, master-plan) AND seeds Phase 1 with explicit tasks for the heavy docs. Each heavy doc gets its own dedicated agent during plan execution — fresh context, focused thinking, full lifecycle (worktree, simplify, PR). Schema design is feature work; treating it that way produces better schemas.
