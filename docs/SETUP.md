# Setup

About an hour if the Meta side is new to you, twenty minutes if it is not.

---

## 1. Meta / WhatsApp Business Platform

1. Create an app at [developers.facebook.com](https://developers.facebook.com)
   and add the **WhatsApp** product.
2. In **WhatsApp → API Setup**, note the **Phone number ID** and the test number,
   or register your own business number.
3. Generate a **permanent access token**. The temporary one in the dashboard
   expires in 24 hours and will strand you. Create a System User in Business
   Settings, give it access to the WhatsApp asset, and generate a token from
   there with `whatsapp_business_messaging` and `whatsapp_business_management`.
4. In **App Settings → Basic**, copy the **App Secret**. This is what signs the
   webhooks and you need it in step 3 below.

> **Test numbers only message numbers you have explicitly added** as recipients
> in the dashboard. If your test message never arrives, this is almost always why.

## 2. Import the workflow

n8n → **Workflows** → **Import from File** → `workflow.json`.

Node type versions move between n8n releases. If a node imports with a warning,
open it, confirm the parameters look sensible, and re-save. The logic is in the
code nodes, which are version-stable.

## 3. Environment variables

Set these on the n8n instance, not inside the workflow:

```bash
WHATSAPP_APP_SECRET=your_meta_app_secret
ENQUIRY_SHEET_ID=1AbC...        # from the Google Sheets URL
TEAM_EMAIL=enquiries@yourcompany.com
```

On Docker, add them to your compose file and restart. In n8n Cloud, use
**Settings → Variables**.

The workflow refuses to process a webhook if `WHATSAPP_APP_SECRET` is missing,
rather than silently accepting unverified requests.

## 4. Credentials

Three, created in n8n under **Credentials**:

| Credential | Type | Notes |
|---|---|---|
| Anthropic API | Header Auth | Header name `x-api-key`, value your key |
| WhatsApp Graph API | Header Auth | Header name `Authorization`, value `Bearer YOUR_PERMANENT_TOKEN` |
| Google Sheets | OAuth2 | Standard n8n flow |
| Gmail | OAuth2 | Or replace the node with Slack |

Attach them to the *Extract enquiry details*, *Send WhatsApp reply*, *Tell them
a person is coming*, *Log to sheet* and *Notify the team* nodes.

## 5. The sheet

Create a Google Sheet with a tab named **Enquiries** and the header row from
[`SHEET.md`](SHEET.md). Column names must match exactly — the append node maps
by name.

## 6. Webhook

1. **Activate** the workflow in n8n. The webhook node then exposes a production
   URL ending `/webhook/whatsapp-enquiry`.
2. In Meta → **WhatsApp → Configuration → Webhook**, set the callback URL to that
   production URL and pick any verify token string.
3. Meta sends a `GET` with `hub.challenge` to verify. The workflow's webhook node
   handles `POST` only, so **verify once manually**: temporarily set the webhook
   node to accept `GET` and respond with the challenge, complete verification,
   then set it back to `POST`. Alternatively point verification at a separate
   one-node workflow. This trips up almost everyone the first time.
4. Subscribe to the **`messages`** field. Do not subscribe to everything — you
   will drown in delivery receipts, which the workflow discards anyway.
5. n8n must be reachable over **HTTPS with a valid certificate**. Meta will not
   deliver to self-signed. For local development use a tunnel.

## 7. Test

```bash
# Replace with your n8n URL. This will fail signature verification, which is
# correct — it proves the check works.
curl -X POST https://your-n8n/webhook/whatsapp-enquiry \
  -H 'Content-Type: application/json' \
  --data @samples/inbound-text.json
```

Expect an error in the execution log: *"Webhook signature did not verify"*.
That is the security check doing its job.

For an end-to-end test, message the number from a phone registered as a
recipient. Watch the execution in n8n, then check the sheet.

Try all three paths:

- **Complete:** *"Hi, 2 adults to Dubai, 15-22 August, budget around £1,800"*
- **Incomplete:** *"Do you do Umrah packages?"* → should ask for dates and passengers
- **Complex:** *"I need to change the dates on booking REF1234"* → should hand to a human

---

## Data and retention

You are processing customer personal data — names, phone numbers, travel plans,
sometimes health or accessibility information. That last category is special
category data under UK and EU law.

Decide deliberately:

- **Where n8n is hosted**, and whether that is an acceptable jurisdiction for
  your customers' data.
- **How long execution data is kept.** n8n stores full payloads by default.
  `Settings → Workflow → Save execution data` and the `EXECUTIONS_DATA_MAX_AGE`
  environment variable control this. Message content sitting in execution logs
  forever is a liability nobody intended to create.
- **Who can see the sheet.** A Google Sheet shared with "anyone with the link"
  containing customers' names, phone numbers and travel dates is a breach
  waiting to be discovered.
- **What your privacy notice says**, and whether the automated first reply should
  mention it.

None of this is legal advice. If you operate in the UK or EU, put it in front of
whoever handles your data protection obligations before you go live.

---

## Troubleshooting

**Webhook verification fails on the Meta side.** The `GET` challenge issue in
step 6. Meta sends `GET` to verify and `POST` to deliver.

**Signature verification fails on real messages.** The webhook node needs
`rawBody` enabled — re-serialising the JSON changes the bytes and the HMAC no
longer matches. Check the app secret is the *App Secret*, not the access token.

**Replies never arrive.** With a test number, the recipient must be added in the
dashboard. On a live number, check you are inside the 24-hour customer service
window — outside it, only approved template messages can be sent, which this
workflow does not implement yet.

**Extraction returns nonsense.** Look at `rawModelOutput` in the failed
execution. Usually the model wrapped the JSON in prose despite the instruction —
tighten the prompt in *Set config* rather than loosening the parser.

**Duplicate rows.** Meta occasionally redelivers a webhook if your endpoint is
slow to acknowledge. Add a dedupe step keyed on `messageId` — a good first
contribution.
