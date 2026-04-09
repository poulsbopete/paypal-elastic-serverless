---
slug: explore-telemetry
id: ylcnef3giyz0
type: challenge
title: Explore Telemetry — Trading Platform Observability
teaser: Investigate order flow, latency, and risk events across 9 financial services
  using ES|QL.
notes:
- type: text
  contents: |
    ## Explore Financial Platform Telemetry

    You now have live data from 9 trading services. In this challenge you'll use ES|QL to explore the full observability picture — order flow, latency distribution, error patterns, and subsystem health.

    This is what replacing CAL looks like: **one query language across all signals**.
tabs:
- id: mkivx88pvqg4
  title: Demo App
  type: service
  hostname: es3-api
  path: /
  port: 8090
- id: yioapkasfcj7
  title: Chaos Controller
  type: service
  hostname: es3-api
  path: /chaos
  port: 8090
- id: mz3qbyyy9bpl
  title: Elastic Serverless
  type: service
  hostname: es3-api
  path: /app/dashboards#/list?_g=(filters:!(),refreshInterval:(pause:!f,value:30000),time:(from:now-30m,to:now))
  port: 8080
  custom_request_headers:
  - key: Content-Security-Policy
    value: 'script-src ''self'' https://kibana.estccdn.com; worker-src blob: ''self'';
      style-src ''unsafe-inline'' ''self'' https://kibana.estccdn.com; style-src-elem
      ''unsafe-inline'' ''self'' https://kibana.estccdn.com'
  custom_response_headers:
  - key: Content-Security-Policy
    value: 'script-src ''self'' https://kibana.estccdn.com; worker-src blob: ''self'';
      style-src ''unsafe-inline'' ''self'' https://kibana.estccdn.com; style-src-elem
      ''unsafe-inline'' ''self'' https://kibana.estccdn.com'
- id: imavv3tksqks
  title: Terminal
  type: terminal
  hostname: es3-api
difficulty: basic
timelimit: 1200
enhanced_loading: null
---

# Explore Live OpenTelemetry Telemetry

Now that your scenario is running, let's explore the data flowing into Elastic. Open the **Elastic Serverless** tab.

---

## What's Generating Telemetry

The platform runs several background generators simultaneously:

| Generator | What It Produces |
|-----------|-----------------|
| **9 scenario services** | Application logs, traces, errors |
| **Host metrics** | CPU, memory, disk, network for 3 cloud hosts |
| **Kubernetes metrics** | Node, pod, container metrics |
| **Nginx metrics + logs** | Access logs, error logs, request spans |
| **MySQL logs** | Slow query + error logs |
| **VPC flow logs** | Network flow telemetry |
| **Distributed traces** | Multi-service request chains |

---

## Explore #1 — Logs via ES|QL

1. In the **Elastic Serverless** tab → **Discover**
2. Switch to **ES|QL** mode (top-left toggle)
3. Run this query:

```esql
FROM logs*
| WHERE @timestamp > NOW() - 5 MINUTES
| KEEP service.name, body.text, severity_text, @timestamp
| SORT @timestamp DESC
| LIMIT 50
```

You should see a stream of logs from multiple services. Once you confirm data is flowing, try filtering to errors only:

```esql
FROM logs*
| WHERE @timestamp > NOW() - 15 MINUTES
| WHERE severity_text == "ERROR"
| STATS error_count = COUNT(*) BY service.name
| SORT error_count DESC
```

**Things to notice:**
- `service.name`: `auction-engine`, `card-printing-system`, `digital-marketplace`, `packaging-fulfillment`, `cloud-inventory-scanner`, `nginx-proxy`, `mysql-primary`, and more
- `severity_text`: `INFO`, `WARN`, `ERROR`
- `body.text` contains the raw log message and error type

---

## Explore #2 — APM / Services

1. In the **Elastic Serverless** tab → **Applications → Service inventory**
2. You should see **7 services** — the 5 application services plus `nginx-proxy` and `mysql-primary`
   > The remaining 2 network/infrastructure services (`wifi-controller`, `network-controller`, `firewall-gateway`, `dns-dhcp-service`) emit logs only — no traces — so they won't appear here
3. Click any service to see latency, throughput, and error rate
4. Open a transaction to see the **distributed trace waterfall**

