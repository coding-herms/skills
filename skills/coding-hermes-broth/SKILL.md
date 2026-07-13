---
name: coding-hermes-broth
description: >-
  The fleet supervisor — monitors all projects, heals broken foremen,
  rebalances schedules and models, detects orphans, and produces HTML
  fleet reports. The "broth" that holds all the stars together.
version: 1.0.0
metadata:
  hermes:
    tags: [coding-hermes, broth, supervisor, fleet, maintenance]
---

# Coding Hermes Broth — Fleet Supervisor

The broth is the fleet-level maintenance layer. It watches every star (foreman),
heals broken ones, rebalances models and schedules, detects orphan projects,
and produces HTML fleet reports. Runs every few hours — it's the foundation
that keeps the fleet alive.

**The broth does NOT write code. It does NOT spawn workers.** It monitors,
heals, rebalances, and reports.

## The Three-Tier Model

```
You (Strategy) → Broth (Fleet) → Stars (Projects) → Workers (Code)
```

| Tier | Runs | Model Type | Purpose |
|------|------|------------|---------|
| You | When you want | — | Strategy, architecture, new capabilities |
| **Broth** | Every 4 hours | Reasoning model | Heal, audit, rebalance, report |
| Star | Per-project schedule | Reasoning model (PAYG) | Scan tasks, compile prompts, spawn workers |
| Worker | Spawned per task | Coding model (prepaid) | Write code, run tests, commit |

### Escalation Path

```
Worker → Star → Broth → You
```

Each level must exhaust its own capabilities before escalating:
- **Worker** hits a compile error → fixes it. Doesn't ask the star.
- **Star** sees a rate limit → switches providers. Doesn't wait for broth.
- **Broth** detects 3 stars stalled → heals them. Doesn't bother you.
- **You** only hear about structural problems — new capabilities needed,
  architecture decisions, fleet-wide model migrations.

---

## The Broth Loop

### Phase 0 — Auto-Heal (run FIRST, before any audit)

Fix without asking. Use the `cronjob` tool for state transitions.
Never edit `jobs.json` directly — it bypasses scheduler state.

#### 0A. Schema Auto-Fix

Compare every cron job against the canonical schema in `coding-hermes-cron-schema`.
Fix:
- `kind: "every"` → convert to `kind: "interval"`, parse minutes from display
- Missing `schedule.expr` on cron-kind jobs → copy from `schedule.display`
- Missing `schedule.minutes` on interval-kind jobs → parse from display
- `schedule` is a string (legacy format) → parse into dict `{"kind": "...", ...}`
- Mismatched `schedule_display` (top-level) vs `schedule.display` (nested) → sync

**Why auto-heal:** Missing `expr` crashes the ENTIRE scheduler. String-format
schedules are silently broken. These are JSON schema bugs, not policy decisions.

#### 0B. State Auto-Fix

For every star cron job:
- `state: "completed"` + `enabled: true` → set `state: "scheduled"`
- `state: "paused"` + no `paused_reason` → set `state: "scheduled"`, `enabled: true`
- `state: "paused"` + has `paused_reason` → respect it (user intentionally paused)

#### 0C. Stale/Overdue Stars

For every enabled star:
- If `last_run_at` is null/never OR staleness > 2× expected interval:
  - Set `next_run_at` to 5 minutes in the PAST (forces fire on next tick)

#### 0D. Orphan Detection

Scan your projects directory for directories with `.coding-hermes/tasks.md`
that have NO matching star cron (matching by workdir). Auto-create a star:
- Model: the configured star model/provider (from config)
- Schedule: every 120m (if tasks pending) or every 360m (if idle)
- Skills: `coding-hermes-star`, `coding-hermes-cron-schema`
- Set `next_run_at` to 5 min in the past

Skip git worktrees (where `.git` is a file, not a directory).

#### 0E. Duplicate Star Detection

When two stars share a workdir, pause the legacy one (the one WITHOUT
`coding-hermes-cron-schema` in skills). Set `enabled: false`,
`state: "paused"`, with a reason explaining which star replaced it.
Never auto-delete.

#### 0F. Missing Task Board Detection

Star exists but `.coding-hermes/tasks.md` is missing → create it with bootstrap tasks:
```markdown
# Task Board — <project>
## [ ] INIT — Set up memory namespace, quality gates, code graph
## [ ] SPEC — Audit specs vs implementation, queue gap tasks
## [ ] DOC — Verify documentation matches current code
## [ ] CI — Check CI pipeline health, fix failing jobs
```

### Phase 2 — Fleet Audit

#### 2A. Firing Report

For every star: did it fire on time? Compare `last_run_at` against expected
schedule. Grade: ✅ (on time), 🟡 (slightly late), 🔴 (missed/never/disabled).

#### 2B. Errors

For every 🔴 job: check `last_error`, `last_status`, `state`. Common causes:
- `state: completed` → job finished, won't run again (fix in Phase 0B)
- `enabled: false` + no `paused_reason` → accidentally disabled (re-enable)

#### 2C. Results

For stars that DID fire: git activity, task progress (`[x]` vs `[ ]` count),
CI status, error rate. Grade: 🟢 (progress), 🟡 (running but unclear), 🔴 (failing).

#### 2D. Rebalance

