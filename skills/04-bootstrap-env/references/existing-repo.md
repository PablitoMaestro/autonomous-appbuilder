# Recipe — Existing Repo (Detect + Adapt)

The detect-and-adapt path. The repo already has code; the job is to figure out what stack it's on, what's already provisioned, and what's missing — then fill the gaps without touching working code.

This recipe is a diagnostic tool first, a provisioning tool second. Most of the work is reading.

## When this recipe runs

Pick this recipe when ANY of:

- The repo has tracked code beyond a `README.md` (`git ls-files | wc -l > 5`).
- A `package.json` exists at root or in `apps/*`.
- Any framework config exists: `next.config.*`, `vite.config.*`, `astro.config.*`, `nuxt.config.*`, `remix.config.*`, `svelte.config.*`, `tsconfig.json` with framework refs, `pyproject.toml`, `Cargo.toml`.
- A `CLAUDE.md` or `AGENTS.md` already documents conventions.

If the repo is empty (`git ls-files` returns nothing or only license/readme), use a greenfield recipe instead — `nextjs-supabase-vercel.md` is the default.

## Detection pass

Read these in parallel — do not start writing anything yet. The output of this phase is a single mental table.

| Source | What to extract |
|---|---|
| `package.json` (root + every `apps/*/package.json`) | `packageManager` field, `scripts.dev`, `scripts.build`, `scripts.typecheck`, `scripts.test:e2e`, `scripts.db:*`, dependencies (Next.js, Vite, Astro, `@supabase/ssr`, `@mux/mux-player-react`, etc.) |
| Lockfile | Confirms manager: `pnpm-lock.yaml` → pnpm, `bun.lockb` → bun, `package-lock.json` → npm, `yarn.lock` → yarn |
| `pnpm-workspace.yaml` / `turbo.json` / `nx.json` | Monorepo? Where do apps live? |
| `next.config.*` / `vite.config.*` / `astro.config.*` / `nuxt.config.*` | Framework + version + custom port/proxy settings |
| `apps/*/.env.example` | Authoritative key inventory — every key the app reads |
| `apps/*/.env.local` (if present, not tracked) | What's already populated locally |
| `supabase/config.toml` | Local Supabase configured? Which port? |
| `docker-compose*.yml` | Other local services (postgres, redis, mailpit) |
| `CLAUDE.md` / `AGENTS.md` / `README.md` | Stated conventions: commands to run, ports, test users, gotchas |
| `docs/runbook.md` | Already exists? If yes, this recipe only updates it; if no, it seeds one |
| `vercel.json` / `.vercel/project.json` | Vercel already linked? |
| `.github/workflows/*.yml` | What CI expects to be green |

After the read pass, synthesize:

```
Detected stack:
- Manager: pnpm 10.x (from pnpm-lock.yaml)
- Framework: Next.js 16 + React 19 (apps/web/package.json)
- Backend: Supabase (supabase/config.toml + @supabase/ssr in deps)
- Optional services: Mux player, Sentry (from deps)
- Dev command: `pnpm dev` → port 3000 (or PORT env override)
- DB types script: `pnpm db:types`
- E2E: Playwright (`pnpm test:e2e`)
- Runbook: docs/runbook.md (exists / missing)
```

If detection is ambiguous (e.g., two frameworks present), stop and ask the human which is the active one. Don't guess.

## Gap analysis

Compare detected state against required state:

1. **Env file gap.** Diff keys: `apps/<app>/.env.example` keys MINUS `apps/<app>/.env.local` keys. Every missing key needs a value or a `<MISSING>` marker.
2. **Local services gap.** For each config-implied service, check it's actually running:
   - `supabase/config.toml` exists → `npx supabase status` should return a running stack.
   - `docker-compose.yml` exists → `docker compose ps` should show services up.
   - If config exists but service is stopped, that's a gap.
3. **CLI auth gap.** For each remote service implied by deps/config, check auth state:
   - `gh auth status` — needed if remote provisioning is in scope.
   - `npx supabase login` (token in `~/.supabase/access-token`) — needed for cloud Supabase.
   - `npx vercel whoami` — needed for Vercel.
   - Missing logins are NOT auto-fixed; they are surfaced in the report.
4. **Generated artifacts gap.** Are typed clients fresh?
   - `packages/shared-types/src/database.ts` exists AND is non-empty AND newer than the latest `supabase/migrations/*.sql` → fresh. Otherwise stale; flag it.
