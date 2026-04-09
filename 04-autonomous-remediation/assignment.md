---
slug: autonomous-remediation
id: 0h7rr7klrbwo
type: challenge
title: Autonomous Remediation — AI-Driven Incident Response
teaser: Resolve the financial fault and explore how Elastic's AI and ML deliver autonomous
  remediation for PayPal-scale trading operations.
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
- id: jz1edp5lfpmx
  title: Demo App
  type: service
  hostname: es3-api
  path: /
  port: 8090
- id: t4y8e30eexkg
  title: Chaos Controller
  type: service
  hostname: es3-api
  path: /chaos
  port: 8090
- id: 3o66hyfwxfwl
  title: Elastic Serverless
  type: service
  hostname: es3-api
  path: /app/observability/alerts?_g=(filters:!(),refreshInterval:(pause:!f,value:30000),time:(from:now-30m,to:now))
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
- id: mnxdlxrrtho0
  title: Terminal
  type: terminal
  hostname: es3-api
difficulty: basic
timelimit: 1200
enhanced_loading: null
---

# Autonomous Investigation & Remediation

In this final challenge, you'll watch Elastic autonomously investigate the fault you injected and remediate it — then verify the full loop has closed.

---

## Step 1 — Watch the Alert Fire

First, confirm the alert rule detected your fault:

**Observability → Alerts**

Within 30–60 seconds of triggering a fault you should see an active alert. If no alert appears yet, wait a minute — the ES|QL rules run on a 30-second interval.

---

## Step 2 — Watch the Workflow Execute

Once an alert fires, the **Significant Event Notification** workflow runs automatically. In the **Elastic Serverless** tab:

**Observability → Workflows**

Find **Fanatics Collectibles Significant Event Notification** and click **Executions** to see:
- `count_errors` — ES|QL query counting recent errors
- `run_rca` — AI agent root-cause analysis
- `create_case` — Kibana case created with RCA findings

> **Note:** The **Remediation Action** workflow is a separate workflow called on-demand — it won't show executions here unless you manually trigger it.

---

## Step 3 — Review the AI Agent Investigation & Remediation

In the workflow execution log, expand the **run_rca** step to see what the AI agent did:

- **Error identification** — which service, which error type
- **Log correlation** — how many errors in the detection window
- **Root cause hypothesis** — what the AI agent determined
- **Remediation executed** — the agent automatically calls the `remediation_action` tool to resolve the fault

You can also chat directly with the AI agent — click **AI Agent** (top right in the Elastic Serverless tab) and ask:

```
What happened with the fault that was just detected? What caused it and how should it be resolved?
```

---

## Step 4 — Confirm the Loop Closed

After the workflow completes, verify end-to-end in the **Elastic Serverless** tab:

1. **Cases** — a new case with the AI's root-cause analysis and remediation summary
2. **Observability → Alerts** — alert moves to "Recovered" state
3. **Discover → ES|QL** — error rate drops back to baseline:
```esql
FROM logs*
| WHERE @timestamp > NOW() - 5 MINUTES
| WHERE severity_text == "ERROR"
| STATS error_count = COUNT(*) BY service.name
| SORT error_count DESC
```
4. **Demo App → Chaos** link — all channels back to **STANDBY**

---

## Step 5 — Try a Cascade (Optional)

Trigger **multiple channels at once** from the **Demo App → Chaos** link to see cascade effects — inject channels 7, 8, and 9 (all GCP network_access services). Watch how the AI agent correlates errors across multiple services back to a single root cause, then resolves all three automatically.

---

## What You've Built

By completing this lab, you've seen the complete Elastic Autonomous Observability loop:

| Stage | Elastic Feature |
|-------|----------------|
| **Instrument** | OTLP ingestion — logs, metrics, traces |
| **Detect** | ES|QL alert rules — real-time anomaly detection |
| **Investigate** | AI Agent with tools — automated root-cause analysis |
| **Remediate** | Elastic Serverless Workflows — auto-remediation + notification |
| **Verify** | Dashboards & APM — confirm resolution |

✅ **Ready to complete when** all fault channels are resolved (no active chaos).
