# PayPal Merchant Observability: Elastic Serverless + OpenTelemetry

An Instruqt workshop demonstrating how Elastic Serverless + OpenTelemetry replaces PayPal's CAL logging and delivers unified, merchant-centric observability at global scale.

**Track URL:** https://play.instruqt.com/manage/elastic/tracks/paypal-elastic-serverless  
**GitHub Repo:** https://github.com/poulsbopete/paypal-elastic-serverless

---

## Overview

This lab puts participants in the role of a PayPal SRE evaluating Elastic Serverless as a CAL replacement. Across **three** challenges, they experience the full observability modernization story: from connecting and exploring live telemetry in Serverless to fault injection and autonomous AI-driven incident remediation powered by ML anomaly detection.

### Story Arc

| Phase | Title | What Happens |
|---|---|---|
| **Challenge 1** | Connect & explore telemetry | Serverless project + Retail Banking demo; learners use Discover, ES\|QL, APM, infrastructure, dashboards, and TS metrics in one challenge |
| **Challenge 2** | Inject a Fault | A payment incident is triggered via the Chaos Controller; learners watch Elastic surface merchant impact in real time |
| **Challenge 3** | Autonomous Remediation | Elastic AI Agent + Workflows investigate and remediate automatically; ML anomaly jobs flag the incident; SLOs show service health |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  Instruqt VM: es3-api (elastic/es3-api-v2)                       │
│                                                                   │
│  ┌─────────────────────┐  OTLP/HTTPS   ┌──────────────────────┐  │
│  │  elastic-launch-demo│──────────────▶│  Elastic Serverless   │  │
│  │  Python FastAPI      │  port 443     │  Observability Project│  │
│  │  port 8090          │  .ingest. URL │  (AWS us-east-1)      │  │
│  └─────────────────────┘               └──────────────────────┘  │
│           │                                       ▲               │
│  ┌────────▼────────┐                              │               │
│  │  NGINX           │──────── Kibana proxy ───────┘               │
│  │  port 8080       │  (ApiKey auth, no login)                    │
│  │  /loading        │  ← PayPal value props + O11Y Survivors iframe │
│  │  /home           │  ← Demo App landing page                    │
│  └─────────────────┘                                              │
└──────────────────────────────────────────────────────────────────┘
```

### PayPal Demo Services

| Service | Function | Simulated Cloud |
|---|---|---|
| `merchant-api` | Merchant profile and settings | AWS us-east-1 |
| `checkout-service` | Checkout session orchestration | AWS us-east-1 |
| `payments-orchestrator` | Payment routing and processing | AWS us-east-1 |
| `fraud-decisioning` | Real-time fraud scoring | GCP us-central1 |
| `settlement-service` | Settlement batch processing | GCP us-central1 |
| `merchant-support-api` | Support ticket and account management | Azure eastus |
| `notification-service` | Email, push, and webhook delivery | Azure eastus |

All 7 services emit **logs, metrics, and traces** via OTLP to Elastic Serverless.  
OTLP data lands in these Elastic data streams:

| Signal | Data Stream |
|---|---|
| Logs | `logs-apm.otel-default` |
| Traces | `traces-apm.otel-default` |
| Metrics | `metrics-apm.otel-default` |

---

## Track Structure

```
paypal-elastic-serverless/
├── track.yml                          # Track metadata (slug, title, enhanced_loading: false — same as elastic-autonomous-observability)
├── config.yml                         # VM definition + secrets block (ESS_CLOUD_API_KEY, LLM_PROXY_PROD)
├── README.md                          # This file
├── docs/
│   └── splash.html                    # PayPal value props page (hosted via rawcdn.githack.com)
├── track_scripts/
│   ├── setup-es3-api                  # Main provisioning script (same baseline as elastic-autonomous-observability, ~850 lines)
│   └── cleanup-es3-api                # Teardown (deletes Elastic Cloud project)
├── 01-connect-and-deploy/
│   ├── assignment.md                  # Connect + explore telemetry (merged former ch1+ch2)
│   ├── setup-es3-api
│   ├── check-es3-api                  # Deployment + Kibana + telemetry service count
│   └── solve-es3-api
├── 02-inject-fault/
│   ├── assignment.md
│   ├── setup-es3-api
│   ├── check-es3-api                  # Accepts both ACTIVE and RESOLVED fault states
│   └── solve-es3-api
└── 03-autonomous-remediation/
    ├── assignment.md
    ├── setup-es3-api
    ├── check-es3-api
    └── solve-es3-api
