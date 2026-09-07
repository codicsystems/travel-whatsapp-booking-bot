# Travel enquiry bot — WhatsApp

An n8n workflow that reads inbound WhatsApp enquiries, pulls out the details a
travel consultant actually needs, replies to fill the gaps, logs everything to a
sheet, and hands anything complicated to a person.

Self-hosted. No SaaS subscription, no per-conversation fee beyond what Meta and
your model provider charge.

---

## The problem it solves

Small tour operators lose more money in the gap between an enquiry arriving and
a quote going out than almost anywhere else in the funnel, because that gap is
still worked by hand.

A typical WhatsApp enquiry reads:

> *Hi, looking for Umrah packages for my parents in early Ramadan, 2 adults, from
> Manchester. What's included?*

Somebody has to read it, work out that the destination is Makkah and Madinah,
that "early Ramadan" needs converting to actual dates, that two adults means two
adults, that departure is Manchester — and then type a reply asking the three
things the customer did not mention. On a Saturday evening, that reply happens
on Monday, and by Monday the customer has messaged two other operators.

This workflow does the reading and the gap-filling in about four seconds, at any
hour, and puts a structured row in front of the consultant on Monday morning
instead of a screenshot.

**What it does not do is quote.** See [Design decisions](#design-decisions).

---

## What it does

```
WhatsApp message
      │
      ├─ verify Meta's signature ─────────── reject if it fails
      │
      ├─ normalise the payload ───────────── drop delivery/read receipts
      │
      ├─ text message? ── no ──────────────► tell them a person is coming
      │                                      email the team
      │  yes
      ▼
   extract details with an LLM
      │
      ├─ complete   ──► confirm + reference ──► log to sheet
      ├─ incomplete ──► ask only what's missing ──► log to sheet
      └─ complex    ──► hand to a human ──► email the team
```

Extracted per enquiry: destination, travel dates as the customer expressed them,
passenger count and composition, budget, trip type, and free-text notes —
accessibility needs, occasions, airline preferences.

---

## Design decisions

These are the parts that matter, and the reasons they are the way they are.

### The model extracts. It never replies.

Every reply in this workflow is templated. The model's only job is turning
unstructured text into a JSON object.

This is deliberate. A language model asked to "reply helpfully to a travel
enquiry" will, sooner or later, quote a price that does not exist, confirm
availability it cannot see, or state a cancellation policy it invented. In
travel that is not an embarrassing screenshot — it is a consumer-law problem,
and in the UK an ATOL-adjacent one.

Extraction is a task models are genuinely reliable at. Improvisation about money
is not a task you should let one near.

### The webhook signature is verified before anything else

Meta signs every webhook with your app secret. The first node after the webhook
recomputes the HMAC and compares it in constant time.

Without this, anyone who discovers your webhook URL can post fabricated
enquiries into your CRM **and cause your business to send WhatsApp messages to
phone numbers of their choosing**. Most n8n WhatsApp tutorials skip this. It is
about fifteen lines.

### Anything uncertain goes to a person

Three separate routes end at a human:

- **Non-text messages** — images, voice notes, locations, documents
- **Extraction failure** — if the JSON does not parse, nothing is guessed
- **Complexity flags** — groups above nine, multi-city itineraries, visa
  questions, changes to an existing booking, medical or accessibility
  requirements, and anything that reads as a complaint

The customer is told a person is picking it up. The team gets an email with the
original message and whatever was extracted.

An automation that quietly mishandles a complaint costs more than one that
handles fewer cases.

### Dates stay in the customer's words

"Early June", "next Eid", "15-22 Aug" are stored as written. Converting
"next Ramadan" to a specific date range is a decision with a wrong answer, and
the consultant is better placed to make it than the model.

---

## What you need

| | |
|---|---|
| **n8n** | Self-hosted or cloud. Tested on 1.7x |
| **WhatsApp Business Platform** | A Meta app with the WhatsApp product added, a phone number, a permanent access token and your app secret |
| **An LLM API key** | Configured for Anthropic in the template. Any provider works — swap the URL and body in one node |
| **Google Sheets** | Or replace that node with Postgres, Airtable or your CRM |
| **Gmail** | Or replace with Slack, Telegram or anything else for team alerts |

Running cost is dominated by Meta's conversation pricing, which varies by
country and category. The model cost is a fraction of a cent per enquiry at
these token counts.

---

## Setup

Full walkthrough in [`docs/SETUP.md`](docs/SETUP.md). In short:

1. Import [`workflow.json`](workflow.json) into n8n.
2. Set four environment variables: `WHATSAPP_APP_SECRET`, `ENQUIRY_SHEET_ID`,
   `TEAM_EMAIL`, and your model API key as an n8n credential.
3. Create the header row in your sheet — [`docs/SHEET.md`](docs/SHEET.md) has it.
4. Point your Meta webhook at the n8n production URL and subscribe to `messages`.
5. Send yourself a test message.

Sample payloads for testing without Meta are in [`samples/`](samples/).

---

## Customising it

**The extraction prompt** lives in the *Set config* node, not buried in a code
node, so you can edit it without touching logic. That is where you add fields —
`departureAirport`, `mealPlan`, `roomType` — and where you tune what counts as
complex for your business.

**The replies** are in the *Compose reply* node. Edit the templates and the
`QUESTIONS` map together if you add required fields.

**The required fields** are the `REQUIRED` array in *Parse and validate*. Adding
one there makes the bot ask for it.

**Swapping the model** means changing one HTTP node: the URL, the auth header,
and the request body shape. The parser expects the response text somewhere it
can find and strips markdown fences either way.

---

## What this is not

- **Not a booking engine.** It captures enquiries. It does not check
  availability, hold inventory or take payment.
- **Not a chatbot.** It asks at most one round of clarifying questions and then
  hands over. Multi-turn conversation state is a much larger problem and mostly
  not one operators need solved.
- **Not GDPR compliance in a box.** You are processing customer personal data.
  Know where your n8n instance is hosted, how long you retain execution data,
  and what your privacy notice says. See [`docs/SETUP.md`](docs/SETUP.md#data-and-retention).

---

## Contributing

Pull requests welcome, particularly:

- **A Postgres node** as an alternative to Google Sheets
- **Arabic and Urdu extraction testing** — the prompt should hold, but nobody has
  tested it properly
- **Meta template message support** for replying outside the 24-hour window
- **A dedupe step** — Meta occasionally redelivers a webhook, and right now that
  produces a duplicate row

Please keep the two safety properties intact: signature verification stays, and
the model does not generate customer-facing text about prices, availability or
policy.

## Licence

[MIT](LICENSE). Use it commercially, modify it, no attribution required.

Built by [Codic Systems](https://codicsystems.com) — booking and operations
software for travel companies, Islamabad.
