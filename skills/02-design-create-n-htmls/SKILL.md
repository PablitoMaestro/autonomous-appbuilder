---
name: 02-design-create-n-htmls
description: Generate N (1–100, default 10) self-contained HTML page design explorations in parallel using subagents. Defaults to landing pages, but supports any page type the user names (pricing, about, dashboard, 404, login, blog, marketing email, etc.). Auto-detects the project's brand, audience, language, and design tokens from repo content (CLAUDE.md, design docs, Tailwind/CSS tokens, README, package.json), then asks the user to confirm or override before dispatching. Use whenever the user wants multiple HTML design explorations, landing page variants, pricing page mockups, dashboard concepts, design direction options, or says things like "explore N designs", "spin up landing ideas", "create design variants", "generate a few landing pages", "show me 5 pricing page designs", "make some HTML concepts", "design N variants of X page" — even if they don't explicitly say "skill" or specify a number.
---

# 02 — Design: Create N HTMLs

Spin up a team of N (1–100) parallel subagents, each producing one self-contained HTML page in a distinct visual direction, all rooted in a shared **Design Brief** detected from the repo (or supplied by the user). The page type defaults to **landing page** but the user can ask for any other page type in the same breath (pricing, about, dashboard, login, 404, blog post, marketing email, settings, etc.).

> **Invocation contract:** This skill is invoked via `Skill('02-design-create-n-htmls')` — typically by `00-build-app` at Phase 2, but also standalone. The caller MUST NOT dispatch design subagents from main context; this skill owns the fanout.
>
> **This skill's responsibility ENDS at HTML generation.** It does NOT pick the best design and does NOT write `.ai/design.md`. Those are TWO separate Phase 1 master-plan tasks (1.1 and 1.2) executed by `05-plan-executor` subagents in their own worktrees with their own audit trail — the same lifecycle as every other master-plan task. The caller MUST NOT inline-pick or inline-distill tokens; that's a deviation hatch.

The skill's value isn't running agents in parallel — that's trivial. The value is producing a Design Brief specific enough to make the outputs cohesive, and detecting it from the repo so the user doesn't have to spell it out every time.

## Workflow at a glance

1. **Detect** — read the repo and produce a draft Design Brief.
2. **Confirm or Override** — show the brief to the user in one structured turn; let them accept, edit any field, or supply a free-form override that supersedes detection.
3. **Seed concepts** — generate N distinct concept seeds using the strategy that matches N.
4. **Dispatch** — fan out N subagents in a single message; each gets the shared brief + its unique seed + slot index.
5. **Report** — table summarising what each agent produced, plus the output directory.

## Phase 1 — Detect the Design Brief

Read these sources in parallel (skip what doesn't exist — don't fail loudly if a project lacks them):

| Signal | Files to read |
|---|---|
| Project intent + voice | `CLAUDE.md`, `AGENTS.md`, `README.md`, top-level `.ai/*.md`, `docs/*.md`, `docs/design*.md` |
| Design tokens | `tailwind.config.*`, `apps/*/tailwind.config.*`, any `globals.css` / `app.css` (look for CSS custom properties — `--color-*`, `--font-*`, `--radius-*`), shadcn `components.json` |
| Brand metadata | `package.json` (name, description, keywords), root-level `LICENSE` (org name), any `brand.md` / `branding.md` |
| Existing UI direction | First marketing/landing component if one exists (e.g., `app/page.tsx`, `pages/index.*`, `src/components/landing/`, `src/app/(marketing)/`) |
| Audience + language | Visible copy in existing UI files; locale config (`NEXT_LOCALE`, `i18n` configs); language used in the README |

Determine the **page type** from the user's invoking request. Default is `landing page`. If the user explicitly asked for another page type — pricing, about, dashboard, login, register, 404, blog post, marketing email, settings, profile, checkout, etc. — use that and treat it as a hard fact (don't override it from repo signals; the user's words win). The page type drives the Hard requirements (a pricing page needs a tier comparison; a 404 needs a recovery action; a dashboard needs at-a-glance KPIs) — pick sensible must-haves for the chosen type.

Synthesize into this **Design Brief** structure:

```
Page type:         <e.g., landing page (default) | pricing | about | dashboard | 404 | login | blog post | …>
Product:           <one line — what this product is>
Audience:          <who uses it>
Language:          <UI copy language — e.g., Polish, English, Japanese>
Tone:              <e.g., institutional, playful, technical, editorial, brutalist, calm>
Native theme:      <dark | light | either — agents pick freely if "either">
Palette anchor:    <1–3 hex values, or "agent's choice anchored on <color name>">
Typography lean:   <e.g., "Manrope display + Inter body", "system serif", "agent's choice — geometric sans">
Design DNA:        <2–3 sentences capturing the visual direction the repo points to>
Hard requirements: <bulleted list of things every page must show — calibrated to the page type. Keep this list tight; the agents need room to explore. e.g., landing: hero-led (the first viewport is roughly half the page's job; structure, section count, CTA approach, copy beats — agent's call); pricing: tier comparison + FAQ; dashboard: KPI overview + activity surface; 404: recovery path + on-brand recovery copy>
```

