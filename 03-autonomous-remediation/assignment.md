---
slug: autonomous-remediation
id: 0h7rr7klrbwo
type: challenge
title: PayPal Workshop — Autonomous Remediation & AIOps Loop
teaser: Close the loop with Elastic AI, workflows, and cases—the PayPal AIOps story
  for faster MTTR without sacrificing auditability.
notes:
- type: text
  contents: |
    ## From alert to action — PayPal AIOps on Serverless

    **Workshop outcome:** See how **Elastic Serverless Workflows** plus an **AI agent** can shorten **MTTD/MTTR**: alert → ES|QL context → **root-cause analysis** → **remediation hook** → **Case** with an audit trail.

    **Demo:** The **Retail Banking Platform Significant Event Notification** workflow is preloaded for this scenario. It mirrors patterns PayPal teams discuss for **AIOps**: structured runbooks as YAML, **ES|QL** as the facts layer, and **Agent Builder** tools (e.g. **`remediation_action`**) for controlled automation.

    **In this challenge:**
    - Watch **Retail Banking Platform Significant Event Notification** run end-to-end (including **`queue_remediation`** where configured)
    - Optionally use **RESOLVE** in Chaos for extra faults that did not get their own alert-driven run
    - Explore **ML anomaly** jobs and **SLO** views if your project includes them
    - Revisit **Observability → Streams** to tie alerts and workflows back to **governed ingest** and **TCO**
    - Use **AI Assistant / AI Agent** for deeper RCA questions
- type: text
  contents: |
    ## While you wait — PayPal AIOps value story

    **PayPal AIOps: Engineering Next-Gen Reliability with Elastic AI** — from fragmented tools and revenue risk to AI-powered detection, tiered storage, and lower TCO.

    <img src="https://raw.githubusercontent.com/poulsbopete/paypal-elastic-serverless/main/docs/wait-slides/paypal-aiops-problem-solution.png" alt="PayPal AIOps slide: problem (siloed Splunk, BigQuery, ServiceNow; revenue risk; slow MTTD/MTTR) vs solution (Elastic AI, 70% faster detection, business impact)" width="100%" style="max-width: min(1600px, 100vw); width: 100%; height: auto; border-radius: 8px; display: block; margin: 0 auto;" />
