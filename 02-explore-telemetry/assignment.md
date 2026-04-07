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

    ### O11Y Survivors — play while you wait

    <iframe src="https://poulsbopete.github.io/Vampire-Clone/" width="100%" height="440" style="border:0;border-radius:8px;background:#0f172a" allow="fullscreen; autoplay" title="O11Y Survivors" loading="lazy"></iframe>

    [Open in new tab](https://poulsbopete.github.io/Vampire-Clone/)
tabs:
- id: o11ywaitch02
  title: O11Y Survivors
  type: service
  hostname: es3-api
  port: 8080
  path: /loading
  new_window: true
- id: chqbb7rksvs2
  title: Demo App
  type: service
  hostname: es3-api
  port: 8080
  new_window: true
- id: g3a3ue0le0nj
  title: Elastic Serverless
  type: service
  hostname: es3-api
  path: /app/dashboards#/list?_g=(filters:!(),refreshInterval:(pause:!f,value:30000),time:(from:now-30m,to:now))
  port: 8080
  new_window: true
- id: imavv3tksqks
  title: Terminal
  type: terminal
  hostname: es3-api
difficulty: basic
timelimit: 1200
enhanced_loading: null
---

# Challenge 2 — Explore Financial Platform Telemetry

## The Story

The PayPal Operations Center is live. 9 financial platform services are emitting telemetry: order-gateway and matching-engine handling trades on AWS, fraud-detector and settlement-processor processing on GCP, and compliance-monitor and audit-logger running on Azure.

Your task: explore the data using ES|QL and understand the platform's health.

---

## Step 1 — Order Flow Analysis

In **Elastic Serverless → Analytics → Discover**, switch to ES|QL mode and run:

**Order volume by service and subsystem:**
```esql
FROM paypal-otel-logs
| STATS trades = COUNT(*) BY service.name, subsystem
| SORT trades DESC
```

**Distributed trace reconstruction — follow an order through the system:**
```esql
FROM paypal-otel-logs
| WHERE transaction.name LIKE "*order*"
| STATS span_count = COUNT(*), avg_latency_ms = AVG(transaction.duration.us) / 1000 BY service.name, transaction.name
| SORT avg_latency_ms DESC
```

**Order gateway throughput over time:**
```esql
FROM paypal-otel-logs
| WHERE service.name == "order-gateway"
| STATS orders = COUNT(*) BY bucket = DATE_TRUNC(1 minute, @timestamp)
| SORT bucket DESC
| LIMIT 30
```

---

## Step 2 — Latency & Performance

**P99 latency by service — identify SLA breaches:**
```esql
FROM paypal-otel-logs
| WHERE transaction.duration.us IS NOT NULL
| STATS
    p50_ms = PERCENTILE(transaction.duration.us, 50) / 1000,
    p99_ms = PERCENTILE(transaction.duration.us, 99) / 1000,
    max_ms = MAX(transaction.duration.us) / 1000
  BY service.name
| SORT p99_ms DESC
```

**Matching engine latency (should be < 500µs for SLA compliance):**
```esql
FROM paypal-otel-logs
| WHERE service.name == "matching-engine" AND transaction.duration.us IS NOT NULL
| STATS
    avg_us = AVG(transaction.duration.us),
    p99_us = PERCENTILE(transaction.duration.us, 99),
    count = COUNT(*)
```

**Settlement processor — batch latency (higher is expected, watch for outliers):**
```esql
FROM paypal-otel-logs
| WHERE service.name == "settlement-processor"
| STATS max_latency_ms = MAX(transaction.duration.us) / 1000, count = COUNT(*) BY service.name
```

---

## Step 3 — Error Analysis

**Error rate by service — identify the most impacted services:**
```esql
FROM paypal-otel-logs
| STATS
    total = COUNT(*),
    errors = COUNT(*) WHERE severity_text == "ERROR"
  BY service.name
| EVAL error_pct = ROUND(errors * 100.0 / total, 2)
| SORT error_pct DESC
```

**Find specific financial system errors:**
```esql
FROM paypal-otel-logs
| WHERE severity_text == "ERROR"
| STATS count = COUNT(*) BY body.text
| SORT count DESC
| LIMIT 15
```

**Risk and compliance errors — regulatory impact:**
```esql
FROM paypal-otel-logs
| WHERE severity_text == "ERROR"
  AND (service.name == "risk-calculator" OR service.name == "compliance-monitor" OR service.name == "fraud-detector")
| KEEP @timestamp, service.name, body.text
| SORT @timestamp DESC
| LIMIT 20
```

---

## Step 4 — Multi-Cloud Health

**Health by cloud provider:**
```esql
FROM paypal-otel-logs
| STATS
    total = COUNT(*),
    errors = COUNT(*) WHERE severity_text == "ERROR"
  BY cloud.provider, cloud.region
| EVAL error_rate = ROUND(errors * 100.0 / total, 2)
| SORT error_rate DESC
```

**Instrument trading activity (order instrument distribution):**
```esql
FROM paypal-otel-logs
| WHERE instrument IS NOT NULL
| STATS trades = COUNT(*) BY instrument
| SORT trades DESC
| LIMIT 10
```

---

## Step 5 — AI Agent Questions

Open the Elastic AI Assistant and ask:

> "Using paypal-otel-logs, show me a complete health summary of all 9 trading services: error rate, P99 latency, and total event count for the last 30 minutes."

> "Which instruments and order types are generating the most errors in paypal-otel-logs?"

> "Compare the latency profile of matching-engine vs. settlement-processor. Which has more variance and why would that be expected?"

---

## ✅ Completion Check

You'll pass this challenge when:
- The generator is running
- At least 50 documents exist in `paypal-otel-logs`
- All 9 financial services are present
- Error telemetry exists in the index
