# Auto Portability Checker

Automated number portability checking for Telnyx. Handles Pylon support tickets requesting portability checks and responds with coverage details and per-number portability results.

## How It Works

1. **Trigger** — A Pylon ticket requests a portability check ("Can we port +3221234567?" or "Do we support porting in Belgium?")
2. **Country Coverage** — Looks up the country in the Global Porting Coverage data (39 countries) to verify support for Local, National, Toll-Free, and Mobile number types
3. **Number Portability** — If phone numbers are provided and coverage is confirmed, calls the Telnyx Portability API (`POST /v2/portability_checks`) to check per-number portability
4. **Reply & Close** — Posts the result on the Pylon ticket and closes it

## Two Flows

- **Country-only** — Returns supported number types, lead times, porting hours, and requirements. No API call.
- **Number-based** — Normalizes to E.164, verifies country coverage, calls the Telnyx portability API, and returns per-number portable/not portable results.

## Files

```
auto-portability-checker/
├── SKILL.md                              # Skill workflow and instructions
├── README.md                             # This file
└── references/
    ├── global-porting-coverage.html      # Source of truth — 39 countries, UK Mobile carrier toggle
    └── porting-coverage.json             # Structured data extracted from HTML
```

## Requirements

- **Pylon API Token** — for fetching/replying to tickets (`PYLON_API_TOKEN`)
- **Telnyx API Key** — for the portability check endpoint (`TELNYX_API_KEY`)

Both keys should be stored in a `.env` file (not included in this repo).

## Telnyx Portability API

```
POST https://api.telnyx.com/v2/portability_checks
Authorization: Bearer $TELNYX_API_KEY
Content-Type: application/json

{"phone_numbers": ["+3221234567"]}
```

Response includes `portable`, `not_portable_reason`, `not_portable_reason_description`, `phone_number_type`, `fast_portable`, and `messaging_capable`.

## Coverage Data

The `references/global-porting-coverage.html` file is the source of truth for country-level porting coverage. It includes:

- 39 countries across EMEA, APAC, and Americas
- Per-country support for Local, National, Toll-Free, and Mobile number types
- Lead times, porting hours, and requirements
- LOA download links and support article links
- UK Mobile carrier toggle (12 supported carriers)

The `references/porting-coverage.json` is a structured extraction of the HTML for programmatic use.

## License

MIT
