---
slug: inject-fault
id: 3cg5yt6yrxyx
type: challenge
title: Simulate a Payment Incident
teaser: Use the Chaos Controller to trigger a realistic payment failure and watch
  Elastic detect, alert, and surface the root cause automatically.
notes:
- type: text
  contents: |
    ## High-Cardinality Metrics at PayPal Scale

    PayPal processes millions of transactions per day across millions of merchants. This creates a fundamental observability challenge: **metric cardinality**.

    Traditional metrics backends struggle when you have:
    - Millions of distinct `merchant.id` values
    - Thousands of `transaction.id` combinations per second
    - Cross-dimensional queries (merchant × region × payment_method × service)

    Elastic's data tier architecture handles this natively. Metrics indexed with OTel semantic conventions can be queried at full cardinality using ES|QL — no pre-aggregation required, no label limits.

    In this challenge, you'll inject a fault into one service and watch Elastic surface the impact across merchant dimensions.
tabs:
- id: hcbmyxjoozks
  title: Terminal
  type: terminal
  hostname: es3-api
- id: ln8arqhor7xf
  title: Demo App
  type: service
  hostname: es3-api
  path: /home
  port: 8090
- id: spnj6adl6w6s
  title: Elastic Serverless
  type: service
  hostname: es3-api
  path: /app/discover#/?_a=(columns:!(service.name,severity_text,body.text),query:(esql:'FROM+logs-apm.otel-*+%7C+WHERE+%40timestamp+>+NOW()+-+30+minutes+%7C+WHERE+severity_text+%3D%3D+%22ERROR%22+%7C+KEEP+service.name%2C+body.text%2C+severity_text%2C+%40timestamp+%7C+SORT+%40timestamp+DESC+%7C+LIMIT+50',language:esql),sort:!(!('@timestamp',desc)))&_g=(filters:!(),refreshInterval:(pause:!f,value:10000),time:(from:now-30m,to:now))
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

# Simulate a Payment Incident

Use the **Chaos Controller** in the Demo App to inject a realistic payment fault, then watch Elastic surface the impact automatically.

---

## Step 1 — Inject a Fault

Click the **Demo App** tab → **Chaos Controller**.

Choose one of the payment incident scenarios:
- **Payments Orchestrator Timeout** — upstream payment network calls failing
- **Fraud Decisioning Latency Spike** — fraud scoring delays blocking checkout
- **Merchant API Auth Failure** — token validation failing for a merchant cohort

Activate the fault. The Demo App will immediately begin generating elevated error rates and latency spikes.

---

## Step 2 — Watch Elastic Detect It

Click the **Elastic Serverless** tab. The error log stream should already be showing the fault.

Run this ES|QL query to see the impact by service:

```esql
FROM logs-apm.otel-*
| WHERE @timestamp > NOW() - 5 MINUTES AND severity_text == "ERROR"
| STATS errors = COUNT(*) BY service.name
| SORT errors DESC
```

Compare the error rates to your pre-fault baseline. You should see a clear spike on the affected service.

---

## Step 3 — Identify the Blast Radius

Find which services are seeing the highest error rates during the incident, and what the error messages are:

```esql
FROM logs-apm.otel-*
| WHERE @timestamp > NOW() - 5 MINUTES AND severity_text == "ERROR"
| STATS errors = COUNT(*), unique_traces = COUNT_DISTINCT(trace.id) BY service.name
| SORT errors DESC
| LIMIT 15
```

Drill into the top affected service to see the actual error messages:

```esql
FROM logs-apm.otel-*
| WHERE @timestamp > NOW() - 5 MINUTES AND severity_text == "ERROR"
| KEEP @timestamp, service.name, body.text, trace.id
| SORT @timestamp DESC
| LIMIT 20
```

---

## Step 4 — Check Alerts

In Kibana, go to **Observability** → **Alerts**. You should see one or more alerts triggered by the fault. Click an alert to see the AI-generated investigation summary.

---

## Step 5 — Watch Elastic Workflows Auto-Remediate

No manual intervention needed. The Elastic Workflows triggered by the alert are already executing the remediation playbook in the background.

In Kibana, go to **Observability** → **Workflows** and open the **PayPal Payment Incident Remediation** workflow. You'll see the execution log — Elastic has already:

1. Detected the anomaly via the alert rule
2. Queried recent error logs to identify the affected service
3. Run AI root cause analysis
4. Executed the remediation action
5. Logged the outcome and updated the alert status

Watch the error rate in **Elastic Serverless** → **Dashboards** drop back to baseline as the workflow completes — without a single manual step.

---

## Step 6 — Ask the AI Agent During the Incident

While the fault is active (or immediately after), open the **AI Assistant** and copy/paste these:

```
What is causing the current spike in errors? Which service is the root cause?
```

```
I have an active payment incident. Give me a step-by-step root cause analysis based on the last 10 minutes of telemetry.
```

```
Which transactions were affected by the current fault? How many users were impacted?
```

```
Compare error rates before and after the incident started. When exactly did the anomaly begin?
```

```
Write an incident report summary for this payment outage that I can share with the PayPal SRE leadership team.
```

This is the exact workflow a PayPal SRE would use during a P1 incident — instead of jumping between CAL, traces, and metrics tools, they get a single conversational interface over all signals simultaneously.
