---
slug: inject-fault
id: 5wgfot0esuqz
type: challenge
title: PayPal Workshop — Inject a Fault & Test Detection
teaser: Simulate a production-like incident in the multi-cloud demo and watch Elastic
  Serverless ES|QL alerts and workflows react—mirroring PayPal operational readiness
  drills.
notes:
- type: text
  contents: |
    ## Incident detection at platform scale

    **PayPal context:** Large payments platforms need **fast, correlated detection** across apps, payments, fraud, and identity—not siloed log searches. This lab uses the demo’s **Chaos Controller** to inject controlled faults so you can see **ES|QL alert rules** and downstream **workflows** fire on real OTLP signals.

    **Demo:** The **Retail Banking Platform** scenario exposes **20 fault channels** (mobile, ACH, cards, claims, policy, auth, fraud, documents, infra-style failures). They are **synthetic** but map to the kinds of cross-service incidents operations teams rehearse.

    **You will:** inject a fault, observe **logs** and **alerts**, and optionally preview how an **AI-assisted** workflow will pick up the story in the next challenge.
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
difficulty: basic
timelimit: 1200
enhanced_loading: null
---

# Inject a fault and watch Elastic detect it

Use the incident simulator to inject a controlled fault into the **Retail Banking Platform** demo, then watch Elastic surface it in **logs**, **alerts**, and (next challenge) **workflows**—the same signal path you would rely on for **merchant- and platform-facing** incidents.

---

## Trigger a fault

1. Open the **Demo App** tab.
2. Find **Retail Banking Platform** and click **Chaos** next to it.
3. You should see **20 fault channels** grouped by subsystem (labels may vary slightly by demo version).
4. Pick a channel from the dropdown and click **Inject Fault**.

A simple first choice is **Ch 1 — Mobile App API Timeout** (`digital_banking`), which stresses `mobile-gateway` and related services—similar to **app/API path** incidents.

---

## What to watch in Elastic Serverless

After you inject, switch to the **Elastic Serverless** tab.

### 1. Logs — error spike

The tab may open on alerts or Discover. For errors, use **Discover → ES|QL** (same OTel index pattern as the scenario workflows):

```esql
FROM logs.otel, logs.otel.*
| WHERE @timestamp > NOW() - 10 MINUTES AND severity_text == "ERROR"
| STATS error_count = COUNT(*) BY service.name
| SORT error_count DESC
```

> Use **`FROM logs*`** if your project does not expose `logs.otel` yet.

You should see elevated errors from affected services within seconds (exact services depend on the channel).

### 2. Alert rules — detection

**Observability → Alerts**

ES|QL rules deployed with the scenario typically fire within **30–60 seconds** after the error pattern appears—comparable to how you would run **scheduled ES|QL** in Serverless for PayPal org workloads.

### 3. AI agent investigation (preview)

**Observability → AI Assistant** (or the workflow execution log in the next challenge)

The alert can drive a workflow that runs an AI investigation: error type from alert context, recent logs, related signals, and a short root-cause summary.

---

## Verify the fault is active

In the **Chaos Controller** (or **Demo App → Chaos**), confirm the channel you injected shows as active or in progress. In **Elastic Serverless**, you should already see log errors and (shortly) alerts for that fault.

---

## Fault channel reference (Retail Banking demo)

| Ch | Name | Subsystem | Primary affected services |
|----|------|-----------|---------------------------|
| 1 | Mobile App API Timeout | digital_banking | mobile-gateway, auth-gateway |
| 2 | Mobile Deposit Processing Failure | digital_banking | mobile-gateway, document-vault |
| 3 | Push Notification Storm | digital_banking | mobile-gateway, auth-gateway |
| 4 | ACH Direct Deposit Delay | payment_processing | payment-engine, mobile-gateway |
| 5 | Bill Pay Execution Failure | payment_processing | payment-engine, member-portal |
| 6 | Wire Transfer OFAC Block | payment_processing | payment-engine, fraud-sentinel |
| 7 | Debit Card Authorization Failure | payment_processing | payment-engine, fraud-sentinel |
| 8 | Claims FNOL Intake Backlog | claims_management | claims-processor, document-vault |
| 9 | Photo Damage Estimation Timeout | claims_management | claims-processor, document-vault |
| 10 | Claims Payment Disbursement Failure | claims_management | claims-processor, payment-engine |
| 11 | Policy Renewal Batch Failure | policy_administration | policy-manager, quote-engine |
| 12 | Underwriting Rules Engine Error | underwriting | quote-engine, policy-manager |
| 13 | VA Loan Rate Lock Failure | underwriting | quote-engine, payment-engine |
| 14 | Biometric Auth Service Degradation | authentication | auth-gateway, mobile-gateway |
| 15 | MFA Delivery Failure | authentication | auth-gateway, mobile-gateway |
| 16 | Fraud Model False Positive Surge | fraud_detection | fraud-sentinel, payment-engine |
| 17 | Member Session Timeout Cascade | member_services | member-portal, auth-gateway |
| 18 | Document Upload Service Failure | document_management | document-vault, member-portal |
| 19 | Database Replication Lag | payment_processing | payment-engine, mobile-gateway |
| 20 | Certificate Expiration Cascade | authentication | auth-gateway, mobile-gateway |

✅ **Ready to continue when** at least one fault channel is active (the **Check** button confirms this against the demo API).
