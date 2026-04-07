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
│  │  /loading        │  ← PayPal value props splash page           │
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
├── track.yml                          # Track metadata (slug, title, enhanced_loading: false)
├── config.yml                         # VM definition + secrets block (ESS_CLOUD_API_KEY, LLM_PROXY_PROD)
├── README.md                          # This file
├── docs/
│   └── splash.html                    # PayPal value props page (hosted via rawcdn.githack.com)
├── track_scripts/
│   ├── setup-es3-api                  # Main provisioning script (~1500 lines)
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

`track_scripts/setup-es3-api` runs once at track start and:

1. **Creates an Elastic Cloud Serverless project** (Observability tier, AWS us-east-1) using `ESS_CLOUD_API_KEY`
2. **Generates an API key** (`admin` user, `.encoded` format for OTLP `ApiKey` auth)
3. **Derives the OTLP endpoint** from the ES URL (`.es.` → `.ingest.`, port forced to `443`)
4. **Installs NGINX** as a Kibana proxy on port 8080 with auto-injected `ApiKey` auth header
5. **Clones and configures** `elastic-launch-demo` (branch `feat/back-navigation-noc-chaos`)
6. **Applies runtime patches** to the demo app:
   - Renames all Fanatics services/channels/displays → PayPal equivalents
   - Fixes OTLP error handling (`raise_for_status()` + `HTTPStatusError` logging)
   - Ensures chaos channel dropdown is always populated (static fallback)
   - Fixes back-navigation buttons to use `/home` instead of `history.back()`
7. **Launches the fanatics scenario** with the Elastic Serverless credentials
8. **Creates Kibana Workflows** (PayPal-specific alert automation playbooks)
9. **Creates Kibana Dashboards** (PayPal-specific, imported via `.ndjson`)
10. **Creates SLOs** for all 7 services
11. **Creates ML anomaly detection jobs** (see below)
12. **Creates index aliases** (`logs.otel`, `traces.otel`, `metrics.otel`) once data streams exist
13. **Installs diagnostic helper commands** in `/usr/local/bin`

---

## ML Anomaly Detection Jobs

Three ML jobs are created automatically and start scanning telemetry in real time:

| Job ID | Detector | Data |
|---|---|---|
| `paypal-service-error-spike` | `high_count` errors by `service.name` | `logs-apm.otel-*` (ERROR only) |
| `paypal-transaction-volume-anomaly` | `count` by `service.name` | `logs-apm.otel-*` (all logs) |
| `paypal-checkout-degradation` | `high_count` errors in checkout pipeline | `logs-apm.otel-*` (checkout/payments/fraud services) |

View in Kibana: **Analytics → Machine Learning → Anomaly Detection → Anomaly Explorer**

---

## ES|QL Quick Reference

All queries use the `logs-apm.otel-*` index pattern (data stream alias `logs.otel` also works after setup):

```esql
-- Sanity check: confirm data is flowing
FROM logs-apm.otel-*
| STATS total = COUNT(*), services = COUNT_DISTINCT(service.name)

-- Errors by service (last 30 min)
FROM logs-apm.otel-*
| WHERE @timestamp > NOW() - 30 minutes AND severity_text == "ERROR"
| STATS errors = COUNT(*) BY service.name
| SORT errors DESC

-- Recent error log messages for a specific service
FROM logs-apm.otel-*
| WHERE service.name == "payments-orchestrator" AND severity_text == "ERROR"
| KEEP @timestamp, service.name, body.text, trace.id
| SORT @timestamp DESC
| LIMIT 20
```

### Key Field Names (OTel → Elastic mapping)

| OTel Concept | Field in Elastic | Example |
|---|---|---|
| Service identifier | `service.name` | `checkout-service` |
| Log message body | `body.text` | `Payment declined: timeout` |
| Log severity | `severity_text` | `ERROR`, `INFO`, `WARN` |
| Trace correlation ID | `trace.id` | `4f8a2c3d...` |
| Timestamp | `@timestamp` | ISO-8601 |
| Host name | `host.name` | `paypal-prod-us-east-1` |

---

## CAL → OpenTelemetry Field Mapping

