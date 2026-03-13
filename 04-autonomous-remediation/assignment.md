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
    1. Trigger on: `error_rate > 5%` for `checkout-service` for any `merchant.tier == "premium"`
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
  path: /
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

## Step 3 — Review the Elastic Workflow Execution

In Kibana, go to **Stack Management** → **Rules** (or **Observability** → **Rules**) to see the pre-configured alert rules. Each rule is linked to a Workflow that runs on trigger.

Click any triggered rule to see the workflow execution log — each step is recorded with input, output, and timing.

---

## Step 4 — Run the Investigation Yourself

Practice the full ES|QL-based incident investigation workflow:

```esql
FROM logs.otel
| WHERE @timestamp > NOW() - 10 MINUTES
| WHERE severity_text IN ("ERROR", "CRITICAL")
| STATS errors = COUNT(*), affected_merchants = COUNT_DISTINCT(merchant.id)
  BY service.name
| SORT errors DESC
```

```esql
FROM traces-*
| WHERE @timestamp > NOW() - 10 MINUTES AND error == true
| STATS error_spans = COUNT(*), p99 = PERCENTILE(transaction.duration, 99)
  BY service.name
| SORT error_spans DESC
```

---

## Step 5 — Executive Summary

The PayPal Merchant Observability platform you've just explored delivers:

- **CAL replacement** with OTel semantic conventions — no proprietary schema
- **Unified signals** — logs, metrics, and traces in one query engine
- **Merchant-centric investigation** — `merchant.id` as a first-class pivot
- **High-cardinality metrics** — millions of merchant dimensions, no pre-aggregation
- **Autonomous response** — AI Agent + Workflows reduce MTTR without human intervention
- **Zero-ops infrastructure** — Elastic Serverless scales automatically, no cluster management
