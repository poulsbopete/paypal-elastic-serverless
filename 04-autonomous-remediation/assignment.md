---
slug: autonomous-remediation
id: xyzabc123def
type: challenge
title: Autonomous Remediation — AI-Driven Incident Response
teaser: Resolve the financial fault and explore how Elastic's AI and ML deliver autonomous remediation for PayPal-scale trading operations.
notes:
  - type: text
    contents: |
      ## Autonomous Remediation

      You've observed a fault cascade across the trading pipeline. Now it's time to resolve it and explore how Elastic's built-in AI and ML capabilities can autonomously detect, diagnose, and remediate financial incidents.

      **In this challenge:**
      - Resolve the active fault channels
      - Explore ML anomaly job results
      - Review SLO burn rates
      - Use the AI Assistant for deep RCA
      - Understand the autonomous workflow architecture
tabs:
  - id: 1bvf5tqoejoy
    title: Demo App
    type: service
    hostname: es3-api
    port: 8080
    new_window: true
  - id: 4jnxpfyebyjl
    title: Elastic Serverless
    type: service
    hostname: es3-api
    path: /app/observability/alerts?_g=(filters:!(),refreshInterval:(pause:!f,value:30000),time:(from:now-30m,to:now))
    port: 8080
    new_window: true
  - id: 7k2mplqavjol
    title: Terminal
    type: terminal
    hostname: es3-api
difficulty: basic
timelimit: 1200
---

# Challenge 4 — Autonomous Remediation

## Step 1 — Resolve the Fault

In the **Demo App** tab → **Chaos Controller**:
1. Find the active fault channel (e.g., Channel 1 — Order Book Inconsistency)
2. Click **Resolve** to clear the fault
3. Observe the service health tiles return to green

The matching engine's order book is now re-synced. Watch the error rate drop in real-time.

---

## Step 2 — Recovery Telemetry Analysis

In **Elastic Serverless → Discover** (ES|QL mode):

**Observe the recovery — error rate should be dropping:**
```esql
FROM paypal-otel-logs
| WHERE @timestamp > NOW() - 30 minutes
| STATS errors = COUNT(*) WHERE severity_text == "ERROR"
  BY service.name, bucket = DATE_TRUNC(2 minutes, @timestamp)
| SORT bucket DESC, errors DESC
| LIMIT 30
```

**Full incident timeline — what happened and when:**
```esql
FROM paypal-otel-logs
| WHERE severity_text == "ERROR"
| STATS
    first_error = MIN(@timestamp),
    last_error = MAX(@timestamp),
    peak_errors = COUNT(*)
  BY service.name
| SORT peak_errors DESC
```

**Latency recovery — confirm matching engine is back within SLA:**
```esql
FROM paypal-otel-logs
| WHERE service.name == "matching-engine" AND transaction.duration.us IS NOT NULL
| STATS p99_ms = PERCENTILE(transaction.duration.us, 99) / 1000
  BY bucket = DATE_TRUNC(2 minutes, @timestamp)
| SORT bucket DESC
| LIMIT 15
```

---

## Step 3 — ML Anomaly Detection Review

Navigate to **Kibana → Machine Learning → Anomaly Detection**.

Review the three anomaly jobs:

| Job | Purpose | Expected Anomaly Score |
|-----|---------|----------------------|
| `paypal-service-error-spike` | Error rate spikes per service | High during fault |
| `paypal-trade-volume-anomaly` | Order volume drops or surges | Medium during fault |
| `paypal-trading-pipeline-degradation` | Order gateway + matching engine | High during fault |

Ask the AI Assistant to explain the anomaly:

> "The ML job paypal-service-error-spike detected an anomaly in the order-gateway service. Explain what this means and what it tells us about the health of the trading pipeline."

---

## Step 4 — SLO Burn Rate Analysis

Navigate to **Kibana → Observability → SLOs**.

Review the three SLOs created for the financial platform:

| SLO | Target | Indicator |
|-----|--------|-----------|
| **Availability SLO** | 95% | NOT event.outcome:failure |
| **Latency SLO** | 85% requests < 2s | transaction.duration.us ≤ 2,000,000 |
| **Error Rate SLO** | 95% | NOT event.outcome:failure |

Check the **burn rate** for each SLO — did the fault burn into our error budget?

---

## Step 5 — Advanced ES|QL Analytics

**Build an incident MTTR report:**
```esql
FROM paypal-otel-logs
| STATS
    total_events = COUNT(*),
    error_events = COUNT(*) WHERE severity_text == "ERROR",
    services_affected = COUNT_DISTINCT(service.name) WHERE severity_text == "ERROR"
| EVAL overall_error_rate = ROUND(error_events * 100.0 / total_events, 2)
```

**Cross-cloud incident impact (did one cloud provider take more errors?):**
```esql
FROM paypal-otel-logs
| STATS
    total = COUNT(*),
    errors = COUNT(*) WHERE severity_text == "ERROR"
  BY cloud.provider
| EVAL error_rate = ROUND(errors * 100.0 / total, 2)
| SORT error_rate DESC
```

**Settlement impact — did any T+2 settlements miss their deadline?**
```esql
FROM paypal-otel-logs
| WHERE service.name == "settlement-processor" AND severity_text == "ERROR"
| STATS count = COUNT(*), error_types = VALUES(body.text) BY service.name
```

**Order types most impacted during fault:**
```esql
FROM paypal-otel-logs
| WHERE severity_text == "ERROR" AND order.type IS NOT NULL
| STATS errors = COUNT(*) BY order.type
| SORT errors DESC
```

---

## Step 6 — AI-Powered Root Cause Analysis

Ask the Elastic AI Assistant these questions for a complete executive summary:

> "Generate a complete incident report for the last 30 minutes from paypal-otel-logs. Include: services affected, peak error rates, latency impact, estimated duration, and recommended preventive measures."

> "Based on the telemetry in paypal-otel-logs, did the order-gateway fault cascade to the matching-engine and risk-calculator? Show evidence from the data."

> "What Elastic features (ML anomaly detection, SLOs, alerting, workflows) would provide the fastest time-to-detect for an OMS-BOOK-IMBALANCE event? Recommend an optimal monitoring setup."

> "Compare PayPal's current CAL logging architecture vs. the OpenTelemetry + Elastic Serverless approach demonstrated here. What are the top 3 operational benefits for PayPal's financial platform?"

---

## Step 7 — The Autonomous Architecture

**How Elastic enables zero-touch remediation for PayPal:**

1. **Detection** — ML anomaly jobs detect error spikes within 1 bucket span (5 minutes)
2. **Alerting** — Custom threshold rules fire immediately when errors exceed baseline
3. **AI Diagnosis** — AI Assistant provides instant root cause analysis across all 9 services
4. **Workflows** — Automated remediation: circuit breakers, incident tickets, runbook execution
5. **SLO Tracking** — Real-time burn rate monitoring with proactive escalation

This replaces the manual, CAL-dependent process that required 3–5 systems and 30+ minutes to diagnose.

---

## ✅ Completion Check

You'll pass this challenge when:
- The demo app is healthy
- The generator is still running
- No active fault channels remain
- Substantial telemetry has accumulated in `paypal-otel-logs`