- type: text
  contents: |
    ## While you wait — Streams & TCO

    **Elastic Streams** (under **Observability**) is where you **govern** OpenTelemetry
    ingestion—routing, partitioning, retention, and quality—*before* everything accrues
    in Elasticsearch. Tighter streams mean **less junk indexed** on **Elastic Cloud
    Serverless**, which shows up as **lower ingest and retention spend** and fewer
    noisy alerts for operators.

    Use **[Observability TCO Comparison](https://o11y-compare.vercel.app/)** as a
    conversation starter with leadership: adjust metrics/logs assumptions, then map
    “what we stopped indexing or retained shorter” to **TCO** (*tool output is
    illustrative only—not a vendor quote*).

    In the first challenge you will open **Streams** and relate it to the **`logs.otel`**
    path used by this Retail Banking scenario.
- type: text
  contents: "## While You Wait — Play O11y Survivors! \U0001F3AE\n\nSetup takes a
    few minutes. Survive the anomaly storm while Elastic provisions your environment:\n\n<iframe
    src=\"https://poulsbopete.github.io/Vampire-Clone/\" width=\"100%\" height=\"800\"
    frameborder=\"0\" allowfullscreen style=\"border-radius:8px;display:block;\"></iframe>\n"
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
difficulty: basic
timelimit: 1200
enhanced_loading: null
---

# Autonomous investigation & remediation

In this final **PayPal Elastic Serverless** challenge, you’ll watch Elastic **investigate** the fault you injected and **remediate** it through the demo integration—then verify the loop closed. This is the **AIOps** narrative from the workshop sidebar: fewer handoffs, faster resolution, **documented** in **Cases**.

---

## Step 1 — Watch the alert fire

First, confirm the alert rule detected your fault:

**Observability → Alerts**

Within 30–60 seconds of triggering a fault you should see an active alert. If no alert appears yet, wait a minute — the ES|QL rules run on a 30-second interval.

---

## Step 2 — Watch the workflow execute

Once an alert fires, the **Significant Event Notification** workflow runs automatically. In the **Elastic Serverless** tab:

**Observability → Workflows**

Find **Retail Banking Platform Significant Event Notification** (search **Retail Banking** or **banking**) and open **Executions**. A typical successful run includes:
- `count_errors` — ES|QL over recent errors, typically `FROM logs.otel, logs.otel.*` with `severity_text == "ERROR"`
- **`queue_remediation`** — runs **next**: writes a pending doc to **`banking-remediation-queue`** in Elasticsearch; the demo app’s **remediation poller** picks it up and **auto-resolves** the chaos channel (you should see **Active Channels** clear within about a **poll interval**, without clicking **RESOLVE**)
- `run_rca` — AI agent (**banking-ops-analyst**) performs RCA after remediation is already queued (RCA documents impact; it should not queue duplicate remediations)
- `create_case` — Kibana case with RCA context (audit trail for managers and partner teams—similar to how PayPal orgs track **severity-1** follow-up)
- `audit_log` — records the run

> **Multiple active channels:** Each workflow run is usually driven by **one** alert (one fault context). If you injected several faults at once, you may see several executions over time—or click **RESOLVE** next to any channel that stays active after its alert workflow has finished.

---

## Step 3 — Review the AI agent investigation & remediation

In the workflow execution log, expand the **run_rca** step to see what the AI agent did:

- **Error identification** — which service, which error type
- **Log correlation** — how many errors in the detection window
- **Root cause hypothesis** — what the AI agent determined
- **Remediation** — the **Significant Event Notification** workflow already ran **`queue_remediation`** early in the run; the agent should not duplicate that. For **manual** remediation from a case, the separate **Remediation Action** workflow still uses the same queue pattern

You can also chat directly with the AI agent — click **AI Agent** (top right in the Elastic Serverless tab) and ask:

```
What happened with the fault that was just detected? What caused it and how should it be resolved?
```

---

## Step 4 — Confirm the loop closed

After the workflow completes, verify end-to-end in the **Elastic Serverless** tab:

1. **Cases** — a new case with the AI's root-cause analysis and remediation summary
2. **Observability → Alerts** — alert moves to "Recovered" state
3. **Discover → ES|QL** — error rate drops back toward baseline (same pattern as **`count_errors`** in the workflow):
```esql
FROM logs.otel, logs.otel.*
| WHERE @timestamp > NOW() - 5 MINUTES AND severity_text == "ERROR"
| STATS error_count = COUNT(*) BY service.name
| SORT error_count DESC
```
4. **Chaos Controller** — the channel that triggered this alert should move back to **STANDBY** (or drop from **Active Channels**) after remediation; clear any stragglers with **RESOLVE** if you had multiple simultaneous faults

---

## Step 5 — Try a cascade (optional)

From **Demo App → Chaos**, inject **multiple channels at once** to mimic a broader incident. For example, trigger **channels 14, 15, and 16** (authentication and fraud_detection: biometric degradation, MFA delivery failure, fraud false-positive surge) and watch how workflows and the AI assistant correlate errors across **auth-gateway**, **mobile-gateway**, **payment-engine**, and **fraud-sentinel**—a useful pattern for **identity + risk + payments** correlations.

Another coherent set is **channels 1, 2, and 3** (all **digital_banking**): mobile API timeout, mobile deposit failure, and push notification storm.

---

## What you’ve built — PayPal workshop recap

By completing this lab, you’ve walked the **Elastic Autonomous Observability** loop in a **PayPal-relevant** story (OTel in, ES|QL detection, AI-assisted investigation, workflow-driven remediation, case audit trail):

| Stage | Elastic feature | PayPal workshop angle |
|-------|-------------------|------------------------|
| **Instrument** | OTLP ingestion — logs, metrics, traces | OTel as the path beyond fragmented **CAL**-era silos |
| **Detect** | ES|QL alert rules — scheduled queries | One language for logs + metrics investigations |
| **Investigate** | AI Agent with tools — automated RCA | **AIOps** assist for on-call and L2 |
| **Remediate** | Serverless Workflows — hooks + notifications | Controlled automation with guardrails |
| **Verify** | Dashboards & APM — confirm resolution | Merchant- and platform-facing health views |

✅ **Ready to complete when** all fault channels are resolved (no active chaos).
