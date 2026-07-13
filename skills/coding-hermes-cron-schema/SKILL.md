---
name: coding-hermes-cron-schema
description: >-
  Canonical cron shape for the Coding Hermes fleet. Shared by broth and
  stars — the single source of truth for schedule schema, model/provider
  patterns, skills arrays, workdir conventions, cron creation, and
  infrastructure cron types. When a star is broken, heal it against this spec.
version: 1.0.0
metadata:
  hermes:
    tags: [coding-hermes, cron, schema]
---

# Coding Hermes Cron Schema — Canonical Shape

This skill defines the valid shape of every coding-hermes cron job.
The broth loads it to heal broken stars. Stars load it to validate
their own config. One source, zero drift.

---

## Cron Categories

### 1. Star Crons (code-producing)

Per-project autonomous coding loops. Load `coding-hermes-star` and
`coding-hermes-cron-schema`. Model: reasoning model on PAYG provider
(configured in config skill). Skills: `["coding-hermes-star", "coding-hermes-cron-schema"]`.

Required toolsets: `["terminal","file","web","search","skills","memory"]`
(no `delegation`, no `cronjob`).

### 2. Infrastructure Crons (memory & health)

Maintain the fleet's long-term memory and operational health.
**Load-bearing — if they fail, stars lose context across ticks.**

| Cron Type | Skills | Agent? | Broth Treatment |
|-----------|--------|--------|-----------------|
| Memory sync | memory skill or no_agent script | No (script) | **Never pause or rebalance** |
| Data loaders | Project-specific | Optional | **Never rebalance model** |
| Health check / watchdog | None | No (script) | **Never touch** |

### 3. Scheduler Trigger Cron

In the target architecture, static star crons are replaced by ONE trigger
cron that asks the scheduler daemon what to run each tick:

```json
{
  "name": "scheduler-trigger",
  "state": "scheduled",
  "enabled": true,
  "schedule": {"kind": "cron", "display": "* * * * *", "expr": "* * * * *"},
  "repeat": {"times": null, "completed": 0},
  "skills": [],
  "no_agent": true,
  "script": "trigger-scheduler.py",
  "model": null,
  "provider": null,
  "workdir": null,
  "deliver": "origin"
}
```

**Key differences from stars:** `no_agent: true`, `skills: []`, fires every
60 seconds, script-only (pings scheduler HTTP endpoint). The script does NOT
contain scheduling logic — it asks the scheduler daemon "what should run?"
The daemon computes urgency, packs the weight budget, and spawns stars.

---

## Valid Star Cron Shape

```json
{
  "name": "<project>-star",
  "state": "scheduled",
  "enabled": true,
  "schedule": {"kind": "cron", "display": "*/15 * * * *", "expr": "*/15 * * * *"},
  "schedule_display": "*/15 * * * *",
  "repeat": {"times": null, "completed": 0},
  "skills": ["coding-hermes-star", "coding-hermes-cron-schema"],
  "enabled_toolsets": ["terminal","file","web","search","skills","memory"],
  "model": "<your-star-model>",
  "provider": "<your-payg-provider>",
  "workdir": "/home/you/<project>",
  "deliver": ["<platform>:<chat_id>:<topic_id>"],
  "last_status": "ok"
}
```

### Schedule Schema

| Kind | Required Fields | Valid Values |
|------|----------------|--------------|
| `cron` | `kind`, `display`, `expr` | `*/15 * * * *`, `0 */2 * * *` |
| `interval` | `kind`, `display`, `minutes` | `minutes: 120`, `minutes: 360` |
| ❌ `every` | INVALID — convert to `interval` | Never use `kind: "every"` |

**Hard rules:**
- Cron-kind MUST have `schedule.expr`. Missing = scheduler crashes.
- `schedule_display` (top-level) must match `schedule.display` (nested).
- `kind: "every"` is legacy. Convert to `kind: "interval"` + parse minutes from display.
- The `every Nm` CLI format is valid for creation. Internally it becomes `kind: "interval", minutes: N`.
- **The schedule in the example is just that — an example.** Preserve each star's existing frequency when converting formats.

### State Machine

| State | Meaning | Scheduler Behavior |
|-------|---------|-------------------|
| `scheduled` + `enabled: true` | Active, fires on schedule | ✅ Fires |
| `completed` + `enabled: true` | Finished, won't run again | ❌ Never fires — BUG |
| `paused` + has `paused_reason` | Intentionally stopped | ❌ Skipped — respect it |
| `paused` + no `paused_reason` | Accidentally paused | ❌ Skipped — FIX |

---

## Model-Provider Patterns

**Stars MUST be on PAYG.** If a star's provider locks (prepaid plans DO lock
when exhausted), the star can't load → can't rebalance → can't spawn workers
→ project dead. PAYG is always rescuable: add credit, star comes back online.

| Role | Model Type | Billing | When |
|------|------------|---------|------|
| Star (active) | Reasoning | **PAYG** | Always — presence non-negotiable |
| Star (idle) | Reasoning | **PAYG** | Always — same rule |
| Worker (heavy) | Coding | Prepaid | Complex tasks, new features |
| Worker (light) | Coding | Prepaid | Bug fixes, small changes |
| Worker (budget) | Coding | Flat-rate | Idle project, minor work |
| Infrastructure (no_agent) | — | — | No model — script only |

