# Setup Guide — v2

---

## Prerequisites

- n8n ≥ 1.0.0 (self-hosted or Cloud)
- PostgreSQL (required for production queue mode; SQLite fine for dev)
- Redis (required for queue mode only)

---

## Step 1 — Run n8n

**Development (Docker, SQLite):**
```bash
docker run -it --rm \
  -p 5678:5678 \
  -e N8N_BASIC_AUTH_ACTIVE=true \
  -e N8N_BASIC_AUTH_USER=admin \
  -e N8N_BASIC_AUTH_PASSWORD=changeme \
  -e WEBHOOK_URL=https://your-domain.com/ \
  --env-file .env \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

**Production (Docker Compose + Queue Mode):**
```bash
# Copy and fill in .env.example → .env
cp .env.example .env

docker compose up -d   # starts main + 2 workers + Redis + Postgres
```

---

## Step 2 — Set Environment Variables

Create `.env` from this template:

```bash
# ── Sheet IDs ─────────────────────────────────────────────────────
LOG_SHEET_ID=
LEADS_SHEET_ID=
CRM_SHEET_ID=
METRICS_SHEET_ID=

# ── Security ──────────────────────────────────────────────────────
WEBHOOK_SECRET=<generate: openssl rand -hex 32>

# ── API Keys ──────────────────────────────────────────────────────
HUNTER_API_KEY=
PAGERDUTY_ROUTING_KEY=

# ── Notifications ─────────────────────────────────────────────────
REPORT_EMAIL_LIST=exec@company.com

# ── Runtime ───────────────────────────────────────────────────────
N8N_HOST=https://n8n.yourcompany.com
NODE_ENV=production
NOTION_TICKETS_DB_ID=
```

---

## Step 3 — Create Credentials

Go to **Settings → Credentials → + Add Credential** for each:

| Name (exact) | Type | Notes |
|---|---|---|
| `Google Sheets — Service Account` | Google Sheets OAuth2 | Enable Sheets API; share all sheets with service account email |
| `OpenAI API` | OpenAI | Requires GPT-4o access (Tier 2+) |
| `Slack — Automation Bot` | Slack API | Scopes: `chat:write`, `chat:write.public` |
| `Gmail — Automation Account` | Gmail OAuth2 | Scopes: `gmail.modify` |
| `HubSpot — Private App` | HTTP Header Auth | Header: `Authorization: Bearer TOKEN` |
| `Clearbit API` | HTTP Header Auth | Header: `Authorization: Bearer KEY` |
| `Notion API` | HTTP Header Auth | Headers: `Authorization: Bearer TOKEN`, `Notion-Version: 2022-06-28` |
| `Stripe API` | HTTP Header Auth | Header: `Authorization: Bearer sk_live_...` |
| `Analytics API` | HTTP Header Auth | Header per your analytics provider |
| `CRM API` | HTTP Header Auth | Header per your CRM |
| `SMTP — Notifications` | SMTP | Port 587, TLS |

---

## Step 4 — Create Google Sheets

Create one Google Sheet with these tabs:

**LOG_SHEET_ID** (shared logger sheet):
```
application_logs: timestamp | level | traceId | workflow | event | message | data | error | durationMs | env
error_log:        timestamp | traceId | severity | workflowName | workflowId | executionId | lastNode | errorMessage | errorStack | n8nUiUrl
```

**LEADS_SHEET_ID:**
```
leads_v2: leadId | traceId | tier | scoreTotal | firstName | lastName | email | phone | jobTitle | company | companySize | companyIndustry | companyCountry | techStack | emailStatus | source | utmSource | utmCampaign | enrichmentSuccess | ingestedAt | enrichedAt
```

**CRM_SHEET_ID:**
```
contacts: syncId | traceId | eventType | sourceSystem | email | firstName | lastName | phone | company | jobTitle | lifecycleStage | leadStatus | country | hubspotId | hubspotStatus | dealName | dealAmount | dealStage | receivedAt | lastSyncedAt | hasErrors | errorSummary
```

**METRICS_SHEET_ID:**
```
daily_metrics: pipelineId | traceId | reportDate | pipelineStatus | dataCompleteness | pageviews | sessions | newUsers | conversions | conversionRate | bounceRate | avgSessionDuration | newContacts | dealsCreated | dealsClosed | closeRate | pipelineValue | grossRevenue | totalRefunds | netRevenue | transactionCount | avgTransactionValue | refundRate | revenuePerSession | revenuePerContact | leadToCloseRate | analyticsQuality | crmQuality | revenueQuality | transformedAt | startTime
```

---

## Step 5 — Create Notion Database (Workflow 04)

Create a Notion database with these properties:

| Property | Type |
|----------|------|
| Ticket ID | Title |
| Subject | Text |
| Customer Email | Email |
| Customer Name | Text |
| Category | Select |
| Subcategory | Text |
| Priority | Select |
| Assigned Team | Select |
| Status | Select |
| Sentiment | Select |
| Frustration Score | Number |
| Plan Type | Select |
| Source | Select |
| Escalate | Checkbox |
| SLA Deadline | Date |
| Submitted At | Date |
| AI Confidence | Number |
| Trace ID | Text |

Share the database with your Notion integration. Copy the database ID from the URL (32-char hex string) → set `NOTION_TICKETS_DB_ID`.

---

## Step 6 — Create Slack Channels

Create these channels and invite your bot (`/invite @YourBotName`):

```
#sales-hot-leads       # HOT lead alerts
#sales-pipeline        # WARM lead notifications
#email-review-queue    # Emails flagged for human review
#support-{team}        # Dynamic — one per support team
#oncall-p1-alerts      # P1 critical ticket alerts
#automation-logs       # WARN-level log entries
#oncall-alerts         # ERROR/FATAL log entries
#automation-errors     # CRM sync errors, pipeline failures
#daily-metrics         # Daily ETL report
```

---

## Step 7 — Import Workflows

**Import order is mandatory:**

```bash
# Option A: n8n CLI
n8n import:workflow --input=workflows/00_shared_error_handler.json
n8n import:workflow --input=workflows/00_shared_logger.json
n8n import:workflow --input=workflows/01_lead_enrichment.json
n8n import:workflow --input=workflows/02_ai_email_responder.json
n8n import:workflow --input=workflows/03_crm_sync.json
n8n import:workflow --input=workflows/04_support_ticket_classifier.json
n8n import:workflow --input=workflows/05_etl_pipeline.json

