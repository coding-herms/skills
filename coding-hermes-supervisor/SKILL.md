# coding-hermes-supervisor

**Status:** Active  
**Category:** orchestration  
**Layer:** Supervisor (Phase 5)

---

## Purpose

The supervisor provides FLEET-LEVEL oversight. Unlike the foreman (one project, one tick), the supervisor looks across ALL projects and makes strategic decisions.

---

## Trigger

The supervisor runs:
- When the user asks "how is the fleet doing?"
- When `/fleet status` is invoked
- Periodically (every 5-10 foreman ticks) for health checks
- After major events (deployments, outages, model changes)

---

## Supervisor Responsibilities

### 1. Fleet Health Check

Query the scheduler API:

```bash
curl -s http://localhost:9090/api/v1/status
curl -s http://localhost:9090/api/v1/projects
```

Check for:
- Projects that haven't run in >24h (starving)
- Projects with >3 consecutive failures (broken)
- High error rates (>50% ticks failing)
- Concurrency saturation (all 8 slots always full)

### 2. Anti-Starvation Detection

For each enabled project, check: `time_since_last_tick`.

If `time_since_last_tick > 2 × interval`:
- Project may be starved.
- Increase its priority by 1 (temporarily).
- Note in DuckBrain: `/fleet/events/` with level=warn.

### 3. Failure Pattern Detection

If a project has failed 3+ consecutive ticks:
- It's likely broken — something changed (API key expired, repo corrupted, dependency broke).
- PAUSE the project automatically: `curl -X POST http://localhost:9090/api/v1/projects/<name>/pause`
- Notify the user: "⚠️ <project> has been paused after 3 consecutive failures."
- Write finding to DuckBrain: `/fleet/findings/<project>-broken-<date>`

### 4. Model Provider Health

Check if the configured models are actually working:

```bash
# Simple health probe — does the model respond?
hermes chat -q "Say 'ok' and nothing else." -m <model> --provider <provider> --max-turns 1
```

If the model fails 3 times:
- Flag in DuckBrain: `/fleet/config/model-status/<model>` → `{"healthy": false}`
- Suggest switching to a fallback model.

### 5. Cost Monitoring

Track per-project and fleet-wide costs:

```
Query scheduler.db:
  SELECT project_name, SUM(cost_usd), COUNT(*) 
  FROM ticks 
  WHERE status = 'completed' 
  GROUP BY project_name
```

Report:
- Top 5 most expensive projects
- Projects exceeding daily cost budget
- Total fleet cost

### 6. Fleet Rebalancing

When user runs `/fleet rebalance`:
1. Compute how long each project has waited.
2. Suggest priority adjustments.
3. Apply them via API: `PUT /api/v1/projects/<name> {"priority": <new>}`
4. Force evaluate: `POST /api/v1/evaluate`

---

## Supervisor Loop

```
Every 10 minutes OR on user request:

1. Health check → status from scheduler
2. Starvation scan → check last_tick for all projects
3. Failure scan → check consecutive failures
4. Model health → probe configured models
5. Cost report → query tick costs
6. Dashboard refresh → verify dashboard shows current data
7. DuckBrain write → /fleet/supervisor/latest-tick
```

---

## Escalation Path

| Severity | Condition | Action |
|----------|-----------|--------|
| **CRITICAL** | Scheduler down | Alert user immediately. Try restart. |
| **HIGH** | >3 projects failing | Pause failing projects. Alert user. |
| **MEDIUM** | >2 projects starving | Auto-rebalance priorities. |
| **LOW** | 1 project failing | Note in DuckBrain. Monitor. |
| **INFO** | Fleet healthy | Log. Continue. |

---

## Report Format

```
## Supervisor Report — <timestamp>

### Fleet Health
- Active projects: <N>/<total>
- Running ticks: <N>/<max>
- Uptime: <duration>
- Recent outcomes: <completed>/<failed>/<timeout>

### Starvation Watch
- <project>: last run <time> ago (interval: <interval>) ⚠️
- <project>: healthy ✓

### Failures
- <project>: <N> consecutive failures — PAUSED
- No failures detected ✓

### Costs
- Fleet total: $<amount>
- Top spender: <project> ($<amount>)

### Action Items
- [ ] <action>
```
