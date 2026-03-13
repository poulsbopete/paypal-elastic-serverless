---
slug: explore-telemetry
id: eqaxxn8koq3y
type: challenge
title: CAL Log Replacement — Explore OTel Telemetry
teaser: Query merchant logs, traces, and metrics using ES|QL to experience how OpenTelemetry
  replaces CAL with a unified, queryable schema.
notes:
- type: text
  contents: |
    ## Why CAL Needs Replacing

    CAL has served PayPal well, but it was designed for a different era. When a merchant opens a support ticket about a failed transaction, a PayPal SRE today must:

    1. Find the transaction ID in CAL logs (specialist query syntax required)
    2. Switch to a separate trace tool to correlate the log to a distributed trace
    3. Cross-reference yet another tool for service metrics
    4. Manually reconstruct the timeline across all three

    With Elastic + OTel, all three signal types share the same schema. `trace.id` is native. `merchant.id` is a first-class attribute. One query engine handles everything.
- type: text
  contents: |
    ## OTel Semantic Conventions at PayPal

    OpenTelemetry semantic conventions provide a shared field vocabulary across all services:

    | Field | Meaning | Example |
    |---|---|---|
    | `service.name` | Which microservice emitted this signal | `checkout-service` |
    | `merchant.id` | The merchant affected by this event | `M-98231` |
    | `transaction.id` | The payment transaction being processed | `TXN-44821903` |
    | `trace.id` | Distributed trace spanning all services | `4f8a2c...` |
    | `severity_text` | Log level | `ERROR` |
    | `cloud.region` | Where the service is running | `us-east-1` |
    | `http.response.status_code` | HTTP response code | `503` |

    Every service in this lab emits all of these fields automatically via OTel SDK auto-instrumentation.
tabs:
- id: l4xw8tzn8iec
  title: Terminal
  type: terminal
  hostname: es3-api
- id: rtkpw3ta6y5i
  title: Demo App
  type: service
  hostname: es3-api
  path: /
  port: 8090
- id: dbuch240epcz
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
difficulty: basic
timelimit: 0
enhanced_loading: null
---

# CAL Log Replacement — Explore Live OTel Telemetry

With live OTLP telemetry flowing from all 7 PayPal microservices, let's demonstrate what CAL replacement looks like in practice.

---

## Step 1 — Find Errors Across All Services

Open the **Elastic Serverless** tab → **Discover** → switch data view to `logs.otel`.

Run this ES|QL query to see errors by service:

```esql
FROM logs.otel
| WHERE severity_text == "ERROR"
| STATS error_count = COUNT(*) BY service.name
| SORT error_count DESC
```

This is the equivalent of a CAL query but with zero specialist syntax — and it covers all 7 services simultaneously.

---

## Step 2 — Investigate a Specific Merchant

Simulate a merchant complaint. Run the following query to see all errors for a specific merchant:

```esql
FROM logs.otel
| WHERE merchant.id IS NOT NULL AND severity_text == "ERROR"
| STATS errors = COUNT(*) BY merchant.id, service.name
| SORT errors DESC
| LIMIT 10
```

Pick any `merchant.id` from the results and drill in:

```esql
FROM logs.otel
| WHERE merchant.id == "<paste merchant id here>"
| KEEP @timestamp, service.name, severity_text, body.text, trace.id
| SORT @timestamp DESC
| LIMIT 20
```

---

## Step 3 — Trace a Transaction End-to-End

Take any `trace.id` from the results above and view the full distributed trace:

In Kibana, go to **APM** → **Traces** and paste the `trace.id` into the search box. You'll see every service hop that participated in that transaction — something impossible with CAL alone.

---

## Step 4 — Check the Service Map

In Kibana, go to **APM** → **Service Map**. You should see all 7 PayPal microservices with live dependency lines. Hover any service to see its P99 latency and error rate.
