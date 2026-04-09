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

# Challenge 2 — Explore Financial Platform Telemetry

## The Story

The PayPal Operations Center is live. 9 financial platform services are emitting telemetry: order-gateway and matching-engine handling trades on AWS, fraud-detector and settlement-processor processing on GCP, and compliance-monitor and audit-logger running on Azure.

Your task: explore the data using ES|QL and understand the platform's health.

---

## Step 1 — Order Flow Analysis

In **Elastic Serverless → Analytics → Discover**, switch to ES|QL mode and run:

**Order volume by service and subsystem:**
```esql
FROM logs.otel
| STATS trades = COUNT(*) BY resource.attributes.service.name, attributes.subsystem
| SORT trades DESC
```

**Distributed trace reconstruction — follow an order through the system (spans):**
```esql
FROM traces.otel
| WHERE transaction.name LIKE "*order*" OR span.name LIKE "*order*"
| STATS span_count = COUNT(*), avg_latency_ms = AVG(transaction.duration.us) / 1000 BY resource.attributes.service.name, transaction.name
| SORT avg_latency_ms DESC
```

**Order gateway throughput over time:**
```esql
FROM logs.otel
| WHERE resource.attributes.service.name == "order-gateway"
| STATS orders = COUNT(*) BY bucket = DATE_TRUNC(1 minute, @timestamp)
| SORT bucket DESC
| LIMIT 30
```

---

## Step 2 — Latency & Performance

**P99 latency by service — identify SLA breaches (trace spans):**
```esql
FROM traces.otel
| WHERE transaction.duration.us IS NOT NULL
| STATS
    p50_ms = PERCENTILE(transaction.duration.us, 50) / 1000,
    p99_ms = PERCENTILE(transaction.duration.us, 99) / 1000,
    max_ms = MAX(transaction.duration.us) / 1000
  BY resource.attributes.service.name
| SORT p99_ms DESC
```

**Matching engine latency (should be < 500µs for SLA compliance):**
```esql
FROM traces.otel
| WHERE resource.attributes.service.name == "matching-engine" AND transaction.duration.us IS NOT NULL
| STATS
    avg_us = AVG(transaction.duration.us),
    p99_us = PERCENTILE(transaction.duration.us, 99),
    count = COUNT(*)
```

**Settlement processor — batch latency (higher is expected, watch for outliers):**
```esql
FROM traces.otel
| WHERE resource.attributes.service.name == "settlement-processor"
| STATS max_latency_ms = MAX(transaction.duration.us) / 1000, count = COUNT(*) BY resource.attributes.service.name
```

---

## Step 3 — Error Analysis

**Error rate by service — identify the most impacted services:**
```esql
FROM logs.otel
| STATS
    total = COUNT(*),
    errors = COUNT(*) WHERE severity_text == "ERROR"
  BY resource.attributes.service.name
| EVAL error_pct = ROUND(errors * 100.0 / total, 2)
| SORT error_pct DESC
```

**Find specific financial system errors:**
```esql
FROM logs.otel
| WHERE severity_text == "ERROR"
| STATS count = COUNT(*) BY body.text
| SORT count DESC
| LIMIT 15
```

**Risk and compliance errors — regulatory impact:**
```esql
FROM logs.otel
| WHERE severity_text == "ERROR"
  AND (resource.attributes.service.name == "risk-calculator" OR resource.attributes.service.name == "compliance-monitor" OR resource.attributes.service.name == "fraud-detector")
| KEEP @timestamp, resource.attributes.service.name, body.text
| SORT @timestamp DESC
| LIMIT 20
```

---

## Step 4 — Multi-Cloud Health

**Health by cloud provider:**
```esql
FROM logs.otel
| STATS
    total = COUNT(*),
    errors = COUNT(*) WHERE severity_text == "ERROR"
  BY resource.attributes.cloud.provider, resource.attributes.cloud.region
| EVAL error_rate = ROUND(errors * 100.0 / total, 2)
| SORT error_rate DESC
```

**Instrument trading activity (order instrument distribution):**
```esql
FROM logs.otel
| WHERE attributes.instrument IS NOT NULL
| STATS trades = COUNT(*) BY attributes.instrument
| SORT trades DESC
| LIMIT 10
```

---

## Step 5 — AI Agent Questions

Open the Elastic AI Assistant and ask:

> "Using OTLP logs (`logs.otel`) and traces (`traces.otel`), show me a complete health summary of all 9 trading services: error rate, P99 latency, and total event count for the last 30 minutes."

> "Which instruments and order types are generating the most errors in OTLP logs (`logs.otel`)? (Hint: inspect log record attribute fields in the Discover field list.)"

> "Compare the latency profile of matching-engine vs. settlement-processor. Which has more variance and why would that be expected?"

---

## ✅ Completion Check

You'll pass this challenge when:
- The generator is running
- At least 50 documents exist in OTLP log streams (`logs.otel`)
- All 9 financial services are present
- Error telemetry exists in the index
