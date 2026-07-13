# Coding Hermes Skills — The Autonomous Coding Fleet

Skills that power a self-managing fleet of AI coding agents — a broth that watches over everything, stars that run individual projects, and a scheduler that decides who runs when.

## The Metaphor

> A **broth** is the base — the foundation that holds everything together.  
> **Stars** float in the broth, each one a project, burning fuel and producing light.  
> The **scheduler** decides which stars get fuel and when.

```
┌─────────────────────────────────────────────────┐
│                  BROTH (Supervisor)               │
│   Watches all stars. Heals broken ones.          │
│   Rebalances. Reports. Keeps the fleet alive.    │
│                                                   │
│   ★ Star A    ★ Star B    ★ Star C    ★ Star D  │
│   (foreman)   (foreman)   (foreman)   (foreman)  │
│      │            │            │            │      │
│   Workers     Workers     Workers     Workers    │
└─────────────────────────────────────────────────┘
```

## Skill Index

| Skill | File | Purpose |
|-------|------|---------|
| **Config** | `skills/coding-hermes-config/SKILL.md` | First skill to load. Gathers API keys, model choices, delivery targets, budget settings. |
| **Broth** | `skills/coding-hermes-broth/SKILL.md` | The fleet supervisor — monitors all projects, heals broken foremen, rebalances schedules, produces HTML reports. |
| **Star** | `skills/coding-hermes-star/SKILL.md` | Per-project foreman — self-heals, scans tasks, spawns workers, verifies quality, commits, learns. Full 10-step SDLC loop. |
| **Cron Schema** | `skills/coding-hermes-cron-schema/SKILL.md` | Canonical cron shape. Defines valid schedule schemas, model-provider patterns, state machine. Single source of truth. |

## Architecture

### Three-Tier Model

```
You (Strategy) → Broth (Fleet) → Stars (Projects) → Workers (Code)
```

| Tier | Runs | Model Type | Purpose |
|------|------|------------|---------|
| **You** | When you want | — | Strategy, architecture, new capabilities |
| **Broth** | Every 4 hours | Reasoning model (PAYG) | Heal, audit, rebalance, report |
| **Star** | Per-project schedule | Reasoning model (PAYG) | Scan tasks, compile prompts, spawn workers |
| **Worker** | Spawned per task | Coding model (prepaid/flat-rate) | Write code, run tests, commit |

### The Golden Rule

**Foremen (stars) must always be reachable.** If a foreman can't load because its provider is locked, the entire project goes dark — no rebalancing, no worker spawning, nothing. Foremen stay on PAYG (always add credit to unblock). Workers burn prepaid buckets getting coding done — when one locks, the foreman switches to the next.

### Scheduler

A Go daemon (`coding-herms/scheduler`) replaces static cron with a **weight-budget knapsack**:

- **Weight** (1–100): How much concurrency budget a project consumes per tick
- **Priority** (1–10): How aggressively the scheduler tries to run it
- **Urgency = Priority × Decay**: Guarantees starvation is impossible

The scheduler packs projects greedily by urgency into a fixed budget (default 100). Lightweight projects squeeze into gaps that heavier projects can't fit.

## Quick Start

### 1. Install the Skills

```bash
# Clone this repo into your Hermes skills directory
git clone https://github.com/coding-herms/skills.git ~/.hermes/coding-hermes-skills

# Or symlink individual skills
ln -s ~/.hermes/coding-hermes-skills/skills/coding-hermes-config ~/.hermes/skills/coding-hermes-config
ln -s ~/.hermes/coding-hermes-skills/skills/coding-hermes-broth ~/.hermes/skills/coding-hermes-broth
ln -s ~/.hermes/coding-hermes-skills/skills/coding-hermes-star ~/.hermes/skills/coding-hermes-star
ln -s ~/.hermes/coding-hermes-skills/skills/coding-hermes-cron-schema ~/.hermes/skills/coding-hermes-cron-schema
```

### 2. Run the Config Skill

Load `coding-hermes-config` and follow the guided setup. It will ask for:
- Your LLM providers and API keys
- Which model for the broth (supervisor)
- Which models for stars (foremen) and workers
- Where to deliver reports
- Your weight budget and scheduling preferences

### 3. Install the Scheduler

```bash
git clone https://github.com/coding-herms/scheduler.git
cd scheduler
make build
make deploy-install
sudo systemctl start coding-hermes-scheduler
```

### 4. Add Your First Project

```bash
curl -X POST http://localhost:9090/api/v1/projects \
  -H 'Content-Type: application/json' \
  -d '{"name":"my-project","weight":10,"priority":5,"workdir":"/home/you/my-project","repo_url":"https://github.com/you/my-project"}'
```

### 5. Create the Star (Foreman)

The broth will auto-detect projects with `.coding-hermes/tasks.md` and create foremen for them. Or you can create one manually using Hermes:

```python
cronjob(action="create", name="my-project-star",
  schedule="every 120m",
  skills=["coding-hermes-star", "coding-hermes-cron-schema"],
  enabled_toolsets=["terminal","file","web","search","skills","memory"],
  workdir="/home/you/my-project")
```

## Requirements

- **Hermes Agent** — these skills are designed for the Hermes AI agent runtime
- **Scheduler** — the Go daemon from `coding-herms/scheduler`
- **DuckBrain** (optional) — for long-term fleet memory across sessions
- **GitReins** (optional) — for automated quality gates

## License

MIT
