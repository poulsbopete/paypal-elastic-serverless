---
slug: autonomous-remediation
id: txxh8fjd5e75
type: challenge
title: Autonomous Investigation & Remediation
teaser: Trigger a complex incident and watch Elastic's AI Agent and Workflows automatically
  investigate, correlate, and remediate without human intervention.
notes:
- type: text
  contents: |
    ## Autonomous Observability at PayPal Scale

    At PayPal, incidents happen at any hour across a global, multi-cloud platform. The mean time to detect (MTTD) and mean time to resolve (MTTR) are directly tied to merchant GMV impact.

    **The challenge:** Traditional alert → human → investigate → resolve loops are too slow at scale.

    **The Elastic answer:** Autonomous workflows that trigger automatically on alert conditions, run investigation playbooks against live telemetry, and execute remediation steps — all without waking up an SRE at 3 AM.

    In this challenge, you'll trigger a multi-service incident and observe the Elastic AI Agent and Workflows handling the full investigation and remediation lifecycle.
- type: text
  contents: |
    ## Elastic Workflows: Alert → Investigate → Remediate

    Elastic Workflows are event-driven automation pipelines built directly into the Elastic platform.

    **For PayPal, a typical workflow might:**
    1. Trigger on: `error_rate > 5%` for `checkout-service` sustained for 2 minutes
    2. Investigate: Run ES|QL to identify affected merchant cohort and root cause service
    3. Correlate: Pull the distributed trace for the highest-impact transaction
    4. Notify: Send a Slack alert with pre-populated incident context
    5. Remediate: Trigger a circuit breaker reset or scale-out action via webhook

    The entire cycle runs in seconds — not the minutes or hours it takes with manual investigation.
tabs:
- id: rgehg1r7tdor
  title: Terminal
  type: terminal
  hostname: es3-api
- id: d8fldomxjrix
  title: Demo App
  type: service
  hostname: es3-api
  path: /home
  port: 8090
- id: cx7pccogcrsi
  title: Elastic Serverless
  type: service
  hostname: es3-api
  path: /app/observability/alerts?_g=(filters:!(),refreshInterval:(pause:!f,value:10000),time:(from:now-1h,to:now))
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

# Autonomous Investigation & Remediation

Trigger a high-impact payment incident and observe Elastic's AI Agent and Workflows executing the full investigation and remediation lifecycle.

---

## Step 1 — Trigger a Multi-Service Incident

Click **Demo App** → **Chaos Controller**. Activate a complex incident such as:
- **Checkout Latency Spike — High Value Merchants**
- **Payment Authorization Failure Spike**
- **Cross-Region Payment Routing Failure**

These scenarios cascade across multiple services, creating the kind of complex, ambiguous failure that is hardest to diagnose manually.

---

## Step 2 — Watch the AI Agent Investigate

Click **Elastic Serverless** → **Observability** → **Alerts**.

When an alert fires, click it and look for the **AI Investigation** panel. The AI Agent will:
1. Query the affected service's recent error logs
2. Correlate with metrics to identify the performance degradation pattern
3. Pull distributed traces to find the root cause service
4. Generate a plain-language incident summary with recommended actions

---

## Step 3 — Review ML Anomaly Detection

Elastic's machine learning jobs are continuously scanning your PayPal transaction telemetry for anomalies.

In Kibana, go to **Machine Learning** → **Anomaly Detection** (left nav, under **Analytics**). You'll see three pre-configured jobs:

| Job | What it detects |
|---|---|
| `paypal-service-error-spike` | Unusually high error rates per service — catches P1 payment failures |
| `paypal-transaction-volume-anomaly` | Traffic drops or surges per service — catches outages and attack patterns |
| `paypal-checkout-degradation` | Error surges specifically in the checkout → payments → fraud pipeline |

Click any job → **View results** → **Anomaly Explorer** to see the anomaly swimlane and severity scores. Inject a fault in the **Demo App** and watch the ML model flag it as an anomaly in real time.

---

## Step 4 — Review the Elastic Workflow Execution

In Kibana, go to **Stack Management** → **Rules** (or **Observability** → **Rules**) to see the pre-configured alert rules. Each rule is linked to a Workflow that runs on trigger.

Click any triggered rule to see the workflow execution log — each step is recorded with input, output, and timing.

---

## Step 4 — Run the Investigation Yourself

Practice the full ES|QL-based incident investigation workflow.

Error surge by service in the last 10 minutes:

```esql
FROM logs.otel
| WHERE @timestamp > NOW() - 10 MINUTES
| WHERE severity_text IN ("ERROR", "CRITICAL")
| STATS errors = COUNT(*), unique_traces = COUNT_DISTINCT(trace.id) BY service.name
| SORT errors DESC
```

Recent error log messages from the top affected service:

```esql
FROM logs.otel
| WHERE @timestamp > NOW() - 10 MINUTES AND severity_text == "ERROR"
| KEEP @timestamp, service.name, body.text, trace.id
| SORT @timestamp DESC
| LIMIT 25
```

Error rate trend over time (spot the fault injection spike):

```esql
FROM logs.otel
| WHERE @timestamp > NOW() - 30 MINUTES
| WHERE severity_text == "ERROR"
| STATS errors = COUNT(*) BY service.name, BUCKET(@timestamp, 1 minute)
| SORT @timestamp DESC
```

---

## Step 5 — Ask the AI Agent for the Full Story

Open the **AI Assistant** in Kibana and copy/paste these questions to close the loop on the entire observability journey:

```
Walk me through everything that happened during the incident we just simulated — detection, root cause, remediation, and recovery.
```

```
What is the current SLO status for each of the 7 PayPal services? Which services are at risk?
```

```
Based on the last 30 minutes of telemetry, which services are the most likely candidates for a future incident?
```

```
If PayPal adopted Elastic Serverless to replace CAL, what would the top 3 operational benefits be based on what you can see in this environment?
```

```
Generate an executive briefing for PayPal leadership that summarizes the value of this observability platform in 3 bullet points.
```

```
How would you build a SLO for the checkout-service with a 99.9% error rate target using what you see in this Elastic environment?
```

```
The ML anomaly detection jobs just ran — which service shows the highest anomaly score and what does that suggest about the health of the payment pipeline?
```

These questions are intentionally executive and forward-looking — ideal for running live in a customer presentation to show Elastic AI reasoning over real PayPal-shaped data.

---

## Step 6 — Executive Summary

The PayPal Merchant Observability platform you've just explored delivers:

- **CAL replacement** with OTel semantic conventions — no proprietary schema
- **Unified signals** — logs, metrics, and traces in one query engine
- **Service-centric investigation** — `service.name`, `trace.id`, `body.text` queryable together
- **High-cardinality telemetry** — ES|QL handles millions of spans and log events without pre-aggregation
- **ML anomaly detection** — Unsupervised models learn normal baselines and alert on deviations automatically
- **Autonomous response** — AI Agent + Workflows reduce MTTR without human intervention
- **Zero-ops infrastructure** — Elastic Serverless scales automatically, no cluster management
