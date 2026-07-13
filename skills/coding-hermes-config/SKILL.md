---
name: coding-hermes-config
description: >-
  Guided configuration for the Coding Hermes autonomous fleet. Load this first —
  it walks you through every configurable value: which LLM providers to use,
  which models for each role, where to deliver reports, your scheduling budget,
  and DuckBrain setup. Produces a complete config map.
version: 1.0.0
metadata:
  hermes:
    tags: [coding-hermes, config, setup, fleet]
---

# Coding Hermes Config — Fleet Configuration

Load this skill FIRST. It will ask you for every piece of configuration data
the fleet needs, explain WHY each value matters, and produce a config map
you can reference from the broth and star skills.

**Do NOT skip this.** The broth and star skills assume these values exist.
Without them, foremen crash on missing API keys, report to nowhere, or burn
the wrong billing account.

## What You Need Before Starting

- API keys for your LLM providers (at minimum: one PAYG provider for foremen,
  one or more prepaid/flat-rate providers for workers)
- A delivery target (Telegram group + topic ID, Discord channel, or "local")
- Path to your project directory
- Rough idea of how many projects you're running and how fast you want them

---

## Phase 1 — Provider Configuration

### Why It Matters

The fleet has a **golden rule**: foremen must ALWAYS be reachable.
If a foreman's provider locks (prepaid plans DO lock when exhausted),
the foreman can't load → can't rebalance → can't spawn workers →
project dead until the billing cycle resets.

**Foremen stay on PAYG** — you can always add credit to unblock them.
**Workers burn prepaid/flat-rate buckets** — when one locks, the foreman
auto-switches to the next available plan.

### Ask the User

Go through these one at a time with `clarify`. Build the provider palette
as you go. Store each answer before asking the next.

**Step 1: Broth (Supervisor) Provider**

The broth does light work — auditing, rebalancing, reports — every few hours.
A flat-rate or cheap provider works well. It does NOT need the same power
as the stars.

Ask:
> For the BROTH (fleet supervisor): which provider and model should it use?
> This role does light work (auditing, rebalancing, HTML reports).
> A flat-rate or budget model works well here.

Record: `broth_provider`, `broth_model`, `broth_api_key_env`, `broth_billing`

**Step 2: Star (Foreman) Provider**

Each star orchestrates ONE project. It inspects, plans, and spawns workers.
It MUST be reachable at all times. This MUST be PAYG.

Ask:
> For STARS (per-project foremen): which provider and model?
> Stars must always be reachable — use a PAYG provider.
> If this provider locks, all projects go dark.

Record: `star_provider`, `star_model`, `star_api_key_env`

**Step 3: Worker Providers (at least 2)**

Workers do the heavy coding. They burn quota. When one bucket locks,
the star switches to the next. List at least two — one for heavy tasks
(new features, refactors) and one for light tasks (bug fixes, small changes).

Ask:
> For WORKERS (coding agents): list at least two model/provider combos.
> Workers burn prepaid/flat-rate quota. When one locks, stars switch to the next.
> Heavy worker (features, refactors): which model/provider?
> Light worker (bug fixes, small changes): which model/provider?
> Budget worker (idle projects, minor work — optional): which model/provider?

Record each: `worker_heavy_provider`, `worker_heavy_model`, etc.

### Provider Palette (Compile After Answers)

| Role | Provider | Model | Billing | Can Lock? |
|------|----------|-------|---------|-----------|
| Broth | `<answer>` | `<answer>` | `<answer>` | If flat-rate: yes |
| Star | `<answer>` | `<answer>` | **PAYG** | No — add credit |
| Worker (heavy) | `<answer>` | `<answer>` | Prepaid | Yes — auto-switch |
| Worker (light) | `<answer>` | `<answer>` | Prepaid | Yes — auto-switch |
| Worker (budget) | `<answer>` | `<answer>` | Flat-rate | Yes — auto-switch |

---

## Phase 2 — Delivery Configuration

### Why It Matters

Every star tick produces output: task progress, commits pushed, errors
encountered. Without a delivery target, output goes nowhere and you fly blind.
Stars that run silently are indistinguishable from crashed agents.

### Ask the User

**Step 4: Broth Delivery Target**

Ask:
> Where should the broth send fleet reports?
> Platform? Chat/Channel ID? Thread/Topic ID (if applicable)?

Record: `broth_delivery` as `platform:chat_id:thread_id` or `local`

**Step 5: Per-Project Delivery Targets**

For each project, ask:
> Where should <project-name> star output go?
> Use a separate thread/channel per project.

Record each as `star_delivery_<project>`

**Step 6: Delivery Format**