# Option B: n8n UI
# Workflows → + → Import from File → select .json
```

---

## Step 8 — Assign Credentials

For each imported workflow:
1. Open workflow → click any node with a warning icon (⚠️)
2. Select the matching credential from the dropdown
3. Save

---

## Step 9 — Activate

**Activate in this order:**

1. `[SHARED] Global Error Handler` — activate first
2. `[SHARED] Structured Logger` — activate second
3. All business workflows — any order

To activate: open workflow → toggle **Active** switch (top right) → confirm.

Webhook URLs become live upon activation. Copy the **Production URL** (not Test URL) for each webhook workflow.

---

## Step 10 — Smoke Test

```bash
# Test wf.lead-enrichment
curl -s -X POST https://your-n8n.com/webhook/v1/leads/ingest \
  -H "Content-Type: application/json" \
  -d '{"firstName":"Test","lastName":"User","email":"test@stripe.com","source":"smoke-test"}' \
  | python3 -m json.tool

# Expected: {"status":"accepted","leadId":"LEAD-...","tier":"HOT","score":65,"traceId":"tr-lead-..."}

# Test wf.support-ticket-classifier
curl -s -X POST https://your-n8n.com/webhook/v1/support/tickets \
  -H "Content-Type: application/json" \
  -d '{"subject":"URGENT: Production API down","body":"All endpoints returning 503 for the past 20 minutes","email":"cto@enterprise.com","name":"Enterprise CTO","planType":"enterprise"}' \
  | python3 -m json.tool

# Expected: {"status":"created","priority":"P1_critical","assignedTeam":"engineering",...}
```

After each test:
- Check Google Sheets `application_logs` for the log entries
- Check the relevant Slack channels for notifications
- Check n8n UI → Executions for the full execution graph
