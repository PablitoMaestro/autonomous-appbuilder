# `.ai/agent-logs/` — what each file is for

Agent-generated audit trail. Every `/00-build-app` run drops files here. **If you want to read what happened in a run, start at `run-summary-<...>.md`** — that's the human-readable timeline. The rest is supporting detail.

---

## File types

| File pattern | What it is | When | Audience |
|--------------|-----------|------|----------|
| **`run-summary-<YYYY-MM-DD-HHMM>.md`** | Chronological orchestration timeline. ASCII tree, one line per dispatch/return, phase headers. | Every run, written by main agent | **Start here.** Human reading post-run. |
| `autonomous-decisions-<date>.md` | Load-bearing decisions with WHY. | Autonomous-mode runs only (`gates=none`) | Auditor checking "why did the agent pick X?" |
| `findings-<date>-<plan-slug>.md` | Per-cohort engineering findings from `05-plan-executor`. Decision-grade summaries. | One per `05-plan-executor` invocation | Engineer reviewing what shipped per wave |
| `review-findings-<date>.md` | Per-task review verdicts from `06-plan-reviewer`. | Only if `review=on` or `review=full` | Escalation, post-review triage |
| `smoke-<date>.md` + `smoke-*.png` | 4-gate smoke test report + screenshots. | Phase 7 (every run) | Pre-launch sanity check |
| `hooks-<date>.jsonl` | Raw event stream — every Skill / Agent / canon-edit / git-op tool call. | Every run, written by hook layer | Machine. Diagnostics, audit pipeline. **Gitignored.** |

---

## Reading a run cold

You see `run-summary-2026-05-04-1342.md` and want to know what happened.

1. **Open `run-summary-<...>.md`** — read top-to-bottom. The ASCII tree shows phases → waves → cohorts → tasks chronologically with timestamps and PR numbers.
2. **Decision unclear?** Open `autonomous-decisions-<date>.md` and grep for the topic (e.g., "stack pick", "design pick"). Each entry has `Why:` and `How to apply:`.
3. **Engineering detail in a cohort?** Open `findings-<date>-<plan-slug>.md` and find the wave block. Cohort findings have task-level summaries.
4. **Smoke result?** Open `smoke-<date>.md` — 4 gates with pass/fail and links to screenshots.
5. **Review pass ran?** Open `review-findings-<date>.md` — per-task verdict tables.
6. **Need machine-grade audit?** Read `hooks-<date>.jsonl` — every tool call timestamped. Use `jq` to filter.

---

## Why so many files

Each file has a different lifecycle, audience, and writer:

- `run-summary` is the **narrative** — single-author (main agent), chronological, scannable.
- `autonomous-decisions` captures **why** at the moment of decision; sparse and stable.
- `findings` is the **engineering layer** — multi-cohort, technical, decision-grade.
- `smoke` is **evidence** — 4 specific gates, structured, pre-launch artifact.
- `review-findings` is the **audit layer** — per-task verdicts from a separate phase.
- `hooks-*.jsonl` is the **firehose** — raw event stream for machine consumption.

If you only ever read one: `run-summary-<...>.md`.

---

## What's NOT here

- Per-task implementation details — those live in `master-plan-points/<id>.md` (the cards).
- Code review comments — those live in PR descriptions on GitHub.
- Production deploy logs — those live in your hosting provider's dashboard / `docs/runbook.md`.
