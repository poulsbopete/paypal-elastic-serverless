# PayPal Merchant Observability: Elastic Serverless + OpenTelemetry

An Instruqt workshop demonstrating how Elastic Serverless + OpenTelemetry replaces PayPal's CAL logging and delivers unified, merchant-centric observability at global scale.

**Track URL:** https://play.instruqt.com/manage/elastic/tracks/paypal-elastic-serverless  
**GitHub Repo:** https://github.com/poulsbopete/paypal-elastic-serverless

---

## Overview

This lab puts participants in the role of a PayPal SRE evaluating Elastic Serverless as a CAL replacement. Across four challenges, they experience the full observability modernization story: from connecting a live multi-cloud environment to autonomous AI-driven incident remediation powered by ML anomaly detection.

### Story Arc

| Phase | Title | What Happens |
|---|---|---|
| **Challenge 1** | Connect & Deploy | Elastic Serverless project is provisioned; 7 PayPal microservices start emitting OTLP telemetry; learners verify live data in Kibana |
| **Challenge 2** | Explore Telemetry | ES\|QL queries replace CAL log lookups; logs, metrics, and distributed traces are explored across all services |
| **Challenge 3** | Inject a Fault | A payment incident is triggered via the Chaos Controller; learners watch Elastic surface merchant impact in real time |
| **Challenge 4** | Autonomous Remediation | Elastic AI Agent + Workflows investigate and remediate automatically; ML anomaly jobs flag the incident; SLOs show service health |

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
│   ├── assignment.md                  # Challenge content + tabs
│   ├── setup-es3-api                  # Per-challenge health check
│   ├── check-es3-api                  # Completion check
│   └── solve-es3-api                  # Auto-solve
├── 02-explore-telemetry/
│   ├── assignment.md
│   ├── setup-es3-api
│   ├── check-es3-api
│   └── solve-es3-api
├── 03-inject-fault/
│   ├── assignment.md
│   ├── setup-es3-api
│   ├── check-es3-api                  # Accepts both ACTIVE and RESOLVED fault states
│   └── solve-es3-api
└── 04-autonomous-remediation/
    ├── assignment.md
    ├── setup-es3-api
    ├── check-es3-api
    └── solve-es3-api
```

---

## What Gets Provisioned at Setup

`track_scripts/setup-es3-api` is aligned with **[elastic-autonomous-observability](https://play.instruqt.com/manage/elastic/tracks/elastic-autonomous-observability)**. It runs once at track start and:

1. **Creates an Elastic Cloud Serverless Observability project** (AWS us-east-1) using `ESS_CLOUD_API_KEY`
2. **Generates an Elasticsearch API key** for the `admin` user (`.encoded` form)
3. **Installs NGINX** on port **8080** as a Kibana reverse proxy (Basic auth to the project)
4. **Serves** `/loading` and `/chatbot` static pages
5. **Starts** the JSON credentials server on port **8081**
6. **Clones** `elastic-launch-demo` (branch `feat/back-navigation-noc-chaos`), **patches** `scenarios/__init__.py` to remove **Claro** from the scenario list, installs deps, and starts **systemd** `elastic-demo` on port **8090** with **`ACTIVE_SCENARIO=banking`**
7. **POSTs `/api/setup/launch`** with `scenario_id: banking` so the **Retail Banking Platform** deploys into the project (alerts, workflows, dashboards, ML, etc. come from that deployment — not from extra bash in this repo)

Shell aliases: `demo-logs`, `demo-status`, `demo-deployments`, `demo-chaos`, `demo-restart` (see **Diagnostic Commands**).

---

## ES|QL Quick Reference

After the scenario is running, use Discover’s data views (for example **All logs** / `logs-*`) or ES|QL against `logs*`, `metrics*`, `traces-*`:

```esql
FROM logs*
| STATS total = COUNT(*), services = COUNT_DISTINCT(service.name)
| LIMIT 10

FROM logs*
| WHERE @timestamp > NOW() - 30 minutes AND severity_text == "ERROR"
| STATS errors = COUNT(*) BY service.name
| SORT errors DESC
```

### Common fields (OTel in Elastic)

| Concept | Typical field | Example |
|---|---|---|
| Service | `service.name` | `auction-engine` |
| Log body | `message` or `body.text` | (depends on integration) |
| Severity | `log.level` / `severity_text` | `ERROR` |
| Trace | `trace.id` | hex string |
| Time | `@timestamp` | ISO-8601 |

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

Run these in the **Terminal** tab on the Instruqt VM:

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
- A **separate notes slide** (second `notes` entry) titled **“While You Wait — Play O11y Survivors!”** with an `<iframe>` to [Vampire-Clone](https://poulsbopete.github.io/Vampire-Clone/) at **height 800** (no extra O11Y tab).
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
vim 02-explore-telemetry/assignment.md

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
| "Unknown column @timestamp" in ES\|QL | Wrong data view / no data yet | Pick **All logs** or `logs*`; widen time range; wait for scenario deploy |
| Empty chaos fault dropdown | No active deployment | Open Demo App → **Launch**; or `demo-restart` then redeploy |
| ML / SLOs missing | Scenario not finished pushing assets | Wait for deployment progress; refresh Kibana |