| CAL Field | OTel Semantic Convention | Notes |
|---|---|---|
| `CAL_TYPE` | `service.name` | Standard across all CNCF tooling |
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
| `demo-otlp-test` | Test OTLP auth + connectivity; shows exact HTTP status and data stream existence |
| `demo-otlp-errors` | Grep app logs for OTLP warnings and errors |
| `demo-deployments` | Show running scenarios and their Kibana URL |
| `demo-chaos` | Show chaos channel states |
| `demo-status` | Show full demo app status JSON |
| `demo-restart` | Restart the demo app (systemd) |

### Common OTLP troubleshooting

| Symptom | What to run | Expected fix |
|---|---|---|
| "Unknown column @timestamp" in ES\|QL | `demo-otlp-test` | If HTTP 200 → wait 30s for data; if 401/403 → API key issue |
| Generators log "Sent X" but no Kibana data | `demo-otlp-errors` | Look for `OTLP HTTP 401` lines |
| Empty chaos fault dropdown | `demo-restart` | Reloads static channel registry |
| Demo App still spinning | `demo-logs` | Check for `ModuleNotFoundError` — run `demo-restart` |

---

## Key Design Decisions

### `enhanced_loading: false`

Setting `enhanced_loading: false` prevents a "multiple starts" race condition observed in testing. The `/loading` path on NGINX serves the PayPal value props splash page while the VM provisions.

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

### Demo App Patching

The `elastic-launch-demo` app is patched at runtime (not forked) to preserve upstream updates. All PayPal customizations live in `track_scripts/setup-es3-api` as Python and bash patch blocks. The key patches are:

| Patch target | What it does |
|---|---|
| `scenarios/fanatics/scenario.py` | Renames all services, chaos channels, and display strings to PayPal equivalents |
| `scenarios/fanatics/services/*.py` | Renames service module files to match the new import paths |
| `app/main.py` | Returns static channel registry when no deployment is active (fixes empty fault dropdown) |
| `app/telemetry.py` | Injects `raise_for_status()` + catches `HTTPStatusError` to surface OTLP 4xx errors |
| `elastic_config/deployer.py` | Accepts HTTP 405 as a valid OTLP endpoint probe response |
| `app/landing/static/index.html` | Removes broken Elastic Observability and API Endpoints sections |
| `app/chaos_ui/static/index.html` | Fixes back button to use `/home` instead of `history.back()` |
| `app/dashboard/static/index.html` | Fixes back button to use `/home` |

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

---

## Track IDs

| Resource | Value |
|---|---|
| Track slug | `paypal-elastic-serverless` |
| Track ID | `q7432fqzit6g` |
| Instruqt org | `elastic` |
| GitHub repo | `github.com/poulsbopete/paypal-elastic-serverless` |
| Upstream inspiration | `elastic/elastic-autonomous-observability` (financial scenario; this track is **not** `elastic-autonomous-observability-copy`) |
| Demo app repo | `github.com/poulsbopete/elastic-launch-demo` (branch `feat/back-navigation-noc-chaos`) |

---

## Full Troubleshooting Reference

| Symptom | Likely Cause | Fix |
|---|---|---|
| "The host for this script was not found" | Sandbox config with empty VMs is linked | Unlink via GraphQL: `setTrackSandboxConfigVersion` with empty `configVersionID` |
| Secrets not available in setup script | `ESS_CLOUD_API_KEY` not in Elastic org secret store | Add secret in Instruqt org settings |
| Demo app never healthy | Python deps failed / import error | `journalctl -u elastic-demo -n 50` on VM |
| Kibana not loading | NGINX not started | `systemctl status nginx` on VM |
| "Unknown column @timestamp" in ES\|QL | `logs-apm.otel-*` data stream empty | `demo-otlp-test` → check HTTP status |
| OTLP silently failing (HTTP 401) | API key not in correct `.encoded` format | Check `agent variable get ES_API_KEY` — should be base64-encoded `id:api_key` |
| Empty chaos fault dropdown | No active deployment registered | `demo-restart` — static registry fallback serves all channels |
| ML jobs not appearing in Kibana | Data streams didn't exist at setup time | Jobs auto-start 60s after data flows; check ML → Anomaly Detection |
| SLOs not created | Kibana not reachable at setup time | Re-run creation manually or restart track |
