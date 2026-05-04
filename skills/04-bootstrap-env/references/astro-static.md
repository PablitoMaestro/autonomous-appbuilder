# Recipe — Astro Static

Static-first content sites — blogs, docs, marketing pages, portfolios. Astro ships zero JS by default and adds islands of interactivity (React/Vue/Svelte/Solid) only where needed. MDX is the usual companion for long-form content. Deploy target is any static host (Vercel / Netlify / Cloudflare Pages). Local-first; remote hosting is optional.

## What this recipe provisions

- **Local:** Astro dev server (Vite under the hood) on port 4321, optional MDX integration, optional UI framework integration (React/Vue/Svelte), generated content-collection types.
- **Remote (optional):** GitHub repo, Vercel project (preset `astro`) or Netlify/Cloudflare Pages site linked to the repo. Optional CMS integration via Astro's marketplace (`astro add`) — Sanity, Contentful, Storyblok, Hygraph, Decap, Keystatic, etc.
- **Env files:** `.env.local` (real values), `.env.example` (schema with placeholders). Astro reads `.env*` at the project root.
- **Runbook:** `docs/runbook.md` with deploy + content-collections + secrets + monitoring sections.

## Required env keys

Most Astro static sites need **zero env keys** — that's the whole point. Add keys only for the integrations the project actually wires.

| Key | Local source | Remote source | When required |
|---|---|---|---|
| `PUBLIC_SITE_URL` | `http://localhost:4321` | Deployed URL (Vercel / Netlify) | Always (used for `<link rel="canonical">`, sitemap, OG tags) |
| `PUBLIC_<CMS>_PROJECT_ID` | CMS dashboard | CMS dashboard | If a CMS is wired |
| `<CMS>_API_TOKEN` | CMS dashboard (read token) | CMS dashboard (read token) | If a CMS is wired and needs auth |
| `ASTRO_DB_REMOTE_URL` | `<MISSING>` (local uses libSQL file) | Astro Studio / Turso dashboard | Only if `@astrojs/db` is installed |
| `ASTRO_DB_APP_TOKEN` | `<MISSING>` (local uses libSQL file) | Astro Studio / Turso dashboard | Only if `@astrojs/db` is installed |
| `SENTRY_DSN` (optional) | `<MISSING>` | Sentry → Project settings → DSN | Only if Sentry is wired |

**Important convention:** Astro exposes `PUBLIC_*` (and `import.meta.env.PUBLIC_*`) to client-side bundles. Anything **without** the `PUBLIC_` prefix is server-only (runs at build time or in SSR endpoints) and must never be referenced from a client island. Mirror Vite's contract — don't invent your own.

## Provisioning steps

### Step 1 — Preflight

```bash
node --version          # ≥ 20 (Astro 5 requires 18.20.8 / 20.3 / 22+)
pnpm --version          # ≥ 10
gh auth status          # for remote provisioning (optional)
```

If `node` or `pnpm` are missing, install them via the system package manager (`brew`, `nvm`, `corepack`). Don't try to script Node installation.

### Step 2 — Repo skeleton (if empty repo)

Astro projects are typically single-package, not monorepos. Default to a flat layout unless the user explicitly wants a monorepo.

```bash
mkdir -p docs .ai
git init
echo "node_modules\ndist\n.astro\n.env\n.env.local\n.env.*.local\n" > .gitignore
```

### Step 3 — Scaffold Astro

```bash
pnpm create astro@latest .
# Prompts:
#   - tmpl: "Empty project" or "Blog" (pick "Empty" for a clean slate; "Blog" for content-collections wired up)
#   - install deps: yes
#   - typescript: strict
#   - git: no (already done)
```

This writes `astro.config.mjs`, `tsconfig.json`, `package.json`, and a `src/` tree (`src/pages/`, `src/layouts/`, `src/components/`, `src/content/` if blog template).