Based on pending task count and recent commits:
- **Speed up:** 3+ pending tasks + active commits → faster schedule
- **Slow down:** 0 pending + 0 commits in 48h → slower schedule

**Never rebalance model/provider on user-set projects.**

### Phase 5 — Fleet Report

Every broth run produces an HTML fleet report. Write a self-contained
Python script that:
1. Reads cron state
2. Computes per-project health metrics
3. Generates HTML with fleet table, metrics bar, what-got-done section

Format: link + MEDIA + summary. Use dark theme, mobile-first.
Use `<pre class="mermaid">` for Mermaid diagrams (NOT `<div>`).

### Phase 6 — CI Check

Batch-fetch CI status for all projects with GitHub remotes via `gh run list`.
Classify: passing / failing / queued / no_ci / billing-blocked.
Report failures in the HTML report.

---

## Tool Health Monitoring

Every broth tick, sample agent logs for infrastructure tool usage:
- Quality gate calls (guard + judge)
- Memory calls (recall + remember)
- Code graph calls (impact analysis)
- Solution cache calls (submit + discover)

Report in HTML. Degraded tool health = stars losing memory and quality.

---

## Star Model Architecture

**The golden rule:** Stars must ALWAYS be reachable. If a star's provider
locks, the entire project goes dark — no rebalancing, no worker spawning,
nothing. Stars stay on PAYG because you can always add credit to unblock them.
Workers burn prepaid buckets getting coding done. When a bucket locks, the
star (still alive on PAYG) switches workers to the next available plan.

| Role | Type | Billing | Why |
|------|------|---------|-----|
| **Star (active)** | Reasoning model | **PAYG** | Must always load. Always unblockable. |
| Star (idle) | Reasoning model | **PAYG** | Same rule — presence non-negotiable. |
| **Broth** | Reasoning model | Flat-rate | Light work. If bucket locks, fleet healing pauses but stars keep running. |
| **Worker (heavy)** | Coding model | **Prepaid** | Burn quota. Star switches if bucket locks. |
| **Worker (light)** | Coding model | **Prepaid** | Bug fixes. Same bucket-switching logic. |
| **Worker (budget)** | Coding model | Flat-rate | Idle project, minor work. |

**Critical rules:**
- Stars NEVER use prepaid providers. Locked prepaid = star can't load = project dead.
- Worker bucket exhaustion is normal — star auto-switches. This is by design.
- The broth must NEVER change a star's provider away from PAYG.
- Speed and worker model rebalancing are fine — provider is not.

---

## Infrastructure Crons (Memory & Health)

The fleet has infrastructure crons that maintain long-term memory and
operational health. These are load-bearing — if they fail, stars lose
context across ticks.

| Cron Type | Skills | Agent? | Broth Treatment |
|-----------|--------|--------|-----------------|
| Memory sync | memory skill or no_agent script | No (script) | **Never pause or rebalance** |
| Data loaders | Project-specific | Optional | **Never rebalance model** |
| Health check / watchdog | None | No (script) | **Never touch** — system-level |

**Why they matter:** When a star loads context before coding, it queries
project memory. If the sync cron hasn't run, the namespace is stale or
empty — the star has no memory of past decisions, pitfalls, or patterns.

---

## Delivery Targeting

**Always use explicit delivery.** Auto-detection can strip thread IDs
in group chats — star reports silently land in the wrong channel.
Every cron MUST use an explicit delivery target: `platform:chat_id:thread_id`.

---

## Pitfalls

- **Don't run stars one at a time to fix the fleet.** When many stars are
  stale, run the broth. Manual star runs bypass the self-heal pipeline.
- **Don't change a star's model to a coding model.** Stars inspect and
  delegate — they use reasoning models. Coding models are for workers only.
- **Stars on prepaid is a failure mode.** If a star's prepaid bucket locks,
  it can't load → can't rebalance → can't spawn workers → project dead.
- **Worker bucket exhaustion is normal.** When a worker's prepaid bucket
  locks, the star auto-switches. This is by design, not a bug.
- **Phase 0 should NOT change schedules** — only fix schema validity and
  zombie state. Phase 2D handles speed changes.
- **LLM-based enforcement is unreliable — use scripts for deterministic rules.**
  If the broth repeatedly ignores a rule (e.g., "never change star models"),
  replace that logic with a Python script running as a `no_agent` cron.
- **`repeat` field is polymorphic** — can be `int` or `dict`. Dict format
  is standard: `{"times": null, "completed": N}`. Always write in dict format.
- **Cron skill-not-found is a silent failure.** A cron referencing a skill
  that doesn't exist on disk logs a warning every tick but the scheduler
  marks it `ok`. Validate every skill reference against the filesystem.
- **Mermaid diagrams: use `<pre class="mermaid">`, not `<div>`.** The div
  tag fails to render. Always `<pre>` for Mermaid blocks.
- **Schedule auto-detection vs. explicit config.** When the broth runs
  `cronjob list`, the jobs are JSON. Parse programmatically — don't rely
  on regex or embedded snippets for field extraction.
- **Don't deliver silently.** Every broth run must produce output, even if
  the fleet is healthy. "Fleet healthy, 0 issues" is a valid report.
  Silence is indistinguishable from a crash.
- **Schedule schema: `kind: "every"` is legacy.** Convert to `kind: "interval"`
  + parse minutes from display on every auto-heal pass.