```

**Learner UI:** Challenge `tabs:` intentionally list only **Demo App**, **Chaos Controller**, and **Elastic Serverless**—no **Terminal** tab. Do not add `type: terminal` to `assignment.md`; facilitators use Instruqt **Shell** / SSH for VM diagnostics (see **Diagnostic Commands**).

---

## What Gets Provisioned at Setup

`track_scripts/setup-es3-api` is aligned with **[elastic-autonomous-observability](https://play.instruqt.com/manage/elastic/tracks/elastic-autonomous-observability)**. It runs once at track start and:

1. **Creates an Elastic Cloud Serverless Observability project** (AWS us-east-1) using `ESS_CLOUD_API_KEY`
2. **Generates an Elasticsearch API key** for the `admin` user (`.encoded` form)
3. **Installs NGINX** on port **8080** as a Kibana reverse proxy (Basic auth to the project)
4. **Serves** `/loading` and `/chatbot` static pages
5. **Starts** the JSON credentials server on port **8081**
6. **Clones** `elastic-launch-demo` (branch `feat/back-navigation-noc-chaos`), **patches** `scenarios/__init__.py` to remove **Claro** from the scenario list, **replaces** `elastic_config/workflows/significant_event_notification.yaml` with a PayPal copy that runs **`queue_remediation` before `run_rca`** so chaos faults **auto-resolve** via the demo’s Elasticsearch remediation-queue poller without waiting for the AI step, then installs deps and starts **systemd** `elastic-demo` on port **8090** with **`ACTIVE_SCENARIO=banking`**
7. **POSTs `/api/setup/launch`** with `scenario_id: banking` so the **Retail Banking Platform** deploys into the project (alerts, workflows, dashboards, ML, etc. come from that deployment — not from extra bash in this repo)

Shell aliases: `demo-logs`, `demo-status`, `demo-deployments`, `demo-chaos`, `demo-restart` (see **Diagnostic Commands**).

---

## ES|QL Quick Reference

After the scenario is running, use Discover’s data views (for example **`logs.otel`**, **All logs**, or `logs-*`) or ES|QL against OTel log streams, `metrics*`, `traces-*`.

Banking alert rules and workflows use OpenTelemetry logs — in ES|QL prefer:

```esql
FROM logs.otel, logs.otel.*
```

If that returns no rows, use **`FROM logs*`** instead.

```esql
FROM logs.otel, logs.otel.*
| STATS total = COUNT(*), services = COUNT_DISTINCT(service.name)
| LIMIT 10

