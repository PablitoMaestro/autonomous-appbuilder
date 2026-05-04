---
name: 04-bootstrap-env
description: Provision the dev environment for a project — Supabase, Vercel, GitHub repo, env files, runbook — based on the picked tech stack. Picks the right per-stack recipe, runs CLIs, populates `apps/*/.env.local`, writes `.env.example`, seeds `docs/runbook.md`. Asks the human only for credentials it cannot self-provision; in autonomous mode, proceeds with localhost-only when secrets aren't supplied. Use whenever the user says "bootstrap the env", "set up Supabase and Vercel", "provision the dev stack", "wire up the credentials", "make `pnpm dev` work", or has a master plan ready and needs the local stack working before code execution can start.
---

# 04 — Bootstrap Env

Take a locked `.ai/techstack.md` + `.ai/environments.md` and bring the dev environment to a state where `pnpm dev` (or the stack's equivalent) boots successfully. Recipes are stack-specific and live in `references/`; this skill picks the right one and runs it.

> **Invocation contract:** This skill is invoked via `Skill('04-bootstrap-env')` — typically by `00-build-app` at Phase 4, but also standalone. The caller MUST NOT run `pnpm install` / write `.env.local` / scaffold `globals.css` themselves. Design-token implementation belongs to Phase 5 master-plan tasks (e.g., 1.1), not to this phase. Phase 4's contract: the dev server boots green, nothing more.

The skill's value isn't running CLIs — it's the discipline of provisioning the right services with the right config, populating envs without ever inventing secret values, and writing a runbook so future humans (or agents) can reproduce the setup.

## Inputs

The orchestrator (or standalone caller) must provide:
- `.ai/techstack.md` — names the stack
- `.ai/environments.md` — names the env tiers (local / staging / prod) and which services live in each
- `mode`: `supervised` (asks for missing creds) | `autonomous` (asks once, proceeds with localhost-only if no reply)

## Workflow at a glance

1. **Read inputs** — techstack + environments.
2. **Pick recipe** — match the stack to one of `references/<stack>.md`.
3. **Preflight** — check CLIs are installed (`gh`, `npx supabase`, `npx vercel`, etc.). Install missing ones via the project's package manager (most are workspace devDependencies in modern repos).
4. **Provision local** — always succeeds without external creds. `npx supabase start`, generate types, verify `pnpm dev` boots.
5. **Provision remote (optional)** — GitHub repo, Supabase cloud project, Vercel project. Asks for any creds the agent cannot self-provision (Vercel team token, domain ownership). In autonomous mode, this ask is one-shot with a 5-minute soft timeout.
6. **Write env files + runbook** — `apps/*/.env.local`, `apps/*/.env.example`, `docs/runbook.md`.
7. **Verify** — boot the dev server, confirm green, optionally hit `/healthz` or equivalent.
8. **Report** — what's wired, what's still missing, where to find the runbook.

## Phase 1 — Recipe selection

Match the stack to a recipe; first match wins:

| Stack signal | Recipe |
|---|---|
| Next.js + Supabase + Vercel (default greenfield) | `references/nextjs-supabase-vercel.md` |
| Vite-based SPA / browser game (Phaser, etc.) | `references/vite-spa.md` |
| Astro static site | `references/astro-static.md` |
| CLI / library (Bun/Node/Deno) | `references/cli-bun.md` |
| Existing repo with detected stack | `references/existing-repo.md` |
| Stack not yet covered | Stop. Ask the user to confirm the closest recipe + adapt, or to add a new recipe under `references/`. |

Each recipe is a step-by-step provisioning script (CLIs to run, in order, with the env keys each step writes). Read the recipe in full before running any step.

## Phase 2 — Stack-specific preflight

The orchestrator's Step 0.5 already ran universal preflight (node, pnpm, git, chrome-devtools-mcp). Don't duplicate. Here, check ONLY what's stack-specific (now that we know the stack from `.ai/techstack.md`):

- Stack package manager (`bun` for CLI/lib stacks, `pnpm` already verified for default stack).
- Stack-specific CLIs (`npx supabase` for Supabase stacks, `npx vercel` for Vercel deploys, etc.). Modern stacks pin them as workspace devDependencies — `pnpm install` provisions them. If missing, install the missing one only.
- **Docker Desktop is running** — only if the stack needs it (Supabase local does; static-site stacks don't).
- CLI auth (`gh auth status`, `npx supabase login` status, `npx vercel whoami`) for stacks that need remote provisioning.
- The current working directory is the repo root (not a worktree, not a subdir).

If any preflight fails, stop and report the missing piece concretely — don't try to install Docker or fix system PATH.

## Phase 3 — Provision local (always)

Local provisioning must succeed without ANY external credentials. This is the floor — even autonomous mode without creds reaches this state.

Per the recipe:
- Spin up local services (`npx supabase start`, `docker compose up -d`, etc.).
- Generate types from local schema (`pnpm db:types` or equivalent).
- Run any seed scripts the recipe specifies.
- Verify the dev server boots (`pnpm dev` returns green within 60s).

If local provisioning fails, fix at the root cause (missing Docker, wrong port, conflicting service). Don't paper over by skipping the failing step.

## Phase 4 — Provision remote (optional)

Remote provisioning needs creds. The recipe lists which creds + how to source them:

- **GitHub repo**: requires `gh auth status` to be authenticated. Repo created with `gh repo create`.
- **Supabase cloud project**: requires `npx supabase login`. Project created with `supabase projects create`.
- **Vercel project**: requires `npx vercel login`. Project linked with `vercel link`.

For each cred the agent cannot self-source:
- **Supervised mode**: ask the user with a clear ask — what cred, why, where to get it. Wait for reply.
- **Autonomous mode**: send the ask once, proceed in 5 minutes. Skip remote provisioning for missing creds — the local stack still works, so the master plan can still execute. Append a "skipped remote provisioning: <reasons>" entry to the autonomous decision log.

**Never invent a value for a secret.** If the agent doesn't have a real cred, leave the env key as `<MISSING>` in `.env.example` and explain in the runbook that this key is human-supplied.

## Phase 5 — Env files + runbook

Write three things:

1. **`apps/<app>/.env.local`** — populated with real values (local services + any remote creds successfully sourced). Never commit. Ensure `.gitignore` covers it.
2. **`apps/<app>/.env.example`** — schema with placeholder values (`<your-supabase-anon-key>`). Committed. Lists every key.
3. **`docs/runbook.md`** — operational reference for humans + agents:
   - How to start the local stack (`pnpm install`, `npx supabase start`, `pnpm dev`).
   - Where each env var comes from (CLI command, dashboard URL, manual entry).
   - Deploy procedure (per env tier).
   - Backup / DR steps (skeleton — flesh out as services accumulate).
   - Sentry / monitoring URLs (if wired).
   - Cron jobs (if any).
   - Incident contacts (placeholder — human fills).

The runbook must let a fresh agent (or a new human) re-provision from scratch using only the doc + the cred sources it lists.

## Phase 6 — Verify

After provisioning, run the verification commands from `.ai/techstack.md`:
- `pnpm install` (clean state)
- `pnpm dev` — wait 60s for green; `curl localhost:<port>/healthz` if the stack supports it.
- `pnpm typecheck` — must pass (proves generated types are valid).
- `pnpm build` — must pass (proves nothing's missing for production build).

If any verify step fails, fix at the root cause. Don't flag-skip a failure into the runbook.

## Phase 6.5 — Flip Phase 0 task + populate CLAUDE.md

Once Phase 6 verifies green (`pnpm dev` boots, build passes):

1. Flip master-plan task `0.5 Env bootstrapped + runbook seeded` to `[x]`. Use a single Edit on `.ai/master-plan.md`.
2. **Populate CLAUDE.md "Local Development" section** (currently a placeholder from `01-discover-idea`'s skeleton). Use Edit (not Write) — the rest of CLAUDE.md is untouched. Add real commands:

```markdown
## Local Development

**Prerequisites:** <node version, pnpm/bun, docker if applicable>

```bash
# First-time setup
<install command>

# Running the stack
<services start command>     # e.g., npx supabase start
<dev server command>         # e.g., pnpm dev → http://localhost:3000

# Database (if applicable)
<db reset command>
<types regen command>

# Quality / CI
<typecheck>
<lint>
<build>
<test>
```

**Test users:** see `.ai/environments.md`.
**Runbook:** see `docs/runbook.md`.
```

Match the actual commands defined in the project's `package.json` scripts (or stack equivalent). Don't invent commands that don't exist.

## Phase 7 — Report

```
**Env bootstrap complete (or partial).**

Local services:
- ✅ Supabase local (postgres + auth + storage on port 54321)
- ✅ Next dev server (port 3000)

Remote services:
- ✅ GitHub repo: <url>
- ✅ Supabase cloud project: <project-id> (region: <region>)
- ⚠️ Vercel project: skipped — VERCEL_TOKEN not provided
- ⚠️ Domain: skipped — requires manual DNS

Env files:
- apps/web/.env.local: 14 keys populated, 2 marked <MISSING>
- apps/web/.env.example: 16 keys
- docs/runbook.md: written

Next:
- pnpm dev → localhost:3000 (working now)
- For Vercel: `npx vercel login` then re-run `/04-bootstrap-env phase=remote`
- For domain: see docs/runbook.md § Domain setup
```

In autonomous mode, append the same report to the decision log instead of presenting it interactively.

## Universal quality rails

1. **Never invent secrets.** Real creds come from CLIs or the human. `<MISSING>` is a valid placeholder; a hallucinated key is not.
2. **Local-first.** Local provisioning must succeed without ANY external creds. If the agent can't do that, the recipe is broken.
3. **Runbook is the contract.** A future agent reading only the runbook should be able to re-provision the env from scratch. If it can't, the runbook is incomplete.
4. **Recipe-driven.** Don't reinvent provisioning logic in the SKILL body — read the relevant `references/<stack>.md` and follow it.
5. **Failure means stop and report.** Don't paper over a failed CLI with a skip-and-pray; bubble the error up so the human (or orchestrator) can decide.

## When the orchestrator invokes this skill

The orchestrator passes:
- `mode`: `supervised` | `autonomous`
- `decision_log_path`: where to append autonomous decisions
- `gates=all`: if set, this skill ALSO requires explicit human approval before running remote provisioning (default: only ask for missing creds)

Standalone invocation: assume `mode=supervised`, no decision log, no `gates=all`.

## Reference files

- `references/nextjs-supabase-vercel.md` — default greenfield recipe (read first)
- `references/vite-spa.md` — SPA / browser-game shell
- `references/astro-static.md` — content sites
- `references/cli-bun.md` — CLI / library projects
- `references/existing-repo.md` — detect-and-adapt path

When adding a new stack, drop a new `references/<stack>.md` following the same shape and add a row to the recipe-selection table above.

## Why this skill is shaped this way

- **Stacks are the right unit of variation.** Provisioning steps are tightly coupled to the stack; one big SKILL body would conflate them. Per-stack recipes keep each path readable and replaceable.
- **Local-first because it's the only floor.** Remote provisioning depends on creds the agent may not have; local provisioning depends on Docker + a recipe. By guaranteeing local always works, the orchestrator can always reach Phase 5 (execute) even when remote is incomplete.
- **No invented secrets.** A skill that hallucinates env values is worse than one that fails — fake creds get committed and break in subtle ways. `<MISSING>` is a contract.
- **Runbook over docs-rot.** Keeping operational steps in one runbook (vs scattered across READMEs) makes re-provisioning trivial and prevents drift.