**If the repo is sparse** (no design docs, no marketing UI, brand info is just a stub) — don't fabricate a brief. Tell the user honestly that you couldn't deduce enough, and ask the eight Design Brief fields directly. Better to ask once than to confidently invent a direction that misses the mark.

If detection finds *some* signal but leaves fields ambiguous, fill what's confident and mark uncertain fields with `?` so the user can confirm or correct in Phase 2. The why: the user reads the brief in seconds — flagging uncertainty is faster than burying it.

## Phase 2 — Confirm or Override

Present the Design Brief to the user as one structured message:

```
**Detected Design Brief**
- Page type: landing page  ← (or whatever the user named)
- Product: …
- Audience: …
- Language: …
- Tone: …
- Native theme: …
- Palette anchor: …
- Typography lean: …
- Design DNA: …
- Hard requirements:
  - …

**Configuration**
- Number of designs (N): 10
- Output directory: docs/design-ideas/<YYYY-MM-DD>_<HH-MM>_<page-type-slug>/   (e.g., …_landing-explorations/, …_pricing-explorations/)
- Commit + push when done: no

Reply with:
- "ok" / "go" → use the defaults above
- A new value for any field (e.g., "N=20, page type: pricing, language: English, tone: brutalist")
- A free-form style override → wholesale replaces Design DNA + palette + typography
```

Wait for the user before dispatching. If they hand you a free-form override, restate the final brief inline so they can see what the agents are about to receive.

**Soft warning at N > 25**: warn the user this will spend significant tokens, and offer to batch (e.g., 25 per turn). Proceed only if they confirm.

**Default output dir:** `docs/design-ideas/<YYYY-MM-DD>_<HH-MM>_<slug>/`. Get the timestamp by running `date +%Y-%m-%d_%H-%M`. The `<slug>` is a short kebab-case derived from the page type (e.g., `landing-explorations`, `pricing-explorations`, `dashboard-explorations`, `404-explorations`). Always create the directory before dispatching.

**Commit + push default is "no".** This skill runs in many repos — committing without explicit consent is the wrong default. Only stage, commit, and push when the user said yes in Phase 2.

## Phase 3 — Concept seeds

The strategy depends on N. The reason to vary it: with small N, an orchestrator-curated list yields predictable, deliberate variety; with large N, a taxonomy-claim pattern produces more genuine differentiation than the orchestrator could brainstorm by hand.

### N ≤ 10 — orchestrator-curated

Before dispatching, generate exactly N one-line concept seeds. Each seed must be **specific, not categorical**:

- ✓ "oversized rotated display type used as background architecture"
- ✗ "bold typography"

- ✓ "side-by-side split canvas where left half is a fixed monolithic statement and right half scrolls"
- ✗ "asymmetric layout"

Each agent receives one assigned seed. Aim for variety across layout structure, typographic rhythm, spatial logic, and content hierarchy — not loud visual gimmicks.

### N > 10 — taxonomy-claim

Provide every agent with the same taxonomy plus their slot index. The agent picks one option from each axis, then commits to a specific concept inside that combination. Slot index is the tiebreaker so adjacent agents don't collide.

**Taxonomy:**

| Axis | Options |
|---|---|
| Type-led direction | display-type-as-architecture · editorial-serif-restraint · monospace-technical · kinetic-stretched-type · numeric-led · text-only-no-images |
| Spatial logic | asymmetric-grid · centered-classical · split-canvas · rail-of-strips · oversize-margins · edge-to-edge-cinema |
| Tonal direction | monochrome-editorial · single-bold-accent · gradient-glass · matte-document · saturated-poster · paper-tactile |
| Hierarchy stunt | numbered-sections · floating-tags · marquee-band · sticky-side-table · stacked-quotes · diagram-as-headline |

The agent picks ONE option per axis, ensuring the four together would not collide with adjacent slot indices.

## Phase 4 — Dispatch

Fan out all N subagents in **one message** with parallel tool calls. Each agent receives:

