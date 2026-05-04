# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository. It is the entry point — for product/planning context follow the links into [`.ai/`](.ai/README.md).

## Project overview

> _TODO — one-line product description._ Read [`.ai/PRD.md`](.ai/PRD.md) for the full picture (scope, users, constraints, NFRs, decisions).

The project has a Next.js frontend and an **optional** FastAPI side-car. Use FastAPI only when something genuinely cannot live as a Next.js Route Handler / Server Action. Default to Next.js-only.

## Project language

> _TODO — set the language convention for this project._

Default convention: **code, identifiers, and commit messages in English**; planning docs in whatever language the team and stakeholders share. If non-English, keep code English regardless.

## Planning structure

**Reading order:** [`PRD`](.ai/PRD.md) → [`functional-spec`](.ai/functional-spec.md) → [`master-plan`](.ai/master-plan.md) → [`techstack`](.ai/techstack.md) → task card.

The full doc index — what each `.ai/` file owns and how they cross-reference — lives in [`.ai/README.md`](.ai/README.md). This file does not duplicate it.

| File | Read this when… |
|------|-----------------|
| [`.ai/README.md`](.ai/README.md) | You need to know which doc owns what concept |
| [`.ai/PRD.md`](.ai/PRD.md) | You need scope, constraints, or "is X in or out?" |
| [`.ai/functional-spec.md`](.ai/functional-spec.md) | You need the user stories that justify a task |
| [`.ai/master-plan.md`](.ai/master-plan.md) | You're picking what to work on next, or updating a status |
| [`.ai/techstack.md`](.ai/techstack.md) | You're adding a dependency or changing infrastructure |
| [`.ai/database-architecture.md`](.ai/database-architecture.md) | You're touching schema, RLS, triggers, or storage buckets |
| [`.ai/design.md`](.ai/design.md) | You're adding UI — tokens, responsive, motion, components |
| [`.ai/site-map.md`](.ai/site-map.md) | You're adding, renaming, or removing a route |
| [`.ai/environments.md`](.ai/environments.md) | You're configuring envs, secrets, branching, or test users |
| [`.ai/master-plan-points/<id>.md`](.ai/master-plan-points/_template.md) | Working a specific task — record decisions + tests there |

## Repo layout (pnpm monorepo)

```
<project-root>/
├── apps/
│   ├── web/              # Next.js + React + TS + Tailwind + shadcn/ui (frontend + serverside via Route Handlers / Server Actions)
│   └── api/              # FastAPI — OPTIONAL side-car. Skip entirely if not needed.
├── packages/
│   └── shared-types/     # Generated Supabase DB types + future TS contracts
├── supabase/             # Migrations, seed.sql, local config (Supabase CLI)
├── .ai/                  # Planning docs — see .ai/README.md for the index
├── package.json          # pnpm workspace root
├── pnpm-workspace.yaml   # Workspace packages
└── tsconfig.base.json    # Shared strict TS config
```

## Local development

**Prerequisites:** Docker Desktop, Node.js 22 (see `.nvmrc`), pnpm 10+, Python 3.12+ with `uv` (only if using `apps/api/`). The Supabase CLI is pinned as a workspace devDependency, so `pnpm install` provisions it — invoke via `npx supabase` or `pnpm exec supabase`.

```bash
# First-time setup
pnpm install                 # Install JS/TS deps for the whole workspace
cd apps/api && uv sync       # Optional — only if using FastAPI

# Running the stack
npx supabase start           # Start local Supabase (Postgres, Auth, Storage, Realtime)
pnpm dev                     # Next.js on http://localhost:3000
pnpm api:dev                 # Optional — FastAPI on http://127.0.0.1:8000

# Database
npx supabase db reset        # Drop → migrations → seed.sql (clean state)
pnpm db:types                # Regenerate packages/shared-types/src/database.ts from live schema
pnpm db:reset                # Full reset (db reset + any post-reset hooks defined in scripts/)

# Quality / CI — run from repo root
pnpm typecheck               # tsc --noEmit across all workspace packages
pnpm lint                    # ESLint apps/web
pnpm build                   # Production build of apps/web
pnpm test:e2e                # Playwright e2e (auto-runs `supabase db reset` first)
```

## Key directories & files

