# Recipe — Bun CLI / Library

For TypeScript projects shipping a CLI binary or a publishable library on npm using Bun as the runtime + toolchain. npm registry is the only "remote" service. CI is optional.
## What this recipe provisions

- **Local:** Bun runtime, TypeScript config, `bun test`, Bun's bundler, local `bun link` for CLI smoke testing.
- **Remote (optional):** GitHub repo, npm publish access (interactive `npm login` or CI token), GitHub Actions release workflow.
- **Env files:** `.env.example` only (npm + GitHub tokens for CI). Most projects don't need `.env.local`.
- **Runbook:** `docs/runbook.md` with publish flow, version bump, CI dispatch, npm 2FA notes.

## Required env keys

Most CLI/library projects need **zero env vars at runtime**. The keys below exist only for publishing + CI:

| Key | Local source | Remote source | Used by |
|---|---|---|---|
| `NPM_TOKEN` | `<MISSING>` (use interactive `npm login` instead) | npm dashboard → Access Tokens → "Automation" type | GitHub Actions publish job |
| `GITHUB_TOKEN` | `gh auth token` (read-only) | Auto-injected by Actions | Optional release-notes scripts |

**Important:** for local publishing, prefer interactive `npm login` over committing a token. `NPM_TOKEN` belongs in GitHub Actions secrets, never in `.env.local`. Scoped packages (`@org/pkg`) need `publishConfig.access=public` in `package.json` AND publish rights on the scope.

## Provisioning steps

### Step 1 — Preflight

```bash
bun --version           # ≥ 1.1
node --version          # ≥ 20 (some tooling still shells out via node)
gh auth status          # for remote provisioning (optional)
npm --version           # for `npm login` / `npm publish`
```

If `bun` is missing: `curl -fsSL https://bun.sh/install | bash`. Don't auto-install on a managed environment — surface the missing piece and stop.

### Step 2 — Repo skeleton

For an empty directory:

```bash
mkdir -p src tests
bun init -y             # generates package.json, tsconfig.json, .gitignore, README.md
# Then rewrite package.json per Step 3
```

Extend `.gitignore` beyond Bun's defaults: `node_modules`, `dist`, `*.tgz`, `.env.local`, `.env*.local`, `coverage`.

### Step 3 — package.json shape

**For a CLI binary** (entry point installed as a system command):

```json
{
  "name": "<pkg-name>",
  "version": "0.0.1",
  "type": "module",
  "bin": {
    "<cli-name>": "./dist/cli.js"
  },
  "files": ["dist", "README.md"],
  "scripts": {
    "build": "bun build ./src/cli.ts --outdir ./dist --target node --minify",
    "dev": "bun run --watch ./src/cli.ts",
    "test": "bun test",
    "typecheck": "bun x tsc --noEmit",
    "prepublishOnly": "bun run build"
  },
  "engines": { "node": ">=20" },
  "devDependencies": { "@types/bun": "latest", "typescript": "^5.5.0" }
}
```

The first line of `src/cli.ts` must be `#!/usr/bin/env node` so the bin shim works after install.

**For a library** (published exports map):

```json
{
  "name": "@<scope>/<pkg-name>",
  "version": "0.0.1",
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    },
    "./package.json": "./package.json"
  },
  "files": ["dist", "README.md"],
  "scripts": {
    "build": "bun build ./src/index.ts --outdir ./dist --target node && bun x tsc --emitDeclarationOnly --outDir ./dist",
    "test": "bun test",
    "typecheck": "bun x tsc --noEmit",
    "prepublishOnly": "bun run build"
  },
  "publishConfig": { "access": "public" },
  "engines": { "node": ">=20" }
}
```

Bun's bundler doesn't emit `.d.ts` — fall through to `tsc --emitDeclarationOnly` for type definitions.

