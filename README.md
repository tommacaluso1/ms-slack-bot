# n8n Workflows: Slack ↔ Freshdesk Enquiry + KB Drafting

This package now includes **two separate n8n workflow JSON files** for reliable automatic import:
1. `workflow_1_slack_enquiries_to_freshdesk_tickets.json` → **Slack Enquiries → Freshdesk Tickets**
2. `workflow_2_freshdesk_resolved_to_kb_draft.json` → **Freshdesk Resolved → Knowledge Base Draft**

A legacy combined bundle (`n8n_export.json`) is also present, but use the two separate files for automation/import APIs.

This guide explains **what every node does**, **how to configure credentials/env vars**, and **how to test end-to-end**.

---

## 1) Import the workflows correctly

Use the two single-workflow files (recommended for UI and API automation):

1. In n8n, go to **Workflows** and import `workflow_1_slack_enquiries_to_freshdesk_tickets.json`.
2. Import `workflow_2_freshdesk_resolved_to_kb_draft.json`.
3. Confirm both workflow names appear exactly:
   - `Slack Enquiries → Freshdesk Tickets`
   - `Freshdesk Resolved → Knowledge Base Draft`
4. Open each workflow and complete credential mappings before activation.

For API-based imports, upload each file as a standalone workflow object.

---

## 2) Prerequisites checklist

- n8n has a public HTTPS URL.
- Slack app created and installable in your workspace.
- Freshdesk API key available.
- You know your target Slack channel ID (`C...`).
- You know (or can leave blank) Freshdesk group/product IDs.

---

## 3) Environment variables (required + optional)

Set these in your n8n runtime environment (Docker `.env`, Kubernetes env, systemd env, etc.).

```bash
# Slack
SLACK_CHANNEL_ID=C0123456789
SLACK_IGNORE_USER_IDS=U0BOTUSER,U0ANOTHERBOT
SLACK_IGNORE_THREAD_REPLIES=true
SLACK_POST_ACK=true
SLACK_ACK_CHANNEL_ID=C0123456789

# Freshdesk tickets
FRESHDESK_DOMAIN=yourdomain
FRESHDESK_API_KEY=your_freshdesk_api_key
FRESHDESK_GROUP_ID=
FRESHDESK_PRODUCT_ID=
FRESHDESK_TICKET_SOURCE=3
FRESHDESK_DEFAULT_PRIORITY=2
FRESHDESK_DEFAULT_STATUS=2

# Freshdesk Solutions (KB)
FRESHDESK_KB_CATEGORY_NAME=Common Issues
FRESHDESK_KB_FOLDER_VISIBILITY=3
FRESHDESK_KB_LANGUAGE=en
KB_MIN_CONFIDENCE=0.75

# OpenAI
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4.1-mini
```

### What each variable controls

- `SLACK_CHANNEL_ID`: Channel watched by Slack Trigger.
- `SLACK_IGNORE_USER_IDS`: Comma-separated users to ignore; include the bot user ID to prevent loops.
- `SLACK_IGNORE_THREAD_REPLIES`: `true` ignores non-root thread replies.
- `SLACK_POST_ACK`: If `true`, posts a thread acknowledgment when ticket is created.
- `SLACK_ACK_CHANNEL_ID`: Where error/alert messages go (defaults to intake channel if unset in expressions).

- `FRESHDESK_DOMAIN`: Your Freshdesk subdomain (without protocol).
- `FRESHDESK_API_KEY`: Used as Basic Auth username; password is literal `X`.
- `FRESHDESK_GROUP_ID` / `FRESHDESK_PRODUCT_ID`: Optional routing fields for ticket create.
- `FRESHDESK_TICKET_SOURCE`: Ticket source code (for example `3` for chat).
- `FRESHDESK_DEFAULT_PRIORITY`: Base priority if no stronger urgency mapping applies.
- `FRESHDESK_DEFAULT_STATUS`: Usually `2` (Open).

