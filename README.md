# Transition Buddy Data

Reference data for the Transition Buddy app suite. Most categories update
annually; `cfr/` updates irregularly (see below).

## Structure
- `pay-tables/` — Military basic pay by grade and YOS (DFAS)
- `va-rates/` — VA disability compensation rates (VA.gov)
- `bah/` — Basic Allowance for Housing rates (DTMO) — embedded in app
- `cfr/` — 38 CFR Part 4, Schedule for Rating Disabilities (eCFR.gov) — see `cfr/SCHEMA.md`

## Update Schedule
- **January** — Pay tables (DFAS publishes new rates)
- **December** — VA rates (COLA adjustment effective Dec 1)
- **January** — BAH rates (DTMO publishes new rates)
- **As amended** — CFR Part 4 has no fixed schedule; amendments are irregular
  federal rulemaking, not an annual cycle. The app checks for drift at
  runtime against eCFR's live amendment date (`lib/ecfr.ts`); this dataset's
  per-condition `lastVerified` field is the complementary static record of
  when each entry's text was last hand-checked.

## Usage
App fetches `latest.json` from each directory on launch.
Falls back to embedded data if offline.