If the user wants a UI framework for islands, add it now (one is enough — don't preinstall all four):

```bash
pnpm astro add react        # or: vue / svelte / solid / preact
```

### Step 4 — Add MDX integration

```bash
pnpm astro add mdx
```

The `astro add` command edits `astro.config.mjs`, installs `@astrojs/mdx`, and registers the integration in one step. Always prefer it over hand-editing the config.

If the project needs sitemap + RSS too:

```bash
pnpm astro add sitemap
pnpm add @astrojs/rss
```

### Step 5 — Install + verify

```bash
pnpm install
pnpm dev &           # background
sleep 5
curl -fsS http://localhost:4321 > /dev/null && echo "OK booted" || echo "FAIL boot"
# Stop the dev server (note PID; never pkill -f)
```

If boot fails: read the dev-server log, fix the root cause. Common: port 4321 taken (`pnpm dev --port 4322`), missing integration peer dep, MDX frontmatter syntax error in a `.md` file under `src/content/`.

### Step 6 — Write env files

Astro reads `.env`, `.env.local`, `.env.<MODE>`, `.env.<MODE>.local` at the **project root** (not in `apps/web/`). Only `PUBLIC_*` keys reach client bundles; the rest are build-time / server-only.

```bash
# .env.local — real values (gitignored)
PUBLIC_SITE_URL=http://localhost:4321
# Add CMS keys only if wired, e.g.:
# PUBLIC_SANITY_PROJECT_ID=<from Sanity dashboard>
# SANITY_API_TOKEN=<from Sanity dashboard, read-only>

# .env.example — placeholder schema (committed)
PUBLIC_SITE_URL=http://localhost:4321
# PUBLIC_SANITY_PROJECT_ID=<your-sanity-project-id>
# SANITY_API_TOKEN=<your-sanity-read-token>
```

Confirm `.gitignore` covers `.env`, `.env.local`, and `.env.*.local`. The default Astro template gets this right; double-check after editing.

### Step 7 — GitHub repo (remote, optional)

```bash
gh repo create <project-name> --private --source=. --push
```

Skip if `gh auth status` not authenticated. In autonomous mode, log and continue.

### Step 8 — Vercel project (remote, optional)

```bash
npx vercel login
npx vercel link
# Vercel auto-detects Astro (preset = "astro"); no buildCommand override needed.
# Push env vars used at build time:
npx vercel env add PUBLIC_SITE_URL production preview
# Add CMS keys here too if wired.
```

For Netlify or Cloudflare Pages instead:
- **Netlify:** `npx netlify-cli init` → preset auto-detects Astro (publish dir `dist`, build `astro build`).
- **Cloudflare Pages:** `npx wrangler pages project create <name>` → build command `astro build`, output dir `dist`.

Skip if not logged in. Don't deploy yet — that's an opt-in step.

### Step 9 — Seed runbook

Write `docs/runbook.md`:

```markdown
# Runbook — <project-name>

## Local development
- Prerequisites: Node 20+, pnpm 10+
- First boot: `pnpm install` then `pnpm dev` → http://localhost:4321
- Build: `pnpm build` → static output in `dist/`
- Preview prod build: `pnpm preview`

## Content collections
- Markdown / MDX content lives under `src/content/<collection>/`
- Schema defined in `src/content/config.ts` via `defineCollection({ schema: z.object({…}) })`
- Astro generates types into `.astro/` on every `pnpm dev` / `pnpm build`; commit `src/content/config.ts` but not `.astro/`
- After editing the schema, restart `pnpm dev` to refresh generated types

## Env vars (where each comes from)
- `PUBLIC_SITE_URL`: localhost in dev; deployed URL in prod (set in Vercel/Netlify/CF dashboard)
- `PUBLIC_*`: exposed to client islands — never put secrets here
- Non-PUBLIC keys: server-only (build-time + SSR endpoints); safe for read tokens, never write tokens

## Deploy
- Preview: push to a branch; Vercel/Netlify/CF auto-deploys
- Prod: merge to `main` (auto-deploy), or `npx vercel --prod`
- Astro static output is fully cacheable; CDN config defaults are fine

## Backup / DR
- Source of truth is the Git repo + the CMS (if wired)
- Static output is regenerable from `pnpm build`; no data to back up unless `@astrojs/db` is in use
- If `@astrojs/db` is in use: snapshot the libSQL/Turso DB on the Studio dashboard schedule

## Monitoring / alerts
- Sentry: <DSN> (set after Sentry signup) — wire via `@sentry/astro` integration
- Lighthouse CI in PRs (optional): `npx @lhci/cli@latest autorun` against `pnpm preview`

## Incident contacts
- [Human contact info — placeholder, fill manually]
```

### Step 10 — Final verify

```bash
pnpm install
pnpm astro check     # type-checks .astro / .ts / content collections
pnpm build           # full static build → dist/
pnpm preview &       # serves dist/ on http://localhost:4321
sleep 5
curl -fsS http://localhost:4321 > /dev/null && echo "OK all green" || echo "FAIL preview"
```

All four must pass. `astro check` is the Astro-native equivalent of `tsc --noEmit` — it also validates content-collection schemas. If any step fails, fix root cause before reporting success.

## Failure modes + fixes

- **`pnpm create astro` aborts on dirty directory** → it refuses to scaffold into a non-empty dir unless `.` is empty (or only contains `.git`). Move existing files aside or scaffold into a fresh subdir, then merge.
- **Port 4321 taken** → `pnpm dev --port 4322` (or set `server.port` in `astro.config.mjs`). Update `PUBLIC_SITE_URL` for local if needed.
- **`PUBLIC_*` var read as `undefined` on the client** → make sure the prefix is exactly `PUBLIC_`, the file is `.env.local` (not `.env.development.local`) at the project root, and the dev server was restarted after the edit.
- **Server-only env var leaked to a client island** → Astro will warn at build time. Move the read into the `.astro` frontmatter (server) and pass the value as a prop to the island, or guard with `import.meta.env.SSR`.
- **Content collection types are stale after schema edit** → kill `pnpm dev`, run `pnpm astro sync` (regenerates `.astro/types.d.ts`), then restart.
- **MDX imports fail with "Cannot find module"** → confirm `@astrojs/mdx` is in `astro.config.mjs` integrations (the `astro add mdx` step) and that the file uses `.mdx`, not `.md`.
- **Vercel build fails with "framework not detected"** → set the preset manually in the Vercel project settings (Framework Preset = "Astro"); auto-detect requires `astro` in `package.json` dependencies.
- **`@astrojs/db` keys missing in prod** → expected on first deploy. Ask the human for `ASTRO_DB_REMOTE_URL` + `ASTRO_DB_APP_TOKEN` from Astro Studio / Turso; leave `<MISSING>` until then. Local dev keeps working against the libSQL file.
