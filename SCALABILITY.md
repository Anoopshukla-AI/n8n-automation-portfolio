# Scalability Architecture

## Throughput Baseline (Single n8n Instance)

A default n8n deployment (2 vCPU, 4GB RAM) handles approximately:

| Workflow Type | Throughput | Bottleneck |
|--------------|-----------|-----------|
| Webhook (no AI) | ~200 req/min | n8n execution queue |
| Webhook (with AI) | ~40 req/min | OpenAI API latency (~1-3s) |
| Gmail poll | ~60 emails/poll cycle | Gmail API rate limit |
| ETL pipeline | 1 run / schedule | Source API limits |

---

## Scaling Strategies

### 1. Queue Mode (Primary Scaling Lever)

The most impactful change. Decouples webhook reception from execution:

```bash
# docker-compose.yml — queue mode setup
services:
  n8n-main:
    image: n8nio/n8n
    environment:
      EXECUTIONS_MODE: queue
      QUEUE_BULL_REDIS_HOST: redis
      N8N_CONCURRENCY_PRODUCTION_LIMIT: 10   # per worker

  n8n-worker:
    image: n8nio/n8n
    command: worker
    environment:
      EXECUTIONS_MODE: queue
      QUEUE_BULL_REDIS_HOST: redis
    deploy:
      replicas: 4     # scale this independently

  redis:
    image: redis:7-alpine
```

With 4 workers × 10 concurrency = **40 concurrent executions**.
Scale workers horizontally for linear throughput increase.

### 2. Webhook Response Decoupling

All webhook-triggered workflows use `responseMode: responseNode`, which means:

- The webhook immediately hands off to the execution queue
- The caller receives the response from `respond.*` node (not after full execution)
- The heavy processing (AI calls, API enrichment) runs async

This gives **sub-100ms HTTP acknowledgment** regardless of workflow complexity.

### 3. Parallel Execution Design

Workflows 01 and 05 fan out to multiple HTTP calls at the same time:

```
fn.Init ──┬── http.Clearbit    ──┐
          └── http.Hunter     ──┴── fn.Merge  (both branches wait)
```

n8n executes these branches in parallel. For 3 sources each taking 2s, total
wait is ~2s not ~6s. This pattern is used in every workflow that calls multiple APIs.

### 4. Rate Limit Handling

Per-node retry configuration:

```
retryOnFail: true
maxTries: 3
waitBetweenTries: 1500ms   (Clearbit, Hunter — fast APIs)
waitBetweenTries: 3000ms   (HubSpot — stricter limits)
waitBetweenTries: 5000ms   (ETL sources — batch APIs)
```

For Stripe's 429 (rate limit), the 3×5000ms retry sequence provides:
- Attempt 1: immediate
- Attempt 2: +5s
- Attempt 3: +10s
Total backoff: 15s before final failure — appropriate for Stripe's 429 TTL.

For high-volume scenarios, add a **Wait node** (500ms) before each external API
call to implement request throttling at the workflow level.

### 5. Execution History Management

Large execution histories degrade UI performance. Tune these env vars:

```bash
EXECUTIONS_DATA_MAX_AGE=72         # hours before pruning
EXECUTIONS_DATA_PRUNE=true
DB_POSTGRESDB_DATABASE=n8n         # prefer PostgreSQL over SQLite in production
```

For the highest throughput, disable execution saving for successful runs
on high-frequency webhook workflows:

```json
"saveDataSuccessExecution": "none"   // vs "last" or "all"
```

The shared logger writes to Google Sheets regardless, so observability is preserved.

---

## Database Choice

| Database | Max Concurrent Executions | Notes |
|----------|--------------------------|-------|
| SQLite (default) | ~10 | Dev/staging only |
| PostgreSQL | ~200+ | Required for queue mode |
| PostgreSQL + PgBouncer | ~500+ | Connection pooling for high throughput |

```bash
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=your-postgres-host
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n
DB_POSTGRESDB_PASSWORD=<secure-password>
```

