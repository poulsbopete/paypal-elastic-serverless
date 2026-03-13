# PayPal Merchant Observability: Elastic Serverless + OpenTelemetry

An Instruqt workshop demonstrating how Elastic Serverless + OpenTelemetry replaces CAL logging and delivers unified, merchant-centric observability at PayPal scale.

**Track URL:** https://play.instruqt.com/manage/elastic/tracks/paypal-elastic-serverless

---

## Overview

This lab puts participants in the role of a PayPal SRE evaluating Elastic Serverless as a CAL replacement. Across four challenges, they experience the full observability modernization story: from connecting a live multi-cloud environment to autonomous AI-driven incident remediation.

### Story Arc

| Phase | What Happens |
|---|---|
| **Environment loads** | "Why Elastic" tab shows the PayPal → Elastic value proposition while the VM provisions |
| **Challenge 1** | Connect to Elastic Cloud, verify 7 live microservices emitting OTel telemetry |
| **Challenge 2** | Run ES\|QL queries to explore merchant logs, traces, and metrics — CAL replacement in action |
| **Challenge 3** | Inject a payment fault via the Chaos Controller, watch Elastic surface merchant impact |
| **Challenge 4** | Observe Elastic AI Agent + Workflows auto-investigate and remediate a complex incident |

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Instruqt VM: es3-api (elastic/es3-api-v2)              │
│                                                          │
│  ┌────────────────┐  OTLP   ┌──────────────────────┐   │
│  │  elastic-launch│────────▶│  Elastic Serverless   │   │
│  │  -demo         │         │  Observability Project│   │
│  │  (port 8090)   │         │  (AWS us-east-1)      │   │
│  └────────────────┘         └──────────────────────┘   │
│                                        ▲                 │
│  ┌────────────────┐  proxy             │                 │
│  │  NGINX         │────────────────────┘                 │
│  │  (port 8080)   │                                      │
│  │  /loading      │ ← PayPal value props splash page     │
│  │  /             │ ← Kibana proxy (no login required)   │
│  └────────────────┘                                      │
└─────────────────────────────────────────────────────────┘
```

### PayPal Demo Services

| Service | Function | Cloud |
|---|---|---|
| `merchant-api` | Merchant profile and settings | AWS us-east-1 |
| `checkout-service` | Checkout session orchestration | AWS us-east-1 |
| `payments-orchestrator` | Payment routing and processing | AWS us-east-1 |
| `fraud-decisioning` | Real-time fraud scoring | GCP us-central1 |
| `settlement-service` | Settlement batch processing | GCP us-central1 |
| `merchant-support-api` | Support ticket and account management | Azure eastus |
| `notification-service` | Email, push, and webhook delivery | Azure eastus |

All 7 services emit **logs, metrics, and traces** via OTLP to Elastic Serverless.

---

## Track Structure

```
paypal-elastic-serverless/
├── track.yml                          # Track metadata (slug, title, enhanced_loading: false)
├── config.yml                         # VM definition + secrets block
├── docs/
│   └── splash.html                    # PayPal value props page (hosted on GitHub)
├── track_scripts/
│   ├── setup-es3-api                  # Main provisioning script
│   └── cleanup-es3-api                # Teardown (deletes Elastic Cloud project)
├── 01-connect-and-deploy/
│   ├── assignment.md                  # Challenge content + tabs
│   ├── setup-es3-api                  # Health-check before handoff
│   ├── check-es3-api                  # Completion check
│   └── solve-es3-api                  # Auto-solve (launches paypal scenario)
├── 02-explore-telemetry/
├── 03-inject-fault/
└── 04-autonomous-remediation/
```

---

## Key Design Decisions

### `enhanced_loading: false`

The track uses `enhanced_loading: false` (matching the upstream `elastic-autonomous-observability-copy` baseline). This is intentional:

- `enhanced_loading: true` caused a "multiple starts" race condition in testing
- `type: website` tabs (like "Why Elastic") are always visible regardless of this setting
- The `/loading` path on the NGINX proxy serves the PayPal value props page until Kibana is ready

### Secrets via `config.yml`

Secrets are declared in `config.yml` under the `es3-api` VM's `secrets:` block:

```yaml
secrets:
- name: ESS_CLOUD_API_KEY
- name: LLM_PROXY_PROD
```

Instruqt injects these from the organization's secret store automatically. **Do not use a sandbox config** — linking a `sandboxConfig` with an empty `virtualmachines: []` overrides the VM definitions in `config.yml` and causes "The host for this script was not found" errors.

### PayPal Scenario Injection

The track setup script dynamically injects a `PayPalScenario` class into the `elastic-launch-demo` app at runtime (in `track_scripts/setup-es3-api`). If the PayPal scenario fails to launch, it falls back to the `fanatics` scenario so the lab is never broken.

### "Why Elastic" Tab

Challenge 1 includes a `type: website` tab pointing to the externally hosted `docs/splash.html` via `rawcdn.githack.com`. This tab is always visible — even during VM provisioning — making it the ideal place to deliver the PayPal + Elastic value proposition while participants wait for setup to complete.

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

### Test the Track

```bash
instruqt track test --track elastic/paypal-elastic-serverless
```

Or use the Instruqt web UI: https://play.instruqt.com/manage/elastic/tracks/paypal-elastic-serverless

### Update the Splash Page

Edit `docs/splash.html`, commit, push to `main`, then get the new raw CDN URL:

```
https://rawcdn.githack.com/poulsbopete/paypal-elastic-serverless/<commit-sha>/docs/splash.html
```

Update the URL in `01-connect-and-deploy/assignment.md` under the `pp_why_elastic` tab, then `instruqt track push --force`.

---

## CAL → OpenTelemetry Field Mapping

| CAL Field | OTel Semantic Convention |
|---|---|
| `CAL_TYPE` | `service.name` |
| `TXN_ID` | `trace.id` / `transaction.id` |
| `MERCHANT_ID` | `merchant.id` |
| `STATUS` | `http.response.status_code` |
| `DURATION_MS` | span `duration` |
| *(none)* | `deployment.environment`, `cloud.region` |

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| "The host for this script was not found" | A sandbox config with empty VMs is linked to the track | Unlink via GraphQL: `setTrackSandboxConfigVersion` with empty `configVersionID` |
| Secrets not available in setup script | `ESS_CLOUD_API_KEY` not in Elastic org secret store | Add the secret in Instruqt org settings |
| PayPal loading page not visible | CDN URL in `assignment.md` is stale | Update `pp_why_elastic` tab URL to new `rawcdn.githack.com` commit SHA |
| Demo app unhealthy | Python deps failed to install | Check `journalctl -u elastic-demo` on the VM |
| Kibana not loading | NGINX proxy not started | Check `systemctl status nginx` on the VM |

---

## Track IDs

| Resource | ID |
|---|---|
| Track slug | `paypal-elastic-serverless` |
| Track ID | `1tpbmpeq2nao` |
| Instruqt org | `elastic` |
| GitHub repo | `github.com/poulsbopete/paypal-elastic-serverless` |
| Base track | `elastic/elastic-autonomous-observability-copy` |
