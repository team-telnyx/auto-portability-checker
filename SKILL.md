---
name: auto-portability-checker
description: "Check number porting coverage and per-number portability from Pylon tickets requesting portability checks."
---

# Auto Portability Checker

Handle Pylon issues/tickets that request number portability checks for Telnyx.

## Triggers

- A Pylon issue/ticket requesting a portability check
- Keywords: "portability check", "can we port", "is this number portable", "porting eligibility", "check if we support porting in [country]"
- Any Pylon ticket where the customer asks whether Telnyx can port their number(s) or whether a country is supported

## Testing restriction (temporary)

**Only process tickets where the sender email is `stephenr@telnyx.com`.** Do NOT reply to or close tickets from any other sender. This restriction will be lifted after the testing phase.

To verify the sender, check `message.author.user.email` or `message.email_info.from_email` from the ticket messages. Only proceed if the email matches `stephenr@telnyx.com`.

## Pylon API

**Base URL:** `https://api.usepylon.com`
**Auth:** `Authorization: Bearer $PYLON_API_TOKEN`
**Token stored in:** `~/.openclaw/workspace/france-rio-provider/.env` (`PYLON_API_TOKEN`)

| Action | Method | Endpoint | Notes |
|--------|--------|----------|-------|
| List issues | GET | `/issues?start_time={ISO}&end_time={ISO}&limit=50` | Required params: start_time, end_time (ISO 8601) |
| Get issue | GET | `/issues/{id}` | Returns issue details including `requester` |
| Read messages | GET | `/issues/{id}/messages` | Returns messages with `author.user.email`, `email_info.from_email` |
| Reply (customer-facing) | POST | `/issues/{id}/reply` | Body: `{"message_id": "...", "body_html": "...", "email_info": {"to_emails": ["..."]}}` |
| Internal note | POST | `/issues/{id}/note` | Body: `{"body_html": "..."}` — not visible to customer |
| Close ticket | PATCH | `/issues/{id}` | Body: `{"state": "closed"}` |

**Rate limits:** Issues 10/min, Messages 20/min, Reply 10/min

## Telnyx Portability API

**Endpoint:** `POST https://api.telnyx.com/v2/portability_checks`
**Auth:** `Authorization: Bearer $TELNYX_API_KEY`
**Content-Type:** `application/json`

**Request body:**
```json
{
  "phone_numbers": ["+3221234567", "+442071234567", "+61281234567"]
}
```
- All numbers must be in E.164 format (+ prefix, country code, no spaces)
- Batch supported — pass all numbers in one request

**Response (201):**
```json
{
  "data": [
    {
      "record_type": "portability_check_result",
      "phone_number": "+3221234567",
      "phone_number_type": "local",
      "carrier_name": null,
      "messaging_capable": false,
      "portable": true,
      "fast_portable": false,
      "not_portable_reason": null,
      "not_portable_reason_description": null
    }
  ]
}
```

**Response fields:**
- `portable` (boolean) — whether the number can be ported to Telnyx
- `not_portable_reason` (string|null) — `null` if portable; reason code if not (e.g. `"no_coverage"`, `"invalid_phone_number"`)
- `not_portable_reason_description` (string|null) — human-readable explanation (e.g. `"We do not have coverage for this phone number."`, `"The phone number is invalid."`)
- `fast_portable` (boolean) — whether the number is FastPort eligible
- `phone_number` (string) — the E.164 number this result is for
- `phone_number_type` (string|null) — inferred type (e.g. `"local"`, `"mobile"`, `null` if unknown)
- `carrier_name` (string|null) — current carrier name (usually `null`)
- `messaging_capable` (boolean) — whether the number supports messaging

**Error responses:**
- `401` — Unauthorized (check API key)
- `422` — Unprocessable entity (check message field for details)

**cURL example:**
```bash
curl -X POST https://api.telnyx.com/v2/portability_checks \
  -H "Authorization: Bearer $TELNYX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"phone_numbers": ["+3221234567", "+3221234568"]}'
```

## Workflow

### Step 1 — Fetch and parse the Pylon ticket

1. Fetch the ticket: `GET /issues/{id}`
2. Read messages: `GET /issues/{id}/messages`
3. **Verify sender** — check `message.author.user.email` or `message.email_info.from_email` on the first customer message. If sender is NOT `stephenr@telnyx.com` → skip this ticket. Do NOT reply or close.
4. Parse the ticket subject + body for:
   - Phone number(s) — if present → Step 2 (number-based flow)
   - Country name only — if no numbers → Step 3 (country-only flow)
5. Record the `message_id` from the first customer message (needed for reply)

### Step 2 — Number-based flow

1. **Normalize** all numbers to E.164 format.
2. **Infer country** from the E.164 country code for each number.
3. **Look up country coverage** in `references/porting-coverage.json`.
   - If the country is not in the file → draft ticket reply: "Telnyx does not currently support porting in [country]."
   - If the country exists but the inferred number type (Local/National/Toll-Free/Mobile) is not supported → reply with what IS supported and stop. Do NOT call the portability API.
4. **Call the Telnyx portability API:**
   ```
   POST https://api.telnyx.com/v2/portability_checks
   Authorization: Bearer $TELNYX_API_KEY
   Content-Type: application/json

   {"phone_numbers": ["+3221234567", "+3221234568"]}
   ```
