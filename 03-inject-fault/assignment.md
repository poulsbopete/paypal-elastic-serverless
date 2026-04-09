---
slug: inject-fault
id: 5wgfot0esuqz
type: challenge
title: Inject Fault — Financial Incident Detection
teaser: Use the Chaos Controller to inject a trading platform fault and watch Elastic
  detect it automatically.
notes:
- type: text
  contents: |
    ## Financial Incident Simulation

    The PayPal trading platform handles billions of dollars in daily order flow. A single fault in the matching engine or order gateway can cascade into settlement failures and regulatory reporting gaps.

    In this challenge, you'll use the **Chaos Controller** to inject a financial incident and observe how Elastic Serverless detects it in real-time.
tabs:
- id: lrubz3y8tquk
  title: Demo App
  type: service
  hostname: es3-api
  path: /
  port: 8090
- id: almy7jlst2rp
  title: Chaos Controller
  type: service
  hostname: es3-api
  path: /chaos
  port: 8090
- id: ytffh7ojjgec
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
- id: mns4y27ssjnf
  title: Terminal
  type: terminal
  hostname: es3-api
difficulty: basic
timelimit: 1200
enhanced_loading: null
---

# Inject a Fault and Watch Elastic Detect It

Now it's time to break something. The incident simulator lets you inject realistic faults into the simulated environment — then watch Elastic light up.

---

## Trigger a Fault

1. Open the **Demo App** tab
2. Find your running **Fanatics Collectibles** deployment and click the **Chaos** link next to it
3. You'll see 20 fault channels organized by subsystem and cloud provider
4. Select a channel from the dropdown and click **Inject Fault**

---

## What to Watch in Elastic Serverless

Once you trigger a fault, switch to the **Elastic Serverless** tab and watch:

### 1. Logs — Error Spike

The tab opens directly to the error log stream. Or navigate there manually:

**Discover → ES|QL** and run:

```esql
FROM logs*
| WHERE @timestamp > NOW() - 10 MINUTES
| WHERE severity_text == "ERROR"
| STATS error_count = COUNT(*) BY service.name
| SORT error_count DESC
```

You should see an error spike from the affected service within seconds.

### 2. Alert Rules — Detection
**Observability → Alerts**

The ES|QL alert rules will fire within 30–60 seconds of the error spike. You'll see an active alert appear.

### 3. AI Agent Investigation
**Observability → AI Assistant** (or the Workflows execution log)

The alert triggers a workflow that calls the AI agent. The agent:
- Identifies the error type from the alert tags
- Queries recent error logs for context
- Searches for related events
- Produces a root-cause analysis summary

---

## Verify the Fault is Active

In the Terminal tab:

```bash
demo-chaos
```

You should see the triggered channel with `"state": "ACTIVE"`.

---

## Fault Channel Reference (Fanatics Scenario)

| Ch | Name | Subsystem | Cloud |
|----|------|-----------|-------|
| 1 | MAC Address Flapping | network_core | Azure |
| 2 | Spanning Tree Topology Change | network_core | Azure |
| 3 | BGP Peer Flapping | network_core | Azure |
| 4 | Firewall Session Table Exhaustion | security | Azure |
| 5 | Firewall CPU Overload | security | Azure |
| 6 | SSL Decryption Certificate Expiry | security | Azure |
| 7 | WiFi AP Disconnect Storm | network_access | GCP |
| 8 | WiFi Channel Interference | network_access | GCP |
| 9 | Client Authentication Storm | network_access | GCP |
| 10 | DNS Resolution Failure Over VPN | network_services | Azure |
| 11 | DHCP Lease Storm | network_services | Azure |
| 12 | Auction Bid Latency Spike | commerce | AWS |
| 13 | Payment Processing Timeout | commerce | AWS |
| 14 | Product Catalog Sync Failure | commerce | AWS |
| 15 | Print Queue Overflow | manufacturing | AWS |
| 16 | Quality Control Rejection Spike | manufacturing | AWS |
| 17 | Fulfillment Label Printer Failure | logistics | GCP |
| 18 | Warehouse Scanner Desync | logistics | GCP |
| 19 | Orphaned Cloud Resource Alert | cloud_ops | GCP |
| 20 | Cross-Cloud VPN Tunnel Flapping | cloud_ops | GCP |

✅ **Ready to continue when** at least one fault channel is active (verified by `demo-chaos` showing `"active": true`).
