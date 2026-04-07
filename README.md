# n8n Automation Portfolio — v2

**Senior n8n Developer · Automation Architect · AI Integration Engineer**

> 7 production-grade workflows built to the engineering standards of high-scale SaaS teams. Shared infrastructure, structured observability, HMAC security, circuit-breaker fault tolerance, and modular design — not just automations, but an automation *platform*.

[![n8n](https://img.shields.io/badge/n8n-1.0%2B-FF6D5A?style=flat-square&logo=n8n)](https://n8n.io)
[![OpenAI](https://img.shields.io/badge/GPT--4o-integrated-412991?style=flat-square&logo=openai)](https://openai.com)
[![Quality](https://img.shields.io/badge/code_quality-FAANG--grade-0EA5E9?style=flat-square)]()
[![Available](https://img.shields.io/badge/available-within_2_weeks-22C55E?style=flat-square)]()

---

## What separates v2 from v1

| Concern | v1 | v2 |
|---------|----|----|
| **Node naming** | `Function - Validate & Normalize` | `fn.ValidateAndNormalize` — typed prefix system |
| **Logging** | Ad-hoc `console.log` + inline Slack | Dedicated `[SHARED] Structured Logger` — fans out to Sheets + Slack with Block Kit |
| **Error handling** | Inline Slack node per workflow | `[SHARED] Global Error Handler` — PagerDuty for CRITICAL, Slack for ERROR, full audit log |
| **Security** | HMAC commented out | Constant-time `crypto.timingSafeEqual` HMAC verification |
| **Retry logic** | `maxTries: 3` on some nodes | All external HTTP nodes: `retryOnFail: true, maxTries: 3-4, waitBetweenTries: 1500-5000ms` |
| **AI safety** | None | AI output stripped of self-disclosure; deterministic business rules always override AI |
| **Scoring** | Inline if/else | Table-driven scoring engine — O(n) extension with zero logic changes |
| **Idempotency** | None | SHA-256 content-addressed IDs (same email + source = same leadId, always) |
| **Partial success** | Basic | HTTP 207 Multi-Status; full destination state machine (`pending/success/failed`) |
| **Data quality** | Missing | Per-source quality flags: `GOOD / DEGRADED / MISSING / ERROR` |
| **Circuit breaker** | None | ETL pipeline continues with degraded data rather than aborting |
| **Trace IDs** | Some | Every workflow stamps `traceId` at entry; propagated to all logs and responses |

---

## Repository Structure

```
n8n-portfolio-v2/
│
├── workflows/
│   │
│   ├── [SHARED] — Deploy these first
│   │   ├── 00_shared_error_handler.json    # Global error handler (errorTrigger)
│   │   └── 00_shared_logger.json           # Structured logger (executeWorkflow target)
│   │
│   ├── [DOMAIN] — Business workflows
│   │   ├── 01_lead_enrichment.json         # wf.lead-enrichment
│   │   ├── 02_ai_email_responder.json      # wf.ai-email-responder
│   │   ├── 03_crm_sync.json               # wf.crm-sync
│   │   ├── 04_support_ticket_classifier.json # wf.support-ticket-classifier
│   │   └── 05_etl_pipeline.json           # wf.etl-pipeline
│   │
├── docs/
│   ├── ARCHITECTURE.md                     # System design + data flows
│   ├── SCALABILITY.md                      # Scaling playbook
│   ├── RUNBOOKS.md                         # Operational runbooks
│   └── SETUP_GUIDE.md                      # Step-by-step deployment
│
├── payloads/
│   └── sample_webhook_payloads.json
│
└── README.md
```

---

## Infrastructure Workflows (Deploy First)

### `[SHARED] Structured Logger`
A reusable sub-workflow invoked via `Execute Workflow` node from every business workflow. Accepts a structured log envelope and fans out to:
- **Console** (always) — captured by any log aggregator connected to n8n stdout
- **Google Sheets** `application_logs` tab (always)
- **Slack** `#automation-logs` for WARN, `#oncall-alerts` for ERROR/FATAL

All log entries carry: `traceId`, `timestamp`, `level`, `workflow`, `event`, `message`, `data`, `error`, `durationMs`.

### `[SHARED] Global Error Handler`
Set as `settings.errorWorkflow` on every business workflow. Fires on any uncaught exception. Triages by severity (CRITICAL vs ERROR), dispatches to Slack with Block Kit (including a deep-link button to the failed execution), writes to a dedicated `error_log` sheet, and fires PagerDuty for CRITICAL workflows.

---

## Business Workflows

### `wf.lead-enrichment` — Lead Generation & Enrichment
**Entry:** `POST /v1/leads/ingest`

**Architecture highlights:**
- **Idempotency:** SHA-256 content-addressed `leadId` — re-submitting the same lead is safe
- **Parallel enrichment:** Clearbit and Hunter.io called concurrently; each `continueOnFail: true`
- **Table-driven scoring:** 10-signal scoring table (0–100). Add a new signal by adding one row — no logic changes needed
- **Tier classification:** HOT (≥65) / WARM (≥40) / COLD (<40)
- **Block Kit notifications:** Rich Slack messages showing matched scoring signals per lead

**Data flow:**
```
Webhook → [Validate + Idempotency Key] → [Clearbit ‖ Hunter] → [Merge + Score]
       → [gate: COLD?] → reject (HTTP 200)
       → [Sheets upsert] → [gate: HOT?] → Block Kit alert → log
```

---

### `wf.ai-email-responder` — AI Email Auto-Responder
**Entry:** Gmail poll (every minute)

**Architecture highlights:**
- **RFC 3834 loop prevention:** Checks `auto-submitted`, `x-auto-response-suppress`, `precedence`, subject patterns — prevents infinite reply loops
- **Two-model strategy:** GPT-4o-mini (temperature 0.05) for classification; GPT-4o (temperature 0.65) for reply generation
- **Deterministic safety layer:** Regex patterns for legal language, urgency, and low confidence override AI routing decisions — AI never has the final word on safety
- **AI self-disclosure guard:** Strips any accidental AI self-identification from generated replies before sending

**Routing logic:**
```
Gmail → [Preprocess + loop guard] → [GPT-4o-mini classify]
      → [Parse + override rules]
      → [gate: Human review?]
           → YES: Block Kit → #email-review-queue
           → NO:  [GPT-4o generate reply] → [Safety strip] → Gmail send → log
```

---

### `wf.crm-sync` — CRM Sync System
**Entry:** `POST /v1/crm/events`

**Architecture highlights:**
- **Constant-time HMAC:** `crypto.timingSafeEqual` prevents timing-based signature bypass attacks
- **409 Conflict tolerance:** HubSpot duplicate contact treated as idempotent success, not an error
- **Destination state machine:** Each destination tracked as `pending → success | failed`
- **HTTP 207 Multi-Status:** Returns partial success if one destination fails, preserving data at the other
- **Comprehensive error record:** Error arrays carry `{ destination, statusCode, message, category }` for root-cause analysis

**State machine:**
```
pending ──[HTTP success]──▶ success
pending ──[HTTP 409]──────▶ success (idempotent)
pending ──[HTTP 4xx/5xx]──▶ failed  → Slack alert + HTTP 207
```

---

### `wf.support-ticket-classifier` — AI Support Ticket Classifier
**Entry:** `POST /v1/support/tickets`

**Architecture highlights:**
- **Pre-signal injection:** Regex patterns (urgency, legal, billing, security keywords) detected before AI — seeded into prompt as structured context
- **SLA-aware output:** AI returns `sla_minutes` per priority; workflow computes ISO 8601 `slaDeadline` timestamp
- **Enforcement cascade:** Business rules applied in strict priority order (security > legal > urgency > enterprise plan)
- **Dynamic Slack routing:** P1 → `#oncall-p1-alerts`; P2+ → `#support-{team}` with team name interpolated

**Priority routing:**
```
P1_critical → #oncall-p1-alerts (15-min SLA)
P2_high     → #support-{assignedTeam} (60-min SLA)
P3_medium   → #support-{assignedTeam} (240-min SLA)
P4_low      → #support-{assignedTeam} (1440-min SLA)
```

---

### `wf.etl-pipeline` — Daily ETL Data Pipeline
**Entry:** Schedule (daily, 6AM ET)

**Architecture highlights:**
- **Single context node:** `fn.InitContext` is the single source of truth for all date ranges, IDs, and config — prevents date drift between parallel branches
- **Circuit-breaker pattern:** Each source independently assessed as `GOOD / DEGRADED / MISSING / ERROR`; pipeline continues with whatever is available
- **Quality-aware status:** `COMPLETE` (3/3 sources good) / `DEGRADED` (2/3) / `FAILED` (≤1); only FAILED aborts reporting
- **Safe math helpers:** `safe()`, `pct()`, `avg()`, `r2()` — all division guarded, all output rounded at the output layer
- **HTML email report:** Responsive CSS grid with quality badges; no external dependencies

**Quality decision tree:**
```
COMPLETE  → Sheets + AI summary + Slack + Email
DEGRADED  → Sheets + AI summary (with quality caveat) + Slack + Email
FAILED    → Error log + Slack alert only (no misleading report)
```

---

## Node Naming Convention

All nodes follow a typed prefix system for instant visual scanning:

| Prefix | Type | Example |
|--------|------|---------|
| `trigger.` | Entry point | `trigger.WebhookIngest` |
| `fn.` | Function (JavaScript) | `fn.ValidateAndNormalize` |
| `gate.` | IF / Switch routing | `gate.IsHot?` |
| `http.` | HTTP Request | `http.Clearbit` |
| `ai.` | LLM / AI node | `ai.ClassifyIntent` |
| `sink.` | Write to destination | `sink.Sheets.StoreLead` |
| `notify.` | Send notification | `notify.Slack.Hot` |
| `log.` | Execute Logger sub-workflow | `log.LeadQualified` |
| `action.` | External action (send email, etc.) | `action.GmailSendReply` |
| `respond.` | Respond to Webhook node | `respond.Accepted` |

---

## Credential Reference

All credentials use n8n's encrypted credential store, referenced by name.

| Credential Name | Type | Used In |
|-----------------|------|---------|
| `Google Sheets — Service Account` | Google Sheets OAuth2 | All workflows |
| `OpenAI API` | OpenAI | 02, 04, 05 |
| `Slack — Automation Bot` | Slack API | All workflows |
| `Gmail — Automation Account` | Gmail OAuth2 | 02 |
| `HubSpot — Private App` | HTTP Header Auth | 03 |
| `Clearbit API` | HTTP Header Auth | 01 |
| `Notion API` | HTTP Header Auth | 04 |
| `Stripe API` | HTTP Header Auth | 05 |
| `Analytics API` | HTTP Header Auth | 05 |
| `CRM API` | HTTP Header Auth | 05 |
| `SMTP — Notifications` | SMTP | 05 |

---

## Environment Variables

```bash
# Sheet IDs
LEADS_SHEET_ID=1abc...          # Workflow 01
CRM_SHEET_ID=1def...            # Workflow 03
METRICS_SHEET_ID=1ghi...        # Workflow 05
LOG_SHEET_ID=1jkl...            # Shared Logger + Error Handler

# Security
WEBHOOK_SECRET=<32+ char secret>  # Workflows 01, 03, 04

# API Keys (use env vars, never hardcode)
HUNTER_API_KEY=...
PAGERDUTY_ROUTING_KEY=...

# Notifications
REPORT_EMAIL_LIST=exec@company.com,data@company.com

# Runtime
N8N_HOST=https://n8n.yourcompany.com
NODE_ENV=production
NOTION_TICKETS_DB_ID=...
```

---

## Setup

See **[docs/SETUP_GUIDE.md](docs/SETUP_GUIDE.md)** for full deployment instructions.

**Import order matters:**
1. `00_shared_error_handler.json` — activate first
2. `00_shared_logger.json` — activate second
3. Business workflows (any order)

---

## About

Senior automation engineer specialising in n8n, API integration, and AI-augmented business processes. Experienced building production automation infrastructure for SaaS, agencies, and enterprise clients.

Focused on: reliability, observability, security, and maintainability — not just "it works".

**Available to join within 2 weeks.**

---

*n8n · GPT-4o · HubSpot · Google Workspace · Slack · Notion · Stripe · PagerDuty*