FROM logs.otel, logs.otel.*
| WHERE @timestamp > NOW() - 30 minutes AND severity_text == "ERROR"
| STATS errors = COUNT(*) BY service.name
| SORT errors DESC
```

### Common fields (OTel in Elastic)

| Concept | Typical field | Example |
|---|---|---|
| Service | `service.name` | `mobile-gateway`, `payment-engine`, … |
| Log body | `message` or `body.text` | (depends on integration) |
| Severity | `log.level` / `severity_text` | `ERROR` |
| Trace | `trace.id` | hex string |
| Time | `@timestamp` | ISO-8601 |

### Retail Banking metrics (discover fields; avoid Fanatics)

This track deploys **`banking` (Retail Banking)** only. Saved queries or dashboards that reference **Fanatics Live** metrics (`auction.*`, `card_printing.*`, `cloud_inventory.*`) will fail with **`Unknown column`**—those fields are not in this scenario.

**Discover what exists, then aggregate:** On some **`metrics*`** streams ES|QL may not expose **`@timestamp`** (you will see `Unknown column [@timestamp]`). Use **logs** for time-windowed counts, or omit the time filter on metrics for a coarse index roll-up.

```esql
FROM metrics* METADATA _index
| STATS docs = COUNT(*) BY _index
| SORT docs DESC
| LIMIT 15
```

```esql
FROM logs.otel, logs.otel.*
| WHERE @timestamp > NOW() - 30 MINUTES AND service.name IS NOT NULL
| STATS samples = COUNT(*) BY service.name
| SORT samples DESC
| LIMIT 20
```

Then open **Discover** on the busiest `metrics-*` stream, pick a **numeric** field that appears for your services, and use it in **`FROM metrics*`** or **`TS metrics*`** with **backticks** around dotted names. Do not copy **`auction.*` / `card_printing.*` / `cloud_inventory.*`** examples from other demos.

**`TS metrics*` examples (Retail Banking)** — swap field names if Discover shows different dotted paths:

```esql
TS metrics*
| WHERE @timestamp > NOW() - 15 MINUTES
| EVAL minute = DATE_TRUNC(1 minute, @timestamp)
| STATS tps = AVG(`payment_engine.transactions_per_sec`) BY minute
| SORT minute DESC
```

```esql
TS metrics*
| WHERE @timestamp > NOW() - 15 MINUTES
| EVAL minute = DATE_TRUNC(1 minute, @timestamp)
| STATS p95_ms = PERCENTILE(`mobile_gateway.api_latency_ms`, 95) BY minute
| SORT minute DESC
```

---

## CAL → OpenTelemetry Field Mapping

| CAL Field | OTel Semantic Convention | Notes |
|---|---|---|
| `CAL_TYPE` | `resource.attributes.service.name` | OpenTelemetry `service.name` resource attribute |
| `TXN_ID` | `trace.id` | Native distributed trace correlation |
| `MERCHANT_ID` | `merchant.id` | Custom resource attribute |
| `STATUS` | `http.response.status_code` | Numeric HTTP status |
| `DURATION_MS` | span `duration` | Nanoseconds in OTLP, auto-converted |
| *(none)* | `deployment.environment`, `cloud.region` | New observability dimensions |

---

## Diagnostic Commands

For facilitators or advanced troubleshooting, run these on the **es3-api** host (Instruqt **Shell** or SSH). Learners complete the lab using the **Demo App**, **Chaos Controller**, and **Elastic Serverless** tabs only.

| Command | What it does |
|---|---|
| `demo-logs` | Stream live demo app logs (`journalctl -f`) |
| `demo-deployments` | Show running scenarios and their Kibana URL |
| `demo-chaos` | Show chaos channel states |
| `demo-status` | Show full demo app status JSON |
| `demo-restart` | Restart the demo app (systemd); clears in-memory deployments — you may need to **Launch** again from the Demo App |

### Empty Discover or “no active deployment”

- **Discover empty:** widen the time range (e.g. **Last 15 minutes**) and confirm the **Retail Banking** scenario finished deploying (Demo App progress bar).
- **Chaos Controller:** “NO ACTIVE DEPLOYMENT” means the demo scenario is not running — open **Demo App**, confirm Kibana URL/API key if prompted, and click **Launch** (or wait for auto-launch to finish after track start).

| Symptom | What to try |
|---|---|
| Empty chaos fault dropdown | `demo-restart`, then redeploy from Demo App |
| Demo App still spinning | `demo-logs` — check for `ModuleNotFoundError` |
| Kibana proxy errors | `systemctl status nginx` on the VM |

---

## Key Design Decisions

### O11Y Survivors + wait slides (same pattern as `elastic-autonomous-observability`)

The upstream track [elastic-autonomous-observability](https://play.instruqt.com/manage/elastic/tracks/elastic-autonomous-observability) uses:

- **`enhanced_loading: false`** — notes-only while the sandbox provisions.
- **Wait slides** in `assignment.md` `notes:`: (1) workshop context, (2) **PayPal AIOps** image slide, (3) **Streams & TCO** copy with link to [Observability TCO Comparison](https://o11y-compare.vercel.app/), (4) **O11y Survivors** `<iframe>` to [Vampire-Clone](https://poulsbopete.github.io/Vampire-Clone/) at **height 800** (no extra O11Y tab).
- **Challenge 1** includes an **Observability → Streams** step so learners see **governed ingest** tied to **`logs.otel`** before the game tab.
- **Demo App tabs on port `8090`**: `Demo App` → `/`, **Chaos Controller** → `/chaos`; **Elastic Serverless** on **`8080`** via NGINX with the same **CSP** header overrides as the upstream track.

The VM **`/loading`** page (NGINX) still includes value props + the game for anyone who opens that URL directly.

### Secrets via `config.yml`

Secrets are declared under the `es3-api` VM's `secrets:` block in `config.yml`:

```yaml
secrets:
- name: ESS_CLOUD_API_KEY
- name: LLM_PROXY_PROD
```

Instruqt injects these from the organization's secret store automatically.

> **Do not use a sandbox config** — linking a `sandboxConfig` with an empty `virtualmachines: []` overrides the VM definitions in `config.yml` and causes "The host for this script was not found" errors.

### OTLP Endpoint Derivation

The OTLP endpoint is derived from the Elasticsearch URL at setup time:

```bash
# ES URL:   https://<project>.es.<region>.aws.elastic.cloud[:port]
# OTLP URL: https://<project>.ingest.<region>.aws.elastic.cloud:443
OTLP_URL=$(echo "$ES_URL" | sed 's/\.es\./\.ingest\./; s|:[0-9]\{2,5\}$||; s|/$||')
OTLP_URL="${OTLP_URL}:443"
```

The Elastic Serverless OTLP ingest endpoint always listens on port 443, regardless of what port the Elasticsearch endpoint uses.

### Provisioning baseline

`track_scripts/setup-es3-api` starts from the **elastic-autonomous-observability**-style flow, then **defaults to Retail Banking** (`banking`) and **removes Claro** from the demo’s scenario registry via a small patch to `elastic-launch-demo` (branch `feat/back-navigation-noc-chaos`).

---

## Development Workflow

### Prerequisites

- [Instruqt CLI](https://docs.instruqt.com/reference/cli/overview) — `instruqt track push`
- Access to the `elastic` Instruqt organization
- `ESS_CLOUD_API_KEY` configured in the Elastic org's secret store

### Push to Instruqt

```bash
cd paypal-elastic-serverless
instruqt track validate
instruqt track push --force
```

> `--force` is safe here because it updates the track without clearing secrets.  
> The checksum in `track.yml` is auto-updated by Instruqt after each push.

### Iterative Development

```bash
# Edit assignment content
vim 01-connect-and-deploy/assignment.md

