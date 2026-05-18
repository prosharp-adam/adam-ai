# Adam — Monitoring Stack

Adam ships with a self-contained Prometheus + Loki + Grafana stack defined in
`docker-compose.yml` (and the dev-time `docker-compose.local.yml`). Bringing
the stack up gives you a ready-to-use dashboard for tracking sync progress,
recent items per connector, and exceptions filtered out of the wall of logs.

## What's wired up

| Container         | Role                                                                                                                                                              | Port              |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------- |
| `adam-prometheus` | Time-series store. **OTLP push receiver** is enabled, so the .NET hosts (`adam-api`, `adam-worker`, `adam-mcpserver`) send metrics directly — no scraping needed. | 9090              |
| `adam-loki`       | Log store.                                                                                                                                                        | 3100              |
| `adam-promtail`   | Tails Docker container stdout/stderr for any container with the label `logging=adam` and ships entries to Loki. Multi-line .NET stack traces are merged.          | 9080 (in-network) |
| `adam-grafana`    | Pre-provisioned with both datasources and the **Adam → Sync Overview** dashboard.                                                                                 | 3005              |

Login to Grafana with `admin` / `${GRAFANA_PASSWORD}` (default `admin`).

## How application metrics get into Prometheus

`Adam.Aspire.Host.ServiceDefaults.Extensions.AddOpenTelemetryExporters()` reads
the configuration key `OpenTelemetry:Prometheus:OtlpEndpoint`. When that value
is set, it adds an OTLP HTTP-protobuf metrics exporter pointed at Prometheus's
`/api/v1/otlp/v1/metrics` endpoint.

The compose files set this env var on `adam-api`, `adam-worker`, and
`adam-mcpserver`, plus `OTEL_SERVICE_NAME` (so each shows up as a distinct
`job` label in Prometheus).

OTLP counter / histogram naming follows the Prometheus OTLP convention:

| OpenTelemetry instrument          | Prometheus series                    |
| --------------------------------- | ------------------------------------ |
| `adam.tickets.synced` (counter)   | `adam_tickets_synced_total`          |
| `adam.documents.synced` (counter) | `adam_documents_synced_total`        |
| `adam.pullrequests.synced`        | `adam_pullrequests_synced_total`     |
| `adam.sync.errors`                | `adam_sync_errors_total`             |
| `adam.embedding.queue_depth`      | `adam_embedding_queue_depth` (gauge) |

Sync counters are tagged with **`source`** (`jira`, `clickup`, `azuredevops`,
`confluence`, `github`, `googledrive`) — see the `source` dimension in the
dashboard's per-source breakdowns.

## How "last processed item" lists work

Each sync pipeline (`TicketSyncPipeline`, `DocumentSyncPipeline`,
`PullRequestSyncPipeline`) emits a structured info log per item it actually
syncs:

```
ItemSynced kind=ticket source=jira id=jira:OJ-123
ItemSynced kind=pullrequest source=azuredevops id=azuredevops:pr:7464
```

Promtail captures these from `adam-worker`'s stdout, and the dashboard's
**Last processed items per source** panel uses Loki's `regexp` filter to
extract the `kind`, `source`, and `id` fields so you can see exactly which
Jira keys / PR numbers / document IDs were just processed.

## How "exceptions filtered" works

The dashboard's **Exceptions & failures** panel queries Loki for log lines
whose .NET log prefix is `fail:` or `crit:`, plus anything containing
"Exception", "stack trace", or "unhandled" (case-insensitive). Promtail's
multiline stage merges indented continuation lines so you see the full stack
trace as a single entry.

## Files

```
monitoring/
├── grafana/
│   └── provisioning/
│       ├── datasources/datasources.yml          # Prometheus + Loki, auto-provisioned
│       └── dashboards/
│           ├── dashboards.yml                   # provider config
│           └── adam-overview.json               # the dashboard itself
├── loki/loki-config.yml                         # single-binary, filesystem store, 7-day retention
├── promtail/promtail-config.yml                 # Docker SD, label filter logging=adam
└── prometheus/
    ├── prometheus.yml                           # OTLP receiver enabled via CLI flag
    └── rules/                                   # add recording / alerting rules here
```

## Modifying the dashboard

The dashboard is provisioned with `allowUiUpdates: true`, so you can edit it
in Grafana. To make changes permanent, export the JSON
(_Dashboard settings → JSON Model_) and overwrite
`monitoring/grafana/provisioning/dashboards/adam-overview.json`.
