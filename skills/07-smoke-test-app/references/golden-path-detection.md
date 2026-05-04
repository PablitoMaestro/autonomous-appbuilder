# Golden Path Detection

How `07-smoke-test-app` derives the "one core CRUD flow" check (Check 4) without being told what to walk.

## The contract

Check 4 must walk a **realistic user journey** — one that crosses ≥ 1 boundary (e.g. write → read; create → list; submit → confirmation). The walkthrough comes from the master plan's #1 user story, so smoke verifies what the PRD claimed mattered most.

## Source of truth

In priority order:

1. **Explicit override** — orchestrator passed `golden_path: "<description>"` in the addendum. Use it verbatim. Stop.
2. **`.ai/functional-spec.md`** — first user story that's `[x]` in master-plan. Format: `As a <user>, I want to <action> so that <outcome>` → walk that action.
3. **`.ai/master-plan.md`** Phase 5 (or whichever phase is labeled "Core features" / "MVP") — first task that's `[x]` AND has a clear user-facing verb (create / submit / view / list). Walk that.
4. **PRD MVP item #1** — fallback. Walk whatever it described.
5. **Inference fallback** — if none of the above yield a story, walk the homepage CTA. If the page has no CTA, declare Check 4 `skipped` and report.

## Mapping a story to browser steps

A story like:

> As a medical student, I want to create a flashcard deck so that I can study it later.

Maps to:

1. Land on the authenticated home (assumes Check 3 — auth — already passed).
2. Find an entry point: nav link "Decks", or empty-state CTA "Create your first deck".
3. Click into create flow.
4. Fill the minimum-required form fields with realistic data (deck name + 1 card).
5. Submit.
6. Verify redirect to a deck-detail or list page.
7. Verify the new deck/card appears in that list (READ side of the round-trip).

The pattern: **CREATE → confirm appears in READ**. That's the round-trip check.

For non-CRUD stories:
- "Send a message" → submit → confirm message appears in conversation thread.
- "Vote on a poll" → submit → confirm vote count incremented OR "you voted" indicator appears.
- "Watch a video" → click play → confirm video element starts (currentTime > 0 within 5 s).
- "Download a report" → click → confirm Content-Disposition header on the response, or file appears in download folder.

If the round-trip can't be observed in the browser (e.g., async email send), check the closest observable proxy (e.g., success toast appeared) and note in the smoke log that the actual side-effect wasn't directly verified.

## Field-filling heuristics

Use realistic test data, not lorem ipsum:
- Names: from `.ai/environments.md` test users (e.g., `admin@example.test`) or sensible defaults.
- Text fields: 1-2 short, plausible sentences.
- Numbers: small positive integers.
- Dates: today + a week (avoid past dates which may fail validation).
- Required-but-not-asked-by-the-story fields: minimum valid value (one option from a select, etc.).

If a required field has no obvious value and no default in the form, log the field name and the smoke check `skipped` rather than guessing.

## Selecting the browser tool

Use the lightest tool that supports both DOM interaction and console/network capture:

| Tool | Best for |
|---|---|
| `chrome-devtools-mcp` | Default. Has DOM, console, network, a11y, perf trace. |
| `claude-in-chrome` | When the auth flow uses real Chrome session cookies (e.g., OAuth provider with strict CORS). |
| `agent-browser` | When walking multiple viewports (desktop + mobile) in one pass. |

`07-smoke-test-app/SKILL.md` defaults to `chrome-devtools-mcp` because it's the most capable for the 4-check matrix.

## Capturing failure context

When Check 4 fails, capture:
- Screenshot at the moment of failure (the broken state).
- Console output from the last 10 seconds (warn + error level only).
- Network requests made during the failed step (URL + status + response body if < 2 KB).
- The exact step that failed (e.g., "step 4 — form submit returned 500").

Save all to `.ai/agent-logs/smoke-evidence-<date>/check-4-<step>.{png,log,json}`.

## When the master plan has no completed user-story task

Edge case: smoke runs but Phase 5 hasn't shipped any user-facing features yet (e.g., user invoked `/07-smoke-test-app` after only foundational tasks). Behavior:

- Mark Check 4 as `skipped` with reason: "no `[x]` user-story task in master-plan to derive golden path".
- Don't fail Check 4 — it's not broken, it's not applicable yet.
- Suggest in the smoke log: "Run smoke again after Phase 5 ships its first feature."

## When `golden_path` override conflicts with the plan

If the orchestrator passes `golden_path` but the master plan suggests a different one, use the override. Note the discrepancy in the smoke log:

```
Check 4 — golden path: "<override description>" (orchestrator override)
  Note: master-plan.md's first user story would have been "<derived description>"; override took precedence.
```

This makes auditing easy — the human can see what the agent walked vs. what the plan recommended.

## Anti-patterns

- ❌ Walking every user story. Smoke is narrow on purpose. ONE story.
- ❌ Asserting on cosmetic details (colors, exact text). Story-level outcomes only.
- ❌ Using lorem ipsum. Realistic-looking data catches real bugs (max-length validation, regex on email field, etc.).
- ❌ Manufacturing test data that bypasses validation (e.g., directly inserting via SQL). Walk the user flow.
- ❌ Skipping the READ side of CREATE → READ. The whole point is the round-trip; one-way "submit success" isn't enough.