- `FRESHDESK_KB_CATEGORY_NAME`: Category to place generated KB content under.
- `FRESHDESK_KB_FOLDER_VISIBILITY`: Folder visibility (default `3`, Agents).
- `FRESHDESK_KB_LANGUAGE`: Folder/article language.
- `KB_MIN_CONFIDENCE`: Minimum confidence required before KB drafting.
- `OPENAI_API_KEY`: API key used by HTTP Request nodes calling OpenAI Responses API.
- `OPENAI_MODEL`: OpenAI model name (recommended `gpt-4.1-mini` for cost/performance).

---

## 4) Credentials setup in n8n

## A) Slack credentials

You need Slack credentials for:
- `Slack Trigger` nodes
- `Slack` (post message) nodes

In n8n:
1. Open **Credentials → New Credential → Slack API** (or OAuth2 Slack depending on n8n version).
2. Enter client/app details from your Slack app.
3. Ensure the app has required scopes (see section 5).
4. Save and test.
5. Open each workflow and select the Slack credential on all Slack nodes.

## B) Freshdesk authentication

These workflows use `HTTP Request` nodes with header-based Basic auth expression:
- username = `FRESHDESK_API_KEY`
- password = `X`

No separate n8n credential object is required by this export, but you can convert these nodes to a shared credential later if preferred.

## C) OpenAI authentication
- OpenAI calls are implemented with HTTP Request nodes against `https://api.openai.com/v1/responses`.
- Auth header uses `OPENAI_API_KEY` (`Bearer` token).
- No additional n8n credential is required unless you prefer central credential management.

---

## 5) Slack app setup details

Create or update your Slack app.

### Required bot scopes
- `channels:history`
- `channels:read`
- `chat:write`
- `groups:history` (for private channels)
- `groups:read` (for private channels)
- `users:read`

### Event subscriptions
1. Go to Slack app → **Event Subscriptions**.
2. Enable events.
3. Use the Slack Trigger webhook URL from n8n.
4. Subscribe to channel message events for the bot integration used by n8n’s Slack Trigger configuration.
5. Reinstall app after scope/event changes.

### Signing secret verification
- In n8n Slack setup, configure Slack Signing Secret where required by your version.
- Ensure your n8n URL is publicly reachable with valid TLS cert.

---

## 6) Freshdesk automation rule for workflow 2

Create a Freshdesk automation that sends resolved/closed ticket IDs to n8n.

**Condition**
- Ticket status changes to **Resolved** or **Closed**.

**Action**
- Webhook:
  - Method: `POST`
  - URL: `https://<your-n8n-host>/webhook/freshdesk-resolved-kb-draft`
  - Header: `Content-Type: application/json`
  - Body:
    ```json
    {
      "ticket_id": "{{ticket.id}}",
      "updated_at": "{{ticket.updated_at}}"
    }
    ```

`ticket_id + updated_at` is used for dedupe/idempotency.

---

## 7) Node-by-node guide: Workflow 1

**Workflow name:** `Slack Enquiries → Freshdesk Tickets`

1. **Slack Trigger**
   - Purpose: Listen for “new message in configured channel”.
   - Key settings:
     - Channel = `{{$env.SLACK_CHANNEL_ID}}`
     - Ignore users = `{{$env.SLACK_IGNORE_USER_IDS}}`
     - Resolve IDs = on

2. **Filter and Deduplicate** (Code)
   - Purpose:
     - Ignore bot/subtype/edit events.
     - Optionally ignore thread replies (`SLACK_IGNORE_THREAD_REPLIES`).
     - Skip ignored users.
     - Deduplicate with workflow static data using event key + TTL cleanup.
   - Outcome:
     - Returns no items to stop flow when event should be skipped.

3. **Normalize Text** (Code)
   - Purpose:
     - Build normalized enquiry text from message + attachment/file hints.
     - Build a deterministic fallback classification object (used if AI call fails/parses badly).

4. **OpenAI Classify Enquiry** (HTTP Request)
   - Purpose:
     - Calls OpenAI Responses API to classify the message into strict JSON fields (`intent`, `summary`, `urgency`, `tags`, etc.).
   - Reliability:
     - `Continue On Fail` enabled so workflow can still proceed with fallback logic.

