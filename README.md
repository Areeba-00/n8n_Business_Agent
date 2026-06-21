# Vapi Voice Agent + Lawmatics — n8n Workflow (Inbound & Outbound)

A single importable n8n workflow that turns Vapi (or Retell) into a phone-based AI
receptionist for a law firm:

- **Inbound** — while on a call, the voice agent calls n8n "tools" to **check availability**
  and **book appointments** in Lawmatics, then speaks back a natural confirmation.
- **Outbound** — n8n initiates AI calls through the Vapi API (one lead on demand, or on a schedule).

```
INBOUND   Vapi/Retell ──(tool call)──▶ Webhook ▶ Parse ▶ Switch ┌▶ check_availability ─────────────▶ Respond
                                                                 ├▶ book_appointment ▶ Lawmatics ▶ Respond
                                                                 └▶ fallback ───────────────────────▶ Respond
                              ◀── spoken result ── { "results":[ {toolCallId, result} ] }

OUTBOUND  Manual / Schedule ▶ Define Lead ▶ Has Phone? ▶ POST https://api.vapi.ai/call
```

---

## 1. Import

1. In n8n: **Workflows → ⋯ (top-right) → Import from File** → choose `vapi-lawmatics-voice-agent.json`.
2. You'll see two flows on one canvas (INBOUND on top, OUTBOUND below) with sticky-note instructions.

## 2. Create the two credentials

**Credentials → Create new:**

| Credential | Type | Fields |
|---|---|---|
| **Lawmatics Header Auth** | *Header Auth* | Name = `Authorization`, Value = `Bearer YOUR_LAWMATICS_TOKEN` |
| **Vapi Bearer Auth** | *Bearer Auth* | Token = `YOUR_VAPI_PRIVATE_KEY` (Vapi Dashboard → Org Settings → API Keys, **private** key) |

Then open the three HTTP Request nodes and select the matching credential (they're pre-wired by name;
n8n just needs you to confirm the selection after import).

> Lawmatics uses OAuth2. A long-lived access token in a Header Auth credential is the quickest path.
> For auto-refreshing tokens, swap the credential for an **OAuth2 API** credential using Lawmatics'
> authorize/token URLs and set the node's *Authentication* to *Predefined/Generic OAuth2*.

---

## 3. Configure Vapi — INBOUND tools

1. Activate the workflow, then copy the **Production URL** from the **Vapi Tool Calls (Inbound)** webhook node.
2. In the Vapi Dashboard, create two **Custom Tools** and set each tool's **Server URL** to that webhook URL.
3. Use these parameter schemas:

**Tool `check_availability`**
```json
{
  "type": "object",
  "properties": {
    "date": { "type": "string", "description": "Requested date in YYYY-MM-DD, or natural language like 'next Tuesday'." }
  },
  "required": []
}
```

**Tool `book_appointment`**
```json
{
  "type": "object",
  "properties": {
    "name":     { "type": "string", "description": "Caller's full name." },
    "email":    { "type": "string", "description": "Caller's email." },
    "phone":    { "type": "string", "description": "Caller's phone in E.164, e.g. +15551234567." },
    "datetime": { "type": "string", "description": "Chosen start time in ISO 8601, e.g. 2026-07-02T14:00:00." },
    "reason":   { "type": "string", "description": "Short reason / matter type for the consultation." },
    "duration_minutes": { "type": "number", "description": "Optional, defaults to 30." }
  },
  "required": ["name", "phone", "datetime"]
}
```

4. Add both tools to your Vapi **Assistant**, and in your system prompt instruct it to call
   `check_availability` to offer times and `book_appointment` once the caller confirms.
5. To take **inbound calls**, assign that assistant to a Vapi phone number
   (Dashboard → Phone Numbers → select number → assign assistant).

The webhook returns exactly what Vapi expects, so no extra mapping is needed:
`{ "results": [ { "toolCallId": "...", "result": "...single-line string..." } ] }` with HTTP 200.

---

## 4. OUTBOUND calls

- Open **Define Lead** and set `vapiAssistantId`, `vapiPhoneNumberId`, plus the lead's `name`/`phone`/`email`/`reason`.
- Click **Run Outbound Call (Manual)** to place one test call, **or** activate the **Outbound Schedule** trigger.
- For real bulk dialing, replace **Define Lead** with a Lawmatics **GET /v1/prospects** (filter by your "to-call"
  status) followed by a **Loop Over Items** node feeding the Vapi call node.

> The sample lead intentionally uses a fake number (`+15551234567`) and placeholder IDs, so the workflow
> **won't dial anyone** until you fill in real values.

---

## 5. Lawmatics field notes (verify against your account)

Appointments in Lawmatics are **Events**. The workflow uses:

| Step | Endpoint | Body sent |
|---|---|---|
| Create lead | `POST https://api.lawmatics.com/v1/prospects` | `first_name, last_name, email, phone` |
| Create appointment | `POST https://api.lawmatics.com/v1/events` | `name, description, start_date, end_date, all_day, contact_id` |

Depending on your firm's configuration you may need to add fields such as `event_type_id`
(from `GET /v1/event_types`) or `users` (assignee IDs). Edit the JSON body in
**Lawmatics: Create Appointment** to add them. Field/endpoint names should be confirmed against your
account's live API docs at https://docs.lawmatics.com (request bodies require login to view).

If you'd rather de-duplicate the caller instead of always creating a prospect, replace
**Lawmatics: Create Prospect** with a finder call (`GET /v1/contacts/find_by_phone?phone=...`) and only
create when not found.

---

## 6. Using Retell AI instead of Vapi

The **Parse Tool Call** node already understands Retell's payload (`body.name` + `body.args`).
You only need to change the response shape, because Retell expects a plain object:

- Open **Respond to Vapi** and set the JSON body to:
  ```
  ={{ { "result": $json.result } }}
  ```
- For outbound, swap the **Vapi: Create Outbound Call** node's URL/body for Retell's
  `POST https://api.retellai.com/v2/create-phone-call` (`from_number`, `to_number`, `override_agent_id`),
  authenticated with your Retell API key via Bearer Auth.
- Point your Retell **Custom Function** URL at the same inbound webhook.

---

## 7. Test the inbound webhook without a phone call

```bash
curl -X POST 'https://YOUR-N8N/webhook/vapi-inbound' \
  -H 'Content-Type: application/json' \
  -d '{
    "message": {
      "toolCallList": [
        { "id": "call_test_1", "type": "function",
          "function": { "name": "check_availability", "arguments": { "date": "2026-07-02" } } }
      ]
    }
  }'
```

Expected response:
```json
{ "results": [ { "toolCallId": "call_test_1", "result": "I have a few openings on Thursday, July 2: 9:00 AM, 11:00 AM, 2:00 PM. Which time works best for you?" } ] }
```

Swap `function.name` to `book_appointment` with the fuller arguments to exercise the Lawmatics path.

> While testing in the editor, use the webhook's **Test URL** (`/webhook-test/vapi-inbound`) and click
> **Listen for test event**; the **Production URL** (`/webhook/vapi-inbound`) only works once the workflow is **Active**.
