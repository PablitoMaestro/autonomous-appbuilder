# Eval Strategy (Deferred from v0)

Skill-creator's standard "spawn subagents per test case, grade against assertions, view results" loop is too expensive for full e2e at this scope (10–20 h × N iterations = weeks). Per the design spec, evals are deferred until after first hand-validated runs of `/00-build-app`.

This file specifies the 3-tier test plan to run when iteration becomes worthwhile.

## Tier 1 — Per-sub-skill unit evals

The 5 NEW sub-skills (`01-discover-idea`, `03-scaffold-ai-canon`, `04-bootstrap-env`, `06-plan-reviewer`, `07-smoke-test-app`) are independently testable. The 2 reused skills (`02-design-create-n-htmls`, `05-plan-executor`) already work in production.

### Per skill, ~3 evals each (~15 evals total):

**`01-discover-idea`** — input: empty repo + one-line idea. Verify:
1. PRD has all required sections (`MVP scope`, `Out of scope`, `Tech stack`, `Success criteria`).
2. Stack pick matches the tier rule for the input shape (e.g. game idea → Phaser+Vite).
3. Out-of-scope items don't reappear in MVP scope.

**`03-scaffold-ai-canon`** — input: locked PRD + design + master-plan stub. Verify:
1. All 5 canon docs generated (techstack-light, site-map, functional-spec, environments, expanded master-plan); `database-architecture.md` is NOT among them — it's Phase 1 task 1.1.
2. Master plan has ≥ 1 task; each task has a card file.
3. Cross-references resolve (no doc duplicates a fact that's owned by another doc).

**`04-bootstrap-env`** — input: locked techstack/environments + Docker available. Verify:
1. Local stack boots (`pnpm dev` returns 200 within 60 s).
2. `.env.local` has all keys from `.env.example`.
3. Recipe selected matches the picked stack.

**`06-plan-reviewer`** — input: a sample plan with mixed-quality `[x]` tasks. Verify:
1. `[x]` stays `[x]` in all cases (review never downgrades).
2. `simplify` skill ran on every task (`simplify_ran: yes` in every report).
3. Verdict distribution is sane (not 100% `improved` — would mean reviewers are rewriting).

**`07-smoke-test-app`** — input: a known-good repo + a known-broken repo. Verify:
1. Known-good returns "all green".
2. Known-broken returns specific failed check with evidence path.
3. Repair flag (when set) creates a side-plan in `.ai/plans/`.

### Per-eval cost
- ~5–30 min wall-clock per eval (most expensive: scaffold-ai-canon with full master plan).
- Can run several in parallel via subagents.

## Tier 2 — Orchestrator routing eval

Tests `00-build-app`'s detection + routing logic without actually running the phases. Each eval seeds a fake repo state and verifies the orchestrator picks the right entry phase.

### ~5 evals, each < 2 min:

1. Empty repo → entry_phase = 1.
2. PRD only → entry_phase = 2.
3. PRD + design + canon, no env → entry_phase = 4.
4. All artifacts valid + smoke recent ✅ → entry_phase = "done".
5. `phase=design from=scratch force=yes` on a non-empty repo → entry_phase = 1, archives existing `.ai/` to `.ai/_archive/`.

These are CI-grade — fast enough to run on every change to `00-build-app/SKILL.md` or `phase-detection.md`.

## Tier 3 — End-to-end qualitative

ONE big eval per release-candidate iteration. Pick a small, well-bounded idea (e.g. "a pomodoro timer with a streak counter") and run `/00-build-app "<idea>" gates=none`. Time-box to 4 hours of model time.

### What to grade qualitatively:
1. Did the app actually boot at the end?
2. Does the golden path work?
3. Is the autonomous decision log readable and complete?
4. Does the master plan match the PRD's MVP scope (no scope creep, no missing features)?
5. Was the design pick reasonable for the idea?
6. How many times did Phase 5 retry tasks? (Indicator of plan quality.)

This is human-judged. No assertions — just a 30-min review of the output.

## When to run which tier

- **Per change to a sub-skill:** Tier 1 for that sub-skill (~15 min).
- **Per change to `00-build-app/SKILL.md`:** Tier 2 (~10 min total).
- **Per release-candidate (every few weeks):** Tier 3 (~4 h + 30 min review).
- **Per major refactor (e.g., new stack added to `04-bootstrap-env`):** Tier 1 for the affected sub-skill + Tier 3 with an idea that exercises the new stack.

## Workspace layout

When evals run, put outputs in:
```
.ai/agent-logs/evals/
  iteration-<N>/
    tier-1/
      01-discover-idea/
        eval-1-empty-repo/
          inputs/
          outputs/
          grading.json
        ...
    tier-2/
      ...
    tier-3/
      ...
```

`grading.json` per eval uses skill-creator's schema (`expectations` array with `text`, `passed`, `evidence`).

## Why this is deferred

- The skills are still settling — their interfaces will change as we hand-validate.
- Building eval infrastructure before the skills stabilize means rebuilding it after every interface change.
- One careful hand-run of `/00-build-app` on a fresh repo gives more signal at this stage than 50 automated assertions.

Revisit this file once the orchestrator + sub-skills have shipped 3 hand-validated runs without major refactoring.
