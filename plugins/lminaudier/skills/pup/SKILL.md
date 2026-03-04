---
name: pup
description: >-
  Interact with the Datadog observability platform using the pup CLI.
  Use this skill whenever the user mentions Datadog, monitoring, observability,
  or wants to query metrics, search logs, manage monitors, view dashboards,
  investigate incidents, check SLOs, manage infrastructure tags, or interact
  with any Datadog product. Activates on requests like "check my Datadog monitors",
  "query CPU metrics", "search logs for errors", "list dashboards", "show incidents",
  "what's the SLO status", "list hosts", etc.
allowed-tools:
  - Bash
---

# Pup — Datadog CLI for AI Agents

Pup is a CLI companion that provides 300+ commands across 42 command groups covering
Datadog's observability platform. It returns structured JSON output by default and
auto-detects agent mode when invoked from Claude Code.

## Guardrails

- **Auth first**: always run `pup auth status` before any operation to verify credentials
  are valid. If auth fails, guide the user through `pup auth login` (OAuth2) or
  environment variable setup (`DD_API_KEY` + `DD_APP_KEY` + `DD_SITE`).
- **No blind destructive ops**: never pass `--yes` or set `DD_AUTO_APPROVE=true` unless
  the user explicitly asks to skip confirmations. Always show the resource details
  (get it first) before proposing a delete or modify operation.
- **Prefer JSON output**: use `-o json` (the default) for parsing. Switch to `-o table`
  only when the user wants human-readable display or when piping to column/awk.
- **Scope queries**: always include time ranges (`--from`, `--to`) and filters when
  querying metrics or logs to avoid pulling excessive data.
- **Read before write**: for monitors, dashboards, and SLOs, retrieve the current state
  before proposing changes.

## Command Discovery

Pup is self-documenting. When unsure about a command or its flags:

```bash
# List all top-level command groups
pup --help

# Get help for a specific command group
pup monitors --help

# Get help for a specific subcommand
pup monitors list --help
```

Always use `--help` to discover available flags rather than guessing.

## Authentication

Authentication priority (highest to lowest):
1. `DD_ACCESS_TOKEN` — bearer token
2. OAuth2 tokens — stored in system keychain via `pup auth login`
3. `DD_API_KEY` + `DD_APP_KEY` — API key pair

```bash
# Check current auth status
pup auth status

# Login via OAuth2 (preferred — opens browser)
pup auth login

# Or set environment variables
export DD_API_KEY="<key>"
export DD_APP_KEY="<key>"
export DD_SITE="datadoghq.com"  # or datadoghq.eu, us3.datadoghq.com, etc.
```

## Common Workflows

### Querying Metrics

```bash
# Search for available metrics by name pattern
pup metrics list --filter="system.cpu"

# Query metric values over a time range
pup metrics query --query="avg:system.cpu.user{*}" --from="1h"

# Query with specific host filter
pup metrics query --query="avg:system.cpu.user{host:web-01}" --from="4h"
```

### Searching Logs

```bash
# Search logs with a query
pup logs search --query="service:api status:error" --from="1h"

# Aggregate log counts
pup logs aggregate --query="service:api" --from="1h"
```

### Managing Monitors

```bash
# List all monitors
pup monitors list

# Search monitors by name or tag
pup monitors search --query="cpu"

# Get a specific monitor by ID
pup monitors get 12345678

# Delete a monitor (will prompt for confirmation)
pup monitors delete 12345678
```

### Working with Dashboards

```bash
# List dashboards
pup dashboards list

# Get dashboard details
pup dashboards get <dashboard-id>
```

### Checking SLOs

```bash
# List all SLOs
pup slos list

# Get SLO details and status
pup slos get <slo-id>
```

### Incident Management

```bash
# List open incidents
pup incidents list

# Get incident details
pup incidents get <incident-id>
```

### Infrastructure

```bash
# List hosts
pup infrastructure hosts list

# Get host details
pup infrastructure hosts get <hostname>

# Manage tags
pup tags list --source="users"
pup tags get <hostname>
```

### Security Monitoring

```bash
# List security signals
pup security signals list --from="24h"

# Search security findings
pup security findings list
```

### APM / Services

```bash
# List APM services
pup apm services list

# Get service dependencies
pup apm dependencies list --service="my-service"
```

## Investigating Issues — Recommended Flow

When a user asks to investigate a problem (high CPU, errors, incidents):

1. **Check monitors** — `pup monitors search --query="<relevant term>"` to find
   alerting monitors
2. **Query metrics** — `pup metrics query` with the relevant metric and time range
3. **Search logs** — `pup logs search` filtered by service, host, or error status
4. **Check incidents** — `pup incidents list` to see if an incident is already open
5. **Review dashboards** — `pup dashboards list` to find relevant dashboards, then
   `pup dashboards get` to see widget definitions

Correlate findings across signals before presenting conclusions to the user.

## Global Flags

| Flag | Short | Description |
|------|-------|-------------|
| `--output` | `-o` | Output format: `json` (default), `table`, `yaml` |
| `--yes` | `-y` | Skip confirmation prompts (destructive ops) |
| `--agent` | | Force agent mode |

## Tips

- Pup auto-detects Claude Code and enables agent mode (structured JSON, auto-metadata).
- Use `pup <command> --help` liberally — the built-in help is comprehensive and always
  up-to-date.
- For large result sets, look for `--limit` or pagination flags in the command help.
- When the user asks about a Datadog product you haven't used with pup before,
  run `pup --help` to check if a command group exists for it.