5. **Merge Normalized + AI** (Merge)
   - Purpose: Keeps both the normalized message context and AI response in the same item.

6. **Parse AI Classification** (Code)
   - Purpose:
     - Parse JSON returned by OpenAI robustly (supports plain JSON or JSON embedded in text).
     - Validate and sanitize fields.
     - Fall back to deterministic classification when AI output is missing/invalid.
     - Enforce triage fallback when confidence `< 0.5`.

7. **Build Ticket Payload** (Code)
   - Purpose:
     - Construct Freshdesk `/tickets` payload.
     - Map urgency to priority.
     - Apply env defaults and optional group/product IDs.

8. **Create Freshdesk Ticket** (HTTP Request)
   - Purpose: POST to `/api/v2/tickets`.
   - Auth: Basic header from `FRESHDESK_API_KEY:X`.
   - Continue On Fail: enabled to allow controlled error handling.

9. **Ticket Created?** (IF)
   - Purpose: Branch on whether response has ticket `id`.

10. **Post Ack Enabled?** (IF)
   - Purpose: Check `SLACK_POST_ACK == true`.

11. **Slack Thread Ack** (Slack)
   - Purpose: Reply in original Slack thread with created Freshdesk ticket number + URL.

12. **Slack Error Alert** (Slack)
   - Purpose: Post one compact failure message to ack channel.

---

## 8) Node-by-node guide: Workflow 2

**Workflow name:** `Freshdesk Resolved → Knowledge Base Draft`

1. **Webhook**
   - Purpose: Receive Freshdesk automation call.
   - Expects: `{ "ticket_id": 123, "updated_at": "..." }`.

2. **Validate Input** (Code)
   - Purpose: Ensure `ticket_id` exists and normalize types.

3. **Deduplicate Webhook** (Code)
   - Purpose:
     - Track processed `ticket_id:updated_at` in workflow static data.
     - Cleanup old dedupe keys by TTL.

4. **Get Ticket** (HTTP Request)
   - Purpose: Fetch ticket + conversations via `/tickets/{id}?include=conversations`.

5. **Extract Ticket Content** (Code)
   - Purpose:
     - Gather subject/description.
     - Pull best available latest agent resolution text.

6. **Redact PII** (Code)
   - Purpose:
     - Redact email, phone, likely addresses, likely names using regex.
     - Ensure downstream KB drafting uses redacted text only.

7. **OpenAI Draft KB** (HTTP Request)
   - Purpose:
     - Calls OpenAI Responses API to generate structured KB draft JSON from redacted ticket text.
   - Reliability:
     - `Continue On Fail` enabled to preserve fallback generation path.

8. **Merge Redacted + AI** (Merge)
   - Purpose: Combines redacted ticket context with AI response.

9. **Draft KB JSON** (Code)
   - Purpose:
     - Parse and sanitize AI JSON response.
     - Apply deterministic fallback for any missing/invalid fields.
     - Enforce confidence gate using `KB_MIN_CONFIDENCE`.

10. **KB Gate** (IF)
   - Purpose: Continue only if `should_create_kb=true` and confidence ≥ `KB_MIN_CONFIDENCE`.

11. **List Categories** (HTTP)
   - Purpose: GET existing Solutions categories.

12. **Find Category** (Code)
    - Purpose: Match by `FRESHDESK_KB_CATEGORY_NAME`.

13. **Category Missing?** (IF)
    - If missing → **Create Category** (HTTP POST)
    - Then **Merge Category** + **Normalize Category** to get `category_id`.

14. **List Folders** (HTTP)
    - Purpose: GET folders in selected category.

15. **Find Folder** (Code)
    - Purpose: Match folder by generated `kb_folder_name`.

16. **Folder Missing?** (IF)
    - If missing → **Create Folder** (HTTP POST with visibility/language)
    - Then **Merge Folder** + **Normalize Folder** to get `folder_id`.

