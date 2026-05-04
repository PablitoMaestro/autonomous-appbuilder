# Recipe — Next.js + Supabase + Vercel (default greenfield)

The default stack for SaaS web apps. Provisions local + remote services, populates env files, seeds the runbook.

## What this recipe provisions

- **Local:** Supabase CLI (postgres + auth + storage + studio), Next.js dev server, generated TypeScript types from the schema.
- **Remote (optional):** GitHub repo, Supabase cloud project, Vercel project linked to the repo.
- **Env files:** `apps/web/.env.local` (real values), `apps/web/.env.example` (schema with placeholders).
- **Runbook:** `docs/runbook.md` with deploy + secrets + DR + cron + monitoring sections.

## Required env keys

| Key | Local source | Remote source |
|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | `npx supabase status -o env` → `API_URL` | Supabase dashboard → API → URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | `npx supabase status -o env` → `ANON_KEY` | Supabase dashboard → API → anon key |
| `SUPABASE_SERVICE_ROLE_KEY` | `npx supabase status -o env` → `SERVICE_ROLE_KEY` (rename!) | Supabase dashboard → API → service_role |
| `DATABASE_URL` | `npx supabase status -o env` → `DB_URL` | Supabase dashboard → Connection string |
| `NEXT_PUBLIC_APP_URL` | `http://localhost:3000` | Vercel deployment URL |
| `SENTRY_DSN` (optional) | `<MISSING>` | Sentry → Project settings → DSN |

**Important rename:** Supabase CLI emits `SERVICE_ROLE_KEY` raw, but Next.js apps conventionally use `SUPABASE_SERVICE_ROLE_KEY`. Always rename when copying from CLI output.

## Provisioning steps

### Step 1 — Preflight
```bash
# Confirm CLIs (modern repos pin these as workspace devDependencies, so pnpm install provisions them)
node --version          # ≥ 20
pnpm --version          # ≥ 10
docker info             # Docker Desktop must be running
gh auth status          # for remote provisioning
npx supabase --version  # Supabase CLI
npx vercel --version    # Vercel CLI
```

