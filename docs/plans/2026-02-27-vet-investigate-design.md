# Design: pup vet + pup investigate

**Date:** 2026-02-27
**Status:** Approved
**Authors:** @jedgington + Claude

## Problem

Pup gives API access to individual Datadog domains, but production problems don't live in one domain. When a monitor fires, you need logs AND metrics AND traces AND recent deploys AND related monitors — correlated, not in separate JSON blobs.

Meanwhile, observability health issues (silent monitors, inconsistent tags, coverage gaps) are invisible until something breaks. Humans don't find these without purposefully looking.

## Vision

Pup evolves from "Datadog API wrapper" to "the AI agent's observability brain." Two new capabilities:

- **`pup vet`** — Observability health audit. Surfaces gaps, misconfigs, inconsistencies, and improvement opportunities. Any agent can run this anytime — no incident needed.
- **`pup investigate`** — Cross-domain incident investigation. Takes a signal (monitor, service) and auto-correlates logs, metrics, traces, deploys, and monitors into a structured timeline.

The target audience is AI agents (Claude Code, Cursor, etc.) guided by SREs and developers. Both commands output structured JSON in agent mode and human-readable summaries otherwise.

## Architecture

### Approach: Built-in Rust commands

All logic compiled into the pup binary. Checks are hardcoded in Rust. No YAML DSL, no plugin system, no rule engine. The value is in what we surface, not how extensibly we define it.

Extensibility story for the future: AI agents can define custom vet checks via skills/commands — a more powerful model than static YAML rules.

### Module layout

```
src/
├── ops/                        # Orchestration engine (business logic)
│   ├── mod.rs                  # Shared types: Finding, Severity, TimelineEvent, Scope
│   ├── vet.rs                  # Vet check implementations
│   └── investigate.rs          # Investigation stage pipeline
├── commands/
│   ├── vet.rs                  # Clap wrapper → calls ops::vet
│   └── investigate.rs          # Clap wrapper → calls ops::investigate
```

Business logic in `src/ops/`, thin clap wrappers in `src/commands/`. Same pattern as existing code.

### Shared types

```rust
enum Severity { Critical, Warning, Info }

struct Scope {
    services: Option<Vec<String>>,
    tags: Option<Vec<String>>,
    environment: Option<String>,
}

struct Finding {
    check: String,           // "silent-monitors"
    severity: Severity,
    category: String,        // "monitor-health", "tag-hygiene"
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

struct TimelineEvent {
    time: DateTime<Utc>,
    domain: String,          // "logs", "metrics", "monitors", "events"
    event: String,
    severity: Severity,
    raw_data: Option<Value>, // agent can drill deeper
}

struct Correlation {
    finding: String,
    confidence: String,      // "high", "medium", "low"
    domains: Vec<String>,
}
```

---

## pup vet

### Usage

```
pup vet                              # Run all checks
pup vet --service=web-api            # Scope to a service
pup vet --tags=team:platform         # Scope to a team
pup vet --check=silent-monitors      # Run specific check only
pup vet --severity=critical          # Filter output by severity
pup vet --category=monitor-health    # Filter by category
pup vet list                         # List available checks
```

### Check categories

#### Monitor health (Phase 1a)

All checks powered by a single `monitors list` API call.

| Check | Severity | Detection |
|-------|----------|-----------|
| `silent-monitors` | Critical | Monitors with no notification channels — alerts fire into the void |
| `stale-monitors` | Warning | Monitors in "No Data" state for >7 days — probably abandoned or misconfigured |
| `untagged-monitors` | Warning | Monitors without service/team/env tags — impossible to route or filter |
| `muted-forgotten` | Warning | Monitors muted for >30 days — intended as temporary, now permanent |
| `no-recovery-threshold` | Info | Monitors without recovery thresholds — flapping risk |

#### Tag hygiene (Phase 1b)

Adds service catalog API call; reuses monitors data from Phase 1a.

| Check | Severity | Detection |
|-------|----------|-----------|
| `inconsistent-tags` | Warning | Same concept tagged differently: `env:prod` vs `environment:production` |
| `missing-standard-tags` | Warning | Resources without expected tags (service, team, env) |

#### Future expansion (not phased — added organically as checks prove valuable)

| Category | Checks | Detection |
|----------|--------|-----------|
| Coverage gaps | `unmonitored-services`, `no-slo-services`, `no-dashboard-services` | Things that should be monitored but aren't |
| Correlations | `uncorrelated-services`, `logs-without-traces` | Observability data that isn't connected |
| SLO health | `slo-no-alert`, `slo-budget-burned` | SLOs that exist but aren't actionable |

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

## pup investigate

### Usage

```
pup investigate --monitor=12345          # Start from an alerting monitor
pup investigate --service=web-api        # Investigate a service
pup investigate --from=1h               # Override time window (default: 2h)
```

### Investigation pipeline

Five stages executed in order:

| Stage | Purpose | API calls |
|-------|---------|-----------|
| 1. Signal | Identify what's wrong. Fetch monitor details or service monitors. | `monitors get` or `monitors list` |
| 2. Scope | Determine blast radius. Extract services, environments, tags from signal. | Derived from stage 1 |
| 3. Gather | Pull data across domains in parallel via `tokio::join!`: recent logs (aggregated), key metrics, recent events/deploys, error traces. | `logs aggregate`, `metrics query`, `events list`, `traces search` |
| 4. Correlate | Build chronological timeline. Flag temporal correlations with confidence levels. | Pure computation |
| 5. Summarize | Structured findings for agent consumption. | None |

### Correlation logic (intentionally simple)

