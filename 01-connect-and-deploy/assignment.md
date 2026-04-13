---
slug: connect-and-deploy
id: 5tylvnmrdhym
type: challenge
title: PayPal Workshop — Connect & Explore Telemetry
teaser: Open Elastic Serverless, confirm OTLP is flowing, and practice ES|QL, APM,
  infrastructure, dashboards, and metrics—the former connect + explore challenges
  in one stop.
notes:
- type: text
  contents: |
    ## PayPal merchant observability workshop

    **Goal:** See how **Elastic Cloud Serverless** plus **OpenTelemetry** can consolidate logs, metrics, and traces—and how **ES|QL** and **workflows** support faster detection and response than fragmented logging stacks.

    **Context:** Many PayPal teams evolve from **CAL-style** centralized logging toward **OTel-native** signals and a single analytics plane. This lab uses a **synthetic Retail Banking Platform** scenario (nine services, three clouds) as a **stand-in** for complex payment-adjacent flows: wallets and apps, payments rails, fraud and risk, identity, and back-office processing—not PayPal production data.

    **You will:**
    - Explore **Discover**, **Applications**, **Infrastructure**, and **Dashboards** in Kibana
    - Run **ES|QL** and **TS** queries on OpenTelemetry logs and metrics (`logs.otel`, `metrics*`)
    - Use the **Demo App** and **Chaos Controller** tabs in later challenges; this one stays in **Elastic Serverless**

    **Services in the demo (Retail Banking scenario):**

    | Service | Cloud | Subsystem | PayPal-relevant theme |
    |---------|-------|-----------|------------------------|
    | mobile-gateway | AWS us-east-1 | digital_banking | Consumer apps & APIs |
    | payment-engine | AWS us-east-1 | payment_processing | Money movement & settlement |
    | fraud-sentinel | GCP us-central1 | fraud_detection | Risk & abuse detection |
    | auth-gateway | GCP us-central1 | authentication | Identity & session trust |
    | claims-processor | AWS us-east-1 | claims_management | Case-like back-office work |
    | policy-manager | GCP us-central1 | policy_administration | Rules & policy systems |
    | quote-engine | Azure eastus | underwriting | Decisioning / pricing logic |
    | member-portal | Azure eastus | member_services | Self-service surfaces |
    | document-vault | Azure eastus | document_management | Secure document handling |
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
difficulty: basic
timelimit: 1200
enhanced_loading: null
---

# Connect to Elastic Cloud & deploy

This **PayPal Elastic Serverless** workshop uses a shared **Elastic Cloud Serverless Observability** project, provisioned when the lab starts. Open the **Elastic Serverless** tab—you are proxied in (no manual login).

The VM runs **elastic-launch-demo** with your project’s Kibana URL and API key. The track **auto-launches** the **Retail Banking Platform** (`banking`) scenario as the demo workload and **hides** unrelated scenarios (e.g. Claro NOC) from the picker.

Your workspace has three tabs: **Demo App** (scenario lifecycle), **Chaos Controller** (fault injection in a later challenge), and **Elastic Serverless** (Kibana). This challenge stays in **Elastic Serverless**; avoid **Stop & Teardown** in the Demo App unless you intend to redeploy.

---

## Explore Elastic Serverless

In the **Elastic Serverless** tab, set the time range to **Last 15 minutes** or **Last 30 minutes**.

### Logs — Discover and ES|QL

**Discover →** an OpenTelemetry logs data view (**`logs.otel`**, **All logs**, or `logs*`). **Discover → Try ES|QL** using **`FROM logs.otel, logs.otel.*`** (fallback: **`FROM logs*`**).

**Errors by service:**

```esql
FROM logs.otel, logs.otel.*
| WHERE @timestamp > NOW() - 30 MINUTES AND severity_text == "ERROR"
| STATS errors = COUNT(*) BY service.name
| SORT errors DESC
```

**Counts by service and severity:**

```esql
FROM logs.otel, logs.otel.*
| WHERE @timestamp > NOW() - 30 MINUTES
| STATS cnt = COUNT(*) BY service.name, severity_text
| SORT cnt DESC
```

**Log volume by cloud:**