5. **Interpret results** — for each number in the response:
   - `portable: true` → ✅ Portable (include `phone_number_type` and `fast_portable` in reply)
   - `portable: false` → ❌ Not portable (include `not_portable_reason_description`)
   - API error or missing number → ⚠️ Unable to determine — escalate
6. **Draft reply** and post to the Pylon ticket via `POST /issues/{id}/reply`.
7. **Close the ticket** via `PATCH /issues/{id}` with `{"state": "closed"}` — unless escalation is needed.

### Step 3 — Country-only flow

1. Look up the country in `references/porting-coverage.json` (case-insensitive, also match slug).
2. If not found → draft ticket reply listing the unsupported country/countries. If multiple countries are unsupported, list them together and add the expansion message once as a separate paragraph (not per country):
   - Single unsupported country: "Telnyx does not currently support number porting in [country]. We're always expanding, and we hope that we will soon be able to port this type of number as we continue to expand our network. But we do not have an ETA."
   - Multiple unsupported countries: "Telnyx does not currently support number porting in [country1], [country2], and [country3]. We're always expanding, and we hope that we will soon be able to port these types of numbers as we continue to expand our network. But we do not have an ETA." → post reply → close ticket.
3. If found → draft ticket reply with:
   - Supported number types (Local, National, Toll-Free, Mobile) with status and lead time
   - Porting hours
   - Porting requirements — **always as bullet points, never as a single line.** Each requirement gets its own line. Group by type if different requirements exist per type.
   - Any special notes from the coverage data
   - Support article link
4. Post reply via `POST /issues/{id}/reply`.
5. Close ticket via `PATCH /issues/{id}` with `{"state": "closed"}`.
6. Do NOT call the Telnyx portability API.

## Response format

### Country-only example (Belgium)

> Yes, Telnyx supports number porting in Belgium 🇧🇪.
>
> **Supported number types:**
> - Local: ✅ (4+ business days)
> - National: ✅ (4+ business days)
> - Toll-Free: ✅ (4+ business days)
> - Mobile: ❌ Not supported
>
> **Porting hours:** 8 AM – 5 PM local
>
> **Requirements (Local):**
> - LOA (local address required, area code must match)
> - VAT / TAX ID
> - Latest invoice
> - Proof of local address
>
> **Requirements (National/Toll-Free):**
> - LOA (national address required)
> - VAT / TAX ID
> - Latest invoice
> - Proof of local address
>
> 📖 Full guide: https://support.telnyx.com/en/articles/3266421-belgium-number-porting

### Number-based example

> Portability check for +3221234567 (Belgium 🇧🇪):
>
> ✅ **+3221234567** — Portable
> - Type: Local
> - FastPort: No
> - Country: Belgium
> - Lead time: 4+ business days
> - Porting hours: 8 AM – 5 PM local
>
> If any numbers fail or are inconclusive, they will be listed separately with the reason.

## Escalation

- **When to escalate (do NOT close ticket):**
  - Telnyx portability API returns an error or inconclusive result
  - Number type is supported at country level but the API cannot confirm portability
  - Any unexpected response or ambiguity
- **How to escalate:**
  - Post an internal note via `POST /issues/{id}/note` explaining the situation
  - Do NOT close the ticket — leave it open for a human agent

## Formatting rules

- **Requirements must always be bullet points** — never compress into a single line. Each requirement gets its own bullet.
- Example:
  ```
  Requirements (all types):
  - LOA (national address mandatory)
  - SIRET code (14 digits for business)
  - RIO code (12 characters, dial 3179)
  - Latest invoice
  - Proof of address (within 3 months)
  ```
- If requirements differ by number type, group them under subheadings (e.g. "Requirements (Local):", "Requirements (Toll-Free):"), each with their own bullet list.
- For multiple unsupported countries, list them together in one sentence and add the expansion message once as a separate paragraph.

## Key rules

- **Testing phase:** Only process tickets where sender email is `stephenr@telnyx.com`. Skip all other tickets silently.
- **Never assume** country coverage means a specific number is portable. Always run the API check for number-based requests.
- **Escalate on API failure** — do not guess or infer portability without an API response.
- **Always reply on the same Pylon ticket** the request came from — use `POST /issues/{id}/reply`.
- **Always close the ticket** after a successful reply — unless escalating.
- **US/CA/PR** are handled by the US Porting Team and are NOT in the coverage file. Redirect to: https://support.telnyx.com/en/articles/8673249-us-ca-toll-free-number-porting
- The coverage data file is `references/porting-coverage.json` — 39 countries, 4 number types each.
- Phone numbers must be E.164 before calling the portability API.
- Use HTML in reply bodies (`body_html` field) for formatting in Pylon.
- `TELNYX_API_KEY` is stored in `~/.openclaw/workspace/france-rio-provider/.env`.

## References

- `references/global-porting-coverage.html` — the source of truth. Updated HTML file with all 39 countries, number types, requirements, and carrier toggles.
- `references/porting-coverage.json` — structured porting coverage data extracted from the HTML. 39 countries, 4 number types each. Includes per-country: supported types, lead times, porting hours, requirements, LOA links, support article links, and carrier lists for limited types (e.g. UK Mobile).
