# Changelog

All notable changes to this plugin will be documented here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versioning is [SemVer](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.0] — 2026-05-04

### Added

- Seven composable skills under `skills/`:
  - `00-build-app` — orchestrator (7 phases, 2 modes, 3 strategic checkpoints)
  - `01-discover-idea` — `.ai/PRD.md` + master-plan stub + `CLAUDE.md` skeleton
  - `02-design-create-n-htmls` — N parallel HTML design explorations
  - `03-scaffold-ai-canon` — `.ai/` planning docs + master-plan with Waves table
  - `04-bootstrap-env` — local stack boots green; `.env.local` populated
  - `05-plan-executor` — per-task subagent dispatch with worktree, 3-tier tests, simplify pass, PR-merge-to-main
  - `06-plan-reviewer` — per-`[x]`-task audit subagents, surgical fixes
  - `07-smoke-test-app` — 4 narrow runtime gates
- Two slash commands: `/00-git-push`, `/00-git-merge-PR`
- Hooks layer (`hooks/hooks.json`) — observability for `Skill`, `Agent`, canonical-doc edits, `git`-ops; emits one-line audit summary on `Stop` (Claude Code only)
- Cross-tool packaging:
  - `.claude-plugin/plugin.json` + `.claude-plugin/marketplace.json` (Claude Code)
  - `.codex-plugin/plugin.json` (Codex)
  - Shared `skills/` directory (identical SKILL.md format for both platforms)
  - `.claude/skills` and `.agents/skills` symlinks for dev-mode usage
- Universal contracts baked into every skill: invocation contract, test-coverage contract (unit + integration + e2e with whitelisted skip reasons), worktree+PR contract, granularity contract (1 task = 1 agent), required `simplify` / `frontend-design:frontend-design` invocations inside subagents
- Auto-deployable `.ai/agent-logs/README.md` (template at `skills/00-build-app/references/agent-logs-readme.md`, idempotently copied at Step 0.5 of every run)
- Documentation: `README.md`, `INSTALL.md` (3 install paths: Claude marketplace / Codex manual / dev mode), `LICENSE` (MIT)

[Unreleased]: https://github.com/PablitoMaestro/pablitomaestro-agi-appbuilder/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/PablitoMaestro/pablitomaestro-agi-appbuilder/releases/tag/v0.1.0
