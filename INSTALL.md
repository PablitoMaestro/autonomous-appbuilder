# Install

Three install paths. Pick the one that matches how you work.

## Path 1 — Claude Code (marketplace, recommended)

```
/plugin marketplace add PablitoMaestro/pablitomaestro-agi-marketplace
/plugin install autonomous-appbuilder@pablitomaestro-agi-marketplace
```

The marketplace lives in a [dedicated repo](https://github.com/PablitoMaestro/pablitomaestro-agi-marketplace); this plugin is one of the catalog entries.

After install, run `/reload-plugins` to pick up the skills. Verify with:

```
/help
```

You should see `/autonomous-appbuilder:00-build-app` and the six sub-skills under that namespace.

**Hooks caveat (Claude Code only):** the settings watcher only watches `.claude/` directories that existed at session start. After first install, open `/hooks` once OR restart the session for the hook layer to fire.

## Path 2 — Codex (manual clone + symlink)

Codex doesn't have a centralized marketplace yet, so install by cloning into `~/.agents/plugins/` and symlinking the skills directory into `~/.agents/skills/`:

```bash
mkdir -p ~/.agents/plugins ~/.agents/skills
git clone https://github.com/PablitoMaestro/autonomous-appbuilder \
  ~/.agents/plugins/autonomous-appbuilder
ln -s ~/.agents/plugins/autonomous-appbuilder/skills \
  ~/.agents/skills/autonomous-appbuilder
```

Verify Codex picks them up — start a session and check that the skills appear in your skill listing.

To update:

```bash
cd ~/.agents/plugins/autonomous-appbuilder && git pull
```

## Path 3 — Dev mode (clone & use in-place, both tools)

Useful if you want to **edit the skills** while using them, or if you're evaluating before committing to a permanent install.

```bash
git clone https://github.com/PablitoMaestro/autonomous-appbuilder
cd autonomous-appbuilder
```

The repo ships with two symlinks:

- `.claude/skills → skills/` — Claude Code picks this up as standalone project skills
- `.agents/skills → skills/` — Codex picks this up as repo-scoped team skills

Open the repo with `claude` or `codex`, and the skills are available immediately under their unprefixed names (no namespace, since they're loaded as standalone, not plugin).

For Claude Code dev-mode testing of the **plugin format itself** (with namespacing):

```bash
claude --plugin-dir /path/to/autonomous-appbuilder
```

When you change a skill's `SKILL.md`, run `/reload-plugins` to pick up the edits without restarting.

## Uninstall

**Claude:** `/plugin uninstall autonomous-appbuilder`

**Codex:**

```bash
rm ~/.agents/skills/autonomous-appbuilder
rm -rf ~/.agents/plugins/autonomous-appbuilder
```

**Dev mode:** delete the cloned directory.

## Verifying the install

Run `/00-build-app` (Claude) or invoke the build-app skill (Codex) on a throwaway directory:

```bash
mkdir /tmp/test-build && cd /tmp/test-build
git init
# then invoke the skill
```

The skill should:

1. Run Step 0.5 — tool preflight (Node, pnpm, gh, chrome-devtools-mcp).
2. Auto-create `.ai/agent-logs/` with a navigation `README.md`.
3. Detect that there's no `.ai/master-plan.md` yet and route to Phase 1 (`01-discover-idea`).