- The full Design Brief (verbatim — don't paraphrase between phases)
- The Universal Quality Rails (below)
- Its assigned concept seed (or the taxonomy + slot index for N > 10)
- Its slot number `<i>/<N>` and target filename `design-<NN>.html` (zero-padded so files sort correctly)
- The full output directory path

Use `general-purpose` subagent type. The subagent prompt below already instructs each agent to invoke the `frontend-design` skill — keep that line in every dispatch (it's a no-op when the skill isn't installed, and a real taste-uplift when it is).

**Don't use `isolation: "worktree"` for this dispatch.** Each agent writes to its own filename — they don't conflict. Worktrees would multiply setup cost for no isolation benefit.

### Subagent prompt template

The orchestrator fills the bracketed placeholders before dispatching:

```
You are agent [i] of [N] producing one self-contained HTML [page_type] (e.g., landing page, pricing page, dashboard, 404, login — pull from the Design Brief below).

DESIGN BRIEF:
[paste the full agreed Design Brief, verbatim from Phase 2 — including the Page type and Hard requirements lines, which together tell you what sections this page must contain]

YOUR CONCEPT SEED:
[either the assigned one-liner (N ≤ 10), or the taxonomy + slot-index instruction (N > 10)]

UNIVERSAL QUALITY RAILS — every agent honours these:
1. Self-contained — one HTML file, opens in a browser with no build step. Google Fonts and CDN icon libraries are allowed; everything else inline.
2. Working light/dark theme toggle in the nav. Persist via localStorage. Smooth CSS transition on color/background. Default to the brief's native theme.
3. No generic emoji or stock-icon decoration (no ✅🚀⭐). Use purposeful inline SVG only when it serves the layout. When in doubt, no icon.
4. UI copy in the brief's language — every visible word: nav, headlines, taglines, buttons, body, form labels, placeholders, footer. Code, comments, classnames, and HTML attributes stay in English.
5. Restraint — every section must justify its existence. Three sections done well beats seven done adequately.
6. Distinct from your peers — your slot index is [i]; assume neighbouring slots will do the obvious thing for your concept seed. Push further.

PAGE-TYPE EMPHASIS — read the Page type from the brief and find the load-bearing surface:
- If `landing page`: the hero (the first viewport, before any scroll) is roughly **half the page's job** — the most decisive moment. Make it count, and let your defining visual concept live there first. Beyond that one cue, every structural decision is yours: number of sections, copy hierarchy, single vs. multiple CTAs, length, density, whether to include trust signals at all and where. A weak hero kills a strong page; a strong hero earns the right to a quiet rest.
- For other page types: the load-bearing surface lives elsewhere — pricing in the tier comparison, dashboard in the KPI grid, 404 in the recovery copy, login in the form treatment. Identify it and put your concept's strongest move there.

YOUR WORKFLOW — follow in order:

1. **Invoke the `frontend-design` skill first.** Before brainstorming the concept, before writing a line of code. This skill governs design taste at a level no brief can encode — typography pairings, spacing rhythm, what "good" actually looks like in 2026. Treat it as a hard prerequisite when it's available in your environment. (If the skill isn't installed, skip silently — don't fail.)
2. **Use ultrathink to pick your concept.** Commit to ONE defining visual concept. Specific beats categorical. If it sounds like something you've seen a hundred times, discard and think again.
3. **Write the concept as the file's first line:** `<!-- CONCEPT: ... -->`.
4. **Use ultrathink again before writing the code itself.** Build the page.
5. **Save to:** [output_dir]/design-[NN].html
```

If the project's brief has additional **Hard requirements** (e.g., "must include a PWZ login section", "must show CME credit indicator"), they're already in the Design Brief above — the subagent will see them. Don't restate them in the rails.

## Phase 5 — Report

After all subagents return, output a markdown table:

| File | Concept | Native theme | Accent |
|---|---|---|---|
| design-01.html | … | dark | #4b8eff |
| … | … | … | … |

Then state:
- Total files generated (and any that failed)
- Output directory absolute path
- Whether commit + push happened (only if the user explicitly said yes in Phase 2)

## Why this skill is shaped this way

- **Detection beats interrogation.** Most repos have enough signal to skip a 20-question intake. When they don't, the skill admits it instead of fabricating a brief.
- **Concept seeds beat implicit coordination.** Asking 50 agents to "implicitly diverge" produces clones. Pre-allocation (N ≤ 10) and taxonomy-claim (N > 10) both scale linearly with N because variety is structural, not hoped-for.
- **Universal rails are deliberately minimal.** Domain-specific commands carry 15+ non-negotiables. This skill keeps only what's good across every web project: theme toggle, self-contained, no emoji, distinctness, language match. Everything else is a Design Brief field the user owns.
- **Single conversation.** Detect → confirm → dispatch happens in one back-and-forth. The user is never left hanging or asked to repeat themselves.
- **No worktrees, no commit by default.** Each agent writes a unique filename, so isolation is unnecessary. And the skill runs in many repos — auto-pushing to git would be the wrong default.