---

## Explore #3 — Infrastructure / Hosts

1. In the **Elastic Serverless** tab → **Observability → Infrastructure**
2. You should see 3 hosts — one per cloud provider:
   - `fanatics-aws-host-01`
   - `fanatics-gcp-host-01`
   - `fanatics-azure-host-01`
3. Click a host to see CPU, memory, disk, and network metrics

> **Note:** If hosts don't appear immediately, wait 1–2 minutes for the host metrics generator to send its first batch.

---

## Explore #4 — Dashboards

The deployer created an **Executive Dashboard** pre-configured for your scenario. Find it in:

**Elastic Serverless** tab → **Dashboards** → search "Fanatics" (or "Executive")

---

## Explore #5 — ES|QL Time Series Queries

In the **Elastic Serverless** tab → **Discover** → **ES|QL** mode, try these queries against the live metrics stream.

### Auction health at a glance
```esql
TS metrics*
| WHERE @timestamp > NOW() - 15 MINUTES
| EVAL minute = DATE_TRUNC(1 minute, @timestamp)
| STATS
    active_auctions = AVG(auction.active_auctions),
    bid_latency_ms  = AVG(auction.bid_latency_ms),
    bids_per_min    = AVG(auction.bids_per_min),
    websocket_conns = AVG(auction.websocket_connections)
  BY minute
| SORT minute DESC
```

### Spot a latency spike before users notice
```esql
TS metrics*
| WHERE @timestamp > NOW() - 30 MINUTES
| EVAL minute = DATE_TRUNC(1 minute, @timestamp)
| STATS avg_latency = AVG(auction.bid_latency_ms) BY minute
| EVAL status = CASE(
    avg_latency > 45, "🔴 CRITICAL",
    avg_latency > 30, "🟡 DEGRADED",
    "🟢 HEALTHY"
  )
| SORT minute DESC
```

### Card printing throughput vs queue depth
```esql
TS metrics*
| WHERE @timestamp > NOW() - 20 MINUTES
| EVAL minute = DATE_TRUNC(1 minute, @timestamp)
| STATS
    queue_depth    = AVG(card_printing.queue_depth),
    throughput     = AVG(card_printing.throughput),
    substrate_temp = AVG(card_printing.substrate_temp)
  BY minute
| EVAL backlog_ratio = ROUND(queue_depth / throughput, 2)
| SORT minute DESC
```

### Multi-cloud compliance drift
```esql
TS metrics*
| WHERE @timestamp > NOW() - 30 MINUTES
| EVAL bucket5m = DATE_TRUNC(5 minutes, @timestamp)
| STATS
    aws_compliance   = AVG(cloud_inventory.aws.compliance_pct),
    gcp_compliance   = AVG(cloud_inventory.gcp.compliance_pct),
    azure_compliance = AVG(cloud_inventory.azure.compliance_pct)
  BY bucket5m
| EVAL lowest = LEAST(aws_compliance, gcp_compliance, azure_compliance)
| EVAL at_risk_cloud = CASE(
    lowest == aws_compliance, "AWS",
    lowest == gcp_compliance, "GCP",
    "Azure"
  )
| SORT bucket5m DESC
```

### Log volume by service and severity over time
```esql
FROM logs*
| WHERE @timestamp > NOW() - 30 MINUTES
| EVAL minute = DATE_TRUNC(1 minute, @timestamp)
| STATS
    errors = COUNT(*) WHERE severity_text == "ERROR",
    warnings = COUNT(*) WHERE severity_text == "WARN",
    total  = COUNT(*)
  BY minute, service.name
| SORT minute DESC, total DESC
```

> **Tip:** After triggering a chaos fault in the next challenge, re-run this query to watch the error count spike for the affected service in real time — while healthy services stay flat.

---

## Check Your Work

In the **Terminal** tab, run:

```bash
demo-deployments
```

This shows the active deployment with its scenario, namespace, Confirm it matches what you see in Elastic Serverless.

You can also run:
```bash
demo-status
```

To see all 9 services (7 with traces, 4 network/infra services with logs only) and their telemetry send rates.

✅ **Ready to continue when** you've seen logs, traces, or metrics in Elastic Serverless and confirmed services are healthy.
