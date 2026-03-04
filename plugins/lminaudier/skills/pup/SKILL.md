---
name: pup
description: >-
  Interact with the Datadog observability platform using the pup CLI.
  Use this skill whenever the user mentions Datadog, monitoring, observability,
  or wants to query metrics, search logs, manage monitors, view dashboards,
  investigate incidents, check SLOs, manage infrastructure tags, look up
  Datadog documentation, or interact with any Datadog product. Activates on
  requests like "check my Datadog monitors", "query CPU metrics", "search logs
  for errors", "list dashboards", "show incidents", "what's the SLO status",
  "list hosts", "who's on call", "find Datadog docs on X", etc.
allowed-tools:
  - Bash
  - WebFetch
---

# Pup -- Datadog CLI for AI Agents

Pup is a CLI companion providing 300+ commands across 42 command groups for
Datadog's observability platform. It returns structured JSON by default and
auto-detects agent mode when invoked from Claude Code.

## Guardrails

- **Auth first**: run `pup auth status` before any operation. If it fails,
  try `pup auth refresh` then fall back to `pup auth login`.
- **No blind destructive ops**: never pass `--yes` or set `DD_AUTO_APPROVE=true`
  unless the user explicitly asks. Always fetch and show the resource before
  proposing a delete or modify.
- **Never delete monitors or dashboards directly**: instead, rename with a
  `[MARKED FOR DELETION]` prefix so the owner can review. Only delete if the
  user insists after seeing the details.
- **Scope queries**: always include time ranges (`--duration`, `--from`) and
  filters when querying metrics or logs to avoid pulling excessive data.
- **Read before write**: retrieve current state before proposing changes.
- **Cost awareness**: log indexing and APM span retention cost money. Prefer
  narrow queries and recommend exclusion filters when users report high costs.

## Quick Reference

| Task | Command |
|------|---------|
| Check auth | `pup auth status` |
| Refresh token | `pup auth refresh` |
| Search error logs | `pup logs search --query "status:error" --duration 1h` |
| List monitors | `pup monitors list` |
| Mute a monitor | `pup monitors mute --id 123 --duration 1h` |
| Find slow traces | `pup apm traces list --service api --min-duration 500ms` |
| List active incidents | `pup incidents list --status active` |
| Create incident | `pup incidents create --title "Issue" --severity SEV-2` |
| Query metrics | `pup metrics query --query "avg:system.cpu.user{*}" --duration 1h` |
| List hosts | `pup hosts list` |
| Check SLOs | `pup slos list` |
| Who's on call | `pup on-call who --team my-team` |
| Security signals | `pup security signals list --severity critical` |

## Command Discovery

Pup is self-documenting. When unsure about a command or its flags:

```bash
pup --help              # List all command groups
pup <command> --help    # Command-specific help
pup <cmd> <sub> --help  # Subcommand help
```

Always use `--help` to discover available flags rather than guessing.

## Authentication

Priority (highest to lowest): `DD_ACCESS_TOKEN` > OAuth2 tokens > API keys.

```bash
pup auth login          # OAuth2 browser flow (recommended)
pup auth status         # Check token validity
pup auth refresh        # Refresh expired token (no browser needed)
pup auth logout         # Clear credentials
```

OAuth tokens expire after ~1 hour. When a command fails with 401/403:

```bash
pup auth refresh        # Try refresh first
pup auth login          # If refresh fails, full re-auth
```

### Headless / CI (no browser)

```bash
export DD_API_KEY="<key>"
export DD_APP_KEY="<key>"
export DD_SITE="datadoghq.com"
```

### Datadog Sites

| Site | `DD_SITE` value |
|------|-----------------|
| US1 (default) | `datadoghq.com` |
| US3 | `us3.datadoghq.com` |
| US5 | `us5.datadoghq.com` |
| EU1 | `datadoghq.eu` |
| AP1 | `ap1.datadoghq.com` |
| US1-FED | `ddog-gov.com` |

## Monitors

```bash
pup monitors list
pup monitors list --tags "env:prod"
pup monitors list --status "Alert"
pup monitors get --id 12345
pup monitors mute --id 12345 --duration 1h
pup monitors unmute --id 12345
pup monitors create --name "High CPU" --type "metric alert" \
  --query "avg(last_5m):avg:system.cpu.user{env:prod} by {host} > 80" \
  --message "CPU above 80% @slack-ops"
```

### Monitor Best Practices

When creating or reviewing monitors:

- **Avoid alert fatigue**: use `last_5m` or wider windows, not `last_1m`.
  Set recovery thresholds (e.g., critical=80, critical_recovery=70) to prevent
  flapping.
- **Scope properly**: always filter by `env`, `service`, or `team` -- never
  alert on `{*}` in production.
- **Include context**: add runbook URLs, current value `{{value}}`, host
  `{{host.name}}`, and notification handles (`@slack-ops`, `@pagerduty-oncall`)
  in the message.
- **Actionable only**: if no human action is needed, it should not be an alert.

### Monitor Types

| Type | Use Case |
|------|----------|
| `metric alert` | CPU, memory, custom metrics |
| `query alert` | Complex metric queries |
| `service check` | Agent check status |
| `log alert` | Log pattern matching |
| `composite` | Combine multiple monitors |
| `apm` | APM metrics |

### Downtime vs Muting

| Use | When |
|-----|------|
| Mute monitor | Quick one-off, < 1 hour |
| Downtime | Scheduled maintenance, recurring |

```bash
pup downtime create --scope "env:staging" --duration 2h --message "Maintenance"
pup downtime list
pup downtime cancel --id 12345
```

## Logs

```bash
pup logs search --query "status:error" --duration 1h
pup logs search --query "service:api status:error" --duration 1h --limit 100
pup logs search --query "@http.status_code:>=500" --duration 24h --json
```

