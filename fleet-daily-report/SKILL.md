---
name: fleet-daily-report
description: Generate a deep, interactive, offline-readable HTML fleet report — one subagent per project for maximum depth, then compile into a navigable dashboard with clickable drill-down cards showing full task boards, git history, CI status, DuckBrain entries, risk assessments, and foreman tick logs
version: 2.0.0
category: coding-hermes
---

# Fleet Daily Report — Deep Edition

Generate a self-contained HTML report designed for offline reading. Bane downloads this, reads it for hours, and writes responses in Telegram that send when connectivity returns. The fleet keeps working in the background.

## Design Principles

- **One subagent per project** — each agent dives deep into a single project: full board read, git log, CI history, DuckBrain recall, tick log analysis. No surface-level summaries.
- **Clickable drill-down** — every project card expands to show ALL detail. Nothing hidden behind pagination.
- **Self-contained** — no external resources, no JS frameworks. Pure HTML+CSS. Works offline.
- **Rich formatting** — code blocks, diff previews, commit messages, CI URLs, model assignments. Formatted for reading, not scanning.
- **Cross-linking** — project cards link to related projects (shared repos, shared DuckBrain namespaces).

## Data Gathering — One Subagent Per Project

For EACH of the ~14 active projects (idle projects get lighter treatment — see below):

### Active Projects (has real pending tasks)
Dedicated subagent gathers:
1. **Board:** Read full `.coding-hermes/tasks.md` — list every pending task with model assignment, priority, complexity, dependencies, tags
2. **GitReins:** Read GitReins board if available — count pending, list discrepancies with coding-hermes board
3. **Git log:** `git log --since="48 hours ago" --oneline --stat` — all commits with files changed
4. **Foreman ticks:** Last 5 ticks from scheduler API — status, outcome, commits, model used, duration, spawned workers
5. **DuckBrain:** `duckbrain recall keyPrefix="/projects/[name]" limit=20` — recent context entries
6. **CI:** `gh run list --limit 10` — full CI history with status, duration, URL
7. **Risks:** Zombie detection (idle ticks > 20), blocked tasks, cooldown stuck, model mismatch, skill staleness
8. **NEVER-DONE:** Results of most recent audit — gaps found, tasks created
9. **Recent work narrative:** 2-3 sentence summary of what was built/fixed in last 48h (from commits + board changes)

### Idle Projects (only NEVER-DONE remains)
Lighter treatment — one subagent covering all idle projects:
1. Confirmation that board has only NEVER-DONE
2. Last tick timestamp
3. Cooldown status
4. Any CI failures
5. NEVER-DONE present? If missing, add it

## HTML Structure

The report is a single `.html` file with:

### Top Section — Fleet Dashboard
- Generation timestamp, daemon health
- Fleet stats: total projects, active, idle, total real pending tasks, commits in 48h, CI pass/fail ratio
- Quick-nav: links to each project card (anchor links)

### Problem Dashboard
- All risks sorted by severity: zombie projects, CI failures, blocked tasks, board discrepancies, missing NEVER-DONE
- Each risk item links to the relevant project card

### Active Project Cards — Deep Detail (`<details>` open by default)
Each card shows:
```
┌─────────────────────────────────────────────┐
│ 🟢 PROJECT NAME — 5 pending — 900s — V4 Flash      │
├─────────────────────────────────────────────┤
│ ▼ Task Board (full)                                  │
│   ID | Task | Pri | Cpx | Model | Deps | Tags       │
│   ...all rows...                                             │
│                                                              │
│ ▼ Recent Commits (48h)                              │
│   abc1234 fix: broken auth middleware              │
│   def5678 feat: add rate limiting                          │
│   ...with file stats...                                      │
│                                                              │
│ ▼ Foreman Ticks (last 5)                              │
│   Jul 24 14:30 | completed | 3 commits | 245s      │
│   ...                                                                 │
│                                                              │
│ ▼ CI Status                                                │
│   ✅ build (12m)  ❌ test (3m)  ✅ lint (2m)            │
│   ...                                                                 │
│                                                              │
│ ▼ DuckBrain Context                                      │
│   /projects/name/spec-001 — last updated ...           │
│   ...                                                                 │
│                                                              │
│ ▼ Risk Assessment                                         │
│   ⚠️ 3 blocked tasks                                 │
│   ⚠️ cooldown stuck at 1800s (should be 900s)           │
│   ✅ no zombie detected                                        │
│                                                              │
│ ▼ What Got Done (48h narrative)                   │
│   Built auth middleware (3 endpoints, 12 tests).    │
│   Fixed CI flakiness in integration suite.               │
└─────────────────────────────────────────────────────────┘
```

### Idle Project Summary — Compact Table
```
| Project | Last Tick | Cooldown | NEVER-DONE? | CI |
|---------|-----------|----------|-------------|-----|
| bunker  | Jul 23    | 43200s   | ✅          | ✅ |
```

### Footer
- Report generation metadata
- Link to `coding-hermes-model-router` for model profiles reference
- Total subagents dispatched, total data gathered

## Delivery

1. Save HTML to `~/.hermes/reports/fleet-YYYY-MM-DD-HHMM.html`
2. Deliver `MEDIA:[path]` so it renders inline in Telegram
3. Text summary:
   ```
   📊 Fleet Report — [date] [time]
   🔴 Active: [N] | 🔵 Idle: [N] | ⚠️ Risks: [N]
   📝 [N] commits in 48h | 🧪 CI: [pass]/[fail]
   ```

## Scheduling

Cron: `0 8,14,20 * * *` — three times daily.
Each run is independent — a complete offline-readable snapshot.