**Critical boundaries:**
- Stars NEVER use prepaid providers. Prepaid locks = star can't load = project dead.
- Worker bucket exhaustion is normal — star auto-switches workers. By design.
- NEVER set model/provider on `no_agent` jobs. Burns tokens on script-only runs.
- ID stars by `skills` array containing `coding-hermes-star`, not by name keyword.

---

## Toolset Enforcement (Structural)

Stars MUST exclude `delegation` and `cronjob` toolsets. Skill text saying
"never use delegate_task" is ignored by the LLM if the tool is available.
Stripping the toolset is the ONLY reliable enforcement.

| Toolset | Star? | Reason |
|---------|-------|--------|
| terminal | ✅ | `hermes chat -q` for worker spawn, git, builds |
| file | ✅ | Read/write code, board, configs |
| web | ✅ | Research, doc lookup |
| search | ✅ | Grep codebase, session search |
| skills | ✅ | Load skills |
| memory | ✅ | Memory read/write |
| delegation | ❌ | Workers must use independent sessions, not inherit star's provider |
| cronjob | ❌ | Star self-modifying schedule causes drift |

Workers spawned via `hermes chat -q` inherit the default toolset (including
delegation) — they CAN use subagents. This restriction is star-only.

---

## Infrastructure Cron Shape

```json
{
  "name": "Sync Memory — hourly",
  "state": "scheduled",
  "enabled": true,
  "schedule": {"kind": "cron", "display": "0 * * * *", "expr": "0 * * * *"},
  "repeat": {"times": null, "completed": 0},
  "skills": [],
  "no_agent": true,
  "script": "memory-sync.sh",
  "model": null,
  "provider": null,
  "workdir": null,
  "deliver": "origin"
}
```

**Key differences from stars:** `no_agent: true`, `skills: []`, `script` path
instead of prompt, `model: null` / `provider: null`.

---

## Auto-Heal Classification

When the broth heals the fleet, classify each job:

```python
if "coding-hermes-star" in job.get("skills", []):
    # Star — safe to rebalance speed (not model/provider)
    heal_schema(job)
    heal_state(job)
    rebalance_if_stale(job)
elif job.get("no_agent") and job.get("script"):
    if "trigger-scheduler" in (job.get("script") or ""):
        # Scheduler trigger — critical, never pause, flag if failing
        heal_schema(job)
        # NEVER touch model, provider, speed, or state
    else:
        # Other infrastructure — only fix schema validity
        heal_schema(job)
        # NEVER touch model, provider, speed, or state
else:
    # Unknown — audit but don't modify
```

---

## Delivery Targeting

Always use explicit delivery. Auto-detection can strip thread IDs in group
chats — star reports silently land in the wrong channel. Every star cron
MUST use an explicit delivery target: `platform:chat_id:thread_id`.

---

## Why Agent Cron Is Not Linux Cron

Agent-based cron is fundamentally different from Linux cron (`cronie`/`crontab`).
Understanding this is essential to debugging why stars silently skip ticks
or zombie-lock.

| Linux Cron | Agent Cron |
|-----------|------------|
| Forks a process, forgets it | Spawns a full LLM agent session, waits for completion |
| Job takes 1ms or 1hr — doesn't matter | Job takes 30min → blocks next 2-3 ticks |
| All state is on disk | Running state may be in-memory |
| One broken job = one broken job | One broken job schema = entire scheduler can crash |
| Delivery is stdout → mail | Delivery is a separate call AFTER agent finishes |
| Survives process restart | In-memory state lost on restart → zombie locks cleared |

**Critical implications for stars:**
- A star at 15m that takes 30m per tick will run at most once every 30m —
  the intermediate ticks get skipped because the job is "already running."
- In-memory run state can zombie-lock: if a process hangs, the job is
  permanently dead until restart.
- This is why the scheduler daemon architecture matters — the Go scheduler
  uses OS process tracking, SQLite for durable state, and per-tick timeouts.

---

## Pitfalls

- **Agent updates can silently freeze non-star crons.** After agent updates,
  infrastructure crons (memory sync, data loaders) can stop firing without
  error. Verify `last_run_at` timestamps after every update.
- **Scheduler trigger cron is load-bearing.** If it fails, no star runs —
  the entire fleet goes dark. Treat it as critical infrastructure.
- **Thread ID stripping.** Even with explicit delivery targets, the scheduler
  may drop thread IDs for group topics. Re-create the cron fresh rather than
  updating an existing job's delivery target.
- **Don't re-run broken crons repeatedly — diagnose and fix once.** Repeated
  manual runs with no output burn attention with no outcome. Diagnose root
  cause (check `last_status`, `last_delivery_error`, `repeat` format,
  provider validity), apply fix, run once to verify.
- **`repeat` field is polymorphic** — can be `int` or `dict`. Dict format is
  standard: `{"times": null, "completed": N}`. Always write in dict format.
- **Cron skill-not-found is a silent failure.** A cron referencing a skill
  that doesn't exist on disk logs a warning every tick but the scheduler
  marks it `ok`. Validate every skill reference against the filesystem.
- **Foreman cron prompts go stale.** The prompt baked into each star's cron
  config does NOT auto-update when skill text changes. Keep cron prompts
  minimal — a bootstrap line that loads the skills, not a full inline workflow.