### Log Search Syntax

| Query | Meaning |
|-------|---------|
| `error` | Full-text search |
| `status:error` | Tag equals |
| `@http.status_code:500` | Attribute equals |
| `@http.status_code:>=400` | Numeric range |
| `service:api AND env:prod` | Boolean AND |
| `@message:*timeout*` | Wildcard |

### Cost Control -- Exclusion Filters

High-volume log sources drive costs. Find the noisiest sources:

```bash
pup logs search --query "*" --duration 1h --json \
  | jq 'group_by(.service) | map({service: .[0].service, count: length}) | sort_by(-.count)[:10]'
```

Common exclusion candidates: health checks (`@http.url:"/health"`),
debug logs (`status:debug`), static assets, heartbeats.

## Metrics

```bash
pup metrics list --filter "system.cpu"
pup metrics query --query "avg:system.cpu.user{*}" --duration 1h
pup metrics query --query "sum:trace.express.request.hits{service:api}" --duration 1h
```

## APM / Traces

```bash
pup apm services list
pup apm services list --env production
pup apm traces list --service my-service --duration 1h
pup apm traces list --service api --min-duration 500ms --duration 1h
pup apm traces list --service api --status error --duration 1h
pup apm traces get <trace_id> --json
```

### Key APM Metrics

| Metric | Measures |
|--------|----------|
| `trace.http.request.hits` | Request count |
| `trace.http.request.duration` | Latency |
| `trace.http.request.errors` | Error count |
| `trace.http.request.apdex` | User satisfaction |

### Trace Sampling Awareness

Not all traces are retained. Understand the modes:

| Mode | What's Kept |
|------|-------------|
| Head-based | Random % decided at trace start |
| Error/Slow | All errors and slow traces |
| Retention filters | What's indexed (billed) |

```bash
pup apm retention-filters list
```

Indexed spans cost significantly more than ingested spans -- only index what
is needed for search.

## Incidents & On-Call

```bash
pup incidents list --status active
pup incidents create --title "API Degradation" --severity SEV-2
pup incidents update --id abc-123 --status stable
pup incidents resolve --id abc-123

pup on-call teams list
pup on-call who --team platform-team
pup on-call schedules list
```

## Dashboards

```bash
pup dashboards list
pup dashboards list --tags "team:platform"
pup dashboards get --id abc-123
```

## SLOs

```bash
pup slos list
pup slos get --id slo-123
pup slos history --id slo-123 --duration 30d
```

## Synthetics

```bash
pup synthetics list
pup synthetics results --test-id abc-123
pup synthetics trigger --test-id abc-123
```

## Infrastructure

```bash
pup hosts list --limit 50
pup hosts list --filter "env:prod"
pup hosts get --hostname web-01
pup hosts mute --hostname web-01 --duration 1h

pup tags list --source "users"
pup tags get <hostname>
```

## Events

```bash
pup events list --duration 24h
pup events list --tags "source:deploy"
pup events post --title "Deploy started" --text "v1.2.3" --tags "env:prod"
```

## Security

```bash
pup security signals list --duration 24h
pup security signals list --severity critical
```

## Service Catalog

```bash
pup services list
pup services get --name payment-api
```

## Datadog Documentation Lookup

Datadog publishes an LLM-optimized doc index. Use it to answer questions about
Datadog features, limits, or configuration:

```bash
# Fetch the index and search for a topic
curl -s https://docs.datadoghq.com/llms.txt | grep -i "monitors"

# Fetch a specific doc page as markdown
curl -s https://docs.datadoghq.com/monitors.md
```

You can also use the WebFetch tool to read specific doc pages when the user
asks about Datadog features or limits.

| Topic | URL |
|-------|-----|
| Monitors | https://docs.datadoghq.com/monitors/ |
| Logs | https://docs.datadoghq.com/logs/ |
| APM/Tracing | https://docs.datadoghq.com/tracing/ |
| Metrics | https://docs.datadoghq.com/metrics/ |
| Dashboards | https://docs.datadoghq.com/dashboards/ |
| Synthetics | https://docs.datadoghq.com/synthetics/ |
| Incidents | https://docs.datadoghq.com/service_management/incident_management/ |
| API Reference | https://docs.datadoghq.com/api/ |

## Investigating Issues -- Recommended Flow

When a user asks to investigate a problem (high CPU, errors, incidents):

1. **Check monitors** -- `pup monitors list --status "Alert"` to find active alerts
2. **Query metrics** -- `pup metrics query` with relevant metric and time range
3. **Search logs** -- `pup logs search` filtered by service, host, or error status
4. **Check traces** -- `pup apm traces list --service <svc> --status error` for errors
5. **Check incidents** -- `pup incidents list --status active` for open incidents
6. **Who's on call** -- `pup on-call who --team <team>` if escalation is needed

Correlate findings across signals before presenting conclusions.

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| 401 Unauthorized | Token expired | `pup auth refresh` |
| 403 Forbidden | Missing scope | Check app key permissions |
| 404 Not Found | Wrong ID/resource | Verify resource exists |
| Rate limited | Too many requests | Add delays between calls |

## Global Flags

| Flag | Short | Description |
|------|-------|-------------|
| `--output` | `-o` | Output format: `json` (default), `table`, `yaml` |
| `--yes` | `-y` | Skip confirmation prompts (destructive ops) |
| `--agent` | | Force agent mode |
| `--json` | | Shorthand for `-o json` |

## Tips

- Pup auto-detects Claude Code and enables agent mode (structured JSON output).
- Use `pup <command> --help` liberally -- built-in help is always up to date.
- For large result sets, look for `--limit` or pagination flags via `--help`.
- When the user asks about a Datadog product not listed here, run `pup --help`
  to check if a command group exists for it.