17. **Search Articles** (HTTP)
    - Purpose: Search potential duplicate KB articles by title term.

18. **Find Article Match** (Code)
    - Purpose: Simple title similarity scoring; pick match when score threshold met.

19. **Build Article Payload** (Code)
    - Purpose: Construct article body HTML and payload with `status: 1` (draft).

20. **Article Exists?** (IF)
    - True  → **Update Article Draft** (PUT)
    - False → **Create Article Draft** (POST)

21. **Slack KB Error** (Slack)
    - Purpose: Alert channel when gate fails or critical path errors out.

---

## 9) What you should manually verify after import

For **each workflow**:
1. Slack nodes have valid credential selected.
2. Webhook URLs are reachable from external services.
3. Environment variables are present in running n8n process.
4. Both workflows are activated.
5. Expressions render expected values in node preview.

For **workflow 1** specifically:
- Trigger channel is correct.
- Bot user is included in `SLACK_IGNORE_USER_IDS`.
- `SLACK_POST_ACK` behavior matches your preference.

For **workflow 2** specifically:
- Freshdesk automation webhook URL matches exact node path.
- Freshdesk API key has permissions for tickets + solutions APIs.
- KB category/folder visibility aligns with your internal policy.

---

## 10) Operational notes and troubleshooting

### No Slack events arriving
- Verify Slack app is installed to workspace.
- Verify app is added to the target channel.
- Re-check event subscriptions and scopes.
- Check n8n execution list for trigger webhook validation failures.

### Ticket not created
- Check `Create Freshdesk Ticket` response body.
- Confirm `FRESHDESK_DOMAIN` and `FRESHDESK_API_KEY`.
- Confirm account plan/API permissions allow ticket creation.

### KB not created
- Check `KB Gate` values (`should_create_kb`, `confidence`, env threshold).
- Validate Freshdesk solutions endpoints return data for your account.
- Inspect `Search Articles` and match logic outputs.
- Check OpenAI nodes for auth/model/rate-limit errors.

### OpenAI call failed
- Verify `OPENAI_API_KEY` is set in the same runtime where n8n executes.
- Verify `OPENAI_MODEL` exists in your OpenAI account.
- Confirm outbound network access from n8n to `api.openai.com`.
- The workflows still run with fallback logic, but ticket/KB quality will be lower.

### Duplicate records
- Ensure workflow static data persistence is enabled in your n8n deployment.
- Avoid deleting workflow static data unless required.
- Keep `updated_at` in webhook payload for reliable dedupe.

---

## 11) Test plan (minimum 3)

1. **Slack intake success**
   - Post a non-bot root message in `SLACK_CHANNEL_ID`.
   - Expected: exactly one Freshdesk ticket created; optional thread ack appears.

2. **Slack dedupe and loop prevention**
   - Reprocess same event payload / retry execution.
   - Expected: no duplicate ticket.
   - Post from ignored bot user.
   - Expected: flow exits without ticket.

3. **KB draft success from resolved ticket**
   - Trigger webhook:
     ```bash
     curl -X POST "https://<your-n8n-host>/webhook/freshdesk-resolved-kb-draft" \
       -H "Content-Type: application/json" \
       -d '{"ticket_id":12345,"updated_at":"2026-01-01T10:20:30Z"}'
     ```
   - Expected: article created/updated as **draft** (`status=1`), no auto-publish.

4. **PII redaction check**
   - Use ticket containing email + phone + address-like strings.
   - Expected: redacted placeholders in generated KB content.

5. **Retry/idempotency check**
   - Send same webhook payload twice.
   - Expected: second call exits via dedupe, no duplicate KB article.

---

## 12) Suggested safe go-live sequence

1. Keep both workflows **inactive** while configuring creds/env.
2. Activate workflow 1, test with one Slack message.
3. Activate workflow 2, test with one known resolved ticket.
4. Verify Slack alerts route correctly.
5. Monitor first 24h of executions and adjust thresholds/tags if needed.