If any are missing: install the missing one only. For Docker — stop and ask the user to install Docker Desktop manually (don't try to script it).

### Step 2 — Repo skeleton (if empty repo)

```bash
# Create monorepo structure
mkdir -p apps/web packages/shared-types supabase/migrations supabase/scripts .ai docs

# Workspace root package.json
cat > package.json <<'EOF'
{
  "name": "<project-name>",
  "private": true,
  "scripts": {
    "dev": "pnpm --filter web dev",
    "build": "pnpm --filter web build",
    "typecheck": "pnpm -r typecheck",
    "lint": "pnpm --filter web lint",
    "test:e2e": "pnpm --filter web test:e2e",
    "db:reset": "supabase db reset && pnpm db:seed-storage",
    "db:types": "supabase gen types typescript --local > packages/shared-types/src/database.ts",
    "db:seed-storage": "node supabase/scripts/seed-storage.mjs"
  },
  "devDependencies": {
    "supabase": "^1.x",
    "vercel": "^33.x"
  }
}
EOF

# pnpm workspace
cat > pnpm-workspace.yaml <<'EOF'
packages:
  - "apps/*"
  - "packages/*"
onlyBuiltDependencies:
  - supabase
EOF
```

### Step 3 — Scaffold Next.js app

```bash
cd apps/web
pnpm create next-app@latest . --typescript --tailwind --app --no-src-dir --import-alias "@/*" --eslint --turbopack
# Answer prompts: name=<project-name>, tailwind=yes, app router=yes
cd ../..
pnpm install
```

### Step 4 — Initialize Supabase locally

```bash
npx supabase init                   # creates supabase/config.toml
npx supabase start                  # ~30s; pulls images first time
pnpm db:types                       # generates packages/shared-types/src/database.ts
```

Capture all env values from `npx supabase status -o env`. Rename `SERVICE_ROLE_KEY` → `SUPABASE_SERVICE_ROLE_KEY`.

### Step 5 — Wire `@supabase/ssr`

```bash
cd apps/web
pnpm add @supabase/ssr @supabase/supabase-js
```

Create `apps/web/src/lib/supabase/{client,server,middleware,admin}.ts` per `@supabase/ssr` docs (4 factories). Don't import `@supabase/supabase-js` directly except in `admin.ts`.

### Step 6 — Write env files

```bash
# apps/web/.env.local — real values from `npx supabase status -o env`
NEXT_PUBLIC_SUPABASE_URL=<from CLI>
NEXT_PUBLIC_SUPABASE_ANON_KEY=<from CLI>
SUPABASE_SERVICE_ROLE_KEY=<from CLI, renamed>
DATABASE_URL=<from CLI>
NEXT_PUBLIC_APP_URL=http://localhost:3000

# apps/web/.env.example — placeholder values, committed
NEXT_PUBLIC_SUPABASE_URL=<your-supabase-url>
NEXT_PUBLIC_SUPABASE_ANON_KEY=<your-supabase-anon-key>
SUPABASE_SERVICE_ROLE_KEY=<your-supabase-service-role-key>
DATABASE_URL=postgresql://postgres:postgres@localhost:54322/postgres
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

Ensure `apps/web/.gitignore` covers `.env.local`.

### Step 7 — Verify local boot

```bash
pnpm install         # clean state
pnpm dev &           # background
sleep 5
curl -fsS http://localhost:3000 > /dev/null && echo "✅ booted" || echo "❌ failed"
# Stop the dev server (note PID; never pkill -f)
```

If boot fails: read the dev-server log, fix the root cause. Common issues: missing env key, port 3000 taken, schema not migrated.

### Step 8 — GitHub repo (remote, optional)

```bash
gh repo create <project-name> --private --source=. --push
```

Skip if `gh auth status` not authenticated. In autonomous mode, skip silently and log.

### Step 9 — Supabase cloud project (remote, optional)

```bash
npx supabase login
npx supabase projects create <project-name> --org-id <org-id> --region <region> --db-password <generate>
npx supabase link --project-ref <ref>
npx supabase db push                                 # apply local migrations to remote
```

Capture remote env values from the dashboard or `npx supabase projects api-keys`.

Skip if not logged in. In autonomous mode, log and continue.

### Step 10 — Vercel project (remote, optional)

```bash
npx vercel login
cd apps/web
npx vercel link
# Set env vars on Vercel
npx vercel env add NEXT_PUBLIC_SUPABASE_URL production preview
npx vercel env add NEXT_PUBLIC_SUPABASE_ANON_KEY production preview
npx vercel env add SUPABASE_SERVICE_ROLE_KEY production
npx vercel env add NEXT_PUBLIC_APP_URL production
cd ../..
```

Skip if not logged in. Don't deploy yet — that's `deploy=prod` opt-in.

### Step 11 — Seed runbook

Write `docs/runbook.md`:

```markdown
# Runbook — <project-name>

## Local development
- Prerequisites: Docker Desktop running, Node 20+, pnpm 10+
- First boot: `pnpm install` then `npx supabase start` then `pnpm dev`
- Reset DB: `pnpm db:reset`
- Regenerate types after schema change: `pnpm db:types`

## Env vars (where each comes from)
[table per env key — local source, remote source, rotation procedure]

## Deploy
- Preview: push to a branch; Vercel auto-deploys
- Prod: `npx vercel --prod` from `apps/web/`, OR merge to `main` (auto-deploy if linked)

## Backup / DR
- Supabase: enable PITR (Point-in-Time Recovery) in dashboard for prod
- Migrations are versioned in `supabase/migrations/` — recovery = re-apply
- Restore procedure: [TBD — flesh out when prod data exists]

## Cron jobs
- [List of Vercel Cron entries from vercel.ts, when added]

## Monitoring / alerts
- Sentry: <DSN> (set after Sentry signup)
- Vercel Analytics: enabled by default

## Incident contacts
- [Human contact info — placeholder, fill manually]
```

### Step 12 — Final verify

```bash
pnpm install
pnpm typecheck
pnpm build
pnpm dev &
sleep 5
curl -fsS http://localhost:3000 > /dev/null && echo "✅ all green" || echo "❌ failed"
```

All four must pass. If any fail, fix root cause before reporting success.

## Failure modes + fixes

- **Docker not running** → ask user to start Docker Desktop. Cannot self-resolve.
- **Port 3000 taken** → use `PORT=3004 pnpm dev`. Update `.env.local` `NEXT_PUBLIC_APP_URL` accordingly.
- **`supabase start` hangs on first run** → wait up to 5 min (image pulls). After that, check Docker logs.
- **`pnpm db:types` produces empty file** → schema isn't applied; run `npx supabase db reset` first.
- **GitHub repo creation fails (already exists)** → check existing repo with `gh repo view`; ask user whether to use it.
- **Vercel `link` fails (project not found)** → run `npx vercel link` interactively (autonomous mode: skip and log).
