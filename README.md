# Transition Buddy Data

Reference data for the Transition Buddy app suite. Updated annually.

## Structure
- `pay-tables/` — Military basic pay by grade and YOS (DFAS)
- `va-rates/` — VA disability compensation rates (VA.gov)
- `bah/` — Basic Allowance for Housing rates (DTMO) — embedded in app

## Update Schedule
- **January** — Pay tables (DFAS publishes new rates)
- **December** — VA rates (COLA adjustment effective Dec 1)
- **January** — BAH rates (DTMO publishes new rates)

## Usage
App fetches `latest.json` from each directory on launch.
Falls back to embedded data if offline.
