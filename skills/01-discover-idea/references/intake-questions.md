# Intake Questions Bank

Use these when detection from the repo (`CLAUDE.md`, `README.md`, `.ai/*.md`, `package.json`) doesn't yield enough signal to draft a PRD. The SKILL.md body has the canonical one-shot intake form; this file expands it with branching questions you can pull from when a single round isn't enough.

## Core principle

**Ask the smallest set of questions that produces a tight PRD.** A long intake exhausts the user before the PRD even exists. Aim for one structured turn (≤ 6 questions). Only branch into round 2 when answers genuinely don't constrain stack/scope.

## Detection-first decision tree

Before asking anything, check what the repo already says:

| Signal present | Ask about |
|---|---|
| README has product description | scope only (skip product type / audience) |
| package.json has framework deps | nothing about stack — it's already chosen |
| `CLAUDE.md` lists conventions | nothing about language / code style |
| Existing UI files in repo | nothing about aesthetic — match what's there |
| Empty repo, no idea string | everything in the canonical intake |

## Round 1 — Canonical intake (use this verbatim from SKILL.md)

```
I'd like to ground the PRD before drafting. Answer any of these — skip what you don't care about:

1. **Product type** (multi-pick OK):
   [ ] SaaS web app   [ ] static content / docs   [ ] internal tool   [ ] CLI / library
   [ ] browser game   [ ] desktop app             [ ] mobile             [ ] other: ___

2. **Who uses it** (one line): ___

3. **Top 1–3 things it must do** (MVP scope): ___

4. **Aesthetic direction** (or "agent's choice"): ___

5. **Stack preference** (or "agent picks based on type"): ___

6. **Any non-obvious constraints** (compliance, target deployment, language, integrations): ___

Reply with answers, "go" to use defaults, or skip — autonomous mode will proceed in 5 minutes regardless.
```

## Round 2 — Branching follow-ups (only if round 1 answers are ambiguous)

Pull from these only when the user's reply genuinely doesn't constrain something load-bearing:

### If product type is SaaS but no audience given
- "Is this for the public, a specific company, a niche profession, or yourself?"

### If MVP scope is 1-line vague (e.g. "manage tasks")
- "If we ship just one user flow, which one matters most? E.g. for tasks: create-and-list, or assign-to-team, or recurring-templates?"

### If stack is "agent picks" but constraints suggest a non-default
- Compliance mention (HIPAA, GDPR-strict, financial) → "Any infra that's off-limits (e.g., must be self-hosted, can't use US-only services)?"
- Mention of "real-time" or "collaborative" → "Soft real-time (polling fine) or hard real-time (websockets / live cursors)?"
- Mention of "AI" or "LLM" → "Which provider, or agent's pick (default: Anthropic via direct API; Vercel AI Gateway if multi-provider routing)?"

### If aesthetic is "agent's choice" but tone matters
- Skip — let `02-design-create-n-htmls` handle this in Phase 2 with N variants.

### If constraints mention multi-tenancy
- "B2B with separate orgs (workspace pattern), or B2C with personal accounts?"

### If language wasn't specified
- Default English; auto-detect Polish/etc. only if `CLAUDE.md` or README signals it.

## What NOT to ask

- ❌ Questions whose answer is implied by the product type (e.g., "do you need a database?" for a SaaS app — yes).
- ❌ Open-ended questions that re-ask the user to do the agent's job ("what should the MVP look like?" without options).
- ❌ Anything the design phase will decide (color palette, typography, layout — those are Phase 2).
- ❌ Anything the scaffold phase will decide (table names, route structure, file layout — those are Phase 3).
- ❌ Anything the bootstrap phase will decide (env keys, region, port — those are Phase 4).

## Handling silence in autonomous mode

When the user doesn't reply within 5 minutes:
- Use the canonical defaults below.
- Log every default in the autonomous decision log with `Why: no user reply within timeout; default applied`.

### Canonical defaults

| Field | Default |
|---|---|
| Product type | Inferred from idea text; SaaS web app if idea is unspecified |
| Audience | "general public" if SaaS; "developers" if CLI / library; "myself" if vague |
| MVP scope | Top 3 features the idea text explicitly mentions; 1 if idea is short |
| Out-of-scope | payments, multi-tenancy, mobile-native, AI-by-default, i18n, accessibility-beyond-AA |
| Aesthetic | Agent's choice (Phase 2 will explore N variants) |
| Stack | Tier rules from SKILL.md Phase 3 |
| Language | English |
| Hosting | Default per stack (Vercel for Next/Astro/Vite; npm for CLI) |
| Region | EU if any signal of European audience; US otherwise |

## Example branching session

User runs `/01-discover-idea "a flashcard app for medical students"` in supervised mode.

Round 1: agent skips product type (SaaS web app inferred from "app" + flashcard pattern), asks reduced intake:

```
I'll draft a PRD for a flashcard app for medical students. To ground it:

1. Audience specifics: which country / language / exam focus? (or "agent infers")
2. Top MVP feature beyond create-and-review-cards: spaced repetition? deck sharing? quizzes? (pick 1-2)
3. Anything off-limits (specific compliance, must avoid X service)?

Reply with answers or "go" for defaults.
```

User: "Polish students for LEK exam, spaced repetition is the must-have, no real-time collab needed"

→ Agent has enough. Drafts PRD with: Polish UI, LEK-relevant subject categories in DB schema preview, spaced repetition algorithm in MVP, no collab features, Vercel EU region in tech stack.

No round 2 needed.
