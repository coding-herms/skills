# Coding Hermes — Skills

This repository contains the skill files that power the Coding Hermes autonomous coding fleet. These skills define the processes, not the configuration. Your API keys, model names, and project paths go through the config skill — never hardcoded here.

---

## Skills Overview

| Skill | Layer | What It Does |
|-------|-------|-------------|
| [`coding-hermes-config`](coding-hermes-config/SKILL.md) | Foundation | First-run setup — asks for API keys, models, project paths |
| [`coding-hermes-north-star`](coding-hermes-north-star/SKILL.md) | Foundation | Architecture overview — the full system explained |
| [`coding-hermes-foreman`](coding-hermes-foreman/SKILL.md) | Foreman | Per-project tick loop — inspects, plans, spawns workers |
| [`coding-hermes-supervisor`](coding-hermes-supervisor/SKILL.md) | Supervisor | Fleet-wide oversight — health, starvation, failures, costs |
| [`coding-hermes-broker`](coding-hermes-broker/SKILL.md) | Broker | Scheduling algorithm — weight-budget, urgency, packing |
| [`coding-hermes-worker`](coding-hermes-worker/SKILL.md) | Worker | Code implementation — writes code, runs tests, commits |

---

## Architecture

```
┌──────────────────────────────────────────────┐
│                 SUPERVISOR                     │
│  Fleet health, starvation, failures, costs    │
└────────────────────┬─────────────────────────┘
                     │ queries scheduler API
┌────────────────────▼─────────────────────────┐
│                  BROKER                        │
│  Weight-budget knapsack scheduler             │
│  Urgency = priority × (1 + wait/interval)^d  │
└────────────────────┬─────────────────────────┘
                     │ spawns per tick
┌────────────────────▼─────────────────────────┐
│                  FOREMAN                       │
│  Per-project: inspect → plan → spawn worker   │
│  10-step loop: heal, read, hilo, duckbrain,   │
│  prompt, spawn, guard, judge, commit, write   │
└────────────────────┬─────────────────────────┘
                     │ spawns for coding tasks
┌────────────────────▼─────────────────────────┐
│                  WORKER                        │
│  Write code → run tests → commit → report     │
│  One task per spawn. No planning.             │
└──────────────────────────────────────────────┘
```

---

## Getting Started

### 1. Clone the skills

```bash
git clone https://github.com/coding-herms/skills.git ~/.hermes/skills/coding-hermes/
```

### 2. Run the config skill

In Hermes:
```
Load skill coding-hermes-config and walk me through setup.
```

The config skill will ask you for:
- Your API keys (one per provider)
- Which models to use for foreman vs worker
- Your project paths and repos

### 3. Clone the scheduler

```bash
git clone https://github.com/coding-herms/scheduler.git ~/coding-herms-scheduler
cd ~/coding-herms-scheduler
make build
make migrate
make deploy
```

### 4. Verify

```bash
curl http://localhost:9090/
/fleet status
```

---

## How Skills Load

The skills form a dependency chain:

```
coding-hermes-config        ← ALWAYS loaded first (setup)
        ↓
coding-hermes-north-star    ← architecture reference
        ↓
coding-hermes-broker        ← scheduling logic
        ↓
coding-hermes-foreman       ← per-project execution
        ↓
coding-hermes-supervisor    ← fleet oversight
        ↓
coding-hermes-worker        ← code implementation
```

Each skill references the ones above it. The foreman loads config → north-star → itself → spawns workers. The supervisor loads config → north-star → broker → itself.

---

## What's NOT In These Skills

- **API keys** — handled by `coding-hermes-config` at setup time
- **Model names** — configured per user, stored in DuckBrain
- **Project paths** — configured per user
- **Provider URLs** — configured per user
- **Specific account details** — never in skills

---

## Related Repos

- [`coding-herms/scheduler`](https://github.com/coding-herms/scheduler) — The Go scheduler binary
- [`coding-herms/`](https://github.com/coding-herms) — GitHub organization