| Path | Purpose |
|------|---------|
| `apps/web/src/app/` | Next.js App Router — route groups for marketing/auth/app, separate `admin/` tree. Route inventory in [`.ai/site-map.md`](.ai/site-map.md). |
| `apps/web/src/proxy.ts` | Next.js edge proxy. Calls `updateSession()` on every request to refresh Supabase auth cookies. |
| `apps/web/src/lib/supabase/` | Three-factory `@supabase/ssr` clients: `client.ts` (client components), `server.ts` (RSC / Server Actions / Route Handlers), `middleware.ts` (proxy). Plus `admin.ts` for service-role (server-only, bypasses RLS). Never import from `@supabase/supabase-js` directly except via `admin.ts`. |
| `apps/web/src/lib/auth/` | Auth Server Actions (`signIn`, `signUp`, `signOut`, …). Forms POST via `<form action={…}>`; success → `redirect()`, failure → returned `{error}` consumed by `useActionState`. |
| `apps/web/src/components/ui/` | shadcn/ui primitives. Add via `pnpm dlx shadcn@latest add <component>`. Local primitives (Card, Section, etc.) live alongside. |
| `apps/web/src/components/layout/` | Shared chrome — navs, shells, footer, theme toggle. Per-context shells (marketing / auth / app / admin) keep route groups visually distinct. |
| `apps/web/src/lib/utils.ts` | Shared helpers (e.g., `cn()` from shadcn). |
| `packages/shared-types/src/database.ts` | Auto-generated Supabase types — **never edit by hand**. Regenerate with `pnpm db:types`. |
| `supabase/config.toml` | Local Supabase config (ports, auth, storage). |
| `supabase/migrations/` | Versioned schema migrations (timestamped SQL files). **Never edit committed migrations** — create a new one. |
| `supabase/seed.sql` | Test fixtures applied after `supabase db reset`. |

**Schema changes:** create a NEW migration file (`supabase/migrations/<timestamp>_<name>.sql`), never edit existing ones. After applying, run `pnpm db:types` to refresh TS types, and update [`.ai/database-architecture.md`](.ai/database-architecture.md).

## Git branching

Trunk-based with optional demo branch:

- **`main`** — production, PRs merge here.
- **`preview`** *(optional)* — disposable demo aggregator with a stable preview URL. Rebuild via `git reset --hard main` + merge WIP + force push. **Never PR from `preview` → `main`.**
- **`feature/*`** — short-lived dev branches, PR to `main`.

## Task status workflow