---

## Observability at Scale

### Log Volume Estimation

| Workflow | Log Events per Execution | Frequency | Monthly Volume |
|---------|--------------------------|-----------|---------------|
| wf.lead-enrichment | 2 (ingested + qualified) | ~500/day | ~30K rows |
| wf.ai-email-responder | 2 (classified + replied) | ~200/day | ~12K rows |
| wf.crm-sync | 2 (sync events) | ~300/day | ~18K rows |
| wf.support-ticket-classifier | 1 | ~100/day | ~3K rows |
| wf.etl-pipeline | 1-2 | 1/day | ~60 rows |

**Total: ~63K rows/month** — well within Google Sheets' 10M cell limit.
For higher volume, replace the `sink.GoogleSheets` node in the shared logger
with a Postgres INSERT or a BigQuery streaming insert.

### Distributed Tracing

Every workflow stamps a `traceId` at entry and propagates it:
- Into every log call
- Into all API responses (`traceId` field)
- Into the error handler

This enables filtering the entire lifecycle of a single request across:
- Google Sheets application_logs
- Google Sheets error_log
- Slack messages
- n8n execution history

Query example (Google Sheets / BigQuery):
```sql
SELECT * FROM application_logs
WHERE traceId = 'tr-lead-a1b2c3d4'
ORDER BY timestamp ASC;
```

---

## Multi-Environment Strategy

```
environments/
├── dev/
│   └── .env          N8N_HOST=http://localhost:5678
│                     EXECUTIONS_DATA_MAX_AGE=24
│                     WEBHOOK_SECRET=dev-secret-do-not-use-in-prod
├── staging/
│   └── .env          N8N_HOST=https://n8n-staging.yourcompany.com
│                     LOG_SHEET_ID=<staging-sheet>
└── production/
    └── .env          N8N_HOST=https://n8n.yourcompany.com
                      LOG_SHEET_ID=<production-sheet>
                      EXECUTIONS_MODE=queue
```

Workflow JSON is environment-agnostic — all environment-specific values use
`$env.*` references or named credentials. Deploy the same JSON to all environments;
only the `.env` file changes.

---

## Cost Optimisation

### OpenAI Token Budget

| Workflow | Model | Avg Tokens/Execution | Cost @ $0.15/1M |
|---------|-------|---------------------|-----------------|
| wf.ai-email-responder (classify) | gpt-4o-mini | ~450 | $0.000068 |
| wf.ai-email-responder (reply) | gpt-4o | ~800 | $0.002000 |
| wf.support-ticket-classifier | gpt-4o | ~900 | $0.002250 |
| wf.etl-pipeline (summary) | gpt-4o-mini | ~600 | $0.000090 |

At 500 emails/day: ~$1.00/day for email responder.
At 100 tickets/day: ~$0.23/day for ticket classifier.

**Optimisation levers:**
- Use `gpt-4o-mini` for classification tasks (temperature < 0.2) — quality parity at 10× lower cost
- Cache enrichment results (same domain within 24h) using n8n Static Data
- Set `max_tokens` conservatively — the structured JSON output schemas are small

---

## High-Availability Deployment

```yaml
# Kubernetes deployment sketch
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: n8n-main
spec:
  replicas: 1          # main process — single instance
  template:
    spec:
      containers:
        - name: n8n
          image: n8nio/n8n:latest
          env:
            - name: EXECUTIONS_MODE
              value: queue
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: n8n-worker
spec:
  replicas: 4          # scale workers independently
  template:
    spec:
      containers:
        - name: n8n-worker
          image: n8nio/n8n:latest
          args: ["worker"]
          env:
            - name: EXECUTIONS_MODE
              value: queue
```

With this setup:
- Main process handles webhook receipt and scheduling only
- Workers handle all execution — horizontally scalable
- Redis coordinates the queue
- PostgreSQL stores state
- n8n main process can be updated with zero webhook downtime (workers drain existing jobs)
