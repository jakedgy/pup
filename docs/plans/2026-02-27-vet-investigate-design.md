# Design: `pup vet`

**Date:** 2026-02-27
**Status:** Draft
**Authors:** @jedgington + Claude
**RFC:** https://github.com/datadog-labs/pup/issues/129

## Problem

Observability configurations degrade silently. Monitors lose their notification channels. Traces break across service boundaries. Logs stop correlating with APM. These aren't matters of preference or org-specific style — they're genuinely broken configurations that are invisible until something goes wrong.

No one finds these without a dedicated audit, and no one does a dedicated audit until it's too late.

## Vision

A new top-level command — `pup vet` — that surfaces things which are universally broken or misconfigured, without assuming how your org has chosen to structure Datadog.

The target audience is AI agents (Claude Code, Cursor, etc.) guided by SREs and developers. `pup vet` outputs structured JSON in agent mode and human-readable summaries otherwise.

**Key constraint:** Every check must be universally useful. Silent monitors are broken everywhere. Broken trace correlation is broken everywhere. We don't tell you how to tag or organize — we find things that aren't working as intended.

## Architecture

### Approach: Built-in Rust commands

All logic compiled into the pup binary. Checks are hardcoded in Rust. No YAML DSL, no plugin system, no rule engine. The value is in what we surface, not how extensibly we define it.

Future extensibility story: AI agents defining custom checks via skills/commands — a more powerful model than static YAML rules.

### Module layout

```
src/
├── ops/                        # Business logic (new)
│   ├── mod.rs                  # Shared types: Finding, Severity, Scope
│   └── vet.rs                  # Vet check implementations
├── commands/
│   └── vet.rs                  # Clap wrapper → calls ops::vet
```

Business logic in `src/ops/`, thin clap wrappers in `src/commands/`. Same separation pattern as the existing codebase.

### Shared types

```rust
enum Severity { Critical, Warning, Info }

struct Scope {
    services: Option<Vec<String>>,
    tags: Option<Vec<String>>,
}

struct Finding {
    check: String,           // "silent-monitors"
    severity: Severity,
    category: String,        // "monitor-health"
    count: usize,
    resources: Vec<Resource>,
    recommendation: String,
    fix_hint: Option<String>, // actionable pup command hint
}

struct Resource {
    resource_type: String,   // "monitor", "service", "slo"
    id: String,
    name: String,
    detail: String,
}
```

---

## `pup vet`

### Usage

```
pup vet                              # Run all checks
pup vet --service=web-api            # Scope to a service
pup vet --tags=team:platform         # Scope to a team
pup vet --check=silent-monitors      # Run specific check only
pup vet --severity=critical          # Filter output by severity
pup vet list                         # List available checks
```

### Checks

Initial checks target things that are universally non-functional — broken regardless of how an org uses Datadog.

All initial checks are powered by a single `monitors list` API call.

| Check | Severity | What it catches |
|-------|----------|-----------------|
| `silent-monitors` | Critical | Monitors with no notification channels — alerts fire into the void |
| `stale-monitors` | Warning | Monitors stuck in "No Data" for >7 days — abandoned or misconfigured |
| `untagged-monitors` | Warning | Monitors without any tags — can't be filtered, routed, or grouped |
| `muted-forgotten` | Warning | Monitors muted for >30 days — meant to be temporary, now permanent |
| `no-recovery-threshold` | Info | Monitors without recovery thresholds — flapping risk |

Additional checks added as they prove valuable. Candidates include broken trace/log correlation, USM gaps, SLOs without error budget alerts. Each must meet the bar: universally useful, not org-specific.

### Agent mode output

```json
{
  "status": "success",
  "data": {
    "summary": {
      "total_checks": 5,
      "passed": 3,
      "warnings": 1,
      "critical": 1
    },
    "findings": [
      {
        "check": "silent-monitors",
        "severity": "critical",
        "category": "monitor-health",
        "count": 2,
        "resources": [
          {
            "type": "monitor",
            "id": "12345",
            "name": "High CPU on web-api",
            "detail": "No notification channels configured"
          },
          {
            "type": "monitor",
            "id": "67890",
            "name": "Disk usage > 90%",
            "detail": "No notification channels configured"
          }
        ],
        "recommendation": "Add notification channels so alerts reach on-call responders",
        "fix_hint": "pup monitors get <id> to inspect, then update notification channels"
      }
    ]
  },
  "metadata": {
    "command": "vet",
    "scope": {"tags": null, "service": null},
    "checked_at": "2026-02-27T15:00:00Z",
    "count": 1
  }
}
```

### Human mode output

```
pup vet — 5 checks run

CRITICAL: silent-monitors (2 found)
  Monitors with no notification channels
  - #12345 "High CPU on web-api"
  - #67890 "Disk usage > 90%"
  -> Add notification channels so alerts reach on-call responders

WARNING: muted-forgotten (1 found)
  Monitors muted for >30 days
  - #11111 "Memory pressure on auth-service" (muted 45 days)
  -> Review and unmute, or delete if no longer needed

PASSED: stale-monitors, untagged-monitors, no-recovery-threshold

Summary: 1 critical, 1 warning, 3 passed
```

---

## Future Direction: `pup investigate`

Once `pup vet` establishes the `ops/` module pattern and shared types, the next natural capability is cross-domain incident investigation — taking a signal (monitor alert, service name) and correlating logs, metrics, traces, and events into a timeline.

This is a separate design effort. The investigation use case needs careful thought to avoid being org-specific — different orgs use different Datadog products, and an investigation that assumes APM + logs + metrics may not work for an org that only uses infrastructure monitoring.

The `ops/` module layout and shared types (Severity, Finding, Resource) are designed to extend to this use case when the time comes.

---

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Architecture | Built-in Rust, no rule engine | Value is in the checks, not the meta-framework. YAGNI. |
| Extensibility | Deferred. Future: agents define custom checks via skills | More powerful than YAML rules |
| Naming | `pup vet` not `pup doctor` | `doctor` implies CLI self-diagnostics. `vet` is on-brand. |
| Check bar | Universally useful only | No assumptions about org structure. Broken is broken everywhere. |
| Runbook | Cut from scope | Agents + skills solve this better than a YAML execution engine |
| Investigate | Future direction, separate design | Needs careful thought to avoid org-specificity |

## Not doing

- **Runbook execution engine** — Agents running pup commands directly with reasoning between steps is more powerful than a fixed YAML pipeline.
- **YAML rule DSL** — Delays shipping value. Every new check would require learning the DSL instead of writing Rust.
- **MCP server** — Good future addition but shouldn't be the primary interface. CLI-first.
- **Remediation** — Vet surfaces findings. It doesn't take action.
- **Opinionated checks** — No "you should tag this way" or "you should have N monitors per service." Only genuinely broken configurations.