- **Time-window correlation:** Events within N minutes of each other are linked.
- **Deploy proximity:** Any deploy within 30 minutes before an error spike is flagged as high confidence.
- **Metric threshold:** CPU/memory/latency crossing thresholds near error spikes is flagged as medium confidence.

No ML, no algorithms. Temporal proximity with domain-aware heuristics. The agent reasons about causality — investigate just surfaces the facts.

### Agent mode output

```json
{
  "status": "success",
  "data": {
    "signal": {
      "type": "monitor",
      "id": "12345",
      "name": "High Error Rate on web-api",
      "status": "Alert",
      "triggered_at": "2026-02-27T10:05:00Z"
    },
    "scope": {
      "services": ["web-api"],
      "environments": ["production"],
      "tags": ["team:platform"]
    },
    "timeline": [
      {
        "time": "2026-02-27T10:00:00Z",
        "domain": "events",
        "event": "Deploy web-api v2.3.1",
        "severity": "info"
      },
      {
        "time": "2026-02-27T10:04:00Z",
        "domain": "metrics",
        "event": "p99 latency increased 3x (120ms -> 380ms)",
        "severity": "warning"
      },
      {
        "time": "2026-02-27T10:05:00Z",
        "domain": "logs",
        "event": "Error rate 5x baseline (68/min -> 342/min)",
        "severity": "critical",
        "sample": "NullPointerException in AuthHandler.validate()"
      },
      {
        "time": "2026-02-27T10:05:30Z",
        "domain": "monitors",
        "event": "Monitor 'High Error Rate' triggered",
        "severity": "critical"
      }
    ],
    "correlations": [
      {
        "finding": "Error spike began 5 minutes after deploy v2.3.1",
        "confidence": "high",
        "domains": ["events", "logs"]
      },
      {
        "finding": "Latency degradation preceded error spike by 1 minute",
        "confidence": "medium",
        "domains": ["metrics", "logs"]
      }
    ],
    "summary": {
      "probable_cause": "Deploy v2.3.1 likely introduced regression",
      "impact": "342 errors/min affecting web-api in production",
      "suggested_actions": [
        "Inspect deploy v2.3.1 changes",
        "Check AuthHandler.validate() for recent modifications",
        "Consider rollback if error rate persists"
      ]
    }
  },
  "metadata": {
    "command": "investigate",
    "duration_ms": 3800,
    "api_calls": 5,
    "time_range": {
      "from": "2026-02-27T08:05:00Z",
      "to": "2026-02-27T10:10:00Z"
    }
  }
}
```

### Human mode output

```
pup investigate --monitor=12345

Signal: Monitor "High Error Rate on web-api" — ALERT since 10:05 UTC
Scope:  web-api / production / team:platform

Timeline (last 2 hours):
  10:00  [event]    Deploy web-api v2.3.1
  10:04  [metrics]  p99 latency 3x (120ms -> 380ms)
  10:05  [logs]     Error rate 5x baseline (342/min)
                    Sample: NullPointerException in AuthHandler.validate()
  10:05  [monitor]  "High Error Rate" triggered

Correlations:
  HIGH   Error spike began 5min after deploy v2.3.1
  MEDIUM Latency degradation preceded errors by 1min

Summary: Deploy v2.3.1 likely introduced regression in AuthHandler
Actions: Inspect deploy changes | Check AuthHandler.validate() | Consider rollback
```

### Design constraints

- **No remediation.** Investigate gathers facts. The agent decides what to do.
- **No causal claims.** "Probable cause" with confidence levels, not certainty.
- **One level deep.** No recursive investigation. The agent runs follow-up commands.
- **Parallel gathering.** Stage 3 uses `tokio::join!` to query all domains concurrently.

---

## Phasing

| Phase | Delivers | API calls | Estimated scope |
|-------|----------|-----------|-----------------|
| 1a | `pup vet` — 5 monitor health checks, shared types, agent output | `monitors list` (1 call) | `ops/mod.rs`, `ops/vet.rs`, `commands/vet.rs` |
| 1b | `pup vet` — tag hygiene checks (2-3 checks) | `+service catalog list` | Expand `ops/vet.rs` |
| 2 | `pup investigate` — monitor-triggered cross-domain timeline | `monitors get`, `logs aggregate`, `metrics query`, `events list` | `ops/investigate.rs`, `commands/investigate.rs` |

Additional vet checks (SLOs, coverage, correlations) added organically as they prove valuable. No formal phase — just more checks in `ops/vet.rs`.

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Architecture | Built-in Rust, no rule engine | Value is in the checks, not the meta-framework. YAGNI. |
| Extensibility | Future: agents define custom checks via skills | More powerful than YAML rules, deferred until needed |
| Naming | `pup vet` not `pup doctor` | `doctor` implies CLI self-diagnostics. `vet` is on-brand and clear. |
| Runbook | Cut from scope | Agents + skills solve this better than a YAML execution engine |
| Correlation | Simple temporal heuristics | Agent reasons about causality. Investigate surfaces facts. |
| Phasing | Doctor-first, investigate second | Vet doesn't need an active incident. Every org has findings right now. |

## Not doing

- **Runbook execution engine** — Agents running pup commands directly with reasoning between steps is more powerful than a fixed YAML pipeline.
- **YAML rule DSL** — Delays shipping value. Every new check would require learning the DSL instead of writing Rust.
- **MCP server** — Good future addition but shouldn't be the primary interface. CLI-first.
- **ML/AI correlation** — Simple heuristics with confidence levels. The agent does the reasoning.
- **Remediation** — Investigate and vet surface findings. They don't take action.
