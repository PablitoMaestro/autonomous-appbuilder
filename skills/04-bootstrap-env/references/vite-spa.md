# Recipe — Vite SPA

The default stack for browser games (Phaser), interactive demos, and static-hosted SPAs that don't need SSR. Provisions local + (optional) remote static hosting, populates env files, seeds the runbook.

## What this recipe provisions

- **Local:** Vite dev server (HMR), the chosen framework template (React / Vue / vanilla TS), generated `.env` typing via `vite/client`.
- **Remote (optional):** GitHub repo, Vercel / Netlify / Cloudflare Pages project linked to the repo (static deploy, framework preset = `vite`).
- **Env files:** `apps/web/.env.local` (real values, `VITE_*` prefix for anything client-exposed), `apps/web/.env.example` (schema with placeholders).
- **Runbook:** `docs/runbook.md` with deploy + asset paths + env handling sections.

## Required env keys

| Key | Local source | Remote source |
|---|---|---|
| `VITE_APP_NAME` | hardcode `<project-name>` | same |
| `VITE_APP_URL` | `http://localhost:5173` | hosting deployment URL |
| `VITE_API_BASE_URL` (optional) | `<MISSING>` until backend exists | upstream API URL |
| `VITE_SENTRY_DSN` (optional) | `<MISSING>` | Sentry → Project settings → DSN |
| `VITE_ANALYTICS_ID` (optional) | `<MISSING>` | analytics dashboard |

**Important: only `VITE_*`-prefixed keys are exposed to client code.** Anything without that prefix is invisible at runtime — Vite strips it. Never put real secrets in a `VITE_*` key; SPAs ship the bundle to every browser.

## Provisioning steps

### Step 1 — Preflight
```bash
node --version          # ≥ 20
pnpm --version          # ≥ 10
gh auth status          # for remote provisioning (optional)
npx vercel --version    # only if deploying to Vercel
```

If any required CLI is missing: install only the missing one. No Docker needed — Vite is fully native.

### Step 2 — Repo skeleton (if empty repo)

```bash
mkdir -p apps/web .ai docs

cat > package.json <<'EOF'
{
  "name": "<project-name>",
  "private": true,
  "scripts": {
    "dev": "pnpm --filter web dev",
    "build": "pnpm --filter web build",
    "preview": "pnpm --filter web preview",
    "typecheck": "pnpm -r typecheck",
    "lint": "pnpm --filter web lint"
  },
  "devDependencies": {
    "vercel": "^33.x"
  }
}
EOF

cat > pnpm-workspace.yaml <<'EOF'
packages:
  - "apps/*"
EOF
```

### Step 3 — Scaffold Vite app

```bash
cd apps/web
pnpm create vite@latest . --template react-ts
# Other valid templates: react / vue-ts / vue / svelte-ts / vanilla-ts / vanilla
# For a Phaser browser game: pick vanilla-ts and `pnpm add phaser` after.
cd ../..
```

For Phaser specifically, after scaffold:
```bash
cd apps/web
pnpm add phaser
mkdir -p public/assets   # static assets served from /assets/* at runtime
cd ../..
```

### Step 4 — Install + initial verify

```bash
pnpm install
pnpm dev &              # background
sleep 5
curl -fsS http://localhost:5173 > /dev/null && echo "✅ booted" || echo "❌ failed"
# Stop the dev server (note PID; never pkill -f)
```

Vite's default port is 5173. If taken, Vite auto-bumps to 5174 — pin it explicitly with `--port 5173` in `apps/web/package.json` `dev` script if reproducibility matters.

### Step 5 — Write env files

```bash
# apps/web/.env.local — real values (committed to .gitignore, never to git)
VITE_APP_NAME=<project-name>
VITE_APP_URL=http://localhost:5173
# VITE_API_BASE_URL=<MISSING>
# VITE_SENTRY_DSN=<MISSING>

# apps/web/.env.example — placeholder values, committed
VITE_APP_NAME=<your-app-name>
VITE_APP_URL=http://localhost:5173
VITE_API_BASE_URL=<your-api-base-url>
VITE_SENTRY_DSN=<your-sentry-dsn>
VITE_ANALYTICS_ID=<your-analytics-id>
```

