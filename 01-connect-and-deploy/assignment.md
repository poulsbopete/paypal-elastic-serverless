---
slug: connect-and-deploy
id: xjxmr3qpwxmy
type: challenge
title: Connect to Elastic Cloud & Deploy the PayPal Scenario
teaser: Wire the PayPal merchant observability platform to your Elastic Cloud project
  and launch 7 microservices sending live OpenTelemetry telemetry.
notes:
- type: text
  contents: |
    ## PayPal Observability Modernization

    PayPal operates one of the world's largest payment platforms. Every transaction — checkout, auth, settlement, fraud decisioning — passes through a network of interdependent services running at massive scale across multiple regions.

    Today, observability at PayPal is fragmented:

    - **CAL (Common Application Logging)** is the primary logging system, originally built in the early 2000s. It emits proprietary structured logs that are operationally expensive to query, poorly suited to cross-service correlation, and incompatible with modern tooling.
    - **Traces and metrics live in separate silos.** Correlating a checkout failure from a merchant complaint to a specific trace to a service metric requires hopping between tools.
    - **Cardinality at scale** — millions of merchant IDs, transaction IDs, payment methods, and regions — creates challenges for any metrics backend not designed for high-dimensional data.

    This workshop puts you in the role of a PayPal SRE evaluating **Elastic Serverless + OpenTelemetry** as the unified observability destination.
- type: text
  contents: |
    ## The CAL Replacement Story

    CAL was built for a different era. It solved a real problem — structured logging at scale — but the operational model has aged.

    | Pain Point | Impact |
    |---|---|
    | Proprietary schema | No off-the-shelf tooling interoperability |
    | No trace context | Cross-service investigation is manual |
    | Difficult correlation | Engineers stitch log events across services by hand |
    | Query UX | Specialist knowledge required for every investigation |
    | Separate pipelines | Operational overhead for logs, metrics, traces |

    **The goal:** A shared schema (OTel semantic conventions) understood across all services. Automatic correlation via `trace.id`, `merchant.id`, `transaction.id`. Logs, metrics, and traces queryable together in one place.
- type: text
  contents: |
    ## Elastic Serverless: Zero-Ops Observability

    Elastic Serverless decouples compute and storage — you pay for what you ingest and query, never for idle nodes.

    **Key characteristics:**
    - **No cluster management:** no shard sizing, no JVM tuning, no rolling restarts
    - **Instant provisioning:** a new Observability project is ready in under 60 seconds
    - **Three project types:** Elasticsearch, Observability, Security — each tuned for its workload
    - **Built-in AI:** every project includes an Elastic AI Assistant with direct access to live telemetry
    - **Native OTLP:** no Collector required — services ship directly via OTLP gRPC or HTTP
- type: text
  contents: |
    ## CAL → OpenTelemetry Field Mapping

    | CAL Field | OTel Semantic Convention | Value Gained |
    |---|---|---|
    | `CAL_TYPE` | `service.name` | Standard across all CNCF tooling |
    | `TXN_ID` | `trace.id` / `transaction.id` | Native distributed trace correlation |
    | `MERCHANT_ID` | `merchant.id` (custom attribute) | Pivot any signal by merchant in one field |
    | `STATUS` | `http.response.status_code` | Automatic SLO and error rate computation |
    | `DURATION_MS` | `duration` (span attribute) | Native latency percentile histograms |
    | *(no equivalent)* | `deployment.environment`, `cloud.region` | Infrastructure context baked in |
- type: text
  contents: |
    ## ES|QL: Merchant-Scale Analytics

    ES|QL is Elastic's pipe-based query language for high-volume telemetry. One language across logs, metrics, and traces.

    ```esql
    FROM logs-apm.otel-*
    | WHERE merchant.id == "M-98231" AND severity_text == "ERROR"
    | STATS errors = COUNT(*) BY service.name
    | SORT errors DESC
    ```

    ```esql
    FROM metrics-*
    | WHERE @timestamp > NOW() - 15 MINUTES AND service.name == "checkout-service"
    | STATS p99 = PERCENTILE(transaction.duration, 99) BY merchant.tier, cloud.region
    | SORT p99 DESC
    ```