### Step 4 — TS config

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2023"],
    "types": ["bun"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "skipLibCheck": true,
    "declaration": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "esModuleInterop": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

Use `Bundler` resolution because Bun + the bundler both support it; switch to `NodeNext` for libraries targeting plain Node consumers without a bundler.

### Step 5 — Install + verify

```bash
bun install                     # populates bun.lockb
bun run typecheck               # must pass
bun run build                   # must produce ./dist/cli.js (or ./dist/index.js)
bun test                        # runs *.test.ts under tests/ and src/
ls -la dist/                    # confirm artifacts
```

If `bun build` produces no output: check the script targets the right entry file. If `bun test` finds zero tests: confirm files match `*.test.ts`.

### Step 6 — Write env files

Most CLI/library projects don't need `.env.local`. Write only `.env.example` for the CI publish flow:

```bash
# .env.example — committed
NPM_TOKEN=<your-npm-automation-token>
GITHUB_TOKEN=<auto-injected-by-actions>
```

Add a `.env.local` only if the CLI itself reads runtime config from env (e.g., an API key for a service it wraps) — and document each key.

### Step 7 — GitHub repo (optional)

```bash
gh repo create <pkg-name> --public --source=. --push
# OR --private
```

Skip if `gh auth status` fails. In autonomous mode, log and continue — local builds still work.

### Step 8 — npm publish setup (optional)

For interactive (human) publishing:

```bash
npm whoami                      # check current login
npm login                       # if not logged in; opens browser for OTP
```

For scoped packages:

```bash
npm access list packages @<scope>          # verify org membership + publish rights
# Either set publishConfig.access=public in package.json,
# OR override registry in .npmrc:
echo "@<scope>:registry=https://registry.npmjs.org/" >> .npmrc
echo "//registry.npmjs.org/:_authToken=\${NPM_TOKEN}" >> .npmrc
```

Never commit a `.npmrc` containing a real token — the `${NPM_TOKEN}` form expands at publish time from env, which is safe to commit. For CI: add `NPM_TOKEN` as a GitHub Actions secret (`gh secret set NPM_TOKEN`); use the npm "Automation" token type — it bypasses 2FA for CI but still requires 2FA on the account.

### Step 9 — Seed runbook

Write `docs/runbook.md`:

```markdown
# Runbook — <pkg-name>

## Local development
- Prerequisites: Bun ≥ 1.1, Node ≥ 20 (for type-check / declaration emit)
- First boot: `bun install` then `bun test` then `bun run build`
- Smoke-test CLI: `bun link` here, `bun link <pkg-name>` in a scratch dir, run the bin
- Watch mode: `bun run dev`

## Env vars
- Runtime: none (or list keys the CLI reads via `process.env.*`)
- CI: `NPM_TOKEN` (GitHub Actions secret, npm Automation token type)

## Publish flow
1. Bump version: `npm version patch|minor|major` (creates git tag)
2. `bun run build` — verify `dist/` is fresh
3. `npm publish` (interactive 2FA prompt) OR push the tag and let CI publish
4. `git push --follow-tags`

## CI dispatch
- `.github/workflows/release.yml` triggers on `v*` tags
- Job: `bun install` → `bun test` → `bun run build` → `npm publish`
- Token: `NPM_TOKEN` from secrets

## Rollback
- `npm deprecate <pkg>@<version> "<reason>"` — soft remove
- `npm unpublish` — only allowed within 72h, avoid in practice

## Incident contacts
- [Maintainer email — placeholder]
```

### Step 10 — Final verify

```bash
bun install
bun run typecheck
bun run build
bun test

# CLI-only: end-to-end bin smoke test
bun link
cd /tmp && mkdir bun-cli-smoke && cd bun-cli-smoke
bun link <pkg-name>
bun x <cli-name> --help && echo "✅ CLI invokable" || echo "❌ bin shim broken"
cd - && bun unlink                   # cleanup

# Library-only: dry-run publish to inspect tarball contents
npm publish --dry-run | grep -E "^(npm notice|Tarball)"
```

All steps must pass. If `bun link` fails with a permissions error: confirm the global Bun bin dir is on `$PATH` (`bun pm bin -g`).

## Failure modes + fixes

- **Bun version mismatch** (e.g., a `bun.lockb` from 1.0.x on a 1.1.x machine) → delete `bun.lockb` + `node_modules`, run `bun install` fresh. Pin via `engines.bun` if the project depends on a specific release.
- **`bun test` finds zero tests** → confirm files end in `.test.ts` under `src/` or `tests/`. Bun's runner doesn't honor a custom `testMatch` — rename to fit.
- **npm 2FA prompt blocks CI publish** → switch the npm token to "Automation" type (bypasses 2FA for publish, still protects the account). Classic tokens fail on accounts with 2FA-on-publish enforced.
- **Scope permission denied** (`403 Forbidden — you do not have permission to publish`) → confirm `npm access list packages @<scope>` shows the user with publish rights. New scopes default to private; add `"publishConfig": {"access": "public"}` for free-tier public packages.
- **`bin` shim runs but command not found** after `npm i -g` → confirm the `bin` path is correct relative to the package root (post-publish `dist/` layout, not `src/`), and that the entry file starts with a shebang line.
- **Type declarations missing from published tarball** → `bun build` doesn't emit `.d.ts`; `prepublishOnly` must also run `tsc --emitDeclarationOnly`. Verify with `npm publish --dry-run` and grep for `.d.ts` in the file list.
- **`exports` map breaks consumers** (TypeScript can't find types) → ensure `types` appears *first* in each export entry, before `import` / `require`. Order matters in conditional exports.