# Edit provisioning logic
vim track_scripts/setup-es3-api

# Validate + push
instruqt track validate && instruqt track push --force

# Play the track
open https://play.instruqt.com/manage/elastic/tracks/paypal-elastic-serverless
```

### Update the Splash Page

Edit `docs/splash.html`, commit, and push to `main`. Get the new raw CDN URL:

```
https://rawcdn.githack.com/poulsbopete/paypal-elastic-serverless/<commit-sha>/docs/splash.html
```

Update the tab URL in `01-connect-and-deploy/assignment.md`, then push to Instruqt.

### GitHub Pages — PayPal AIOps slides

The deck **The PayPal AIOps Blueprint** is published as static slides under `docs/aiops-slides/` (14 PNGs exported from the source `.pptx`, plus `index.html` for navigation).

1. In the GitHub repo: **Settings → Pages → Build and deployment → Source**: Deploy from branch **`main`**, folder **`/docs`**.
2. After the site builds, open:
   - **Hub:** `https://<user>.github.io/<repo>/`
   - **Slides:** `https://<user>.github.io/<repo>/aiops-slides/`

Use **← →**, **Space**, **Home** / **End**, or click the image halves / dots to move between slides. Optional captions are in the `LABELS` array inside `docs/aiops-slides/index.html` (the PPTX contains no extractable text, so titles are editable there).

---

## Track IDs

| Resource | Value |
|---|---|
| Track slug | `paypal-elastic-serverless` |
| Track ID | `q7432fqzit6g` |
| Instruqt org | `elastic` |
| GitHub repo | `github.com/poulsbopete/paypal-elastic-serverless` |
| Upstream inspiration | [elastic-autonomous-observability](https://play.instruqt.com/manage/elastic/tracks/elastic-autonomous-observability) — this repo’s `track_scripts/setup-es3-api` is the same baseline; local `elastic-autonomous-observability-copy/` is a reference snapshot |
| Demo app repo | `github.com/poulsbopete/elastic-launch-demo` (branch `feat/back-navigation-noc-chaos`) |

---

## Full Troubleshooting Reference

| Symptom | Likely Cause | Fix |
|---|---|---|
| "The host for this script was not found" | Sandbox config with empty VMs is linked | Unlink via GraphQL: `setTrackSandboxConfigVersion` with empty `configVersionID` |
| Secrets not available in setup script | `ESS_CLOUD_API_KEY` not in Elastic org secret store | Add secret in Instruqt org settings |
| Demo app never healthy | Python deps failed / import error | `journalctl -u elastic-demo -n 50` on VM |
| Kibana not loading | NGINX not started | `systemctl status nginx` on VM |
| "Unknown column @timestamp" in ES\|QL | Wrong index type (common on **`metrics*`**) or empty data | Use **`FROM logs.otel, logs.otel.*`** for time filters; on metrics, drop `WHERE @timestamp` until you confirm the time field in Discover |
| "Unknown column [cloud.provider]" in ES\|QL | OTel field not top-level in this mapping | Group by **`service.name`** or another field Discover shows under **cloud.*** |
| Kibana **Case** closed but Chaos still **ACTIVE** | Case is Kibana state; chaos clears per **remediation queue** doc (**one channel per alert run**) or **RESOLVE** on each card | See challenge 3 **“Cases vs Chaos”**; use **RESOLVE** on each **ACTIVE CHANNELS** card for stragglers |
| Empty chaos fault dropdown | No active deployment | Open Demo App → **Launch**; or `demo-restart` then redeploy |
| ML / SLOs missing | Scenario not finished pushing assets | Wait for deployment progress; refresh Kibana |