Ask:
> Delivery format preference?
> Options: Full reports, Summary only, Silent unless error

Record: `delivery_format`

---

## Phase 3 — Scheduling Configuration

### Why It Matters

The scheduler uses a **weight-budget** model. Total budget = 100 units.
Each project has a weight (how much budget it uses per tick) and a
priority (how aggressively the scheduler tries to run it).

Getting this wrong means some projects starve while others hog resources.

### Ask the User

**Step 7: Budget & Concurrency**

Ask:
> Total weight budget? (default: 100)
> Max concurrent running stars? (default: 8)

Record: `weight_budget`, `max_concurrent`

**Step 8: Interval Range**

Ask:
> Fastest tick interval — priority 1? (default: 20m)
> Slowest tick interval — priority N? (default: 24h)
> Number of priority levels? (default: 10)

Record: `min_interval`, `max_interval`, `num_levels`

**Step 9: Per-Project Weights**

For each project, help the user classify it into an archetype:

| Archetype | Weight | Priority | Behavior |
|-----------|--------|----------|----------|
| Must-run lightweight | 3–8 | 9–10 | Runs every tick, costs almost nothing |
| Heavy lifter | 40–70 | 8–10 | Critical AND expensive, dominates budget |
| Background filler | 3–5 | 1–3 | Rarely urgent, always finds a slot |
| Hog | 70–90 | 7–10 | Starves the fleet — reduce weight or accept solo |
| Idle project | 15–25 | 1–2 | Almost never scheduled, decay forces ~daily |

Ask for each project:
> <project-name>: what archetype? Weight? Priority?

Record each.

---

## Phase 4 — DuckBrain Configuration (Optional)

### Ask the User

**Step 10: DuckBrain Setup**

Ask:
> Enable DuckBrain for long-term fleet memory? (recommended)
> If yes: MCP server command? Namespace? Sync interval?

Record: `duckbrain_enabled`, `duckbrain_namespace`, `duckbrain_sync_interval`

---

## Phase 5 — Quality Gate Configuration

### Ask the User

**Step 11: GitReins Quality Gates**

Ask:
> Enable GitReins quality gates? (recommended)
> Secrets detection? Pre-commit build/lint/tests?
> LLM judge for acceptance evaluation?
> If judge: which model/provider?

Record: `gitreins_enabled`, `llm_judge_enabled`, `judge_model`, `judge_provider`

---

## Phase 6 — Config Map (Output)

After gathering all answers, produce a config map. Store it in DuckBrain
at `/fleet/config` or keep it in a local file:

```yaml
# Coding Hermes Fleet Config
providers:
  broth:
    provider: "<answer>"
    model: "<answer>"
    api_key_env: "<ENV_VAR>"
    billing: "<payg|flat-rate>"
  star:
    provider: "<answer>"
    model: "<answer>"
    api_key_env: "<ENV_VAR>"
    billing: "payg"
  workers:
    heavy:
      provider: "<answer>"
      model: "<answer>"
      api_key_env: "<ENV_VAR>"
    light:
      provider: "<answer>"
      model: "<answer>"
      api_key_env: "<ENV_VAR>"
    budget:
      provider: "<answer>"
      model: "<answer>"
      api_key_env: "<ENV_VAR>"

delivery:
  broth: "<platform>:<chat_id>:<topic_id>"
  stars:
    <project>: "<platform>:<chat_id>:<topic_id>"

scheduling:
  budget: 100
  max_concurrent: 8
  min_interval: 20m
  max_interval: 24h
  priority_levels: 10

projects:
  - name: "<project>"
    weight: 10
    priority: 5
    workdir: "/home/you/projects/<project>"
    repo_url: "https://github.com/you/<project>"

duckbrain:
  enabled: true
  namespace: "coding-hermes"
  sync_interval: "hourly"

quality:
  gitreins: true
  llm_judge: true
  judge_model: "<model>"
  judge_provider: "<provider>"
```

---

## Next Steps

1. Store the config in DuckBrain: `duckbrain_remember(key="/fleet/config", ...)`
2. Load the **Broth** skill to set up the fleet supervisor
3. For each project, create a `.coding-hermes/tasks.md` task board
4. The broth auto-detects projects and creates stars — or create them manually
5. The scheduler daemon starts running stars on schedule

## Pitfalls

- **Don't put stars on prepaid providers.** Locked prepaid = star can't load = project dead.
- **Don't skip config.** Stars crash on missing API keys. Broth delivers to nowhere.
- **Test delivery targets.** Send a test message before creating cron jobs.
- **Start conservative with weights.** You can adjust later. Weight 80 starves the fleet.
- **DuckBrain is recommended.** Without it, stars don't remember across ticks.
