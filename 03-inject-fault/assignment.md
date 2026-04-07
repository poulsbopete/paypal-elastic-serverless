---
slug: inject-fault
id: 5wgfot0esuqz
type: challenge
title: Inject Fault — Financial Incident Detection
teaser: Use the Chaos Controller to inject a trading platform fault and watch Elastic
  detect it automatically.
notes:
- type: text
  contents: |
    ## Financial Incident Simulation

    The PayPal trading platform handles billions of dollars in daily order flow. A single fault in the matching engine or order gateway can cascade into settlement failures and regulatory reporting gaps.

    In this challenge, you'll use the **Chaos Controller** to inject a financial incident and observe how Elastic Serverless detects it in real-time.

    ### O11Y Survivors — play while you wait

    <iframe src="https://poulsbopete.github.io/Vampire-Clone/" width="100%" height="440" style="border:0;border-radius:8px;background:#0f172a" allow="fullscreen; autoplay" title="O11Y Survivors" loading="lazy"></iframe>

    [Open in new tab](https://poulsbopete.github.io/Vampire-Clone/)
tabs:
- id: o11ywaitch03
  title: O11Y Survivors
  type: service
  hostname: es3-api
  port: 8080
  path: /loading
  new_window: true
- id: ywvlnwwrjn9q
  title: Demo App
  type: service
  hostname: es3-api
  port: 8080
  new_window: true
- id: lzlq9yaywlmx
  title: Elastic Serverless
  type: service
  hostname: es3-api
  path: /app/observability/alerts?_g=(filters:!(),refreshInterval:(pause:!f,value:30000),time:(from:now-30m,to:now))
  port: 8080
  new_window: true
- id: mns4y27ssjnf
  title: Terminal
  type: terminal
  hostname: es3-api
difficulty: basic
timelimit: 1200
enhanced_loading: null
---

# Challenge 3 — Inject a Financial Fault

## The Scenario

A rogue trading algorithm has started placing orders that stress the matching engine's order book. Bid/ask spreads are widening, and replication between the primary and replica shards is falling behind.

**Your mission:** Inject the fault, observe the error spike, and let Elastic's alerting + AI Assistant help you diagnose the root cause.

---

## Step 1 — Inject the Fault

In the **Demo App** tab:

1. Click **Chaos Controller** (or the "Market Disruption Simulator" button)
2. Select **Channel 1 — Order Book Inconsistency** from the fault channel list
3. Click **Inject Fault**
4. Observe the service health tiles change from green → red/orange

The fault injects `OMS-BOOK-IMBALANCE` errors into the order-gateway and matching-engine services.

---

## Step 2 — Watch Elastic Detect It

Switch to the **Elastic Serverless** tab. The alerts page should show active alerts firing within 1–2 minutes.

You should see:
- **PayPal — Order Gateway Error Spike** alert → ACTIVE
- **PayPal — High Error Rate (any service)** alert → ACTIVE

If alerts aren't yet visible, run this ES|QL query in **Discover** to confirm errors are flowing:

```esql
FROM paypal-otel-logs
| WHERE severity_text == "ERROR"
  AND (service.name == "order-gateway" OR service.name == "matching-engine")
| STATS errors = COUNT(*) BY bucket = DATE_TRUNC(1 minute, @timestamp)
| SORT bucket DESC
| LIMIT 10
```

---

## Step 3 — Investigate with ES|QL

**Find the specific OMS fault signatures:**
```esql
FROM paypal-otel-logs
| WHERE body.text LIKE "*OMS-BOOK-IMBALANCE*" OR body.text LIKE "*ME-LATENCY-SLA*"
| KEEP @timestamp, service.name, body.text, severity_text
| SORT @timestamp DESC
| LIMIT 20
```

**Measure the error rate spike:**
```esql
FROM paypal-otel-logs
| WHERE @timestamp > NOW() - 15 minutes
| STATS
    total = COUNT(*),
    errors = COUNT(*) WHERE severity_text == "ERROR"
  BY service.name
| EVAL error_pct = ROUND(errors * 100.0 / total, 2)
| WHERE error_pct > 0
| SORT error_pct DESC
```

**Trace cascade impact — did matching engine errors affect settlement?**
```esql
FROM paypal-otel-logs
| WHERE severity_text == "ERROR"
  AND @timestamp > NOW() - 15 minutes
| STATS count = COUNT(*) BY service.name
| SORT count DESC
```

**Compare error rates: before vs. during the fault:**
```esql
FROM paypal-otel-logs
| WHERE service.name IN ("order-gateway", "matching-engine", "risk-calculator")
| STATS errors = COUNT(*) WHERE severity_text == "ERROR"
  BY service.name, bucket = DATE_TRUNC(2 minutes, @timestamp)
| SORT bucket DESC, errors DESC
```

---

## Step 4 — Check the ML Anomaly Jobs

Navigate to **Kibana → Machine Learning → Anomaly Detection**. Look for:
- `paypal-service-error-spike` — should show an anomaly score spike on order-gateway
- `paypal-trading-pipeline-degradation` — elevated anomaly score across the order execution pipeline

---

## Step 5 — AI Agent Root Cause Analysis

Open the **Elastic AI Assistant** and ask:

> "I'm seeing OMS-BOOK-IMBALANCE errors in paypal-otel-logs for the order-gateway and matching-engine services starting in the last 15 minutes. What is the likely root cause and which other services are at risk of cascading failure?"

> "Query paypal-otel-logs for error patterns in the last 10 minutes. Show me the error rate per service and identify the primary fault source."

> "The matching engine has elevated latency. What does 'OMS-BOOK-IMBALANCE' mean in a trading system context and what immediate remediation steps should we take?"

---

## Step 6 — Automated Remediation

Elastic Workflows detect the fault and can automatically:
- Page the on-call SRE via PagerDuty/Slack
- Trigger a circuit breaker to halt new order intake
- Initiate order book reconciliation procedures
- File a regulatory incident report

Navigate to **Kibana → Observability → Alerts** to see the active incidents and triggered workflow actions.

---

## ✅ Completion Check

You'll pass this challenge when:
- The demo app is healthy
- A fault has been injected via the Chaos Controller
- Error telemetry from the fault is present in `paypal-otel-logs`