Status codes and the three-condition `[x]` rule are defined once, in [`.ai/master-plan.md` § Status legend](.ai/master-plan.md#status-legend). When working a task, update **both**:

1. The task card in `.ai/master-plan-points/<id>.md` — fill in implementation, files changed, test results.
2. The status in `.ai/master-plan.md` — flip the inline code.

A task is `[x]` only when code is merged, unit/integration tests pass, AND a visual e2e walkthrough is green. Docs-only tasks use `[-]` for tests and become `[x]` when deliverables are complete.

## Browser tooling for frontend work

Four browser drivers are available, each with a distinct sweet spot. Pick the lightest one that can do the job.

| Tool | Invocation | When to reach for it |
|------|------------|----------------------|
| `claude-in-chrome` (MCP) | MCP tool calls | Quick scouting in your *real* logged-in Chrome — cookies/auth preserved. Cannot emulate viewport below the OS window minimum (~500px on macOS). Console capture is lazy (starts at first `read_console_messages` call). |
| `chrome-devtools-mcp` (MCP) | MCP tool calls | Ad-hoc audits: `lighthouse_audit`, `performance_start_trace`, CDP-grade `emulate` for mobile viewports. **Only tool with Lighthouse/perf-trace.** |
| `agent-browser` (CLI) | Bash (`agent-browser <cmd>`) | Shell-scriptable one-shot emulation. Best when looping across viewports/devices or chaining actions in a pipeline. Same engine as Playwright, no `.spec.ts` needed. |
| `Playwright` | `.spec.ts` test files | The committed `[x]` visual e2e gate — repeatable, CI-grade. Use only when the flow needs to stay green over time. |

All four can read console logs and network requests. The split is about ergonomics, not capability.

**Rule of thumb:**
- "Does it load? What's on it?" → `claude-in-chrome`
- "Is the mobile layout broken? A11y? Perf?" → `chrome-devtools-mcp`
- "Check this across 4 viewports / 6 devices" → `agent-browser` (loop in bash)
- "Will this stay green in CI?" → Playwright

When two tools can do the job, prefer less setup (MCP > CLI > Playwright) unless it's going into CI.

**Parallel worktrees must not share ports or process namespaces.** When dispatching ≥2 agents that each run a dev server, assign each a unique `PORT` (e.g., 3004, 3008) and forbid broad process kills — only kill the exact PID you started. A `pkill -f "next dev"` will wipe every parallel agent's server, not just yours.

## DRY & simplicity — don't repeat, don't overengineer

One principle across code AND docs: reuse what exists, pick the simplest proven path.

- **Single source of truth.** Never duplicate info between files — cross-reference. Before writing new content or creating a helper/file, check if something already covers it.
- **Be brief.** One tight sentence beats three padded ones. A short doc that gets read beats a thorough one that gets skimmed.
- **Prefer the boring path.** Use existing libs, shadcn primitives, stock Supabase/Next patterns before inventing. Custom abstractions, clever generics, and speculative "flexibility" are debt.
- **Don't design for hypothetical futures.** No flags or wrappers "just in case." Abstract on the *third* real duplication, not the first.
- **Trust the stack.** Only guard at true system boundaries (user input, external APIs) — don't re-validate framework guarantees.

When in doubt: link don't duplicate, and pick the duller solution.

The four DRY rules baked into [`.ai/`](.ai/README.md#dry-rules-baked-into-this-folder) (one home per concept, doc index lives only in `.ai/README.md`, status legend lives only in `.ai/master-plan.md`, routes vs code paths split) are the load-bearing constraints — read them before adding any new doc.

## Keeping documentation up to date

All `.ai/` planning documents AND this CLAUDE.md are **living documents**. When your work changes any of the following, update the relevant file **before finishing**:

| File | Update when… |
|------|--------------|
| **`CLAUDE.md` (this file)** | A new convention worth other agents knowing emerges; a new key file/directory was added; a workflow / test / deploy command changes; a recurring pitfall is discovered; a breaking architectural decision is made. Use `Edit` (never `Write`) — only add new content; preserve everything else. |
| [`.ai/database-architecture.md`](.ai/database-architecture.md) | Adding/modifying/removing tables, columns, ENUMs, RLS policies, storage buckets, or triggers. |
| [`.ai/environments.md`](.ai/environments.md) | Environment config changes (new services, access patterns, git strategy, test creds). |
| [`.ai/master-plan.md`](.ai/master-plan.md) | A task's status changes, or a wave closes (move the *Starting point* pointer). |
| [`.ai/master-plan-points/<id>.md`](.ai/master-plan-points/_template.md) | Working a task — fill in implementation, files changed, test results, key decisions. |
| [`.ai/techstack.md`](.ai/techstack.md) | A technology decision changes (new library, replaced service). |
| [`.ai/design.md`](.ai/design.md) | Tokens, palette, typography, motion, signatures, or responsive patterns change. |
| [`.ai/site-map.md`](.ai/site-map.md) | Adding, renaming, or removing a route. |
| [`.ai/PRD.md`](.ai/PRD.md) | Requirements, constraints, or architectural decisions change. |

Stale documentation is worse than no documentation — it actively misleads. If you touch code that a document describes, verify the document is still accurate.

**Threshold for CLAUDE.md updates:** only when something genuinely worth knowing emerged. Don't add noise. A good test: would a fresh agent reading CLAUDE.md from cold benefit from this addition? If not, skip it.

## Most important notes

- **Decide on a worktree before starting work.** If the task needs a new branch, default to spinning up a `git worktree`. Work that stays on the current branch can usually proceed in place.
- **Never switch branches inside the main working directory unless the user explicitly commands it.** Other agents may be working there with uncommitted changes; switching branches in-place silently moves their work onto a branch they didn't choose. Use a worktree instead. If the user does direct an in-place switch, run `git status` first and warn about untracked/unstaged files that will travel with the switch.
- **Worktrees don't carry gitignored files** — `apps/web/.env.local`, Playwright caches, etc. live per-tree. When entering a fresh worktree: `pnpm install`, then create `apps/web/.env.local` (copy from main repo, or regenerate from `npx supabase status -o env`). Same in reverse: removing a worktree doesn't restore files in the main repo.
- **Parallel e2e runs need port + reset isolation.** Only the first `pnpm test:e2e` should run `supabase db reset`; followers race the shared local Supabase and wipe each other's seeds. Convention: launch followers with a per-worktree opt-out env var (e.g. `SKIP_DB_RESET=1`) and a unique `PORT`. Document the exact var name in [`.ai/environments.md`](.ai/environments.md).
- **Keep `CLAUDE.md` up to date** — if your work introduces crucial changes (new conventions, architectural decisions, workflow changes, new key files), update this file before finishing. Always explicitly inform the user when `CLAUDE.md` was modified and what changed.
