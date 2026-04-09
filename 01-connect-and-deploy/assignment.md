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

# Connect to Elastic Cloud & Deploy

> **Workshop note:** The steps below match **[Elastic Autonomous Observability](https://play.instruqt.com/manage/elastic/tracks/elastic-autonomous-observability)** — the **Fanatics Live** scenario is auto-launched at track start. Sidebar notes may still describe a PayPal narrative; use the **live service names** from the demo (for example `auction-engine`, `card-printing-system`) in Kibana and ES|QL until challenge copy is updated.

Your Elastic Cloud Serverless Observability project was **automatically provisioned** when this lab started — open the **Elastic Serverless** tab directly. No login required.

The demo platform is already running on this VM and has been pre-configured with your project's credentials.

---

## Step 1 — Verify the Deployment is Running

Open the **Demo App** tab. You should see the Fanatics Live scenario already deployed and sending telemetry. If the deployment bar is still in progress, wait a moment for it to complete.

You can also confirm in the **Terminal** tab:

```bash
demo-deployments
```

---

## Step 2 — Explore the Demo App

The Demo App is your control panel for this lab. From here you can:

- **View the live dashboard** — real-time service health across all 9 microservices
- **Open the Chaos Controller** — inject and resolve fault channels
- **Monitor deployment status** — see all active Elastic resources

---

## Step 3 — Open Elastic Serverless

Click the **Elastic Serverless** tab. This opens your auto-provisioned Observability project, pre-logged in. Navigate to:

- **Discover → ES|QL** — query live logs from services like `auction-engine`, `card-printing-system`, `digital-marketplace`
- **Applications → Service inventory** — distributed traces from 7 services (4 network services are logs-only)
- **Observability → Infrastructure** — 3 simulated hosts (AWS, GCP, Azure)

> **Tip:** Set the time range to **Last 15 minutes** to see recent telemetry.

---

## What was auto-deployed

The track setup automatically provisioned the full observability stack in your Elastic project:

| Resource | Details |
|----------|---------|
| Alert rules | 20 ES\|QL rules — one per fault channel |
| AI agent | Investigation tools + system prompt |
| Workflows | Alert → investigate → remediate → notify |
| Dashboards | Executive dashboard + OTel dashboards |
| Data views | `logs.otel`, `logs.otel.*`, `metrics-*` |

✅ **You're ready for the next challenge when** you can see logs or services in the Elastic Serverless tab.
