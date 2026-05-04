# autonomous-appbuilder

A cross-tool plugin (Claude Code + Codex) that takes a web idea or a partially-completed repo through seven phases — **idea → PRD → design exploration → scaffold → env bootstrap → autonomous execution → optional review → smoke test** — until a production-ready app runs locally.

> One Claude/Codex session, two modes (supervised by default; fully autonomous when invoked with `gates=none`), resumable from any phase.

## What you get

Seven composable skills, each individually invocable:

| # | Skill | Owns |
|---|---|---|
| 0 | `00-build-app` | Orchestrator. Routes the 7 phases, 2 modes, 3 strategic checkpoints, decision-log audit. |
| 1 | `01-discover-idea` | `.ai/PRD.md` + master-plan stub + `CLAUDE.md` skeleton |
| 2 | `02-design-create-n-htmls` | N parallel HTML design explorations under `docs/design-ideas/` |
| 3 | `03-scaffold-ai-canon` | `.ai/` planning docs + expanded master-plan with Waves table + per-task cards |
| 4 | `04-bootstrap-env` | Local dev stack boots green; `.env.local` populated; runbook seeded |
| 5 | `05-plan-executor` | Per-task subagent dispatch, worktrees, 3-tier tests, simplify pass, PR-merge-to-main |
| 6 | `06-plan-reviewer` | Per-`[x]`-task audit subagents, coverage check, surgical fixes |
| 7 | `07-smoke-test-app` | 4 narrow gates: dev boots, hydrates, auth round-trips, golden CRUD round-trips |

Plus two slash commands: `/00-git-push`, `/00-git-merge-PR` and a hooks layer that logs every `Skill` / `Agent` / canonical-doc edit / `git`-op tool call to `.ai/agent-logs/hooks-<date>.jsonl` (Claude Code only — Codex doesn't have hooks yet).

## Universal contracts (cross-platform)

Every skill in the pipeline obeys these — restated in each skill's `SKILL.md`:

1. **Invocation contract.** Skills are invoked via the `Skill` tool. The caller never reproduces the skill's work inline. The orchestrator's compliance audit (Step 6 of `00-build-app`) verifies every phase has its `Skill('XX')` call in the message history.
2. **Test-coverage contract** (Phase 5 + 6). Every code-shipping task delivers three test tiers: unit + integration + e2e. Skip only via whitelisted reason (`pure-config-no-logic`, `pure-presentation-no-boundaries`, `no-user-visible-surface`, `docs-only-task`).
3. **Worktree + PR contract.** Every parallel agent OR every code-touching task in master-plan phases 1+ runs in its own `git worktree` and merges its own PR to `main`.
4. **Granularity contract.** One master-plan task = one agent dispatch. Bundling tasks is forbidden.
5. **Required skill invocations inside subagents.** `simplify` (mandatory on every diff before commit) + `frontend-design:frontend-design` (only if diff touches `.tsx`/`.jsx`/`.css`/`.scss`/`.html`).

## Cross-tool support

This plugin ships **two manifests** so the same `skills/` directory works in both ecosystems:

- `.claude-plugin/plugin.json` — Claude Code plugin manifest
- `.codex-plugin/plugin.json` — Codex plugin manifest
- `skills/` — shared, identical SKILL.md format on both platforms
- `.claude/skills/` and `.agents/skills/` — symlinks to `skills/` for **dev-mode usage** (clone the repo and work in it directly without installing as a plugin)

See [INSTALL.md](./INSTALL.md) for all three install paths.

## Install (TL;DR)

**Claude Code:**

```
/plugin marketplace add PablitoMaestro/pablitomaestro-agi-marketplace
/plugin install autonomous-appbuilder@pablitomaestro-agi-marketplace
```

**Codex:**

```bash
git clone https://github.com/PablitoMaestro/autonomous-appbuilder ~/.agents/plugins/autonomous-appbuilder
ln -s ~/.agents/plugins/autonomous-appbuilder/skills ~/.agents/skills/autonomous-appbuilder
```

**Dev mode (clone & use in-place, both tools):**

```bash
git clone https://github.com/PablitoMaestro/autonomous-appbuilder
cd autonomous-appbuilder
# .claude/skills and .agents/skills are pre-symlinked
claude    # picks up .claude/skills automatically
codex     # picks up .agents/skills automatically
```

Full instructions in [INSTALL.md](./INSTALL.md).

## Usage

Invoke the orchestrator:

```
/autonomous-appbuilder:00-build-app                   # supervised, detect repo state
/autonomous-appbuilder:00-build-app "<idea>"          # idea as first arg
/autonomous-appbuilder:00-build-app gates=none        # autonomous mode
/autonomous-appbuilder:00-build-app review=on         # add review pass after execute
/autonomous-appbuilder:00-build-app deploy=prod       # add prod deploy pass at end
```

Each sub-skill is also independently invocable:

```
/autonomous-appbuilder:01-discover-idea
/autonomous-appbuilder:05-plan-executor
/autonomous-appbuilder:07-smoke-test-app
```

For Codex, skill discovery happens automatically — Codex picks them up from `.agents/skills/` (repo-scoped) or `~/.agents/skills/` (user-scoped).

## Skill name resolution (bare vs namespaced)

The skill text uses **bare names** (`Skill('01-discover-idea')`) when one skill in this plugin invokes another. This works for two reasons:

1. **Plugin-installed mode (Claude Code marketplace install)** — within the same plugin's context, sibling skills resolve by their bare name. Cross-plugin references to the same skill name are disambiguated by namespace (`/autonomous-appbuilder:01-discover-idea`), but intra-plugin invocation does not require the prefix.
2. **Dev mode (clone + symlink)** — skills are loaded as standalone (no plugin namespace), so bare names are the only form that exists.

External skills our pipeline depends on (e.g., `simplify`, `frontend-design:frontend-design`) are referenced with their full namespace because they live in different plugins.

When a user invokes the orchestrator from a Claude Code chat, they use the namespaced form `/autonomous-appbuilder:00-build-app`. Once the orchestrator runs, all internal `Skill('XX')` calls resolve within this plugin's context.

## What this plugin does NOT contain

- **Stack-specific code generation** — that lives in plugin skills like `vercel:nextjs`, `vercel:shadcn`, `supabase:supabase`. Subagents invoke them when their task touches the relevant stack.
- **Tool wrappers** — `chrome-devtools-mcp`, `claude-in-chrome` are MCP servers, not skills. Subagents call them directly.
- **Project-specific config** — that lives in your project's `CLAUDE.md` / `AGENTS.md` / `.ai/*.md`, not here.

## License

MIT — see [LICENSE](./LICENSE).
