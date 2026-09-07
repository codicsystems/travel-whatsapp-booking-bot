# Sheet layout

Create a Google Sheet with a tab named exactly **Enquiries** and this header row
in row 1. The append node maps by column name, so the names must match.

| Column | Contents |
|---|---|
| `Received` | ISO timestamp of the customer's message |
| `Reference` | `ENQ-nnnnnn`, generated only for complete enquiries |
| `Name` | WhatsApp profile name, if the customer shares it |
| `Phone` | Sender's number in international format |
| `Destination` | As extracted. Blank if not stated |
| `Dates` | In the customer's own words — "early June", "15-22 Aug" |
| `Passengers` | Adults and children separately where stated |
| `Budget` | As stated, with the currency the customer used |
| `Trip type` | Umrah, honeymoon, family holiday, business |
| `Notes` | Accessibility needs, occasions, preferences |
| `Status` | `complete`, `incomplete` or `needs_human` |
| `Original message` | The raw text, so a consultant can always see what was actually said |

Paste this into A1 and it will split across columns:

```
Received	Reference	Name	Phone	Destination	Dates	Passengers	Budget	Trip type	Notes	Status	Original message
```

## Two things worth doing

**Freeze row 1** and add a filter view. You will be scanning this daily.

**Restrict sharing.** This sheet holds customer names, phone numbers, travel
plans and sometimes accessibility information. Share it with named people, never
"anyone with the link".

## Swapping it for a database

Replace the *Log to sheet* node with Postgres, MySQL or Airtable. Suggested
schema:

```sql
CREATE TABLE enquiries (
  id              BIGSERIAL PRIMARY KEY,
  message_id      TEXT UNIQUE NOT NULL,   -- dedupes Meta redelivery
  received_at     TIMESTAMPTZ NOT NULL,
  reference       TEXT,
  contact_name    TEXT,
  phone           TEXT NOT NULL,
  destination     TEXT,
  travel_dates    TEXT,
  passengers      TEXT,
  budget          TEXT,
  trip_type       TEXT,
  notes           TEXT,
  status          TEXT NOT NULL,
  original_text   TEXT NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_enquiries_status ON enquiries (status, received_at DESC);
```

The `UNIQUE` constraint on `message_id` solves the duplicate problem for free —
an `ON CONFLICT DO NOTHING` insert makes redelivery harmless.