- type: text
  contents: |
    ## The PayPal Demo Environment

    This lab deploys a simulated PayPal merchant operations environment with 7 microservices:

    | Service | Function | Cloud |
    |---------|----------|-------|
    | `merchant-api` | Merchant profile and settings | AWS |
    | `checkout-service` | Checkout session orchestration | AWS |
    | `payments-orchestrator` | Payment routing and processing | AWS |
    | `fraud-decisioning` | Real-time fraud scoring | GCP |
    | `settlement-service` | Settlement batch processing | GCP |
    | `merchant-support-api` | Support ticket and account management | Azure |
    | `notification-service` | Email, push, and webhook delivery | Azure |

    All 7 services emit **logs, metrics, and traces** via OTLP to your Elastic Serverless Observability project.
tabs:
- id: ufdoduoqc9ok
  title: Terminal
  type: terminal
  hostname: es3-api
- id: hualhkazscdv
  title: Demo App
  type: service
  hostname: es3-api
  path: /home
  port: 8090
- id: ouotjjliyyi1
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
timelimit: 0
enhanced_loading: null
---

# Connect to Elastic Cloud & Deploy the PayPal Scenario

Your Elastic Cloud Serverless Observability project was **automatically provisioned** when this lab started. Open the **Elastic Serverless** tab — no login required.

The PayPal merchant demo environment is running on this VM and has been pre-configured with your project's credentials.

---

## Step 1 — Verify the Demo App is Healthy

Open the **Terminal** tab and check whether the demo app is running:

```bash
curl http://localhost:8090/health
```

You should see `{"status":"ok"}`. If not, check the service log:

```bash
journalctl -u elastic-demo --no-pager -n 30
```

---

## Step 2 — Check Active Deployments

```bash
curl -s http://localhost:8090/api/deployments | python3 -m json.tool
```

You should see the PayPal scenario listed with `"running": true`. If the list is empty, the scenario is still initializing — wait 30 seconds and retry.

---

## Step 3 — Explore the Demo App

Click the **Demo App** tab. This is your control panel for the lab:

- **Live Service Dashboard** — real-time health across the 7 PayPal microservices
- **Chaos Controller** — inject and resolve payment fault scenarios
- **Deployment Status** — verify active Elastic resources

---

## Step 4 — Open Elastic Serverless

Click the **Elastic Serverless** tab to open Kibana. No login required — authentication is handled automatically via the NGINX proxy.

Review the pre-provisioned assets: **Dashboards**, **APM Service Map**, **AI Investigation Agent**, and **Alert Rules**.

---

## Step 5 — Verify Live Telemetry

In Kibana, open **Discover**, select **ES|QL** as the query language, and run this count query:

```esql
FROM logs-apm.otel-*
| STATS services = COUNT_DISTINCT(service.name), log_count = COUNT(*)
```

You should see 7 distinct services and a growing log count.

> **If you see "Unknown column @timestamp" or "No results":** The OTLP data pipeline is still warming up. Switch to the **Terminal** tab and run:
> ```
> demo-otlp-test
> ```
> This will test OTLP connectivity and show you the exact HTTP status code. If it returns HTTP 200, wait 30 seconds and retry the query. If it returns 401 or 403, the API key needs to be refreshed — run `demo-restart` to re-initialize the app.

---

## Step 6 — Ask the AI Agent

In Kibana, click the **AI Assistant** button (top-right, sparkle icon). Copy and paste each question:

```
Which services are currently sending telemetry and what is the overall health of each?
```

```
What is the error rate across all services in the last 30 minutes? Which service has the highest?
```

```
Give me a summary of the PayPal payment platform telemetry that is flowing into Elastic right now.
```

```
What data views and dashboards are available in this Elastic environment?
```

The AI Agent has direct access to your live telemetry — every answer is grounded in real data from the running services, not a generic response.
