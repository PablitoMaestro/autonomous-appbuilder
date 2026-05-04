---
name: 07-smoke-test-app
description: Boot the app locally and verify it actually runs end-to-end via a browser tool. Four concrete checks — dev server boots in 60s, homepage renders without hydration errors, auth round-trips (if applicable), one core CRUD flow round-trips. Derives the golden path from the master plan's #1 user story. Use whenever the user says "smoke test the app", "verify it boots", "walk the golden path", "is the app actually working?", "does pnpm dev still work?", or wants a fast sanity check after execution / review — even if they don't name the checks.
---

# 07 — Smoke Test App

Fast, narrow verification that an app actually runs after Phase 5 execution (and optionally Phase 6 review). Not a test suite, not an audit — a confidence check that the obvious things work end-to-end before claiming "production-ready, running locally."

> **Invocation contract:** This skill is invoked via `Skill('07-smoke-test-app')` — typically by `00-build-app` at Phase 7, but also standalone. The caller MUST NOT run inline `curl` probes or chrome-devtools-mcp checks themselves and call that "smoke" — the formal smoke gates, the report file, and the repair-or-not decision all live in this skill.

The skill's value is in the **narrowness**: 4 concrete checks, fail fast, report concretely. A green smoke test is meaningful precisely because it doesn't try to be exhaustive.

## Inputs

The orchestrator (or standalone caller) provides nothing required — the skill reads the repo. Optional addendum:
- `golden_path`: explicit description of the flow to walk (overrides auto-detection from master plan)
- `port`: dev server port (default: from `package.json` scripts or 3000)
- `repair`: `true` → on failure, dispatch a repair subagent to create a `smoke-fixes` side-plan instead of just reporting

## Workflow

1. **Detect stack + boot commands** from `package.json` scripts and `.ai/techstack.md`.
2. **Boot the local stack** in dependency order (Supabase → migrations → seed → dev server).
3. **Run the 4 checks** in order; fail fast.
4. **Capture evidence** — screenshots, console logs, network errors.
5. **Write smoke log** — `.ai/agent-logs/smoke-<date>.md` with pass/fail per check.
6. **(Optional) Dispatch repair** if `repair=true` and any check failed.

## Phase 1 — Detect

Read in parallel:

- `package.json` scripts — find `dev`, `db:reset`, `db:seed-storage`, `typecheck`, `build`.
- `.ai/techstack.md` — confirm stack and infra dependencies.
- `.ai/master-plan.md` — find the first user story / first MVP feature; that's the golden path candidate.
- `.ai/site-map.md` — confirm routes the golden path needs.
- Existing test users in `.ai/environments.md` (e.g., `admin@example.test` / password convention — exact format defined per project).

If no master-plan / no site-map: stop and report — smoke test depends on knowing what "working" means.

## Phase 2 — Boot the local stack

Per the stack (commands vary; use what `package.json` actually defines):

```bash
# Stack with database (default greenfield):
pnpm install
npx supabase start              # ~30s for first boot
pnpm db:reset                   # if a reset script exists; else skip
pnpm db:seed-storage            # if it exists
PORT=<port> pnpm dev &          # background; capture PID
```

Wait up to 60s for the dev server to respond on `http://localhost:<port>`. If it doesn't:
- Check the dev-server log for the error.
- **Don't restart blindly.** If port is taken, find a free port; if migration failed, that's a check-1 failure (report).
- Kill the dev server PID you started — never `pkill -f "next dev"` (would kill parallel agents per CLAUDE.md).

## Phase 3 — Run the 4 checks

Use the lightest browser tool that works:
- `chrome-devtools-mcp` — preferred (a11y + console + network + perf trace)
- `claude-in-chrome` — fallback for auth-bearing flows (preserves real Chrome cookies)
- `agent-browser` — fallback for cross-viewport sweeps

### Check 1 — Boot
- ✅ Pass: dev server returns `200` on `/` within 60s, no fatal log errors.
- ❌ Fail: timeout, 500, or fatal log line.

### Check 2 — Homepage renders without hydration errors
- ✅ Pass: page renders with no React hydration warnings, no 404s on JS/CSS chunks, no console `error` level messages.
- ❌ Fail: any `Hydration failed` / `Text content does not match` / blocked chunk.

### Check 3 — Auth round-trips (if auth in master plan)
Skip if no auth phase in `.ai/master-plan.md`.

- Open `/login` (or whatever the site map calls it).
- Use a test user from `.ai/environments.md`.
- ✅ Pass: login redirects to authenticated landing (e.g., `/dashboard`); session cookie present; refresh keeps user logged in.
- ❌ Fail: 401, redirect loop, or session lost on refresh.

### Check 4 — One core CRUD flow
Pick the first user story from `.ai/functional-spec.md` that's `[x]`'d in the master plan. Walk it.

- ✅ Pass: the user can do the thing the story claims (create → read; or list → edit; or post → confirm).
- ❌ Fail: any step blocks (button broken, mutation 500s, list doesn't reflect creation).

## Phase 4 — Capture evidence

For each check:
- Screenshot at the moment of pass/fail.
- Console log dump if there are any `warn`/`error` level entries.
- Network log of any failed requests.

Save under `.ai/agent-logs/smoke-evidence-<date>/check-<N>.{png,log}`.

## Phase 5 — Write smoke log

Append to `.ai/agent-logs/smoke-<YYYY-MM-DD>-<HHMM>.md`:

```markdown
# Smoke Test — <YYYY-MM-DD HH:MM>

**Verdict:** ✅ all green / ⚠️ partial / ❌ failed

## Stack
- <stack name + ports + db>

## Checks
1. **Boot** — ✅/❌ — <evidence path or error excerpt>
2. **Homepage renders** — ✅/❌ — <evidence>
3. **Auth round-trip** — ✅/❌/skipped — <evidence>
4. **Golden path (<story name>)** — ✅/❌ — <evidence>

## Console summary
- <N warnings, M errors — top 3 listed with file:line>

## Network summary
- <N failed requests, top 3 listed>

## Notes
- <any judgment calls, skipped checks with reason>

## Repair plan
- <only if `repair=true` and any check failed; link to side-plan in .ai/plans/>
```

## Phase 6 — Optional repair

If `repair=true` AND any check failed:

Dispatch a single subagent to:
1. Read the smoke log.
2. Triage failures into a new side-plan at `.ai/plans/<date>-smoke-fixes/<date>-smoke-fixes.md` + per-fix cards.
3. Register the side-plan in `master-plan.md`'s "Plany dodatkowe" footer.
4. Yield back to the orchestrator (which can recursively call `05-plan-executor` to fix).

In **autonomous mode**, default `repair=true` because there's no human to triage.

## Why this skill is shaped this way

- **Narrow checks beat exhaustive tests.** A green smoke is meaningful only if its scope is tight enough to be trustworthy. 4 checks is the minimum for "production-ready, running locally" without being a full test suite.
- **Golden path comes from the plan, not from heuristics.** The master plan's #1 user story is what the team committed to as the core value — that's what smoke must verify.
- **Browser-tool tier is a choice, not a default.** `chrome-devtools-mcp` for audits, `claude-in-chrome` for auth, `agent-browser` for sweeps — pick lightest that fits.
- **Repair as a distinct phase.** Smoke test fails ≠ smoke test fixes. The fix is its own work, dispatched as a side-plan that re-enters Phase 5.
- **Process discipline.** Never `pkill -f "next dev"` — there may be parallel agents running. Track the PID you started and only kill that.