```esql
FROM logs.otel, logs.otel.*
| WHERE @timestamp > NOW() - 30 MINUTES
| EVAL bucket5m = DATE_TRUNC(5 minutes, @timestamp)
| STATS events = COUNT(*) BY bucket5m, cloud.provider
| SORT bucket5m DESC
```

**Mobile/API-style traffic (`mobile-gateway`):**

```esql
FROM logs.otel, logs.otel.*
| WHERE @timestamp > NOW() - 15 MINUTES
| WHERE service.name == "mobile-gateway" AND body.text LIKE "*api_request*"
| KEEP @timestamp, body.text
| SORT @timestamp DESC
| LIMIT 25
```

### Applications (APM)

**Observability → Applications** (or **APM**). Expect services such as `mobile-gateway`, `claims-processor`, `payment-engine`, `policy-manager`, `fraud-sentinel`, `auth-gateway`, `quote-engine`, `member-portal`, `document-vault`. Open one → **Transactions** or **Errors**.

### Infrastructure

**Observability → Infrastructure** — hosts such as **`banking-aws-host-01`**, **`banking-gcp-host-01`**, **`banking-azure-host-01`** and related containers.

### Dashboards

**Dashboards** — open scenario dashboards if you like, but **skip or ignore panels** whose ES|QL mentions **`auction.*`**, **`card_printing.*`**, or **`cloud_inventory.*`**. Those metrics belong to the **Fanatics Live** demo, not **Retail Banking**; the columns do not exist in this track and ES|QL will return `Unknown column`.

### Metrics — discover first, then TS

**Discover → Try ES|QL** (or Discover **Metrics** mode, if available). This workshop runs **`banking` (Retail Banking)** only. Any **`TS metrics*`** or chart copied from **Fanatics** material will fail on fields such as `auction.active_auctions`, `card_printing.queue_depth`, or `cloud_inventory.aws.compliance_pct`—**do not use those names here.**

**1 — See which metrics streams actually have data:**

```esql
FROM metrics* METADATA _index
| WHERE @timestamp > NOW() - 30 MINUTES
| STATS docs = COUNT(*) BY _index
| SORT docs DESC
| LIMIT 15
```

**2 — Count samples per service** (uses `service.name`, which is common on OTel metric documents):

```esql
FROM metrics*
| WHERE @timestamp > NOW() - 30 MINUTES AND service.name IS NOT NULL
| STATS samples = COUNT(*) BY service.name
| SORT samples DESC
| LIMIT 20
```

**3 — List `metric.name` values** (works when data is in APM OTel metric indices; if ES|QL reports an unknown column here, skip this query and use the field list in Discover on your busiest stream from step 1):

```esql
FROM metrics-apm.otel-*, metrics-apm.internal-*
| WHERE @timestamp > NOW() - 30 MINUTES
| STATS samples = COUNT(*) BY metric.name
| SORT samples DESC
| LIMIT 40
```

**4 — Time series without guessing column names:** Once you see a numeric gauge or counter field in Discover for your stream, aggregate it with **`FROM metrics*`** (wrap dotted names in **backticks**). Until then, you can still plot **event volume** per minute for a service:

```esql
FROM metrics*
| WHERE @timestamp > NOW() - 15 MINUTES AND service.name == "mobile-gateway"
| EVAL minute = DATE_TRUNC(1 minute, @timestamp)
| STATS samples = COUNT(*) BY minute
| SORT minute DESC
```

For **`TS metrics*`**, use the same index pattern Discover uses for metrics, but only with field names that **already exist** in your project—never Fanatics **`auction.*`**, **`card_printing.*`**, or **`cloud_inventory.*`**.

---

## What the scenario deploys into Kibana

The demo **deployer** pushes assets into your project when the scenario runs—similar to how you would onboard **alert rules**, **dashboards**, and **workflows** for a PayPal org space in Serverless:

| Asset type | What to look for |
|------------|------------------|
| Alert rules | ES\|QL rules aligned to the demo fault channels |
| Workflows | Alert → investigate → remediate flows |
| Dashboards | Executive / operations views for the scenario |
| Data views | `logs.otel` / `logs*`, `metrics*`, `traces*` (OTel-friendly views if present) |

---

## ✅ Ready for the next challenge

Continue when you have **logs or services** in Elastic Serverless and have skimmed **Applications** or **Infrastructure** (and optionally **Dashboards** or a **metrics / TS** query).
