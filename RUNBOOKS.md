# Operational Runbooks

Runbooks for every workflow. Each is linked from the Global Error Handler Slack alert.

---

## RB-001: wf.lead-enrichment fails

**Symptoms:** `#automation-errors` alert from `[SHARED] Global Error Handler` for `wf.lead-enrichment`

**Likely causes and steps:**

### VALIDATION_ERROR: missing required fields
- The upstream form or integration is sending incomplete data
- Check the execution payload in n8n UI → last execution → input
- Fix the sending system or add the missing field

### VALIDATION_ERROR: disposable email domain
- Normal rejection — not an error
- If legitimate emails are being blocked, update `BLOCKED_DOMAINS` list in `fn.ValidateAndNormalize`

### Clearbit or Hunter HTTP 4xx
- Both are `continueOnFail: true` — lead will still be processed with lower score
- Check API key validity in n8n credentials: Settings → Credentials → Clearbit API
- Clearbit: 402 = billing issue; 422 = domain not found (normal); 429 = rate limit

### Google Sheets 403
- Re-authenticate: Settings → Credentials → Google Sheets → reconnect OAuth
- Ensure the service account has Editor access to `LEADS_SHEET_ID`

---

## RB-002: wf.ai-email-responder fails

**Symptoms:** Emails not being replied to, or `#email-review-queue` is empty when it shouldn't be

### LOOP_GUARD error (normal)
- Auto-responder correctly detected and discarded
- No action needed

### OpenAI 429 (rate limit)
- Check usage at platform.openai.com
- Temporarily switch `gpt-4o` to `gpt-4o-mini` in `ai.GenerateReply` node
- Add a Wait node (3000ms) before AI nodes

### AI_REPLY_EMPTY error
- GPT returned an empty or very short response
- Check OpenAI status page: status.openai.com
- The email has been classified but not replied to — check `#email-review-queue`

### Gmail OAuth expired
- Re-authenticate: Settings → Credentials → Gmail — Automation Account
- Google OAuth tokens expire after 7 days of inactivity

---

## RB-003: wf.crm-sync fails

**Symptoms:** 207 or 500 responses, contacts missing from HubSpot

### AUTH_ERROR: invalid webhook signature
- The sending system's webhook secret does not match `WEBHOOK_SECRET` env var
- Rotate `WEBHOOK_SECRET` and update both systems simultaneously
- **Never log the secret value**

### AUTH_ERROR: missing header
- The sending system is not including `x-webhook-signature`
- Update the webhook source to include HMAC signature

### HubSpot 401
- Private App token expired or revoked
- Regenerate token in HubSpot: Settings → Integrations → Private Apps
- Update credential in n8n: Settings → Credentials → HubSpot — Private App

### HubSpot 409 (Conflict)
- Contact already exists — this is treated as success (idempotent)
- Only a problem if `fn.EvaluateHubSpotResult` is incorrectly routing 409 to failure branch

### HubSpot 429
- HubSpot rate limit: 110 req/10s for Private Apps
- Increase `waitBetweenTries` to 10000ms in `http.HubSpotUpsert`

### Partial sync (HubSpot failed, Sheets succeeded)
- Data is in Sheets with `hubspotStatus: FAILED`
- Use Google Sheets to identify affected contacts (filter `hubspotStatus = FAILED`)
- Trigger a re-sync by replaying the original webhook payload

---

## RB-004: wf.support-ticket-classifier fails

**Symptoms:** Tickets not appearing in Notion, P1 alerts missing

### OpenAI classification failure
- `cls.confidence = 0.0` + `cls.escalationReason = 'AI_PARSE_FAILURE'`
- Ticket is still routed to `general_support` as P3
- Check OpenAI API status and key validity

### Notion 404
- Database not found — check `NOTION_TICKETS_DB_ID` env var
- Ensure the Notion integration is shared with the database
- The ticket was classified and Slack was notified — Notion storage failed independently

### Notion 400 (validation error)
- A `select` property value doesn't match an existing option
- Go to Notion database settings and add the missing option, OR
- Update the `fn.EnrichAndEnforce` function to map to valid option names

### P1 alert not firing
- Check that `gate.IsP1?` condition is `equal` to `P1_critical` (not `P1`)
- Verify `#oncall-p1-alerts` channel exists and bot is invited

---

## RB-005: wf.etl-pipeline fails or reports FAILED status

**Symptoms:** No Slack/email report by 6:15AM, or `pipelineStatus: FAILED`

### DEGRADED (1-2 sources failed)
- Report was still sent with available data — quality badge shows degraded sources
- Check which sources failed: look at `analyticsQuality`, `crmQuality`, `revenueQuality` in Sheets
- Investigate the relevant API credentials

### FAILED (all sources failed)
- No report sent
- Check `#automation-errors` for the specific HTTP error codes
- Likely a network issue or simultaneous API outage

### Schedule didn't fire
- Check n8n system time: `docker exec n8n date`
- Timezone: workflow is set to `America/New_York` — check `trigger.DailySchedule` settings
- Manually trigger by clicking "Execute Workflow" in the n8n UI

### Google Sheets write fails
- Data was computed but not stored
- Re-run manually with correct `reportDate` in `fn.InitContext`
- Check Google Sheets credentials and `METRICS_SHEET_ID` env var

### AI summary is "Summary unavailable"
- OpenAI call failed — the report was still sent with metrics
- Check OpenAI API status
- The `fn.BuildReport` function uses `??` fallback, so no crash occurs

---

## RB-000: Shared Logger fails

**Symptoms:** No log entries appearing in Google Sheets; Slack alerts not firing for WARN/ERROR

### Google Sheets sink fails
- `continueOnFail: true` on `sink.GoogleSheets` — logger won't crash
- Console logs still work — check n8n container logs: `docker logs n8n -f | grep ERROR`
- Fix: re-authenticate Google Sheets credential

### Slack sink fails
- `continueOnFail: true` on `sink.Slack` — logger won't crash
- Check bot token and channel membership
- Verify `#oncall-alerts` and `#automation-logs` channels exist

---

## General Debugging Steps

1. **Find the trace:** Every execution has a `traceId` in the Slack alert. Search `application_logs` sheet.
2. **Open n8n execution:** Click the "View Execution" button in the Slack alert (deep-link included).
3. **Check the node that failed:** n8n highlights the failed node in red in the execution view.
4. **Review input data:** Click the node → "Input" tab to see exactly what data it received.
5. **Check credentials:** Most failures are credential expiry. Settings → Credentials → test each one.
6. **Re-run:** Fix the issue, then re-run from n8n UI using "Retry" on the failed execution.
