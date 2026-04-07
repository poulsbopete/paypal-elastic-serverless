---
slug: connect-and-deploy
id: 5tylvnmrdhym
type: challenge
title: Connect & Explore — PayPal Payments Platform
teaser: Deploy Elastic Serverless and observe live telemetry from 9 financial services
  across AWS, GCP, and Azure.
notes:
- type: text
  contents: |
    ## PayPal Payments Platform — Elastic Serverless Observability

    PayPal's legacy CAL (Centralized Application Logging) system is proprietary, siloed, and cannot correlate logs, traces, and metrics. Engineers must hop between 3–5 systems to root-cause a single incident.

    **In this workshop, you will:**
    - Observe 9 live financial platform services emitting OpenTelemetry telemetry
    - Write ES|QL queries to explore trading operations data
    - Investigate latency, error rates, and order flow in real-time
    - Trigger and detect financial incidents with the Chaos Controller
    - Leverage Elastic AI for autonomous remediation

    **The 9 Services (matching the financial scenario):**

    | Service | Cloud | Subsystem |
    |---------|-------|-----------|
    | order-gateway | AWS us-east-1 | Order Management |
    | matching-engine | AWS us-east-1 | Trade Execution |
    | risk-calculator | AWS us-east-1 | Risk Management |
    | market-data-feed | GCP us-central1 | Market Data |
    | settlement-processor | GCP us-central1 | Settlement |
    | fraud-detector | GCP us-central1 | Compliance |
    | compliance-monitor | Azure eastus | Compliance |
    | customer-portal | Azure eastus | Client Services |
    | audit-logger | Azure eastus | Audit |
- type: text
  contents: |
    ## While you wait — PayPal AIOps value story

    **PayPal AIOps: Engineering Next-Gen Reliability with Elastic AI** — from fragmented tools and revenue risk to AI-powered detection, tiered storage, and lower TCO.

    <img src="https://raw.githubusercontent.com/poulsbopete/paypal-elastic-serverless/main/docs/wait-slides/paypal-aiops-problem-solution.png" alt="PayPal AIOps slide: problem (siloed Splunk, BigQuery, ServiceNow; revenue risk; slow MTTD/MTTR) vs solution (Elastic AI, 70% faster detection, business impact)" width="100%" style="max-width: min(1600px, 100vw); width: 100%; height: auto; border-radius: 8px; display: block; margin: 0 auto;" />
- type: text
  contents: "## While You Wait — Play O11y Survivors! \U0001F3AE\n\nSetup takes a
    few minutes. Survive the anomaly storm while Elastic provisions your environment:\n\n<iframe
    src=\"https://poulsbopete.github.io/Vampire-Clone/\" width=\"100%\" height=\"800\"
    frameborder=\"0\" allowfullscreen style=\"border-radius:8px;display:block;\"></iframe>\n"
tabs:
- id: ivy5xxd2wd28
  title: Demo App
  type: service
  hostname: es3-api
  path: /
  port: 8090
- id: d3yp8cepyihk
  title: Chaos Controller
  type: service
  hostname: es3-api
  path: /chaos
  port: 8090
- id: t2xzynorumrr
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
- id: szv5okbgr6iu
  title: Terminal
  type: terminal
  hostname: es3-api
difficulty: basic
timelimit: 1200
enhanced_loading: null
---

# Challenge 1 — Connect & Deploy

## What's Running

Your Elastic Serverless project is already provisioned. A standalone telemetry generator (`paypal-otel-gen`) is sending live data from **9 financial platform services** — order-gateway, matching-engine, risk-calculator, market-data-feed, settlement-processor, fraud-detector, compliance-monitor, customer-portal, and audit-logger — all to the `paypal-otel-logs` index.

The Demo App tab shows the **PayPal Operations Center** (financial scenario), with live service health, active fault channels, and the Chaos Controller for fault injection.

---

## Step 1 — Open the Demo App

Click the **Demo App** tab (or **Chaos Controller** — same app on port 8090). You should see:
- **PayPal Operations Center** — Bloomberg terminal-style dashboard
- **9 services** across AWS, GCP, and Azure in green (healthy) state
- **Chaos Controller** — 20 financial fault channels ready to inject

If the scenario is not running, click **Select Scenario → Financial Trading Platform** and launch it.

---

## Step 2 — Open Elastic Serverless

Click the **Elastic Serverless** tab and navigate to **Analytics → Discover**.

Set the time range to **Last 30 minutes**.

---

## Step 3 — Run Your First ES|QL Queries

Click **Try ES|QL** and explore the financial platform data:

**Count all telemetry by service:**
```esql
FROM paypal-otel-logs
| STATS count = COUNT(*) BY service.name
| SORT count DESC
```

**Look for trading errors:**
```esql
FROM paypal-otel-logs
| WHERE severity_text == "ERROR"
| STATS errors = COUNT(*) BY service.name
| SORT errors DESC
```

**Check order latency by service:**
```esql
FROM paypal-otel-logs
| WHERE transaction.duration.us IS NOT NULL
| STATS p99_ms = MAX(transaction.duration.us) / 1000 BY service.name
| SORT p99_ms DESC
```

**Search for OMS errors (Order Book Imbalance):**
```esql
FROM paypal-otel-logs
| WHERE body.text LIKE "*OMS-BOOK-IMBALANCE*" OR body.text LIKE "*ME-LATENCY-SLA*"
| KEEP @timestamp, service.name, body.text
| SORT @timestamp DESC
| LIMIT 20
```

**View data by cloud provider and region:**
```esql
FROM paypal-otel-logs
| STATS services = COUNT_DISTINCT(service.name), events = COUNT(*) BY cloud.provider, cloud.region
| SORT events DESC
```

---

## Step 4 — Pre-Provisioned Kibana Assets

These assets were automatically created during setup:

1. **PayPal Executive Dashboard** — multi-cloud trading operations overview (Kibana → Dashboards)
2. **3 Data Views** — `logs*`, `traces-*`, `metrics-*`
3. **5 Alert Rules** — high error rate, matching engine latency, order gateway spikes, settlement failures, risk limit breaches
4. **3 ML Anomaly Detection Jobs** — service error spikes, trade volume anomalies, trading pipeline degradation
5. **3 SLOs** — Availability (95%), Latency (85%), Error Rate (95%)

---

## Step 5 — AI Agent Questions

Open the **Elastic AI Assistant** and ask:

> "What is the current error rate for each of the 9 trading services in paypal-otel-logs over the last 30 minutes?"

> "Show me the top error types in paypal-otel-logs, grouped by service.name. What patterns suggest a systemic issue?"

> "Which services have the highest P99 latency? Are any exceeding their SLA thresholds?"

---

## ✅ Completion Check

You'll pass this challenge when:
- The demo app is healthy
- The telemetry generator is running
- At least 10 documents exist in `paypal-otel-logs`
- All 9 financial services are represented in the data
