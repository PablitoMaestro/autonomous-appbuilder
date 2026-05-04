# Checkpoint Prompts — Exact Wording

The 3 supervised checkpoints (gates 1, 2, 3) and an optional 4th (Phase 4 env, only when `gates=all`). Use this exact wording so users learn one consistent UX across runs.

The orchestrator presents each checkpoint as one structured message. After emitting, watch for `git diff` — if the user inline-edits the artifact while you wait, treat it as approval (they voted by editing).

## Checkpoint 1 — After PRD draft (Phase 1 → 2)

```
**Phase 1 — Discovery** complete.

Drafted .ai/PRD.md (~<N> lines).

Key decisions:
- Product type: <e.g. SaaS web app>
- Stack picked: <e.g. Next.js + Supabase + Vercel>  (rule: <e.g. SaaS / unspecified default>)
- MVP: <N features> — <one-line summary>
- Out of scope: <N items>

Reply:
- "ok" / "go" → lock the PRD and proceed to Phase 2 (design exploration, ~10 HTML variants)
- "redo: <feedback>" → I'll regenerate with your feedback prepended
- A specific edit (e.g. "rename product to X", "drop MVP item 3") → I'll apply inline if cosmetic; redo if structural
- "abort" → stop here; .ai/PRD.md left in place
```

## Checkpoint 2 — After design fan-out (Phase 2 → 3)

```
**Phase 2 — Design exploration** complete.

10 HTML variants in: docs/design-ideas/<timestamp>_landing-explorations/

Each is self-contained and opens in a browser. Themes:
- design-01: <concept seed>
- design-02: <concept seed>
- ...
- design-10: <concept seed>

Reply:
- "pick N" / "design-NN" → distill that variant into .ai/design.md and proceed to Phase 3
- "redo with <new direction>" → re-run the fan-out with adjusted brief
- "redo with N=20" → larger fan-out
- "abort" → stop; HTMLs left in place
```

## Checkpoint 3 — After master plan generation (Phase 3 → 4)

```
**Phase 3 — Canon** scaffolded.

Generated:
- .ai/PRD.md (locked from Phase 1)
- .ai/design.md (locked from Phase 2)
- .ai/techstack.md (light — names + 1-line; detailed rationale is Phase 1 task 1.2)
- .ai/site-map.md  (<N routes>)
- .ai/functional-spec.md  (<N user stories>)
- .ai/master-plan.md  — <N tasks across M waves, est. <Y h> with parallel subagents>
- .ai/master-plan-points/  (<N task cards>)
- .ai/environments.md

(Note: `.ai/database-architecture.md` is NOT generated here — it's Phase 1 task 1.1, written by `05-plan-executor` with a dedicated agent.)

This is the largest commitment in the run. Phase 5 may take 10–20 hours.

Reply:
- "ok" / "go" → lock canon and proceed to Phase 4 (env provisioning) → Phase 5 (execute)
- "trim: <task IDs>" → remove those tasks (and cards) before locking
- "reorder: <new wave layout>" → revise the waves table
- "redo master-plan: <feedback>" → regenerate just the master plan
- "abort" → stop; canon left in place
```

## Checkpoint 4 — After env provisioning (Phase 4 → 5) — ONLY when `gates=all`

```
**Phase 4 — Env** provisioning complete.

Local services:
- ✅ <service> on <port>
- ...

Remote services:
- ✅ GitHub repo: <url>
- <or> ⚠️ skipped — <reason>
- ...

Env files:
- apps/<app>/.env.local: <N> keys populated, <M> marked <MISSING>
- docs/runbook.md: written

Phase 5 is the long phase — ~10–20 h unattended once started.

Reply:
- "ok" / "go" → start Phase 5 (execute master plan)
- "wait, I need to set <X> first" → pause; tell me when ready
- "abort" → stop; env left in place
```

## Re-do feedback handling

When the user replies "redo: <feedback>":
- Re-invoke the same phase's sub-skill.
- Pass the feedback string verbatim as a prepended addendum to the sub-skill (e.g., `prior_feedback: "more focus on X, less on Y"`).
- The sub-skill's prompt should treat prior_feedback as a hard constraint on its next pass.
- If two consecutive redos still don't satisfy the user, escalate: present a structured "what's not working?" question rather than silently looping.

## Inline-edit detection

After emitting a checkpoint and before waiting for a reply:
- Snapshot `git diff` of the artifact (PRD, design, master-plan).
- If the user replies with anything OR the diff changes, treat as user input.
- If the diff changed but no text reply came, treat the diff as the answer:
  - If the artifact is now structurally valid (sections present, contents non-empty), treat as approval.
  - If the diff broke the artifact (deleted required sections), treat as a failed edit and ask the user to clarify.

## Abort handling

When the user replies "abort":
- Do NOT delete artifacts already produced.
- Do NOT roll back any committed work.
- Print a short status: "Aborted at Phase <N>. Artifacts left in place. Resume with `/00-build-app` (auto-detect) or `/00-build-app phase=<N+1>` to skip ahead."

## Autonomous mode (`gates=none`)

Skip ALL checkpoints. Each would-be checkpoint becomes a structured entry in `.ai/agent-logs/autonomous-decisions-<date>.md` instead — see `autonomous-decisions.md` for the log format.
