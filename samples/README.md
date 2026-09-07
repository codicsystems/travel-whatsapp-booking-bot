# Sample payloads

Real-shaped Meta webhook bodies for testing the workflow without a live
WhatsApp number.

| File | What it exercises |
|---|---|
| `inbound-text.json` | A complete enquiry — destination, dates, passengers all present. Should confirm and generate a reference. |
| `inbound-incomplete.json` | *"Do you do city breaks?"* — should ask for destination, dates and passenger count. |
| `inbound-complex.json` | Booking change, accessibility requirement and a complaint in one message. Should route to a human on all three counts. |
| `status-receipt.json` | A delivery receipt, not a message. Should be discarded silently by the normalisation node. |

## Using them

These will **fail signature verification**, which is correct — they are not
signed by Meta. To exercise the parsing logic, pin the output of the *Verify
signature* node in n8n and run from there, or temporarily disable that node
while developing.

Do not disable it and forget. It is the only thing standing between your
webhook URL and someone sending WhatsApp messages in your company's name.

To sign a sample properly for an end-to-end test:

```bash
SECRET="your_app_secret"
BODY=$(cat samples/inbound-text.json)
SIG="sha256=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$SECRET" | awk '{print $2}')"

curl -X POST https://your-n8n/webhook/whatsapp-enquiry \
  -H "Content-Type: application/json" \
  -H "X-Hub-Signature-256: $SIG" \
  --data "$BODY"
```