Ensure `apps/web/.gitignore` covers `.env.local` and `.env.*.local` (Vite's default ignore set already does — verify).

Type-check the env shape by adding `apps/web/src/env.d.ts`:
```ts
/// <reference types="vite/client" />
interface ImportMetaEnv {
  readonly VITE_APP_NAME: string
  readonly VITE_APP_URL: string
  readonly VITE_API_BASE_URL?: string
  readonly VITE_SENTRY_DSN?: string
  readonly VITE_ANALYTICS_ID?: string
}
interface ImportMeta { readonly env: ImportMetaEnv }
```

### Step 6 — GitHub repo (remote, optional)

```bash
gh repo create <project-name> --private --source=. --push
```

Skip if `gh auth status` not authenticated. In autonomous mode, skip silently and log.

### Step 7 — Vercel project (remote, optional)

Vite is a first-class Vercel framework preset — no `vercel.json` required for vanilla deploys.

```bash
npx vercel login
cd apps/web
npx vercel link                                       # framework auto-detected as Vite
npx vercel env add VITE_APP_NAME production preview
npx vercel env add VITE_APP_URL production preview
# Add other VITE_* keys as needed
cd ../..
```

Alternative static hosts (use whichever the user prefers):
- **Netlify:** `npx netlify-cli init` → build command `pnpm build`, publish dir `apps/web/dist`.
- **Cloudflare Pages:** `npx wrangler pages project create <project-name>` → connect via dashboard, build dir `apps/web/dist`.

Skip if not logged in. Don't deploy yet — that's `deploy=prod` opt-in.

### Step 8 — Seed runbook

Write `docs/runbook.md`:

```markdown
# Runbook — <project-name>

## Local development
- Prerequisites: Node 20+, pnpm 10+
- First boot: `pnpm install` then `pnpm dev`
- Dev server: http://localhost:5173 (HMR enabled)
- Production preview: `pnpm build` then `pnpm preview` (serves `dist/` on :4173)

## Env vars (where each comes from)
- Only `VITE_*`-prefixed keys reach the browser bundle. Everything else is dropped at build time.
- Never store real secrets in `VITE_*` — every visitor downloads them.
- Local: `apps/web/.env.local` (gitignored).
- Remote: set per-environment via `npx vercel env add` (or hosting dashboard).

## Static assets
- Files in `apps/web/public/` are served at `/` (e.g. `public/assets/sprite.png` → `https://app/assets/sprite.png`).
- Inside code, reference public assets with absolute paths: `<img src="/assets/sprite.png" />`.
- For Phaser games: `this.load.image('hero', '/assets/hero.png')` — leading slash matters.
- Imported assets (`import logo from './logo.svg'`) are hashed + bundled by Vite; use these for module-graph assets.

## Deploy
- Preview: push to a branch; Vercel auto-deploys
- Prod: `npx vercel --prod` from `apps/web/`, OR merge to `main` (auto-deploy if linked)
- Build output: `apps/web/dist/` — pure static files, no server runtime

## Monitoring / alerts
- Sentry: <DSN> (set after Sentry signup)
- Vercel Analytics: enabled by default

## Incident contacts
- [Human contact info — placeholder, fill manually]
```

### Step 9 — Final verify

```bash
pnpm install
pnpm typecheck
pnpm build
pnpm dev &
sleep 5
curl -fsS http://localhost:5173 > /dev/null && echo "✅ all green" || echo "❌ failed"
```

All four must pass. If any fail, fix root cause before reporting success.

## Failure modes + fixes

- **Port 5173 taken** → Vite auto-bumps to 5174 (visible in dev log) but breaks any hardcoded `VITE_APP_URL`. Pin explicitly: `pnpm dev --port 5173 --strictPort` (fail-fast) or change `VITE_APP_URL` to match.
- **Phaser canvas blank / 404 on assets** → check leading slash. `load.image('hero', 'assets/hero.png')` is wrong; must be `/assets/hero.png` or use `import.meta.env.BASE_URL` + relative path. Confirm files live under `apps/web/public/assets/`, not `apps/web/src/assets/`.
- **`import.meta.env.VITE_FOO` is `undefined` at runtime** → key missing the `VITE_` prefix, or `.env.local` not loaded (must be at `apps/web/.env.local`, not repo root). Restart dev server after editing `.env*`.
- **`pnpm build` succeeds but `pnpm preview` shows 404 on routes** → SPA needs hosting fallback. On Vercel/Netlify the Vite preset handles this; for custom hosts, configure rewrite all → `/index.html`.
- **Build output too large / slow** → enable `build.rollupOptions.output.manualChunks` to split vendor bundles. For Phaser, lazy-load scenes via dynamic `import()` so the initial bundle stays small.
- **GitHub repo creation fails (already exists)** → check existing repo with `gh repo view`; ask user whether to use it.
- **Vercel `link` fails (project not found)** → run `npx vercel link` interactively (autonomous mode: skip and log).
