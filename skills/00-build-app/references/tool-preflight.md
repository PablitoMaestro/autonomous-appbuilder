# Tool Preflight — Universal vs Deferred Checks

The orchestrator's Step 0.5. Runs before Phase 1 so the agent fails fast on missing-tool blockers instead of discovering them 30+ minutes into the run.

## Rule of thumb

- **Universal checks happen now** (Step 0.5): tools every greenfield run needs regardless of stack pick.
- **Stack-specific checks deferred** to `04-bootstrap-env`'s Step 1 preflight (which knows the stack).
- **MCP server checks happen now**: catching a missing MCP at Step 0.5 lets the agent surface it; catching it at Phase 7 means the smoke test silently degrades.

## Universal checks (run now)

| Check | How | Critical? | Action if missing |
|---|---|---|---|
| `node --version` ≥ 20 | shell | yes | Hard block — ask user to install Node 20+ |
| `pnpm --version` ≥ 10 | shell | yes (default stack) | Hard block — `npm i -g pnpm@latest` |
| `git --version` | shell | yes | Hard block — install git |
| `gh --version` + `gh auth status` | shell | warn | Warn now; Phase 4 re-prompts if still unauth'd |
| MCP `chrome-devtools-mcp` available | check Claude Code's MCP server list | yes | Hard block — Phase 7 smoke depends on it; user installs via `claude mcp add` |
| Working dir is a git repo (or empty) | `git rev-parse --git-dir` | warn | If non-git non-empty repo: warn + ask whether to `git init` |

## Stack-specific checks (deferred to Phase 4's preflight)

These ONLY make sense once the stack is picked (in Phase 1's PRD):
- `docker info` running — only needed when stack uses Supabase local
- `npx supabase --version` — only needed for Supabase stacks
- `npx vercel --version` + `vercel whoami` — only needed when deploying to Vercel
- Stack's bundler / build tool — `bun --version`, `cargo --version`, etc.

`04-bootstrap-env`'s existing Step 1 preflight already covers these. Don't duplicate.

## MCP server check details

Claude Code exposes installed MCP servers via its config. The orchestrator should check:
1. Is `chrome-devtools-mcp` listed in available MCP servers?
2. (Optional, recommended) `supabase` MCP — speeds up Phase 5 schema work
3. (Optional, recommended) `vercel` MCP — speeds up Phase 4 + deploy

Method: probe by attempting to invoke a no-op tool (e.g., `mcp__chrome-devtools-mcp__list_pages` with no args) and catching the "tool not available" error vs the "no pages open" error. Tool-not-available means the MCP isn't installed.

If `chrome-devtools-mcp` is missing: hard block with message:
> Phase 7 smoke test requires `chrome-devtools-mcp`. Install with:
> `claude mcp add chrome-devtools-mcp` (or your platform's equivalent)
> Then re-run `/00-build-app`.

## Readiness report format

Print one structured block. Color codes are optional but help skim:

```
**Tool preflight — <YYYY-MM-DD HH:MM>**

✅ node 20.11.0
✅ pnpm 10.4.0
✅ git 2.45.1
✅ chrome-devtools-mcp (MCP)
⚠️  gh — not authenticated (`gh auth login` deferrable to Phase 4)
✅ working dir is a git repo (clean tree)

Status: ready (1 warning, 0 blockers)
```

If any blockers:

```
❌ docker — not running (required if stack uses Supabase local — DEFERRED, will re-check at Phase 4)
❌ chrome-devtools-mcp (MCP) — not installed
❌ pnpm — not installed (>= 10 required for default Next.js+Supabase stack)

Status: NOT READY — 1 hard blocker (chrome-devtools-mcp). Cannot proceed.
Resolution: install chrome-devtools-mcp via `claude mcp add chrome-devtools-mcp`, then re-run.
```

## Modes

**Supervised mode:**
- Print the report. Wait for user reply.
- "ok" / no reply within 60 s → proceed if no hard blockers.
- "abort" → stop here.

**Autonomous mode:**
- Print the report (also append to `.ai/agent-logs/autonomous-decisions-<date>.md`).
- No hard blockers → proceed automatically.
- Hard blocker present → pause and write a "HARD BLOCK" entry to the decision log; do NOT proceed.

## Phase 0 task update

When preflight passes, the orchestrator flips master-plan task `0.1 Tool preflight` to `[x]`. If the master plan doesn't exist yet (first run), the orchestrator notes "preflight green" and Phase 1 (`01-discover-idea`) creates the master plan with `0.1 → [x]` pre-filled.

## What this is NOT

- ❌ A full system audit. Don't check disk space, RAM, OS version unless it's a known blocker.
- ❌ A network reachability test. Don't ping every API the app might use. That's runtime, not preflight.
- ❌ A version-pinning enforcer. Check minimums; users may run newer versions fine.
- ❌ Stack-specific. Anything that varies by stack belongs in `04-bootstrap-env`'s preflight.

## Why this lives in the orchestrator (not a separate sub-skill)

Tool preflight is small (~50 lines of checks), runs in main context, and produces a one-line summary. Promoting it to its own sub-skill (`00.5-verify-tools` or similar) adds an indirection level for very little benefit. Keep it inline in `00-build-app/SKILL.md` Step 0.5.

The check logic itself can be a shell one-liner per tool — no need for elaborate scripting.
