# Middle-Out Wiring Checklist

**"All packages pass tests" ≠ "the project is usable."**

This checklist runs AFTER a worker completes coding and BEFORE the foreman marks a task `[x]`. It verifies that the code written by the worker is actually reachable from the outside world.

## Quick Check (30 seconds)

```bash
# 1. Can the binary start?
go build -o /tmp/bin ./cmd/... 2>&1 && timeout 3 /tmp/bin serve 2>&1 | head -5

# 2. Is main.go importing the new package?
grep '<new-package>' cmd/*/main.go || echo "MISSING: not imported in main"

# 3. Are routes registered?
grep -c 'HandleFunc\|Handle(' cmd/*/main.go
```

## Full Checklist

| # | Check | Command | Pass if |
|---|-------|---------|---------|
| 1 | Binary compiles | `go build -o /tmp/bin ./cmd/...` | Exit 0 |
| 2 | Binary starts | `timeout 5 /tmp/bin serve` | Starts, no immediate crash |
| 3 | Health endpoint responds | `curl -s localhost:<port>/health` | HTTP 200 |
| 4 | New package imported in main | `grep '<pkg>' cmd/*/main.go` | Found |
| 5 | New handler registered on router | `grep 'HandleFunc.*<route>' cmd/*/main.go` | Found |
| 6 | Config struct passed to new package | `grep '<Config>' cmd/*/main.go` | Found |
| 7 | Middleware applied (if needed) | `grep 'Use(' cmd/*/main.go` | Found |
| 8 | DB connection passed (if needed) | `grep 'sql.Open\|pgx.Connect' cmd/*/main.go` | Found if DB task |
| 9 | gRPC server registered (if needed) | `grep 'Register.*Server' cmd/*/main.go` | Found if gRPC task |
| 10 | All registered routes have handlers | See route-handler verification below | All match |

## Route-Handler Verification

```bash
# For every registered route, verify the handler function exists
grep -r 'HandleFunc\|Handle(' --include='*.go' . | grep -v '_test.go' | while read line; do
    handler=$(echo "$line" | grep -oP 'HandleFunc\("[^"]+",\s*\K\w+')
    if [ -n "$handler" ]; then
        grep -q "func $handler" --include='*.go' -r . || echo "MISSING HANDLER: $handler"
    fi
done
```

## Per-Language Wiring Patterns

### Go — HTTP Service
```
cmd/<name>/main.go  ← imports all packages, registers routes, starts server
internal/<pkg>/     ← package logic + tests
```
Must have: `main.go`, `go build` succeeds, `serve` starts HTTP listener.

### Go — CLI Tool
```
cmd/<name>/main.go  ← imports commands, wires flags, calls Execute()
internal/cli/       ← cobra commands + tests
```
Must have: `main.go`, `--help` works, at least one subcommand wired.

### Go — gRPC Service
```
cmd/<name>/main.go  ← imports proto stubs, registers server, starts listener
internal/<pkg>/     ← service implementation + tests
proto/              ← .proto definitions
```
Must have: `main.go`, proto generated, server registered, listener started.

### Python — FastAPI
```
main.py             ← imports routers, creates app, calls uvicorn.run
routers/<name>.py   ← route handlers + tests
```
Must have: `main.py`, `uvicorn main:app` works, all routers included.

### TypeScript — Hono/Express
```
src/index.ts        ← imports routes, creates app, calls serve
src/routes/<name>.ts ← route handlers + tests
```
Must have: `index.ts`, `npm start` works, all routes mounted.

## Common Failure Patterns

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| `go build` fails on `cmd/` | Worker wrote package but not main.go | Create main.go importing the package |
| Binary starts but no routes | Handler defined but not registered | Add `mux.HandleFunc(...)` in main.go |
| `curl` gets 404 | Route path mismatch between handler and registration | Align the route strings |
| Config not found | `main.go` doesn't load config before passing to package | Add config.Load() in main |
| DB migration exists but no tables | Worker wrote migration but didn't wire DB connection in main | Add DB init in main.go |
| gRPC reflection fails | Server registered but reflection not enabled | Add reflection.Register() |
| Worker says "done" but nothing wired | Worker implemented package-level code, never touched main.go | Foreman MUST verify wiring before accepting |

## When to Reject a Worker's Work

Reject (do NOT mark `[x]`) if:
- The new package compiles but has no import in `main.go`
- The handler is defined but no route registers it
- The gRPC service is implemented but not registered
- `serve` command exits immediately without listening
- Any registered route returns 501 / "not implemented" / empty

Create tasks for each gap before marking anything complete.