5. **Runbook gap.** `docs/runbook.md` missing → seed it. Present but stale (references commands that don't exist in `package.json`) → update it.

## Fill-in steps

Run these only after the detection + gap analysis is done. Each step is conditional on a gap actually existing — don't run a step that has nothing to do.

### Step 1 — Reconcile env files

For every `apps/<app>` with an `.env.example`:

```bash
# Find missing keys
comm -23 <(sort apps/<app>/.env.example | grep -oE '^[A-Z_][A-Z0-9_]*') \
        <(sort apps/<app>/.env.local 2>/dev/null | grep -oE '^[A-Z_][A-Z0-9_]*')
```

For each missing key:

- If it's a local-Supabase key (`NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `DATABASE_URL`) → source from `npx supabase status -o env`. Remember to rename `SERVICE_ROLE_KEY` → `SUPABASE_SERVICE_ROLE_KEY`.
- If it's `NEXT_PUBLIC_APP_URL` and the dev server is local → write `http://localhost:<detected-port>`.
- If it's a remote secret (Sentry DSN, Mux env key, Stripe key, etc.) → mark as `<MISSING>` in `.env.local` and explain in the report. **Never invent it.**

Append (don't overwrite) so any human-supplied secrets in `.env.local` survive. After writing, re-diff to confirm zero missing keys.

### Step 2 — Boot local services

Only services whose config exists in the repo:

```bash
# Supabase
[ -f supabase/config.toml ] && npx supabase status >/dev/null 2>&1 || npx supabase start

# Docker compose
[ -f docker-compose.yml ] && docker compose ps --format json | grep -q '"State":"running"' || docker compose up -d
```

Wait for each to report healthy before proceeding. If a service fails to start, stop and report — don't paper over.

### Step 3 — Generate types if scripts exist

If `package.json` declares any of these scripts, run them now (idempotent):

```bash
jq -e '.scripts."db:types"' package.json >/dev/null && pnpm db:types
jq -e '.scripts."db:seed-storage"' package.json >/dev/null && pnpm db:seed-storage
```

After regen, run `pnpm typecheck` to confirm the generated types compile against the rest of the codebase. If typecheck fails on generated types, the local schema drifted from the code — flag it in the report and let the human decide (don't auto-migrate).

### Step 4 — Seed `docs/runbook.md` if missing

Use detected commands, not generic placeholders. Skeleton:

```markdown
# Runbook — <project-name detected from package.json>

## Local development
- Manager: <pnpm|bun|npm>
- First boot: `<install>` → `<start local services>` → `<dev>`
- Reset DB: `<db:reset script if present>`
- Regenerate types: `<db:types script if present>`

## Env vars
[Table from detected `.env.example` — local source per key]

## Deploy
[If vercel.json detected → `npx vercel --prod`; else placeholder for human]

## Backup / DR
[Skeleton — flesh out as services accumulate]

## Cron / monitoring / contacts
[Placeholders]
```

If `docs/runbook.md` already exists, do NOT overwrite. Instead, diff against detected commands and append a "## Detected drift" section listing mismatches for the human to reconcile.

### Step 5 — Verify dev server boots

Use the detected dev command + port. Background it, poll, kill the exact PID:

```bash
PORT=<detected> pnpm dev &
DEV_PID=$!
for i in {1..30}; do
  curl -fsS "http://localhost:<detected>" >/dev/null 2>&1 && break
  sleep 2
done
kill "$DEV_PID" 2>/dev/null
```

If boot fails, read the dev-server log, fix the root cause (missing env key, schema drift, port collision). Don't `pkill -f "next dev"` — that wipes parallel agents' servers.

## What this recipe will NOT do

- **No scaffolding.** No `pnpm create next-app`, no new framework configs, no new app directories. The repo's structure is law.
- **No rewrites.** No reformatting `package.json`, no normalizing scripts, no migrating from npm to pnpm.
- **No upgrades.** No bumping framework or library versions, even if they're behind.
- **No new deps beyond `pnpm install`.** If a runtime dep is missing, that's a code bug — flag it, don't `pnpm add` your way around it.
- **No mass file creation.** Only `apps/<app>/.env.local` (gitignored) and `docs/runbook.md` (if missing) get written. Everything else is read-only.
- **No silent overwrites.** Existing `.env.local` values are preserved; existing `docs/runbook.md` gets a drift section appended, not a rewrite.

## Failure modes + fixes

- **Two frameworks detected (Next + Vite both present)** → stop and ask the human which is active. Common in repos mid-migration.
- **`.env.example` missing entirely** → derive the key list from grepping `process.env.` across the source tree, present it to the human for confirmation before writing `.env.example`.
- **Local Supabase config exists but Docker isn't running** → ask user to start Docker Desktop. Cannot self-resolve.
- **`pnpm db:types` produces an empty file** → migrations not applied. Run `npx supabase db reset` only if the user confirms (it wipes local data).
- **Generated types compile-fail** → schema drift. Don't auto-migrate; report the diff and let the human reconcile.
- **Detected port already taken** → set `PORT=<next-free>` in `.env.local` and update `NEXT_PUBLIC_APP_URL` to match. Note in the report.
- **`docs/runbook.md` is stale (references deleted scripts)** → append "## Detected drift" section; don't rewrite. Stale runbooks usually encode tribal knowledge worth preserving.
- **Monorepo with multiple apps, only one is the target** → ask the human which app is in scope before reconciling envs. Default: the one with a `dev` script and a framework config.
